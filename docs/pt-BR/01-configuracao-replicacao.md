# Replicação PostgreSQL para LND

[🇺🇸 English version](../en/01-replication-setup.md)

## Introdução

Um node Lightning pode concentrar anos de operação, canais ativos, histórico de roteamento e informações essenciais para a continuidade do serviço. Embora mecanismos como o Static Channel Backup (SCB) sejam fundamentais para a recuperação dos canais, eles não substituem uma estratégia de alta disponibilidade e proteção do banco de dados utilizado pelo LND.

Quando o LND utiliza PostgreSQL como backend, surge a possibilidade de utilizar os próprios mecanismos nativos do PostgreSQL para manter uma cópia continuamente atualizada do banco em outro servidor.

Este projeto nasceu com esse objetivo: **construir e documentar uma arquitetura de replicação física do PostgreSQL para um node LND, priorizando consistência dos dados, recuperação controlada e proteção contra perda de estado**.

A implementação e os testes documentados neste projeto foram realizados utilizando o seguinte ambiente de referência:

- **Ubuntu Server 24.04 LTS**;
- **PostgreSQL 18.6**;
- **LND 0.21.3-beta**;
- **Nodes utilizados:**
     - Naghust SA | BR⚡LN (02dd543868366e0bc3e498ab7c687d795a30ce0f70d2d034006b4654bbe887af8a)
     - Naghust Replication Test (026eea7a1d59689a4e40c3504cbaca131d712564436ea081db6fd81050b585f0b7)

A arquitetura utiliza dois servidores:

- **Primary** — servidor PostgreSQL utilizado normalmente pelo LND;
- **Standby** — servidor independente que mantém uma réplica física continuamente atualizada do cluster PostgreSQL.

A replicação é realizada por **PostgreSQL Physical Streaming Replication**, utilizando um *physical replication slot*. A comunicação entre os servidores pode ocorrer por uma rede privada, VPN ou outra rede de replicação adequadamente protegida.

O modo normal de operação adotado neste projeto utiliza:

```conf
synchronous_commit = remote_apply
synchronous_standby_names = 'standby1'
```

Com `remote_apply`, uma transação síncrona confirmada pelo ***Primary*** somente retorna ao cliente após o WAL correspondente ter sido aplicado pelo ***Standby*** selecionado como síncrono. Essa escolha privilegia a consistência entre os dois servidores, ao custo de acrescentar a latência da replicação ao caminho de confirmação das transações.

Essa decisão também cria uma consequência operacional importante: **a indisponibilidade do ***Standby*** pode bloquear commits no ***Primary*** mesmo que o servidor principal continue funcionando normalmente**.

Por esse motivo, o projeto também documenta um procedimento de contingência que permite retirar temporariamente a exigência da réplica síncrona e manter o LND operacional enquanto o ***Standby*** é recuperado.

A arquitetura foi concebida com alguns princípios:

- utilizar recursos nativos do PostgreSQL sempre que possível;
- evitar mecanismos automáticos de failover que possam aumentar o risco de *split-brain*;
- manter promoção e recuperação sob controle explícito do operador;
- proteger o estado do LND como prioridade;
- permitir operação temporariamente assíncrona em caso de falha do ***Standby***;
- validar o estado da replicação antes de qualquer promoção;
- documentar procedimentos que possam ser reproduzidos e auditados.

Este repositório documenta a construção dessa arquitetura de forma progressiva.

A primeira parte aborda a preparação do PostgreSQL, criação do ***Standby***, replicação física, ativação do modo síncrono com `remote_apply` e operação em contingência.

As etapas seguintes abordarão o processo de promoção do ***Standby***, recuperação após falha do ***Primary*** e integração desse mecanismo com a operação do LND.

> **Atenção:** os procedimentos descritos neste projeto envolvem diretamente o banco de dados utilizado pelo LND. Antes de aplicá-los em um node em produção, compreenda cada comando, mantenha backups adequados e adapte endereços, caminhos, versões e parâmetros à sua própria infraestrutura.

## 1 — Preparar o PostgreSQL no ***Primary*** para replicação

### 1.1 — Criar um arquivo de configuração específico para a replicação

**Objetivo:** manter os parâmetros da replicação separados do `postgresql.conf` principal, facilitando a manutenção, auditoria e eventual reversão da configuração.

No ***Primary***, execute o comando para criar um arquivo específico dentro do diretório `conf.d` (substitua 18 pela versão major identificada no ***Primary***):

```bash
sudo nano /etc/postgresql/18/main/conf.d/99-lnd-replication.conf
```

Insira:

```conf
# PostgreSQL Physical Replication - LND

# WAL e processos de replicacao
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10

# Limite de WAL retido por replication slots
max_slot_wal_keep_size = 10GB
```

Salve o arquivo (Ctrl+O e Enter) e saia do Nano (Ctrl+X).

#### Entendendo os parâmetros

- `wal_level = replica` determina que o WAL contenha as informações necessárias para suportar replicação física.

- `max_wal_senders = 10` permite até 10 processos `walsender` simultâneos. Esses processos são responsáveis, entre outras funções, por transmitir WAL do ***Primary*** para o ***Standby***.

- `max_replication_slots = 10` permite a existência de até 10 `replication slots`. Mais adiante criaremos um slot específico para o ***Standby***.

- `max_slot_wal_keep_size = 10GB` estabelece um limite para a quantidade de WAL que poderá ser retida em razão de `replication slots`. Neste tutorial adotaremos 10 GB como configuração padrão.

### 1.2 — Permitir conexões ao PostgreSQL pela rede utilizada para replicação

**Objetivo:** permitir que o PostgreSQL aceite conexões na interface de rede do ***Primary*** utilizada para comunicação com o ***Standby***. Neste tutorial utilizamos a rede Tailscale, mas também pode ser utilizada uma rede local ou outra rede privada entre os servidores.

No ***Primary***, execute:

```bash
sudo nano /etc/postgresql/18/main/conf.d/99-lnd-replication.conf
```

Acrescente:

```conf
# Interfaces de rede
listen_addresses = 'localhost,IP_REDE_PRIMARY'
```

Substitua `IP_REDE_PRIMARY` pelo endereço IP da interface do ***Primary***. Por exemplo:

- Tailscale: `100.x.x.x`;
- rede local: `192.168.x.x`;
- outra rede privada: utilize o endereço correspondente dessa interface.

> **Nota:** se o PostgreSQL já estiver configurado para escutar na interface que será utilizada pelo ***Standby***, não é necessário alterar `listen_addresses`.

### 1.3 — Validar o arquivo antes de aplicar

No ***Primary***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT sourcefile, sourceline, name, setting, error
FROM pg_file_settings
WHERE sourcefile LIKE '%99-lnd-replication.conf%'
ORDER BY sourceline;
"
```

Saída esperada:

```text
                            sourcefile                  | sourceline |          name          |        setting            |            error
--------------------------------------------------------+------------+------------------------+---------------------------+------------------------------
 /etc/postgresql/18/main/conf.d/99-lnd-replication.conf |          4 | wal_level              | replica                   |
 /etc/postgresql/18/main/conf.d/99-lnd-replication.conf |          5 | max_wal_senders        | 10                        |
 /etc/postgresql/18/main/conf.d/99-lnd-replication.conf |          6 | max_replication_slots  | 10                        |
 /etc/postgresql/18/main/conf.d/99-lnd-replication.conf |          9 | max_slot_wal_keep_size | 10GB                      |
 /etc/postgresql/18/main/conf.d/99-lnd-replication.conf |         12 | listen_addresses       | localhost,IP_REDE_PRIMARY | setting could not be applied
(5 rows)
```

A consulta permite verificar como o PostgreSQL interpretou os parâmetros configurados. Parâmetros que exigem reinicialização, como listen_addresses, podem apresentar `setting could not be applied` enquanto o PostgreSQL ainda estiver em execução com a configuração anterior. A configuração será confirmada após a reinicialização do serviço.

### 1.4 — Autorizar o ***Standby*** no `pg_hba.conf`

**Objetivo:** permitir que o ***Standby*** estabeleça conexões de replicação com o PostgreSQL do ***Primary***.

No ***Primary***, execute:

```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
```

No final do arquivo, acrescente (substitua `IP_REDE_STANDBY` pelo endereço IP da rede do ***Standby***):

```conf
# Replicacao fisica PostgreSQL - Standby
host    replication    lnd_replicator    IP_REDE_STANDBY/32    scram-sha-256
```

Substitua `IP_REDE_STANDBY` pelo endereço IP da interface do ***Standby***. Por exemplo:

- Tailscale: `100.x.x.x`;
- rede local: `192.168.x.x`;
- outra rede privada: utilize o endereço correspondente dessa interface.

#### 1.4.1 — Validar o `pg_hba.conf`

No ***Primary***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT line_number, type, database, user_name, address, auth_method, error
FROM pg_hba_file_rules
WHERE user_name @> ARRAY['lnd_replicator']::text[];
"
```

Saída esperada:

```text
 line_number | type |   database    |    user_name     |     address     |  auth_method  | error 
-------------+------+---------------+------------------+-----------------+---------------+-------
         XXX | host | {replication} | {lnd_replicator} | IP_REDE_STANDBY | scram-sha-256 | 
(1 row)
```
### 1.5 — Criar o usuário dedicado à replicação

**Objetivo:** criar no ***Primary*** um usuário PostgreSQL exclusivo para a replicação física, com somente os privilégios necessários.

No ***Primary***, execute:

```bash
sudo -u postgres psql
```

Você deverá chegar ao prompt:

```text
postgres=#
```

Agora crie o usuário de replicação sem colocar a senha diretamente no comando.

No prompt `postgres=#` execute:

```sql
CREATE ROLE lnd_replicator WITH REPLICATION LOGIN;
```

Saída esperada:
```text
CREATE ROLE
```

Ainda dentro do `psql`, defina a senha de forma interativa, execute:

```psql
\password lnd_replicator
```

O PostgreSQL solicitará:

```text
Enter new password for user "lnd_replicator":
Enter it again:
```
> **Importante:** escolha uma senha forte e exclusiva para a replicação e guarde-a em local seguro pois será utilizada na preparação do ***Standby***.

#### 1.5.1 — Validar o usuário

Ainda dentro do `psql`, execute:

```sql
SELECT rolname, rolcanlogin, rolreplication, rolsuper
FROM pg_roles
WHERE rolname = 'lnd_replicator';
```

Saída esperada:

```text
    rolname     | rolcanlogin | rolreplication | rolsuper 
----------------+-------------+----------------+----------
 lnd_replicator | t           | t              | f
(1 row)
```

> **Nota:** o usuário `lnd_replicator` é uma conta dedicada exclusivamente à replicação do PostgreSQL. Ele possui permissão para realizar login (`rolcanlogin = t`) e iniciar conexões de replicação (`rolreplication = t`), mas não possui privilégios de superusuário (`rolsuper = f`). Essa configuração segue o princípio do menor privilégio, limitando a conta apenas às permissões necessárias para a replicação e reduzindo o impacto de um eventual comprometimento de suas credenciais.

Saia do `psql` executando:
```psql
\q
```

### 1.6 — Criar o replication slot físico

**Objetivo:** criar no ***Primary*** um slot de replicação dedicado ao ***Standby***. O slot permite que o PostgreSQL acompanhe até qual posição do WAL a réplica consumiu os dados e retenha os WALs ainda necessários.

No ***Primary***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT * FROM pg_create_physical_replication_slot('lnd_pg_standby_01');
"
```

Saída esperada:

```text
     slot_name     | lsn 
-------------------+-----
 lnd_pg_standby_01 | 
(1 row)
```

#### 1.6.1 — Validar o replication slot

No ***Primary***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT slot_name, slot_type, active, restart_lsn
FROM pg_replication_slots
WHERE slot_name = 'lnd_pg_standby_01';
"
```

Saída esperada:

```text
     slot_name     | slot_type | active | restart_lsn 
-------------------+-----------+--------+-------------
 lnd_pg_standby_01 | physical  | f      | 
(1 row)
```

> **Nota:** um replication slot impede que o ***Primary*** descarte WALs que ainda possam ser necessários pelo ***Standby***. Isso aumenta a segurança contra uma réplica temporariamente atrasada, mas também cria um risco: se o ***Standby*** permanecer indisponível por muito tempo, os WALs podem ocupar uma quantidade crescente de disco. Por isso configuramos anteriormente `max_slot_wal_keep_size = 10GB`, estabelecendo um limite para a retenção de WAL causada pelo slot.

### 1.7 — Reiniciar o PostgreSQL Primary e aplicar as configurações

**Objetivo:** reiniciar o PostgreSQL no ***Primary*** para aplicar os parâmetros que exigem reinicialização, especialmente `listen_addresses`, e confirmar que o serviço retornou normalmente.

#### 1.7.1 — Reiniciar o PostgreSQL

No ***Primary***, execute:

```bash
sudo systemctl restart postgresql
```

Se o comando retornar sem saída, isso é normal.

#### 1.7.2 — Verificar o estado do cluster

No ***Primary***, execute:

```bash
pg_lsclusters
```

Saída esperada:

```text
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

#### 1.7.3 — Confirmar os parâmetros efetivamente aplicados

No ***Primary***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT name, setting, unit
FROM pg_settings
WHERE name IN (
    'wal_level',
    'max_wal_senders',
    'max_replication_slots',
    'max_slot_wal_keep_size',
    'listen_addresses'
)
ORDER BY name;
"
```

Saída esperada:

```text
          name          |        setting            | unit 
------------------------+---------------------------+------
 listen_addresses       | localhost,IP_REDE_PRIMARY | 
 max_replication_slots  | 10                        | 
 max_slot_wal_keep_size | 10240                     | MB
 max_wal_senders        | 10                        | 
 wal_level              | replica                   | 
(5 rows)
```

#### 1.7.4 — Confirmar em quais endereços o PostgreSQL está escutando

No ***Primary***, execute:

```bash
sudo ss -ltnp | grep ':5432'
```

Saída esperada:

```text
IP_REDE_PRIMARY:5432
127.0.0.1:5432
[::1]:5432
```

O importante é que o PostgreSQL esteja escutando no endereço da interface de rede do ***Primary*** utilizada para comunicação com o ***Standby***. Neste tutorial utilizamos Tailscale, mas esse endereço também pode pertencer a uma rede local ou outra rede privada. Somente `127.0.0.1:5432` não permite a conexão de um servidor ***Standby*** localizado em outra máquina.

> **Nota:** na saída completa do comando `ss`, pode aparecer `0.0.0.0:*` como endereço remoto. Isso não significa que o PostgreSQL esteja escutando em todas as interfaces. Para identificar onde o PostgreSQL está escutando, observe o endereço local associado à porta `5432`.

## 2 — Instalar a versão correspondente do PostgreSQL no ***Standby***

### 2.1 — Identificar a versão PostgreSQL no ***Primary***

**Objetivo:** descobrir qual versão está efetivamente executando no ***Primary***. Essa informação determinará o que prepararemos no ***Standby***.

No ***Primary***, execute:

```bash
sudo -u postgres psql -X -Atc "SHOW server_version;"
```

Saída esperada:

```text
18.6 (Ubuntu 18.6-1.pgdg24.04+2)
```
> Guarde essa informação. Nesse exemplo, a versão major é 18 e a minor é 6. Ela será utilizada na preparação do ***Standby***.

## 2.2 — Verificar existência do PostgreSQL no ***Standby***

**Objetivo:** verificar se o PostgreSQL já está instalado no ***Standby*** e identificar eventuais clusters PostgreSQL existentes antes de preparar o cluster destinado à replicação.

No ***Standby*** execute:

```bash
psql --version 2>/dev/null || echo "psql não instalado"
echo
pg_lsclusters 2>/dev/null || echo "Nenhum cluster PostgreSQL encontrado"
```

Saída esperada:

```text
psql não instalado

Nenhum cluster PostgreSQL encontrado
```

> **Nota:** se já existir um cluster PostgreSQL em uso, não o remova. Mais adiante será mostrado como manter o cluster existente e preparar outro cluster independente para a réplica.

### 2.3 — Verificar disponibilidade da versão no ***Standby***

**Objetivo:** verificar se o repositório atualmente configurado no ***Standby*** oferece a mesma versão major identificada no ***Primary*** e qual versão minor seria instalada.

no ***Standby***, execute (substitua 18 pela versão major identificada no ***Primary***):

```bash
apt-cache policy postgresql-18
```

Saída esperada:

```text
postgresql-18:
  Instalado: (nenhum)
  Candidato: 18.6-1.pgdg24.04+2
  Tabela de Versão:
     18.6-1.pgdg24.04+2 500
```

Se a saída apresentar uma versão candidata compatível, não é necessário adicionar outro repositório. Anote a versão candidata e prossiga para a etapa 2.5.

Se a saída for:

```text
N: Não foi possível encontrar o pacote postgresql-18
```
ou não houver uma versão candidata da mesma major utilizada pelo ***Primary***, prossiga para a etapa 2.4.

### 2.4 — Adicionar o repositório oficial PostgreSQL (PGDG) no ***Standby***

**Objetivo:** disponibilizar no ***Standby*** versões do PostgreSQL que não estão presentes nos repositórios atualmente configurados.

No ***Standby***, execute o comando para instalar as ferramentas comuns do PostgreSQL:

```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

O script identificará o Ubuntu 24.04 como noble e solicitará confirmação:

```text
This script will enable the PostgreSQL APT repository on apt.postgresql.org on
your system. The distribution codename used will be noble-pgdg.
Press Enter to continue, or Ctrl-C to abort.
```

Pressione Enter para continuar.

Ao final, a saída esperada inclui:

```text
Writing /etc/apt/sources.list.d/pgdg.sources ...
Running apt-get update ...
...
You can now start installing packages from apt.postgresql.org.
```

Confirme que existe uma versão candidata da mesma major utilizada pelo ***Primary***. Sempre que possível, mantenha também os servidores ***Primary*** e ***Standby*** na mesma versão minor.

No ***Standby***, execute (substitua 18 pela versão major identificada no ***Primary***):

```bash
apt-cache policy postgresql-18
```

Saída esperada:

```text
postgresql-18:
  Instalado: (nenhum)
  Candidato: 18.6-1.pgdg24.04+2
  Tabela de Versão:
     18.6-1.pgdg24.04+2 500
```

### 2.5 — Instalar PostgreSQL no ***Standby***

**Objetivo:** instalar no ***Standby*** a mesma versão major do PostgreSQL utilizada pelo ***Primary***. Neste procedimento, permitiremos que o `postgresql-common` crie automaticamente o cluster local padrão. Esse cluster ainda não é a réplica e será preparado posteriormente para receber a cópia física do ***Primary***.

No ***Standby***, execute (Substitua 18 pela versão major identificada no ***Primary***):

```bash
sudo apt install -y postgresql-18
```

Saída esperada:

```text
Creating new PostgreSQL cluster 18/main ...
...
Data page checksums are enabled.
...
syncing data to disk ... ok
```

> **Importante:** neste momento foi criado um novo cluster PostgreSQL local, inicializado com os bancos e estruturas padrão. Ele ainda não contém os dados do ***Primary*** e não deve ser confundido com o futuro ***Standby***.

Confirme que a versão major instalada corresponde à utilizada pelo ***Primary***. Identifique ainda o cluster criado, sua porta, estado e diretório de dados.

No ***Standby***, execute:

```bash
psql --version
echo
pg_lsclusters
```

Saída esperada:

```text
psql (PostgreSQL) 18.6 (Ubuntu 18.6-1.pgdg24.04+2)

Ver Cluster Port Status Owner    Data directory
18  main    5432 online postgres /var/lib/postgresql/18/main
```

## 3 — Preparar o ***Standby*** para receber a réplica

### 3.1 — Testar a conectividade PostgreSQL entre ***Standby*** e ***Primary***

**Objetivo:** antes de mexermos no cluster PostgreSQL do ***Standby***, confirmar que ela consegue alcançar a porta `5432` do ***Primary*** pela rede escolhida para a replicação.

No ***Standby**, execute:

```bash
pg_isready -h IP_REDE_PRIMARY -p 5432
```

Saída esperada:

```text
IP_REDE_PRIMARY:5432 - accepting connections
```

### 3.2 — Validar a autenticação do usuário de replicação

**Objetivo:** confirmar que o usuário `lnd_replicator` consegue autenticar no ***Primary*** a partir do ***Standby*** e que a regra configurada no `pg_hba.conf` está funcionando.

No ***Standby***, execute:

```bash
psql "host=IP_REDE_PRIMARY port=5432 user=lnd_replicator replication=true" -W -c "IDENTIFY_SYSTEM;"
```

O parâmetro `-W` fará o `psql` solicitar a senha interativamente para evitar que seja incluída diretamente no comando e fique registrada no histórico do terminal. Informe a senha definida para o usuário `lnd_replicator`:

```text
Password:
```

Saída esperada:
```text
      systemid       | timeline |  xlogpos  | dbname 
---------------------+----------+-----------+--------
 1234567890123456789 |        1 | 0/1234567 | 
(1 row)
```

> **Nota:** o comando `IDENTIFY_SYSTEM` é executado através do protocolo de replicação do PostgreSQL. Uma resposta válida confirma que o usuário conseguiu estabelecer uma conexão de replicação física com o ***Primary***. Os valores de `systemid`, `timeline` e `xlogpos` variam de acordo com cada cluster e com a posição atual do WAL; portanto, não precisam coincidir com os valores apresentados no exemplo.

### 3.3 — Verificar o cluster PostgreSQL local do ***Standby***

**Objetivo:** identificar o cluster PostgreSQL existente no ***Standby*** antes de prepará-lo para receber a cópia física do ***Primary***.

No ***Standby***, execute:

```bash
pg_lsclusters
```

Saída esperada:

```text
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

Verifique quais bancos existem atualmente nesse cluster:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT datname
FROM pg_database
ORDER BY datname;
"
```

Em uma instalação nova, ainda não utilizada, a saída esperada é:

```text
  datname  
-----------
 postgres
 template0
 template1
(3 rows)
```

> **Importante:** se forem encontrados bancos além de `postgres`, `template0` e `template1`, não prossiga com a remoção ou substituição desse cluster. A presença de outros bancos pode indicar que o servidor PostgreSQL já está em uso e que existem dados que precisam ser preservados.
> 
> Nesse caso, o ***Standby*** ainda poderá ser utilizado para a replicação, mas o cluster existente deverá ser mantido. Será necessário criar um segundo cluster PostgreSQL dedicado à réplica física, utilizando outro diretório de dados e uma porta diferente da utilizada pelo cluster existente.
> 
> Por exemplo, um servidor poderá manter simultaneamente:
> 
> ```text
> 18  main         5432  online  postgres  /var/lib/postgresql/18/main
> 18  lndstandby  5433  online  postgres  /var/lib/postgresql/18/lndstandby
> ```
> Nesse exemplo, `18/main` permanece atendendo os bancos já existentes, enquanto `18/lndstandby` é reservado exclusivamente para receber a réplica física do ***Primary***.
>
> Não utilize `pg_dropcluster`, não apague o diretório de dados e não sobrescreva um cluster que contenha dados que devam ser preservados.

#### 3.3.1 — Cenário alternativo: ***Standby*** com cluster PostgreSQL já em uso

**Objetivo:** preparar a replicação quando o ***Standby*** já possui um cluster PostgreSQL em uso, preservando integralmente esse cluster e criando posteriormente um segundo cluster dedicado à réplica física.

> **Nota:** se o servidor possuir apenas o cluster recém-criado, pule diretamente para a etapa 3.4.

#### 3.3.1.1 — Verificar as portas PostgreSQL em uso

Antes de criar outro cluster, precisamos saber quais portas já estão ocupadas.

No ***Standby***, execute:

```bash
sudo ss -ltnp | grep postgres
```

Saída esperada:

```text
LISTEN 0  200  127.0.0.1:5432  0.0.0.0:*  users:(("postgres",...))
```

Neste exemplo, o cluster PostgreSQL existente está utilizando a porta `5432`. Antes de criar um segundo cluster, escolha uma porta que não esteja em uso. Neste tutorial será utilizada a porta `5433`.

#### 3.3.1.2 — Criar um segundo cluster PostgreSQL

**Objetivo:** criar um cluster independente destinado à réplica, preservando o cluster PostgreSQL que já estaria em uso.

No ***Standby***, execute:

```bash
sudo pg_createcluster 18 lndstandby --port=5433 --start
```

Saída esperada:

```text
...
Ver Cluster     Port Status Owner    Data directory                     Log file
18  lndstandby 5433 online postgres /var/lib/postgresql/18/lndstandby /var/log/postgresql/postgresql-18-lndstandby.log
```

No ***Standby***, execute o comando para confirmar a criação do cluster:

```bash
pg_lsclusters
```

Nesse cenário, o cluster `18/main` permanece em funcionamento na porta `5432`, enquanto o cluster `18/lndstandby`, na porta `5433`, será dedicado à réplica física do ***Primary***. Dessa forma, a preparação da réplica não exige substituir ou interromper o cluster PostgreSQL que já estava em uso.

Saída esperada:

```text
Ver Cluster    Port Status Owner    Data directory                    Log file
18  lndstandby 5433 online postgres /var/lib/postgresql/18/lndstandby /var/log/postgresql/postgresql-18-lndstandby.log
18  main       5432 online postgres /var/lib/postgresql/18/main       /var/log/postgresql/postgresql-18-main.log
```

> **Nota:** evite utilizar hífen (`-`) no nome do cluster. O `pg_createcluster` alerta que nomes contendo hífen podem causar problemas na integração com o `systemd`. Neste tutorial será utilizado o nome `lndstandby`.

### 3.4 — Preparar o cluster ***Standby*** para receber a réplica

**Objetivo:** interromper o cluster PostgreSQL que será transformado em réplica física antes de substituir seu diretório de dados pela cópia do ***Primary***.

> **Importante:** esta etapa começa a preparação efetiva do cluster ***Standby***. Certifique-se de ter identificado corretamente, na etapa 3.3, qual cluster será utilizado para a réplica.

> **Cluster secundário:** se foi necessário criar o cluster dedicado `18/lndstandby` conforme a seção 3.3.1, nos comandos seguintes substitua `main` por `lndstandby`. Não interrompa o `18/main` que já estava em uso.

#### 3.4.1 — Parar o cluster destinado à réplica

No ***Standby***, execute:

```bash
sudo pg_ctlcluster 18 main stop
```

Confirme:

```bash
pg_lsclusters
```

Saída esperada:

```text
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 down   postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

#### 3.4.2 — Remover o diretório de dados do cluster local

**Objetivo:** deixar o diretório de dados vazio para receber a cópia física do ***Primary***.

> **Atenção:** esta operação é destrutiva. Execute-a somente depois de confirmar que o cluster correto está parado e que não contém dados que precisem ser preservados.

No ***Standby***, execute:

```bash
sudo find /var/lib/postgresql/18/main -mindepth 1 -maxdepth 1 -exec rm -rf -- {} +
```

No ***Standby***, execute o comando para verificar:

```bash
sudo ls -la /var/lib/postgresql/18/main
```

Saída esperada:

```text
total 8
drwx------ 2 postgres postgres ... .
drwxr-xr-x 3 postgres postgres ... ..
```

### 3.5 — Configurar a autenticação do ***Standby*** no ***Primary***

**Objetivo:** permitir que os processos executados pelo usuário `postgres` no ***Standby*** autentiquem no ***Primary*** sem expor a senha nos comandos.

#### 3.5.1 — Criar o arquivo `.pgpass`

No ***Standby***, execute:

```bash
sudo -u postgres nano /var/lib/postgresql/.pgpass
```

Adicione uma única linha:

```conf
IP_REDE_PRIMARY:5432:*:lnd_replicator:SENHA_DO_USUARIO
```

Substitua `SENHA_DO_USUARIO` pela senha definida anteriormente para `lnd_replicator`.

Salve o arquivo (Ctrl+O e Enter) e saia do Nano (Ctrl+X).

O arquivo `.pgpass` contém a senha em texto simples. Ele deve pertencer ao usuário `postgres` e ter permissão `600`. O PostgreSQL ignora o arquivo se as permissões permitirem acesso por outros usuários. No ***Standby***, execute:

```bash
sudo chown postgres:postgres /var/lib/postgresql/.pgpass
sudo chmod 600 /var/lib/postgresql/.pgpass
```

No ***Standby***, execute o comando para verificar:

```bash
sudo ls -l /var/lib/postgresql/.pgpass
```

Saída esperada:

```text
-rw------- 1 postgres postgres ... /var/lib/postgresql/.pgpass
```

#### 3.5.2 — Validar a autenticação pelo `.pgpass`

**Objetivo:** confirmar que o usuário `postgres` do ***Standby*** consegue autenticar no protocolo de replicação do ***Primary*** utilizando o `.pgpass`.

No ***Standby***, execute:

```bash
sudo -u postgres psql "host=IP_REDE_PRIMARY port=5432 user=lnd_replicator replication=true passfile=/var/lib/postgresql/.pgpass" -c "IDENTIFY_SYSTEM;"
```

O comando deve executar sem solicitar senha. Isso confirma que o arquivo `.pgpass` está sendo utilizado para autenticar o usuário de replicação.

### 3.6 — Criar a réplica física a partir do ***Primary***

**Objetivo:** copiar o cluster PostgreSQL do ***Primary*** para o ***Standby*** e preparar a cópia para posteriormente iniciar como réplica física.

#### 3.6.1 — Executar o `pg_basebackup`

No ***Standby***, execute:

```bash
sudo -u postgres pg_basebackup \
  -D /var/lib/postgresql/18/main \
  -d "host=IP_REDE_PRIMARY port=5432 user=lnd_replicator application_name=standby1 passfile=/var/lib/postgresql/.pgpass" \
  -X stream \
  -S lnd_pg_standby_01 \
  -R \
  -P
```

Saída esperada:

```text
waiting for checkpoint
...
XXXXXX/XXXXXX kB (100%), 1/1 tablespace
```

> **Nota:** a mensagem `waiting for checkpoint` é normal e pode permanecer bastante tempo antes do início da cópia. Ao concluir com sucesso, o `pg_basebackup` retorna ao prompt do terminal. Os valores apresentados pelo indicador de progresso variam conforme o tamanho do cluster.

### 3.7 — Iniciar a replicação física no ***Standby***

**Objetivo:** iniciar o PostgreSQL do ***Standby*** e confirmar que o cluster copiado pelo `pg_basebackup` inicia em modo de recuperação e começa a receber WAL do ***Primary***.

#### 3.7.1 — Iniciar o cluster ***Standby***

No ***Standby***, execute:

```bash
sudo pg_ctlcluster 18 main start
```

Em seguida, confirme o estado:

```bash
pg_lsclusters
```

Saída esperada:

```text
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

#### 3.7.2 — Confirmar que o cluster está em modo Standby

**Objetivo:** verificar que o PostgreSQL iniciou em modo de recuperação, e não como um Primary independente.

No ***Standby***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT pg_is_in_recovery();
"
```

Saída esperada:

```text
 pg_is_in_recovery
-------------------
 t
(1 row)
```

Ainda no ***Standby***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT status,
       sender_host,
       sender_port,
       slot_name,
       written_lsn,
       flushed_lsn,
       latest_end_lsn
FROM pg_stat_wal_receiver;
"
```

Saída esperada:

```text
 status    | sender_host  | sender_port |      slot_name       | written_lsn | flushed_lsn | latest_end_lsn
-----------+--------------+-------------+----------------------+-------------+-------------+---------------
 streaming | IP_PRIMARY   |        5432 | lnd_pg_standby_01    | ...         | ...         | ...
```

#### 3.7.3 — Verificar a replicação pelo ***Primary***

**Objetivo:** confirmar pelo lado do ***Primary*** que o ***Standby*** está conectado e verificar seu estado de sincronização.

No ***Primary***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT application_name,
       client_addr,
       state,
       sync_state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       write_lag,
       flush_lag,
       replay_lag
FROM pg_stat_replication;
"
```

Saída esperada:

```text
 application_name | client_addr     |   state   | sync_state | sent_lsn | write_lsn | flush_lsn | replay_lsn | ...
------------------+-----------------+-----------+------------+----------+-----------+-----------+------------+----
 standby1         | IP_REDE_STANDBY | streaming | async      | ...      | ...       | ...       | ...        | ...
```

> **Nota:** após a primeira inicialização do ***Standby***, `state` pode aparecer temporariamente como `catchup` enquanto os WALs pendentes são recebidos e aplicados. Aguarde até que o estado passe para `streaming` antes de prosseguir.

### 3.8 — Ativar a replicação síncrona com `remote_apply`

**Objetivo:** fazer com que os commits síncronos do ***Primary*** aguardem até que o WAL correspondente tenha sido aplicado no ***Standby***, usando `standby1` como réplica síncrona.

#### 3.8.1 — Adicionar a configuração de replicação síncrona

No ***Primary***, execute:

```bash
sudo nano /etc/postgresql/18/main/conf.d/99-lnd-replication.conf
```

Ao final do arquivo, acrescente:

```conf
# Replicacao sincrona
synchronous_standby_names = 'standby1'
synchronous_commit = remote_apply
```

Salve o arquivo (Ctrl+O e Enter) e saia do Nano (Ctrl+X).

#### 3.8.2 — Validar a configuração antes do reload

**Objetivo:** confirmar que as novas configurações foram interpretadas corretamente pelo PostgreSQL e que não há erros no arquivo.

No ***Primary***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT sourcefile,
       sourceline,
       name,
       setting,
       applied,
       error
FROM pg_file_settings
WHERE name IN ('synchronous_standby_names', 'synchronous_commit')
ORDER BY sourceline;
"
```

Na saída esperada, confirme principalmente:

```text
|           name            |   setting    | applied | error
+---------------------------+--------------+---------+-------
| synchronous_standby_names | standby1     | t       |
| synchronous_commit        | remote_apply | t       |
 ```

#### 3.8.3 — Aplicar a configuração síncrona

**Objetivo:** recarregar a configuração do PostgreSQL para ativar `standby1` como réplica síncrona e utilizar `remote_apply`.

No ***Primary***, execute:

```bash
sudo pg_ctlcluster 18 main reload
```

Confirme os valores efetivamente carregados. No ***Primary***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT name,
       setting
FROM pg_settings
WHERE name IN ('synchronous_standby_names', 'synchronous_commit')
ORDER BY name;
"
```

Saída esperada:

```text
           name            |   setting    
---------------------------+--------------
 synchronous_commit        | remote_apply
 synchronous_standby_names | standby1
(2 rows)
```

#### 3.8.4 — Confirmar que o ***Standby*** está síncrono

**Objetivo:** verificar no ***Primary*** que `standby1` foi selecionada como réplica síncrona e permanece em `streaming`.


No ***Primary***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT application_name,
       client_addr,
       state,
       sync_state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       write_lag,
       flush_lag,
       replay_lag
FROM pg_stat_replication;
"
```

Na saída esperada, confirme principalmente:

```text
  application_name |   state   | sync_state |
 ------------------+-----------+------------+
  standby1         | streaming | sync
 ``` 

## 4 — Operação em contingência: alternar entre replicação síncrona e assíncrona em caso de falha do ***Standby***

Quando o PostgreSQL está configurado com `synchronous_commit = remote_apply` e o ***Standby*** definido em `synchronous_standby_names` fica indisponível, os commits que dependem da replicação síncrona podem permanecer aguardando a réplica.

Nessa situação, caso seja necessário manter o ***Primary*** e o LND operacionais enquanto o ***Standby*** é recuperado, é possível retirar temporariamente a exigência de uma réplica síncrona.

> **Importante:** ao operar dessa forma, o ***Primary*** continuará aceitando gravações sem aguardar o ***Standby***. Durante esse período, deixa de existir a garantia de que as transações confirmadas no ***Primary*** já estejam aplicadas na réplica.

### 4.1 — Confirmar a indisponibilidade do ***Standby***

No ***Primary***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT application_name,
       client_addr,
       state,
       sync_state
FROM pg_stat_replication;
"
```

Se o ***Standby*** estiver desconectado, a saída será:

```text
 application_name | client_addr | state | sync_state
------------------+-------------+-------+-----------
(0 rows)
```

### 4.2 — Alterar temporariamente para replicação assíncrona

No ***Primary***, abra:

```bash
sudo nano /etc/postgresql/18/main/conf.d/99-lnd-replication.conf
```

Altere somente:

```conf
synchronous_standby_names = 'standby1'
```

para:

```conf
synchronous_standby_names = ''
```

Salve o arquivo (Ctrl+O e Enter) e saia do Nano (Ctrl+X).

> **Nota:** `synchronous_commit = remote_apply` permanece configurado. Ao deixar `synchronous_standby_names` vazio, nenhuma ***Standby*** é exigida para confirmação síncrona. Isso permite posteriormente retornar ao modo `remote_apply` alterando apenas `synchronous_standby_names`.

### 4.3 — Validar a configuração antes do reload

No ***Primary***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT sourcefile,
       sourceline,
       name,
       setting,
       applied,
       error
FROM pg_file_settings
WHERE name IN ('synchronous_standby_names', 'synchronous_commit')
ORDER BY sourceline;
"
```

Confirme principalmente:

```text
synchronous_standby_names |              | t |
synchronous_commit        | remote_apply | t |
```

e que a coluna error esteja vazia.

### 4.4 — Aplicar o modo assíncrono

No ***Primary***, execute:

```bash
sudo pg_ctlcluster 18 main reload
```

Confirme que a mudança foi aplicada. No ***Primary***, execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT name,
       setting
FROM pg_settings
WHERE name IN ('synchronous_standby_names', 'synchronous_commit')
ORDER BY name;
"
```

Confirme principalmente:

```text
           name            |   setting
---------------------------+--------------
 synchronous_commit        | remote_apply
 synchronous_standby_names |
(2 rows)
```

> **Nota:** não é necessário reiniciar o PostgreSQL ou o LND.

#### 4.4.1 — Acompanhar o estado da saúde do slot físico

Enquanto o ***Standby*** estiver indisponível, verifique periodicamente a saúde do slot físico no ***Primary***:

```bash
sudo -u postgres psql -X -d postgres -P pager=off -P expanded=on -c "
SELECT
    slot_name,
    active AS slot_active,
    wal_status,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained,
    pg_size_pretty(safe_wal_size) AS safe_wal_remaining
FROM pg_replication_slots
WHERE slot_name = 'lnd_pg_standby_01';
"
```

Uma saída saudável durante a indisponibilidade do Standby deve apresentar, por exemplo:

```text
slot_name          | lnd_pg_standby_01
slot_active        | f
wal_status         | reserved
wal_retained       | 129 MB
safe_wal_remaining | 10116 MB
```

Se o `safe_wal_remaining` estiver muito próximo de 0, é possível disponibilizar mais espaço em disco para o WAL. Para isso, no ***Primary***, execute:
```bash
sudo -u postgres psql -X -d postgres -c "ALTER SYSTEM SET max_slot_wal_keep_size = '100GB';"
```

Troque o `100GB` pelo valor que achar necessário de acordo com a disponibilidade de espaço em disco no ***Standby***.

Saída esperada:
```text
ALTER SYSTEM
```

Depois, execute:
```bash
sudo -u postgres psql -X -d postgres -c "SELECT pg_reload_conf();"
```

Saída esperada:
```text
 pg_reload_conf
----------------
 t
```

### 4.5 — Retornar ao modo síncrono `remote_apply` após o retorno do ***Standby***

> **Importante:** não restaure o modo síncrono apenas porque o ***Standby*** voltou a responder. Antes, confirme que ele está novamente conectado, em `streaming` e suficientemente atualizado em relação ao ***Primary***.

Depois que o ***Standby*** estiver recuperado, no ***Primary*** execute:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT application_name,
       client_addr,
       state,
       sync_state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn
FROM pg_stat_replication;
"
```

Enquanto `synchronous_standby_names` permanecer vazio, um ***Standby*** saudável deverá aparecer em `streaming`, porém ainda como:

```text
application_name |   state   | sync_state
-----------------+-----------+-----------
standby1         | streaming | async
```

Aguarde até que `sent_lsn`, `write_lsn`, `flush_lsn` e `replay_lsn` estejam próximos ou iguais antes de restabelecer a exigência síncrona.

### 4.6 — Reativar o ***Standby*** síncrono

No ***Primary***, abra:

```bash
sudo nano /etc/postgresql/18/main/conf.d/99-lnd-replication.conf
```

Altere:

```conf
synchronous_standby_names = ''
```

para:

```conf
synchronous_standby_names = 'standby1'
```

No ***Primary***, aplique:

```bash
sudo pg_ctlcluster 18 main reload
```

Por fim, confirme:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT application_name,
       client_addr,
       state,
       sync_state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn
FROM pg_stat_replication;
"
```

O Standby deverá voltar a apresentar:

```text
application_name |   state   | sync_state
-----------------+-----------+-----------
standby1         | streaming | sync
```

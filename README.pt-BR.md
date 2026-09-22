# Replicação PostgreSQL para LND

[🇺🇸 English version](README.md)

Um projeto prático para construção e documentação de uma **arquitetura de replicação física PostgreSQL para LND**, com foco em consistência dos dados, recuperação controlada e proteção contra perda de estado.

O projeto utiliza mecanismos nativos de replicação do PostgreSQL para manter uma cópia independente e continuamente atualizada do cluster PostgreSQL utilizado por um node LND.

## Arquitetura

A arquitetura é baseada em dois servidores:

- **Primary** — servidor PostgreSQL normalmente utilizado pelo LND;
- **Standby** — servidor independente que mantém uma réplica física do cluster PostgreSQL.

A replicação é implementada utilizando **PostgreSQL Physical Streaming Replication** com um *physical replication slot* dedicado.

O modo normal de operação utiliza replicação síncrona com `remote_apply`. A arquitetura mantém intencionalmente o *failover* e a recuperação sob controle explícito do operador, evitando mecanismos de promoção automática que poderiam aumentar o risco de *split-brain* ou divergência não intencional do estado do LND.

## Documentação

### Português

**01 — Configuração da Replicação PostgreSQL**

Abrange:

- preparação do PostgreSQL Primary;
- instalação e preparação do Standby;
- replicação física por streaming;
- physical replication slot;
- replicação síncrona com `remote_apply`;
- operação temporária em modo assíncrono durante contingência;
- retorno seguro ao modo síncrono.

[Leia o tutorial em português →](docs/pt-BR/01-configuracao-replicacao.md)

### English

**01 — PostgreSQL Replication Setup**

Covers:

- preparation of the PostgreSQL Primary;
- installation and preparation of the Standby;
- physical streaming replication;
- physical replication slot;
- synchronous replication with `remote_apply`;
- temporary asynchronous contingency operation;
- safe return to synchronous operation.

[Read the English tutorial →](docs/en/01-replication-setup.md)

## Ambiente de Referência

A implementação e os testes documentados no primeiro tutorial foram realizados utilizando:

- **Ubuntu Server 24.04 LTS**
- **PostgreSQL 18.6**
- **LND 0.21.3-beta**
- **Nodes utilizados:**
  - Naghust SA | BR⚡LN (02dd543868366e0bc3e498ab7c687d795a30ce0f70d2d034006b4654bbe887af8a)
  - Naghust Replication Test (026eea7a1d59689a4e40c3504cbaca131d712564436ea081db6fd81050b585f0b7)

## Agradecimentos

Este projeto foi inspirado, em parte, pelo trabalho sobre replicação PostgreSQL documentado por **Filou (Filouman)**, operador do node Lightning **Nodelou**.

Seu trabalho forneceu uma importante referência prática para a utilização de replicação física por streaming do PostgreSQL e replicação síncrona com `remote_apply` em um ambiente de node Lightning.

A implementação documentada neste repositório foi testada de forma independente e adaptada à arquitetura aqui apresentada, incluindo seus procedimentos de replicação, contingência e operação.

[Projeto PostgreSQL Replication de Filouman →](https://github.com/Filouman/Postgresql_replication)

## Roadmap do Projeto

Este repositório pretende documentar a arquitetura de forma progressiva.

Os tópicos atuais e planejados incluem:

1. replicação física PostgreSQL para LND;
2. promoção controlada do Standby;
3. recuperação após falha do Primary;
4. recuperação e ressincronização da replicação PostgreSQL;
5. integração com os procedimentos operacionais do LND;
6. proteções adicionais para um failover controlado.

## Princípios do Projeto

- Preferir mecanismos nativos do PostgreSQL sempre que possível.
- Proteger o estado do LND como objetivo principal.
- Manter a promoção e a recuperação sob controle explícito do operador.
- Evitar failover automático que possa introduzir risco de split-brain.
- Validar o estado da replicação antes da promoção.
- Permitir operação assíncrona temporária e controlada durante falha do Standby.
- Manter os procedimentos reproduzíveis e auditáveis.

## Aviso Importante

Os procedimentos documentados neste repositório afetam diretamente o banco de dados PostgreSQL utilizado pelo LND.

Antes de aplicá-los a um node Lightning em produção:

- entenda cada comando antes de executá-lo;
- mantenha backups adequados;
- verifique a compatibilidade das versões do PostgreSQL e do LND;
- adapte endereços, caminhos, portas e parâmetros à sua infraestrutura;
- valide o procedimento em um ambiente de testes sempre que possível.

Este projeto documenta uma arquitetura e um procedimento operacional testados, mas não substitui uma estratégia de backup e recuperação de desastres específica para cada infraestrutura.

## Licença

A documentação deste projeto está licenciada sob a
[Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE).

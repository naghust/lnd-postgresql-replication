# PostgreSQL Replication for LND

[🇧🇷 Versão em português](../pt-BR/01-configuracao-replicacao.md)

## Introduction

A Lightning node can accumulate years of operation, active channels, routing history, and information essential to service continuity. Although mechanisms such as the Static Channel Backup (SCB) are fundamental for channel recovery, they do not replace a high-availability strategy and protection of the database used by LND.

When LND uses PostgreSQL as its backend, PostgreSQL's native mechanisms can be used to maintain a continuously updated copy of the database on another server.

This project was created with that objective: **to build and document a PostgreSQL physical replication architecture for an LND node, prioritizing data consistency, controlled recovery, and protection against state loss**.

The implementation and tests documented in this project were performed using the following reference environment:

- **Ubuntu Server 24.04 LTS**;
- **PostgreSQL 18.6**;
- **LND 0.21.3-beta**;
- **Nodes used:**
  - Naghust SA | BR⚡LN (02dd543868366e0bc3e498ab7c687d795a30ce0f70d2d034006b4654bbe887af8a)
  - Naghust Replication Test (026eea7a1d59689a4e40c3504cbaca131d712564436ea081db6fd81050b585f0b7)

The architecture uses two servers:

- **Primary** — the PostgreSQL server normally used by LND;
- **Standby** — an independent server that maintains a continuously updated physical replica of the PostgreSQL cluster.

Replication is performed using **PostgreSQL Physical Streaming Replication**, with a *physical replication slot*. Communication between the servers can take place over a private network, VPN, or another appropriately protected replication network.

The normal operating mode adopted in this project uses:

```conf
synchronous_commit = remote_apply
synchronous_standby_names = 'standby1'
```

With `remote_apply`, a synchronous transaction committed by the ***Primary*** only returns to the client after the corresponding WAL has been applied by the ***Standby*** selected as synchronous. This choice prioritizes consistency between the two servers, at the cost of adding replication latency to the transaction commit path.

This decision also creates an important operational consequence: **the unavailability of the ***Standby*** can block commits on the ***Primary*** even if the primary server itself remains operational**.

For this reason, the project also documents a contingency procedure that allows the synchronous replica requirement to be temporarily removed, keeping LND operational while the ***Standby*** is being recovered.

The architecture was designed around the following principles:

- use native PostgreSQL features whenever possible;
- avoid automatic failover mechanisms that could increase the risk of *split-brain*;
- keep promotion and recovery under explicit operator control;
- prioritize protection of the LND state;
- allow temporary asynchronous operation in the event of a ***Standby*** failure;
- validate the replication state before any promotion;
- document procedures that can be reproduced and audited.

This repository documents the construction of this architecture progressively.

The first part covers PostgreSQL preparation, creation of the ***Standby***, physical replication, activation of synchronous mode with `remote_apply`, and contingency operation.

The following stages will cover the ***Standby*** promotion process, recovery after a ***Primary*** failure, and integration of this mechanism with LND operation.

> **Warning:** the procedures described in this project directly involve the database used by LND. Before applying them to a production node, understand each command, maintain appropriate backups, and adapt addresses, paths, versions, and parameters to your own infrastructure.

## 1 — Prepare PostgreSQL on the ***Primary*** for replication

### 1.1 — Create a dedicated configuration file for replication

**Objective:** keep the replication parameters separate from the main `postgresql.conf`, making configuration maintenance, auditing, and potential rollback easier.

On the ***Primary***, run the command below to create a dedicated file inside the `conf.d` directory (replace 18 with the major version identified on the ***Primary***):

```bash
sudo nano /etc/postgresql/18/main/conf.d/99-lnd-replication.conf
```

Add:

```conf
# PostgreSQL Physical Replication - LND

# WAL and replication processes
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10

# Limit of WAL retained by replication slots
max_slot_wal_keep_size = 10GB
```

Save the file (Ctrl+O and Enter) and exit Nano (Ctrl+X).

#### Understanding the parameters

- `wal_level = replica` determines that the WAL contains the information required to support physical replication.

- `max_wal_senders = 10` allows up to 10 simultaneous `walsender` processes. Among other functions, these processes are responsible for transmitting WAL from the ***Primary*** to the ***Standby***.

- `max_replication_slots = 10` allows up to 10 `replication slots`. Later, we will create a dedicated slot for the ***Standby***.

- `max_slot_wal_keep_size = 10GB` sets a limit on the amount of WAL that may be retained due to `replication slots`. In this tutorial, we will use 10 GB as the default configuration.

### 1.2 — Allow PostgreSQL connections over the network used for replication

**Objective:** allow PostgreSQL to accept connections on the ***Primary*** network interface used for communication with the ***Standby***. In this tutorial, we use the Tailscale network, but a local network or another private network between the servers can also be used.

On the ***Primary***, run:

```bash
sudo nano /etc/postgresql/18/main/conf.d/99-lnd-replication.conf
```

Add:

```conf
# Network interfaces
listen_addresses = 'localhost,IP_REDE_PRIMARY'
```

Replace `IP_REDE_PRIMARY` with the IP address of the ***Primary*** interface. For example:

- Tailscale: `100.x.x.x`;
- local network: `192.168.x.x`;
- another private network: use the corresponding address of that interface.

> **Note:** if PostgreSQL is already configured to listen on the interface that will be used by the ***Standby***, there is no need to change `listen_addresses`.

### 1.3 — Validate the file before applying the configuration

On the ***Primary***, run:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT sourcefile, sourceline, name, setting, error
FROM pg_file_settings
WHERE sourcefile LIKE '%99-lnd-replication.conf%'
ORDER BY sourceline;
"
```

Expected output:

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

The query allows you to verify how PostgreSQL interpreted the configured parameters. Parameters that require a restart, such as `listen_addresses`, may show `setting could not be applied` while PostgreSQL is still running with the previous configuration. The configuration will be confirmed after the service is restarted.

### 1.4 — Authorize the ***Standby*** in `pg_hba.conf`

**Objective:** allow the ***Standby*** to establish replication connections with PostgreSQL on the ***Primary***.

On the ***Primary***, run:

```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
```

At the end of the file, add the following (replace `IP_REDE_STANDBY` with the IP address of the ***Standby*** network):

```conf
# PostgreSQL physical replication - Standby
host    replication    lnd_replicator    IP_REDE_STANDBY/32    scram-sha-256
```

Replace `IP_REDE_STANDBY` with the IP address of the ***Standby*** interface. For example:

- Tailscale: `100.x.x.x`;
- local network: `192.168.x.x`;
- another private network: use the corresponding address of that interface.

#### 1.4.1 — Validate `pg_hba.conf`

On the ***Primary***, run:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT line_number, type, database, user_name, address, auth_method, error
FROM pg_hba_file_rules
WHERE user_name @> ARRAY['lnd_replicator']::text[];
"
```

Expected output:

```text
 line_number | type |   database    |    user_name     |     address     |  auth_method  | error 
-------------+------+---------------+------------------+-----------------+---------------+-------
         XXX | host | {replication} | {lnd_replicator} | IP_REDE_STANDBY | scram-sha-256 | 
(1 row)
```

### 1.5 — Create the dedicated replication user

**Objective:** create a PostgreSQL user on the ***Primary*** exclusively for physical replication, with only the required privileges.

On the ***Primary***, run:

```bash
sudo -u postgres psql
```

You should reach the following prompt:

```text
postgres=#
```

Now create the replication user without placing the password directly in the command.

At the `postgres=#` prompt, run:

```sql
CREATE ROLE lnd_replicator WITH REPLICATION LOGIN;
```

Expected output:

```text
CREATE ROLE
```

Still inside `psql`, set the password interactively by running:

```psql
\password lnd_replicator
```

PostgreSQL will prompt:

```text
Enter new password for user "lnd_replicator":
Enter it again:
```

> **Important:** choose a strong and unique password for replication and store it in a secure location, as it will be used when preparing the ***Standby***.

#### 1.5.1 — Validate the user

Still inside `psql`, run:

```sql
SELECT rolname, rolcanlogin, rolreplication, rolsuper
FROM pg_roles
WHERE rolname = 'lnd_replicator';
```

Expected output:

```text
    rolname     | rolcanlogin | rolreplication | rolsuper 
----------------+-------------+----------------+----------
 lnd_replicator | t           | t              | f
(1 row)
```

> **Note:** the `lnd_replicator` user is an account dedicated exclusively to PostgreSQL replication. It has permission to log in (`rolcanlogin = t`) and initiate replication connections (`rolreplication = t`), but it does not have superuser privileges (`rolsuper = f`). This configuration follows the principle of least privilege, limiting the account to only the permissions required for replication and reducing the impact of a potential compromise of its credentials.

Exit `psql` by running:

```psql
\q
```

### 1.6 — Create the physical replication slot

**Objective:** create a replication slot on the ***Primary*** dedicated to the ***Standby***. The slot allows PostgreSQL to track the WAL position up to which the replica has consumed the data and retain WAL files that are still required.

On the ***Primary***, run:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT * FROM pg_create_physical_replication_slot('lnd_pg_standby_01');
"
```

Expected output:

```text
     slot_name     | lsn 
-------------------+-----
 lnd_pg_standby_01 | 
(1 row)
```

#### 1.6.1 — Validate the replication slot

On the ***Primary***, run:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT slot_name, slot_type, active, restart_lsn
FROM pg_replication_slots
WHERE slot_name = 'lnd_pg_standby_01';
"
```

Expected output:

```text
     slot_name     | slot_type | active | restart_lsn 
-------------------+-----------+--------+-------------
 lnd_pg_standby_01 | physical  | f      | 
(1 row)
```

> **Note:** a replication slot prevents the ***Primary*** from discarding WAL files that may still be required by the ***Standby***. This provides additional protection when a replica is temporarily lagging behind, but it also introduces a risk: if the ***Standby*** remains unavailable for too long, WAL files may consume an increasing amount of disk space. For this reason, we previously configured `max_slot_wal_keep_size = 10GB`, establishing a limit on WAL retention caused by the slot.

### 1.7 — Restart PostgreSQL on the Primary and apply the configuration

**Objective:** restart PostgreSQL on the ***Primary*** to apply parameters that require a restart, especially `listen_addresses`, and confirm that the service returns normally.

#### 1.7.1 — Restart PostgreSQL

On the ***Primary***, run:

```bash
sudo systemctl restart postgresql
```

If the command returns no output, this is normal.

#### 1.7.2 — Check the cluster status

On the ***Primary***, run:

```bash
pg_lsclusters
```

Expected output:

```text
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

#### 1.7.3 — Confirm the parameters actually applied

On the ***Primary***, run:

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

Expected output:

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

#### 1.7.4 — Confirm which addresses PostgreSQL is listening on

On the ***Primary***, run:

```bash
sudo ss -ltnp | grep ':5432'
```

Expected output:

```text
IP_REDE_PRIMARY:5432
127.0.0.1:5432
[::1]:5432
```

The important point is that PostgreSQL is listening on the address of the ***Primary*** network interface used for communication with the ***Standby***. In this tutorial, we use Tailscale, but this address may also belong to a local network or another private network. Having only `127.0.0.1:5432` does not allow a ***Standby*** server located on another machine to connect.

> **Note:** in the complete output of the `ss` command, `0.0.0.0:*` may appear as the remote address. This does not mean that PostgreSQL is listening on all interfaces. To identify where PostgreSQL is listening, look at the local address associated with port `5432`.

## 2 — Install the corresponding PostgreSQL version on the ***Standby***

### 2.1 — Identify the PostgreSQL version on the ***Primary***

**Objective:** determine which version is actually running on the ***Primary***. This information will determine what we will prepare on the ***Standby***.

On the ***Primary***, run:
```bash
sudo -u postgres psql -X -Atc "SHOW server_version;"
```

Expected output:
```text
18.6 (Ubuntu 18.6-1.pgdg24.04+2)
```

> Keep this information. In this example, the major version is 18 and the minor version is 6. It will be used when preparing the ***Standby***.

## 2.2 — Check whether PostgreSQL already exists on the ***Standby***

**Objective:** check whether PostgreSQL is already installed on the ***Standby*** and identify any existing PostgreSQL clusters before preparing the cluster intended for replication.

On the ***Standby***, run:
```bash
psql --version 2>/dev/null || echo "psql não instalado"
echo
pg_lsclusters 2>/dev/null || echo "Nenhum cluster PostgreSQL encontrado"
```

Expected output:
```text
psql não instalado

Nenhum cluster PostgreSQL encontrado
```

> **Note:** if a PostgreSQL cluster is already in use, do not remove it. Later, we will show how to keep the existing cluster and prepare another independent cluster for the replica.

### 2.3 — Check version availability on the ***Standby***

**Objective:** check whether the repository currently configured on the ***Standby*** provides the same major version identified on the ***Primary*** and which minor version would be installed.

On the ***Standby***, run (replace 18 with the major version identified on the ***Primary***):
```bash
apt-cache policy postgresql-18
```

Expected output:
```text
postgresql-18:
  Instalado: (nenhum)
  Candidato: 18.6-1.pgdg24.04+2
  Tabela de Versão:
     18.6-1.pgdg24.04+2 500
```

If the output shows a compatible candidate version, there is no need to add another repository. Note the candidate version and proceed to step 2.5.

If the output is:
```text
N: Não foi possível encontrar o pacote postgresql-18
```

or there is no candidate version with the same major version used by the ***Primary***, proceed to step 2.4.

### 2.4 — Add the official PostgreSQL repository (PGDG) on the ***Standby***

**Objective:** make PostgreSQL versions that are not available in the currently configured repositories available on the ***Standby***.

On the ***Standby***, run the command to install the PostgreSQL common tools:
```bash
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```

The script will identify Ubuntu 24.04 as noble and request confirmation:
```text
This script will enable the PostgreSQL APT repository on apt.postgresql.org on
your system. The distribution codename used will be noble-pgdg.
Press Enter to continue, or Ctrl-C to abort.
```

Press Enter to continue.

At the end, the expected output includes:
```text
Writing /etc/apt/sources.list.d/pgdg.sources ...
Running apt-get update ...
...
You can now start installing packages from apt.postgresql.org.
```

Confirm that a candidate version with the same major version used by the ***Primary*** is available. Whenever possible, also keep the ***Primary*** and ***Standby*** servers on the same minor version.

On the ***Standby***, run (replace 18 with the major version identified on the ***Primary***):
```bash
apt-cache policy postgresql-18
```

Expected output:
```text
postgresql-18:
  Instalado: (nenhum)
  Candidato: 18.6-1.pgdg24.04+2
  Tabela de Versão:
     18.6-1.pgdg24.04+2 500
```

### 2.5 — Install PostgreSQL on the ***Standby***

**Objective:** install on the ***Standby*** the same PostgreSQL major version used by the ***Primary***. In this procedure, we will allow `postgresql-common` to automatically create the default local cluster. This cluster is not yet the replica and will later be prepared to receive the physical copy of the ***Primary***.

On the ***Standby***, run (replace 18 with the major version identified on the ***Primary***):
```bash
sudo apt install -y postgresql-18
```

Expected output:
```text
Creating new PostgreSQL cluster 18/main ...
...
Data page checksums are enabled.
...
syncing data to disk ... ok
```

> **Important:** at this point, a new local PostgreSQL cluster has been created and initialized with the default databases and structures. It does not yet contain data from the ***Primary*** and should not be confused with the future ***Standby***.

Confirm that the installed major version matches the one used by the ***Primary***. Also identify the cluster that was created, its port, status, and data directory.

On the ***Standby***, run:
```bash
psql --version
echo
pg_lsclusters
```

Expected output:
```text
psql (PostgreSQL) 18.6 (Ubuntu 18.6-1.pgdg24.04+2)

Ver Cluster Port Status Owner    Data directory
18  main    5432 online postgres /var/lib/postgresql/18/main
```

## 3 — Prepare the ***Standby*** to receive the replica

### 3.1 — Test PostgreSQL connectivity between the ***Standby*** and the ***Primary***

**Objective:** before making changes to the PostgreSQL cluster on the ***Standby***, confirm that it can reach port `5432` on the ***Primary*** over the network selected for replication.

On the ***Standby***, run:

```bash
pg_isready -h IP_REDE_PRIMARY -p 5432
```

Expected output:

```text
IP_REDE_PRIMARY:5432 - accepting connections
```

### 3.2 — Validate replication user authentication

**Objective:** confirm that the `lnd_replicator` user can authenticate to the ***Primary*** from the ***Standby*** and that the rule configured in `pg_hba.conf` is working.

On the ***Standby***, run:

```bash
psql "host=IP_REDE_PRIMARY port=5432 user=lnd_replicator replication=true" -W -c "IDENTIFY_SYSTEM;"
```

The `-W` parameter will cause `psql` to request the password interactively, preventing it from being included directly in the command and stored in the terminal history. Enter the password defined for the `lnd_replicator` user:

```text
Password:
```

Expected output:

```text
      systemid       | timeline |  xlogpos  | dbname 
---------------------+----------+-----------+--------
 1234567890123456789 |        1 | 0/1234567 | 
(1 row)
```

> **Note:** the `IDENTIFY_SYSTEM` command is executed through the PostgreSQL replication protocol. A valid response confirms that the user was able to establish a physical replication connection with the ***Primary***. The values of `systemid`, `timeline`, and `xlogpos` vary according to each cluster and the current WAL position; therefore, they do not need to match the values shown in the example.

### 3.3 — Check the local PostgreSQL cluster on the ***Standby***

**Objective:** identify the existing PostgreSQL cluster on the ***Standby*** before preparing it to receive the physical copy of the ***Primary***.

On the ***Standby***, run:

```bash
pg_lsclusters
```

Expected output:

```text
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

Check which databases currently exist in this cluster:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT datname
FROM pg_database
ORDER BY datname;
"
```

On a new, unused installation, the expected output is:

```text
  datname  
-----------
 postgres
 template0
 template1
(3 rows)
```

> **Important:** if databases other than `postgres`, `template0`, and `template1` are found, do not proceed with removing or replacing this cluster. The presence of other databases may indicate that the PostgreSQL server is already in use and contains data that must be preserved.
>
> In this case, the ***Standby*** can still be used for replication, but the existing cluster must be preserved. A second PostgreSQL cluster dedicated to the physical replica must be created, using a different data directory and a different port from the one used by the existing cluster.
>
> For example, a server may simultaneously maintain:
>
> ```text
> 18  main        5432  online  postgres  /var/lib/postgresql/18/main
> 18  lndstandby  5433  online  postgres  /var/lib/postgresql/18/lndstandby
> ```
>
> In this example, `18/main` continues serving the existing databases, while `18/lndstandby` is reserved exclusively for receiving the physical replica of the ***Primary***.
>
> Do not use `pg_dropcluster`, do not delete the data directory, and do not overwrite a cluster that contains data that must be preserved.

#### 3.3.1 — Alternative scenario: ***Standby*** with a PostgreSQL cluster already in use

**Objective:** prepare replication when the ***Standby*** already has a PostgreSQL cluster in use, fully preserving that cluster and subsequently creating a second cluster dedicated to the physical replica.

> **Note:** if the server only has the newly created cluster, skip directly to step 3.4.

#### 3.3.1.1 — Check PostgreSQL ports currently in use

Before creating another cluster, we need to determine which ports are already in use.

On the ***Standby***, run:

```bash
sudo ss -ltnp | grep postgres
```

Expected output:

```text
LISTEN 0  200  127.0.0.1:5432  0.0.0.0:*  users:(("postgres",...))
```

In this example, the existing PostgreSQL cluster is using port `5432`. Before creating a second cluster, choose a port that is not in use. In this tutorial, port `5433` will be used.

#### 3.3.1.2 — Create a second PostgreSQL cluster

**Objective:** create an independent cluster intended for the replica while preserving the PostgreSQL cluster that is already in use.

On the ***Standby***, run:

```bash
sudo pg_createcluster 18 lndstandby --port=5433 --start
```

Expected output:

```text
...

Ver Cluster     Port Status Owner    Data directory                     Log file
18  lndstandby 5433 online postgres /var/lib/postgresql/18/lndstandby /var/log/postgresql/postgresql-18-lndstandby.log
```

On the ***Standby***, run the following command to confirm that the cluster was created:

```bash
pg_lsclusters
```

In this scenario, the `18/main` cluster remains operational on port `5432`, while the `18/lndstandby` cluster, on port `5433`, will be dedicated to the physical replica of the ***Primary***. This allows the replica to be prepared without replacing or interrupting the PostgreSQL cluster that was already in use.

Expected output:

```text
Ver Cluster    Port Status Owner    Data directory                    Log file
18  lndstandby 5433 online postgres /var/lib/postgresql/18/lndstandby /var/log/postgresql/postgresql-18-lndstandby.log
18  main       5432 online postgres /var/lib/postgresql/18/main       /var/log/postgresql/postgresql-18-main.log
```

> **Note:** avoid using a hyphen (`-`) in the cluster name. `pg_createcluster` warns that names containing a hyphen may cause problems with `systemd` integration. In this tutorial, the name `lndstandby` will be used.

### 3.4 — Prepare the ***Standby*** cluster to receive the replica

**Objective:** stop the PostgreSQL cluster that will be transformed into a physical replica before replacing its data directory with the copy from the ***Primary***.

> **Important:** this step begins the actual preparation of the ***Standby*** cluster. Make sure you have correctly identified, in step 3.3, which cluster will be used for the replica.

> **Secondary cluster:** if it was necessary to create the dedicated `18/lndstandby` cluster as described in section 3.3.1, replace `main` with `lndstandby` in the following commands. Do not stop the `18/main` cluster that was already in use.

#### 3.4.1 — Stop the cluster intended for the replica

On the ***Standby***, run:

```bash
sudo pg_ctlcluster 18 main stop
```

Confirm:

```bash
pg_lsclusters
```

Expected output:

```text
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 down   postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

#### 3.4.2 — Remove the local cluster data directory contents

**Objective:** leave the data directory empty so that it can receive the physical copy of the ***Primary***.

> **Warning:** this operation is destructive. Execute it only after confirming that the correct cluster is stopped and that it does not contain any data that must be preserved.

On the ***Standby***, run:

```bash
sudo find /var/lib/postgresql/18/main -mindepth 1 -maxdepth 1 -exec rm -rf -- {} +
```

On the ***Standby***, run the following command to verify:

```bash
sudo ls -la /var/lib/postgresql/18/main
```

Expected output:

```text
total 8
drwx------ 2 postgres postgres ... .
drwxr-xr-x 3 postgres postgres ... ..
```

### 3.5 — Configure ***Standby*** authentication to the ***Primary***

**Objective:** allow processes running as the `postgres` user on the ***Standby*** to authenticate to the ***Primary*** without exposing the password in commands.

#### 3.5.1 — Create the `.pgpass` file

On the ***Standby***, run:

```bash
sudo -u postgres nano /var/lib/postgresql/.pgpass
```

Add a single line:

```conf
IP_REDE_PRIMARY:5432:*:lnd_replicator:SENHA_DO_USUARIO
```

Replace `SENHA_DO_USUARIO` with the password previously defined for `lnd_replicator`.

Save the file (Ctrl+O and Enter) and exit Nano (Ctrl+X).

The `.pgpass` file contains the password in plain text. It must belong to the `postgres` user and have `600` permissions. PostgreSQL ignores the file if its permissions allow access by other users. On the ***Standby***, run:

```bash
sudo chown postgres:postgres /var/lib/postgresql/.pgpass
sudo chmod 600 /var/lib/postgresql/.pgpass
```

On the ***Standby***, run the following command to verify:

```bash
sudo ls -l /var/lib/postgresql/.pgpass
```

Expected output:

```text
-rw------- 1 postgres postgres ... /var/lib/postgresql/.pgpass
```

#### 3.5.2 — Validate authentication through `.pgpass`

**Objective:** confirm that the `postgres` user on the ***Standby*** can authenticate to the ***Primary*** replication protocol using `.pgpass`.

On the ***Standby***, run:

```bash
sudo -u postgres psql "host=IP_REDE_PRIMARY port=5432 user=lnd_replicator replication=true passfile=/var/lib/postgresql/.pgpass" -c "IDENTIFY_SYSTEM;"
```

The command should execute without requesting a password. This confirms that the `.pgpass` file is being used to authenticate the replication user.

### 3.6 — Create the physical replica from the ***Primary***

**Objective:** copy the PostgreSQL cluster from the ***Primary*** to the ***Standby*** and prepare the copy to subsequently start as a physical replica.

#### 3.6.1 — Run `pg_basebackup`

On the ***Standby***, run:

```bash
sudo -u postgres pg_basebackup \
  -D /var/lib/postgresql/18/main \
  -d "host=IP_REDE_PRIMARY port=5432 user=lnd_replicator application_name=standby1 passfile=/var/lib/postgresql/.pgpass" \
  -X stream \
  -S lnd_pg_standby_01 \
  -R \
  -P
```

Expected output:

```text
waiting for checkpoint
...
XXXXXX/XXXXXX kB (100%), 1/1 tablespace
```

> **Note:** the `waiting for checkpoint` message is normal and may remain displayed for a considerable amount of time before the copy begins. When successfully completed, `pg_basebackup` returns to the terminal prompt. The values displayed by the progress indicator vary according to the size of the cluster.

### 3.7 — Start physical replication on the ***Standby***

**Objective:** start PostgreSQL on the ***Standby*** and confirm that the cluster copied by `pg_basebackup` starts in recovery mode and begins receiving WAL from the ***Primary***.

#### 3.7.1 — Start the ***Standby*** cluster

On the ***Standby***, run:

```bash
sudo pg_ctlcluster 18 main start
```

Then confirm the status:

```bash
pg_lsclusters
```

Expected output:

```text
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

#### 3.7.2 — Confirm that the cluster is running in Standby mode

**Objective:** verify that PostgreSQL started in recovery mode rather than as an independent Primary.

On the ***Standby***, run:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT pg_is_in_recovery();
"
```

Expected output:

```text
 pg_is_in_recovery
-------------------
 t
(1 row)
```

Still on the ***Standby***, run:

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

Expected output:

```text
 status    | sender_host  | sender_port |      slot_name       | written_lsn | flushed_lsn | latest_end_lsn
-----------+--------------+-------------+----------------------+-------------+-------------+---------------
 streaming | IP_PRIMARY   |        5432 | lnd_pg_standby_01    | ...         | ...         | ...
```

#### 3.7.3 — Check replication from the ***Primary***

**Objective:** confirm from the ***Primary*** side that the ***Standby*** is connected and check its synchronization state.

On the ***Primary***, run:

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

Expected output:

```text
 application_name | client_addr     |   state   | sync_state | sent_lsn | write_lsn | flush_lsn | replay_lsn | ...
------------------+-----------------+-----------+------------+----------+-----------+-----------+------------+----
 standby1         | IP_REDE_STANDBY | streaming | async      | ...      | ...       | ...       | ...        | ...
```

> **Note:** after the first startup of the ***Standby***, `state` may temporarily appear as `catchup` while pending WAL records are received and applied. Wait until the state changes to `streaming` before proceeding.

### 3.8 — Enable synchronous replication with `remote_apply`

**Objective:** make synchronous commits on the ***Primary*** wait until the corresponding WAL has been applied on the ***Standby***, using `standby1` as the synchronous replica.

#### 3.8.1 — Add the synchronous replication configuration

On the ***Primary***, run:

```bash
sudo nano /etc/postgresql/18/main/conf.d/99-lnd-replication.conf
```

At the end of the file, add:

```conf
# Synchronous replication
synchronous_standby_names = 'standby1'
synchronous_commit = remote_apply
```

Save the file (Ctrl+O and Enter) and exit Nano (Ctrl+X).

#### 3.8.2 — Validate the configuration before reload

**Objective:** confirm that the new settings were interpreted correctly by PostgreSQL and that there are no errors in the file.

On the ***Primary***, run:

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

In the expected output, confirm primarily:

```text
|           name            |   setting    | applied | error
+---------------------------+--------------+---------+-------
| synchronous_standby_names | standby1     | t       |
| synchronous_commit        | remote_apply | t       |
```

#### 3.8.3 — Apply the synchronous configuration

**Objective:** reload the PostgreSQL configuration to activate `standby1` as the synchronous replica and use `remote_apply`.

On the ***Primary***, run:

```bash
sudo pg_ctlcluster 18 main reload
```

Confirm the values actually loaded. On the ***Primary***, run:

```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT name,
       setting
FROM pg_settings
WHERE name IN ('synchronous_standby_names', 'synchronous_commit')
ORDER BY name;
"
```

Expected output:

```text
           name            |   setting    
---------------------------+--------------
 synchronous_commit        | remote_apply
 synchronous_standby_names | standby1
(2 rows)
```

#### 3.8.4 — Confirm that the ***Standby*** is synchronous

**Objective:** verify on the ***Primary*** that `standby1` has been selected as the synchronous replica and remains in `streaming`.

On the ***Primary***, run:

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

In the expected output, confirm primarily:

```text
  application_name |   state   | sync_state |
 ------------------+-----------+------------+
  standby1         | streaming | sync
```

## 4 — Contingency operation: switch between synchronous and asynchronous replication in case of ***Standby*** failure

When PostgreSQL is configured with `synchronous_commit = remote_apply` and the ***Standby*** defined in `synchronous_standby_names` becomes unavailable, commits that depend on synchronous replication may remain waiting for the replica.

In this situation, if it is necessary to keep the ***Primary*** and LND operational while the ***Standby*** is being recovered, the synchronous replica requirement can be temporarily removed.

> **Important:** when operating this way, the ***Primary*** will continue accepting writes without waiting for the ***Standby***. During this period, there is no longer a guarantee that transactions committed on the ***Primary*** have already been applied to the replica.

### 4.1 — Confirm that the ***Standby*** is unavailable

On the ***Primary***, run:
```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT application_name,
       client_addr,
       state,
       sync_state
FROM pg_stat_replication;
"
```

If the ***Standby*** is disconnected, the output will be:
```text
 application_name | client_addr | state | sync_state
------------------+-------------+-------+-----------
(0 rows)
```

### 4.2 — Temporarily switch to asynchronous replication

On the ***Primary***, open:
```bash
sudo nano /etc/postgresql/18/main/conf.d/99-lnd-replication.conf
```

Change only:
```conf
synchronous_standby_names = 'standby1'
```

to:
```conf
synchronous_standby_names = ''
```

Save the file (Ctrl+O and Enter) and exit Nano (Ctrl+X).

> **Note:** `synchronous_commit = remote_apply` remains configured. By leaving `synchronous_standby_names` empty, no ***Standby*** is required for synchronous confirmation. This allows you to later return to `remote_apply` mode by changing only `synchronous_standby_names`.

### 4.3 — Validate the configuration before reload

On the ***Primary***, run:
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

Confirm primarily:
```text
synchronous_standby_names |              | t |
synchronous_commit        | remote_apply | t |
```

and that the `error` column is empty.

### 4.4 — Apply asynchronous mode

On the ***Primary***, run:
```bash
sudo pg_ctlcluster 18 main reload
```

Confirm that the change was applied. On the ***Primary***, run:
```bash
sudo -u postgres psql -X -P pager=off -c "
SELECT name,
       setting
FROM pg_settings
WHERE name IN ('synchronous_standby_names', 'synchronous_commit')
ORDER BY name;
"
```

Confirm primarily:
```text
           name            |   setting
---------------------------+--------------
 synchronous_commit        | remote_apply
 synchronous_standby_names |
(2 rows)
```

> **Note:** there is no need to restart PostgreSQL or LND.

#### 4.4.1 — Monitor the Physical Slot Health

While the ***Standby*** is unavailable, periodically check the health of the physical replication slot on the ***Primary***:

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

A healthy output while the Standby is unavailable should look, for example, like this:

```text
slot_name          | lnd_pg_standby_01
slot_active        | f
wal_status         | reserved
wal_retained       | 129 MB
safe_wal_remaining | 10116 MB
```

#### 4.4.2 — Increase the WAL retention limit

If `safe_wal_remaining` is getting close to 0, you can increase the amount of WAL that the replication slot is allowed to retain on the ***Primary***. First, check the currently configured limit:

```bash
sudo -u postgres psql -X -d postgres -c "SHOW max_slot_wal_keep_size;"
```

Expected output:

```text
 max_slot_wal_keep_size
------------------------
 XXGB
(1 row)
```

Then increase the retention limit:

```bash
sudo -u postgres psql -X -d postgres -c "ALTER SYSTEM SET max_slot_wal_keep_size = '100GB';"
```

Replace `100GB` with the desired value, taking into account the available disk space on the ***Primary***.

> **Important:** this parameter does not immediately reserve the specified amount of disk space. It defines the maximum amount of WAL that the replication slot may require PostgreSQL to retain. Therefore, make sure the ***Primary*** has enough free disk space to accommodate this growth.

Expected output:

```text
ALTER SYSTEM
```

Then reload the configuration:

```bash
sudo -u postgres psql -X -d postgres -c "SELECT pg_reload_conf();"
```

Expected output:

```text
 pg_reload_conf
----------------
 t
```

### 4.5 — Return to synchronous `remote_apply` mode after the ***Standby*** returns

> **Important:** do not restore synchronous mode simply because the ***Standby*** has started responding again. First, confirm that it is connected again, in `streaming`, and sufficiently up to date relative to the ***Primary***.

#### 4.5.1 — Check the ***Standby*** Health

After the ***Standby*** has been recovered, on the ***Primary*** run:
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

While `synchronous_standby_names` remains empty, a healthy ***Standby*** should appear in `streaming`, but still as:
```text
application_name |   state   | sync_state
-----------------+-----------+-----------
standby1         | streaming | async
```

Wait until `sent_lsn`, `write_lsn`, `flush_lsn`, and `replay_lsn` are close to or equal to each other before restoring the synchronous requirement.

#### 4.5.2 — Reactivate the synchronous ***Standby***

On the ***Primary***, open:
```bash
sudo nano /etc/postgresql/18/main/conf.d/99-lnd-replication.conf
```

Change:
```conf
synchronous_standby_names = ''
```

to:
```conf
synchronous_standby_names = 'standby1'
```

On the ***Primary***, apply:
```bash
sudo pg_ctlcluster 18 main reload
```

Finally, confirm:
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

The ***Standby*** should return to:
```text
application_name |   state   | sync_state
-----------------+-----------+-----------
standby1         | streaming | sync
```

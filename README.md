# LND PostgreSQL Replication

[🇧🇷 Versão em português](README.pt-BR.md)

A practical project for building and documenting a **PostgreSQL physical replication architecture for LND**, focused on data consistency, controlled recovery, and protection against state loss.

The project uses PostgreSQL native replication mechanisms to maintain an independent, continuously updated copy of the PostgreSQL cluster used by an LND node.

## Architecture

The architecture is based on two servers:

- **Primary** — the PostgreSQL server normally used by LND;
- **Standby** — an independent server that maintains a physical replica of the PostgreSQL cluster.

Replication is implemented using **PostgreSQL Physical Streaming Replication** with a dedicated *physical replication slot*.

The normal operating mode uses synchronous replication with `remote_apply`. The architecture intentionally keeps failover and recovery under explicit operator control, avoiding automatic promotion mechanisms that could increase the risk of split-brain or unintended LND state divergence.

## Documentation

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

## Reference Environment

The implementation and tests documented in the first tutorial were performed using:

- **Ubuntu Server 24.04 LTS**
- **PostgreSQL 18.6**
- **LND 0.21.3-beta**
- **Nodes used:**
     - Naghust SA | BR⚡LN (02dd543868366e0bc3e498ab7c687d795a30ce0f70d2d034006b4654bbe887af8a)
     - Naghust Replication Test (026eea7a1d59689a4e40c3504cbaca131d712564436ea081db6fd81050b585f0b7)

## Project Roadmap

This repository is intended to document the architecture progressively.

Current and planned topics include:

1. PostgreSQL physical replication for LND;
2. controlled Standby promotion;
3. recovery after Primary failure;
4. PostgreSQL replication recovery and re-synchronization;
5. integration with LND operational procedures;
6. additional safeguards for controlled failover.

## Design Principles

- Prefer native PostgreSQL mechanisms whenever possible.
- Protect LND state as the primary objective.
- Keep promotion and recovery under explicit operator control.
- Avoid automatic failover that could introduce split-brain risk.
- Validate replication state before promotion.
- Allow controlled temporary asynchronous operation during Standby failure.
- Keep procedures reproducible and auditable.

## Important Notice

The procedures documented in this repository directly affect the PostgreSQL database used by LND.

Before applying them to a production Lightning node:

- understand each command before executing it;
- maintain appropriate backups;
- verify PostgreSQL and LND version compatibility;
- adapt addresses, paths, ports, and parameters to your infrastructure;
- validate the procedure in a test environment whenever possible.

This project documents a tested architecture and operational procedure, but it is not a substitute for an infrastructure-specific backup and disaster recovery strategy.

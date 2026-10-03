# AuditLedger

![Python](https://img.shields.io/badge/Python-3.11+-blue)
![FastAPI](https://img.shields.io/badge/FastAPI-009688)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-336791)
![Redis](https://img.shields.io/badge/Redis-DC382D)
![Docker](https://img.shields.io/badge/Docker-ready-2496ED)

A double-entry financial ledger and automated reconciliation microservice built for transactional integrity. It eliminates floating-point errors, blocks duplicate settlements during retry storms, encrypts sensitive data at rest, and provides a tamper-evident audit trail via hash-chaining.

## Architecture

```
Client ──► FastAPI Gateway (Pydantic v2 validation, SHA-256 fingerprint)
                │
                ├──► Redis (distributed lock, in-flight barrier, idempotency cache)
                │
                ▼
        Double-Entry Invariant Engine (Σ debits − Σ credits = 0, Decimal(18,4), hash chainer)
                │  atomic ACID commit
                ▼
           PostgreSQL ◄──── Reconciliation Daemon (balance asserter, chain verifier, alerter)
```

## Key Features

- **Double-entry invariant** – Unbalanced entries are rejected with `HTTP 422` before any database I/O.
- **Exact decimal math** – Python `Decimal` (banker's rounding) backed by `NUMERIC(18, 4)`; no IEEE-754 errors.
- **Append-only ledger** – `UPDATE`/`DELETE` blocked by triggers and role permissions; corrections are contra-entries.
- **Two-layer idempotency** – Redis `SETNX` lock (60s TTL) for in-flight retries, plus a `UNIQUE` constraint on `idempotency_hash` in PostgreSQL.
- **Hash-chained audit trail** – Each entry's hash includes the previous one; any historical edit breaks the chain and the system reports `CHAIN_BROKEN`.
- **Field-level encryption** – Account numbers and other sensitive identifiers are encrypted with AES-256-GCM before reaching the database.
- **Automated reconciliation** – Background worker checks balances against postings and verifies the chain, reporting `CLEAN`, `DISCREPANCY_DETECTED`, or `TAMPER_SUSPECTED`.

## Tech Stack

Python · FastAPI · Pydantic v2 · SQLAlchemy 2.0 (async) · PostgreSQL · Redis · Alembic · PyTest-Asyncio · Docker

## API

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/v1/accounts` | Create an encrypted account |
| `POST` | `/api/v1/transactions` | Idempotent transaction execution |
| `GET`  | `/api/v1/accounts/{id}/balance` | Point-in-time and materialized balance |
| `POST` | `/api/v1/reconcile` | Trigger a manual verification sweep |

## Getting Started

```bash
docker-compose up -d          # Postgres, Redis, API
alembic upgrade head          # Run migrations
```

## Testing

```bash
pytest
```

Key scenarios: concurrency replay assault, precision boundary split, and database tamper detection.

## License

Add your license here.

# Indexer configuration

The indexer (`services/indexer`) reads contract events and transactions from a
Soroban RPC node and writes them to the Postgres database the API serves. This
page lists the environment variables it needs to reach those services. Sharding
is covered separately in [`indexer-sharding.md`](indexer-sharding.md) and
Prometheus metrics in [`metrics.md`](metrics.md).

## Soroban RPC

The indexer refuses to start unless at least one RPC URL is set.

| Variable                    | Default | Description                                                                                                                                                                                          |
| --------------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SOROBAN_RPC_URL`           | —       | Default endpoint. Used for every contract whose network has no dedicated URL below.                                                                                                                  |
| `SOROBAN_RPC_URL_TESTNET`   | —       | Endpoint for contracts tracked on `testnet`.                                                                                                                                                         |
| `SOROBAN_RPC_URL_MAINNET`   | —       | Endpoint for contracts tracked on `mainnet`.                                                                                                                                                         |
| `SOROBAN_RPC_URL_FUTURENET` | —       | Endpoint for contracts tracked on `futurenet`.                                                                                                                                                       |
| `SOROBAN_RPC_TIMEOUT`       | `30s`   | Timeout for a single RPC request (Go duration syntax).                                                                                                                                               |
| `SOROBAN_RPC_MAX_RETRIES`   | `3`     | Retries after HTTP 429, HTTP 5xx, or a transport failure such as a timeout. Backoff starts at 500ms, doubles each time, and is capped at 10s. A 429 `Retry-After` header is honoured up to that cap. |

JSON-RPC application errors such as an out-of-range `startLedger` are not
retried. They come back as `sorobanrpc.RPCError`, which keeps the node's error
code.

Public endpoints:

```bash
export SOROBAN_RPC_URL_TESTNET=https://soroban-testnet.stellar.org
export SOROBAN_RPC_URL_FUTURENET=https://rpc-futurenet.stellar.org
# Mainnet has no SDF-hosted public RPC; use a provider such as
# https://developers.stellar.org/docs/data/apis/rpc/providers
```

Public nodes keep about 7 days of history, so a newly tracked contract can
only backfill that far.

## Postgres

| Variable       | Default | Description                                                                                                                                                                      |
| -------------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `DATABASE_URL` | —       | Required. pgx connection string for the database the API reads, migrated with `apps/api/internal/db/migrations`. The indexer pings it on startup and exits if it is unreachable. |

The indexer writes events, invocations, per-contract sync state, network
cursors, Wasm upgrades, the Wasm cache, health scores, and anomaly alerts
through `services/indexer/internal/pgstore`. Every batch insert ignores rows
that already exist, so re-indexing a ledger window is safe.

The store tests run when `POSTGRES_TEST_URL` points at a migrated database:

```bash
POSTGRES_TEST_URL=postgres://sorolens:sorolens@localhost:5432/sorolens?sslmode=disable \
  go test ./services/indexer/internal/pgstore/...
```

## Redis run lock

| Variable   | Default                   | Description                                                                                                                                                                 |
| ---------- | ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `REDIS_URL` | `redis://localhost:6379` | Redis URL for the per-contract run lock. When set, the indexer acquires a Redis lock (`SET key value NX PX <ttl>`) before indexing each contract and renews it while the run is active. The lock is released with a compare-and-delete Lua script so one runner never deletes another's lock. When empty, locking is disabled and only one indexer should run per contract set. |

Redis is checked at startup and an unreachable server prevents the indexer from starting, so two runners can never process the same ledgers.

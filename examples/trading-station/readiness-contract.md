# Trading Station Readiness Contract

## Service

- Name: trading-station
- Primary responsibility: process live market data and trading workflow events

## Required Dependencies

| Dependency | Purpose | Readiness signal | Recovery window |
| --- | --- | --- | --- |
| RabbitMQ | consume trading commands and publish events | connection open, required consumers active, publisher confirms available | 60s |
| Redis | current market state and pub/sub updates | connection healthy, required subscriptions active | 30s |
| Market data WebSocket | live prices | connection open and last message within freshness window | 15s |
| Account WebSocket | account/order events | socket open, heartbeat fresh, listen key fresh, Redis ownership held | 30s |

## Optional Dependencies

| Dependency | Purpose | Degradation behavior |
| --- | --- | --- |
| Analytics sink | non-critical event analytics | log and count failures without blocking trading flow |

## Health Semantics

- Liveness: process and runtime are responsive.
- Readiness: RabbitMQ, Redis, market data WebSocket, consumers, publishers, and
  required background loops are functional.
- Startup: configuration validated and required runtime components initialized.

## WebSocket Runtime Ownership

| Stream | Kind | Ownership key | Readiness signal |
| --- | --- | --- | --- |
| market prices | market-data | optional by shard | socket open, pong fresh, last message within freshness window |
| account events | account-data | `ws-owner:exchange:account:{accountId}` | socket open, pong fresh, listen key fresh, ownership TTL renewed |

Quiet account streams do not require frequent business messages for readiness.
They use heartbeat, listen-key freshness, and ownership state.

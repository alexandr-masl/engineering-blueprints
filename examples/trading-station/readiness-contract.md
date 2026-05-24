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

## Optional Dependencies

| Dependency | Purpose | Degradation behavior |
| --- | --- | --- |
| Analytics sink | non-critical event analytics | log and count failures without blocking trading flow |

## Health Semantics

- Liveness: process and runtime are responsive.
- Readiness: RabbitMQ, Redis, market data WebSocket, consumers, publishers, and
  required background loops are functional.
- Startup: configuration validated and required runtime components initialized.

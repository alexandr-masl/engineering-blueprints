# Service Readiness Contract

## Service

- Name:
- Owner:
- Runtime:
- Primary responsibility:

## Required Dependencies

| Dependency | Purpose | Readiness signal | Recovery window |
| --- | --- | --- | --- |
|  |  |  |  |

## Optional Dependencies

| Dependency | Purpose | Degradation behavior |
| --- | --- | --- |
|  |  |  |

## Consumers

| Broker | Queue/topic | Handler | Required | Failure behavior |
| --- | --- | --- | --- | --- |
|  |  |  |  |  |

## Publishers

| Broker | Exchange/topic | Message type | Required | Readiness signal | Reconnect behavior | Failure behavior |
| --- | --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |  |

## Subscriptions

| Dependency | Channel/feed | Required | Freshness window | Recovery behavior |
| --- | --- | --- | --- | --- |
|  |  |  |  |  |

## WebSocket Streams

| Stream | Kind | Required | Readiness signal | Ownership key | Listen-key/session refresh | Recovery behavior |
| --- | --- | --- | --- | --- | --- | --- |
|  | market-data/account-data/user-data/other |  |  |  |  |  |

For quiet account or user-data streams, define whether readiness uses business
message freshness or heartbeat/listen-key freshness.

## Background Loops and Timers

| Component | Interval | Required | Success signal | Failure behavior |
| --- | --- | --- | --- | --- |
|  |  |  |  |  |

## Startup Sequence

1. Validate configuration.
2. Initialize required dependencies.
3. Start required consumers, subscriptions, and loops.
4. Report ready only after functional checks pass.

## Shutdown Sequence

1. Mark readiness false.
2. Stop accepting new work.
3. Stop consumers and subscriptions.
4. Drain or reject in-flight work.
5. Close dependencies.

## Health Semantics

- Liveness:
- Readiness:
- Startup:

## Required Tests

- [ ] Startup success
- [ ] Invalid configuration failure
- [ ] Required dependency outage
- [ ] Required dependency recovery
- [ ] Required publisher proactive reconnect without a new publish
- [ ] Required WebSocket stream outage and recovery
- [ ] Ownership TTL expiry and reclaim, if ownership is used
- [ ] Listen-key refresh failure and replacement, if listen keys are used
- [ ] Graceful shutdown

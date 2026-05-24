# Graceful Shutdown

## Purpose

Graceful shutdown prevents message loss, partial writes, connection leaks, and
traffic routed to a service that is already stopping.

## Required Sequence

1. Receive termination signal.
2. Mark readiness false.
3. Stop accepting new external work.
4. Stop broker consumers and subscriptions.
5. Drain, finish, retry, or reject in-flight work according to service contract.
6. Flush critical publishes, logs, traces, and metrics.
7. Close dependency connections.
8. Exit within the configured termination grace period.

## Design Rules

- shutdown must be idempotent
- all waits must have bounded timeouts
- background loops must support cancellation
- forced termination must not corrupt durable state
- in-flight work behavior must be documented

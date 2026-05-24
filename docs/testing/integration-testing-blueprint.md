# Integration Testing Blueprint

## Purpose

Integration tests verify that a service behaves correctly with real or realistic
runtime dependencies.

Use integration tests for behavior that mocks cannot prove:

- broker connection and channel recovery
- cache and database outages
- WebSocket reconnect behavior
- health endpoint transitions
- graceful shutdown and drain behavior
- schema, queue, topic, and subscription compatibility

## Baseline Requirements

Integration tests should:

- run dependencies in containers or isolated test environments
- start the service through its production entrypoint where practical
- assert health endpoint state during startup, outage, recovery, and shutdown
- verify logs and metrics for critical lifecycle events
- avoid arbitrary sleeps when an observable condition can be awaited
- clean up queues, keys, topics, records, and containers after each test

## Required Test Groups

### Startup

- valid configuration starts successfully
- invalid configuration fails fast
- missing required dependency prevents readiness
- optional dependency outage degrades explicitly

### Runtime Outage

- required dependency outage changes readiness to false
- service does not accept work it cannot complete
- in-flight work follows defined retry, reject, or drain behavior
- alerts and metrics expose the outage

### Recovery

- dependency restart is detected
- connections are recreated
- consumers and subscriptions resume
- readiness becomes true only after functional recovery

### Shutdown

- readiness changes to false before dependencies are closed
- consumers stop receiving new work
- in-flight work is drained or rejected according to contract
- process exits within the configured timeout

## Test Evidence

Each integration test should make assertions against externally visible behavior,
not only internal method calls.

Useful evidence includes:

- HTTP response from readiness endpoint
- consumed or published messages
- database records
- Redis keys or subscription effects
- WebSocket messages
- metrics samples
- structured log events

## Anti-Patterns

- mocking the dependency being tested
- asserting only that constructors or handlers were called
- using fixed sleep instead of condition polling
- testing liveness when the requirement is readiness
- assuming process uptime means service availability

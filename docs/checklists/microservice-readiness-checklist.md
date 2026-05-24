# Microservice Readiness Checklist

Use this checklist before declaring a microservice production-ready.

## Runtime Contract

- [ ] Required dependencies are documented.
- [ ] Optional dependencies are documented.
- [ ] Consumers, publishers, subscriptions, timers, and background loops are
      documented.
- [ ] Startup and shutdown sequence is documented.
- [ ] Recovery windows are defined.

## Health

- [ ] Liveness checks process responsiveness only.
- [ ] Readiness checks functional required behavior.
- [ ] Readiness includes required consumers, publishers, subscriptions, and
      background loops.
- [ ] Health responses include actionable failure reasons.

## Resilience

- [ ] Required dependency outage removes readiness.
- [ ] Required dependency recovery restores runtime objects.
- [ ] Retry behavior is bounded.
- [ ] Queues and buffers are bounded.
- [ ] Poison messages or invalid payloads cannot block the system indefinitely.

## Observability

- [ ] Metrics expose dependency and component state.
- [ ] Logs include startup, dependency transition, recovery, and shutdown events.
- [ ] Alerts cover sustained readiness failure and stale runtime behavior.

## Tests

- [ ] Startup tests exist.
- [ ] Required dependency outage tests exist.
- [ ] Recovery tests exist.
- [ ] Graceful shutdown tests exist.
- [ ] Tests assert externally visible behavior.

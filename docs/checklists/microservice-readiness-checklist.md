# Microservice Readiness Checklist

Use this checklist before declaring a microservice production-ready.

## Runtime Contract

- [ ] Required dependencies are documented.
- [ ] Optional dependencies are documented.
- [ ] Consumers, publishers, subscriptions, timers, and background loops are
      documented.
- [ ] WebSocket streams document stream kind, readiness signal, ownership key,
      and listen-key/session-key lifecycle where applicable.
- [ ] Startup and shutdown sequence is documented.
- [ ] Recovery windows are defined.

## Health

- [ ] Liveness checks process responsiveness only.
- [ ] Readiness checks functional required behavior.
- [ ] Readiness includes required consumers, publishers, subscriptions, and
      background loops.
- [ ] Quiet account/user-data streams do not depend on business-message
      freshness when heartbeat and listen-key state are the correct signals.
- [ ] Readiness can recover after dependency recovery without waiting for new
      user traffic or broker messages.
- [ ] Health responses include actionable failure reasons.

## Resilience

- [ ] Required dependency outage removes readiness.
- [ ] Required dependency recovery restores runtime objects.
- [ ] Required publishers proactively reconnect after connection or channel
      close.
- [ ] WebSocket ownership is claimed before socket open and lost ownership stops
      the socket and reconnect loop.
- [ ] Redis ownership renew/release uses atomic compare-and-expire or
      compare-and-delete.
- [ ] Listen-key refresh failure recreates the listen key and replaces the
      socket without running duplicate sockets.
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
- [ ] Publisher reconnect tests prove readiness recovers without a new publish.
- [ ] WebSocket integration tests cover close, missed pong, subscription
      restore, ownership reclaim, and listen-key replacement where applicable.
- [ ] Graceful shutdown tests exist.
- [ ] Tests assert externally visible behavior.

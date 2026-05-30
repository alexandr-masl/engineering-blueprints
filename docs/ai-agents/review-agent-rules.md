# Review Agent Rules

## Review Priorities

Review agents should prioritize:

- incorrect readiness semantics
- dead consumers or subscriptions hidden by healthy process state
- missing dependency recovery
- message loss or duplicate processing risks
- unbounded retry queues
- stale WebSocket data treated as healthy
- quiet account/user-data streams incorrectly failing readiness because no
  business message arrived
- Redis ownership renewal or release implemented without atomic compare
  semantics
- WebSocket reconnect loops that continue after ownership is lost
- listen-key replacement that opens a new socket before stopping the old one
- background loops that can die silently
- shutdown behavior that can drop in-flight work unexpectedly
- missing integration tests for runtime lifecycle changes

## Required Checks

Ask these questions during review:

- Can the service be alive but unable to perform required work?
- Does readiness expose that condition?
- What happens when each dependency restarts?
- Are consumers, publishers, and subscriptions recreated after reconnect?
- Do WebSocket streams use the right readiness signal for their stream kind?
- Can two instances open the same WebSocket listener?
- Does ownership loss stop the socket and reconnect loop?
- Are listen keys refreshed before expiry and replaced safely after failures?
- Are retries bounded?
- Is shutdown ordered and timed?
- Are observability signals sufficient for incident response?

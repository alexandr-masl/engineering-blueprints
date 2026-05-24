# Review Agent Rules

## Review Priorities

Review agents should prioritize:

- incorrect readiness semantics
- dead consumers or subscriptions hidden by healthy process state
- missing dependency recovery
- message loss or duplicate processing risks
- unbounded retry queues
- stale WebSocket data treated as healthy
- background loops that can die silently
- shutdown behavior that can drop in-flight work unexpectedly
- missing integration tests for runtime lifecycle changes

## Required Checks

Ask these questions during review:

- Can the service be alive but unable to perform required work?
- Does readiness expose that condition?
- What happens when each dependency restarts?
- Are consumers, publishers, and subscriptions recreated after reconnect?
- Are retries bounded?
- Is shutdown ordered and timed?
- Are observability signals sufficient for incident response?

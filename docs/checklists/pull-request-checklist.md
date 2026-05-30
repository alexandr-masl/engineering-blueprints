# Pull Request Checklist

Use this checklist for changes that affect runtime behavior, dependencies,
health checks, or deployment behavior.

- [ ] The change preserves or updates the service readiness contract.
- [ ] Required and optional dependencies are classified correctly.
- [ ] Readiness semantics match functional behavior.
- [ ] Dependency reconnect behavior is implemented where needed.
- [ ] Required publisher reconnect is proactive and does not wait for the next
      publish.
- [ ] WebSocket stream readiness uses the correct signal for stream kind.
- [ ] Redis TTL ownership, if used, is atomic and stops sockets on ownership
      loss.
- [ ] Listen-key/session-key lifecycle, if used, refreshes before expiry and
      replaces sockets safely after repeated keepalive failure.
- [ ] Consumers, publishers, subscriptions, and background loops expose state.
- [ ] Retry and buffering are bounded.
- [ ] Shutdown behavior is ordered and bounded.
- [ ] Tests cover changed startup, outage, recovery, or shutdown behavior.
- [ ] Broker recovery tests prove readiness recovers without new traffic when
      required publishers are involved.
- [ ] WebSocket tests cover reconnect, missed pong, subscription restore, and
      duplicate listener prevention where applicable.
- [ ] Metrics, logs, and alerts are updated where behavior changed.
- [ ] Documentation is updated for new operational expectations.

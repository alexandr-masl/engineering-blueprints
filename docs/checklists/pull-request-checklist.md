# Pull Request Checklist

Use this checklist for changes that affect runtime behavior, dependencies,
health checks, or deployment behavior.

- [ ] The change preserves or updates the service readiness contract.
- [ ] Required and optional dependencies are classified correctly.
- [ ] Readiness semantics match functional behavior.
- [ ] Dependency reconnect behavior is implemented where needed.
- [ ] Consumers, publishers, subscriptions, and background loops expose state.
- [ ] Retry and buffering are bounded.
- [ ] Shutdown behavior is ordered and bounded.
- [ ] Tests cover changed startup, outage, recovery, or shutdown behavior.
- [ ] Metrics, logs, and alerts are updated where behavior changed.
- [ ] Documentation is updated for new operational expectations.

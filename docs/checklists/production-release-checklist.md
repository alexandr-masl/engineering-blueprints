# Production Release Checklist

Use this checklist before deploying a service or major runtime change to
production.

- [ ] Service readiness contract is current.
- [ ] Configuration and secrets are validated at startup.
- [ ] Required dependency outages are reflected in readiness.
- [ ] Readiness recovers after dependency recovery without waiting for new user
      traffic or broker messages.
- [ ] Required WebSocket streams have documented readiness semantics for
      market-data vs quiet account/user-data behavior.
- [ ] Redis TTL ownership settings are reviewed for TTL, renew interval, and
      failed-renewal threshold.
- [ ] Kubernetes probes match lifecycle semantics.
- [ ] Graceful shutdown has been tested.
- [ ] Metrics and alerts are configured.
- [ ] Logs are structured and do not expose secrets.
- [ ] Rollback plan is documented.
- [ ] Database or broker schema changes are backward compatible.
- [ ] Runbook or incident notes exist for new failure modes.
- [ ] Required publisher reconnect behavior has been tested after broker restart.
- [ ] WebSocket ownership, listen-key, and duplicate listener tests pass where
      applicable.
- [ ] Integration and resilience tests pass.

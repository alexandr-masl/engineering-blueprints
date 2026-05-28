# Production Release Checklist

Use this checklist before deploying a service or major runtime change to
production.

- [ ] Service readiness contract is current.
- [ ] Configuration and secrets are validated at startup.
- [ ] Required dependency outages are reflected in readiness.
- [ ] Readiness recovers after dependency recovery without waiting for new user
      traffic or broker messages.
- [ ] Kubernetes probes match lifecycle semantics.
- [ ] Graceful shutdown has been tested.
- [ ] Metrics and alerts are configured.
- [ ] Logs are structured and do not expose secrets.
- [ ] Rollback plan is documented.
- [ ] Database or broker schema changes are backward compatible.
- [ ] Runbook or incident notes exist for new failure modes.
- [ ] Required publisher reconnect behavior has been tested after broker restart.
- [ ] Integration and resilience tests pass.

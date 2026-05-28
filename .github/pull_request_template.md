# Pull Request Checklist

- [ ] The change preserves or updates the relevant readiness contract.
- [ ] Required and optional dependencies are classified correctly.
- [ ] Readiness reflects functional service behavior.
- [ ] Required publisher reconnect does not wait for the next publish.
- [ ] Startup, outage, recovery, and shutdown behavior are covered where changed.
- [ ] Broker recovery tests prove readiness recovers without new traffic when
      required publishers are involved.
- [ ] Metrics, logs, and alerts are updated where behavior changed.
- [ ] Documentation is updated.

## Summary

Describe the change and the operational behavior it affects.

## Verification

List commands, tests, or manual checks performed.

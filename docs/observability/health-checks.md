# Health Checks

## Purpose

Health checks communicate whether a service process is alive and whether it can
perform required work.

## Endpoints

### Liveness

Liveness answers: is the process running and responsive?

It should not fail only because a recoverable dependency is temporarily down.

### Readiness

Readiness answers: can the service currently perform required work?

It must include required dependency and runtime component state:

- broker consumers
- publishers
- database connections
- Redis connections and subscriptions
- WebSocket connections and freshness
- background loops and timers

### Startup

Startup answers: has initialization completed?

Use this when slow startup would otherwise cause premature restarts.

## Response Shape

Health responses should include:

- overall status
- dependency statuses
- component statuses
- reasons for degraded or unhealthy state
- timestamps for last success and last failure where useful

Avoid exposing secrets, credentials, or sensitive customer data.

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

## Functional Readiness Checks

For services that use MongoDB, RabbitMQ, Redis, WebSockets, timers, or
background workers, readiness must validate functional runtime state, not just
process uptime.

At minimum, include these checks when the dependency is part of the service
contract:

- MongoDB is connected, if required.
- RabbitMQ is connected, if required.
- Expected RabbitMQ consumers are active.
- Required publishers are initialized.
- Required publishers are not stuck in `reconnecting` after the broker has
  recovered.
- Redis is connected, if required.
- Required Redis subscriptions are active, if applicable.
- Required WebSocket streams are connected and fresh, if applicable.
- Required quiet account/user-data WebSocket streams have fresh heartbeat,
  ownership, and listen-key state, even when no business messages are expected.
- Required timers and background loops have completed within their allowed
  success window.

These checks belong in readiness, not liveness. A recoverable dependency outage
should normally remove the service from readiness without forcing a container
restart.

Readiness recovery must not depend on new user traffic or new broker messages.
For example, after RabbitMQ restarts, a required publisher must reconnect
proactively so `/health/ready` can return to healthy even when no new publish is
attempted.

For the architecture rationale and service design rules, see
[Microservice Resilience Blueprint](../architecture/microservice-resilience-blueprint.md).

## Response Shape

Health responses should include:

- overall status
- dependency statuses
- component statuses
- reasons for degraded or unhealthy state
- timestamps for last success and last failure where useful

Avoid exposing secrets, credentials, or sensitive customer data.

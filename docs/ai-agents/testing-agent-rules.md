# Testing Agent Rules

## Core Rules

- Prefer integration tests for broker, Redis, WebSocket, database, and lifecycle
  behavior.
- Use real dependencies in containers where practical.
- Assert externally visible behavior.
- Avoid arbitrary sleeps when a condition can be polled.
- Test readiness transitions, not only happy-path requests.
- Include outage and recovery tests for required dependencies.

## Minimum Evidence

Testing agents should look for evidence from:

- health endpoints
- consumed and published messages
- persisted records
- Redis keys or pub/sub effects
- WebSocket messages
- metrics
- structured logs

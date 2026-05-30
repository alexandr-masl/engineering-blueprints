# Testing Agent Rules

## Core Rules

- Prefer integration tests for broker, Redis, WebSocket, database, and lifecycle
  behavior.
- Use real dependencies in containers where practical.
- Assert externally visible behavior.
- Avoid arbitrary sleeps when a condition can be polled.
- Test readiness transitions, not only happy-path requests.
- Include outage and recovery tests for required dependencies.
- For WebSocket streams, test server-side close, missed pong, subscription
  restore, ownership loss, duplicate listener races, and listen-key replacement
  where applicable.

## Minimum Evidence

Testing agents should look for evidence from:

- health endpoints
- consumed and published messages
- persisted records
- Redis keys or pub/sub effects
- Redis ownership TTL and owner value
- WebSocket messages
- WebSocket status fields and heartbeat timestamps
- metrics
- structured logs

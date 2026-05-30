# WebSocket Service Resilience Test Plan

## Required Tests

- startup with upstream WebSocket available
- readiness false when upstream is unavailable
- reconnect after upstream disconnect
- subscription resubmission after reconnect
- readiness false when data is stale
- quiet account stream readiness uses heartbeat/listen-key freshness
- ping or pong timeout handling
- Redis ownership claim before socket open
- ownership TTL expiry lets another instance restore the stream
- listen-key refresh failure recreates listen key and socket
- duplicate listener race creates only one socket
- malformed message isolation
- slow downstream client handling
- bounded outbound queue behavior
- graceful shutdown closes upstream and downstream connections

## Metrics to Assert

- active connections
- reconnect count
- last message timestamp
- last pong timestamp
- ownership renew failures
- listen-key refresh failures
- stale duration
- dropped message count
- outbound queue depth

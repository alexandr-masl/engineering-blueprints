# WebSocket Service Resilience Test Plan

## Required Tests

- startup with upstream WebSocket available
- readiness false when upstream is unavailable
- reconnect after upstream disconnect
- subscription resubmission after reconnect
- readiness false when data is stale
- ping or pong timeout handling
- malformed message isolation
- slow downstream client handling
- bounded outbound queue behavior
- graceful shutdown closes upstream and downstream connections

## Metrics to Assert

- active connections
- reconnect count
- last message timestamp
- stale duration
- dropped message count
- outbound queue depth

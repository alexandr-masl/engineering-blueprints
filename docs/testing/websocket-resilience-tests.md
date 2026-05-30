# WebSocket Resilience Tests

## Purpose

WebSocket resilience tests prove that services handle connection lifecycle,
message freshness, subscriptions, backpressure, and reconnect behavior.

## Required Scenarios

- upstream unavailable at startup
- upstream disconnect while running
- reconnect with subscription resubmission
- open connection with stale data
- quiet account stream with healthy heartbeat but no business messages
- listen-key or session-key refresh failure
- Redis ownership loss while socket is open
- duplicate listener race between two service instances
- ping or pong failure
- malformed message
- slow downstream client
- downstream client disconnect
- burst of messages beyond normal rate

## Assertions

Tests should assert:

- readiness is false when required WebSocket data is stale or unavailable
- quiet account streams use heartbeat and listen-key freshness instead of
  business-message freshness
- reconnect logic restores required subscriptions
- reconnect checks ownership before opening a replacement socket
- stale open connections are not treated as healthy
- ownership loss stops the socket and reconnect loop
- listen-key replacement stops the old socket before opening the new socket
- slow clients are disconnected, throttled, or backpressured according to policy
- malformed messages are logged and isolated
- message queues are bounded
- metrics expose connection state, reconnect count, last message time, stale
  duration, dropped messages, client count, ownership state, and listen-key
  refresh state

For full integration scenarios, see
[WebSocket Integration Test Blueprint](websocket-integration-test-blueprint.md).

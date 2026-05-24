# WebSocket Resilience Tests

## Purpose

WebSocket resilience tests prove that services handle connection lifecycle,
message freshness, subscriptions, backpressure, and reconnect behavior.

## Required Scenarios

- upstream unavailable at startup
- upstream disconnect while running
- reconnect with subscription resubmission
- open connection with stale data
- ping or pong failure
- malformed message
- slow downstream client
- downstream client disconnect
- burst of messages beyond normal rate

## Assertions

Tests should assert:

- readiness is false when required WebSocket data is stale or unavailable
- reconnect logic restores required subscriptions
- stale open connections are not treated as healthy
- slow clients are disconnected, throttled, or backpressured according to policy
- malformed messages are logged and isolated
- message queues are bounded
- metrics expose connection state, reconnect count, last message time, stale
  duration, dropped messages, and client count

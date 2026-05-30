# WebSocket Integration Test Blueprint

## Purpose

This blueprint defines integration tests for WebSocket services that use
heartbeats, reconnects, Redis TTL ownership, listen-key refresh, or duplicate
listener prevention.

Use these tests for exchange-style clients, market-data streams, account-data
streams, notification streams, and any long-lived WebSocket runtime component.

## Test Environment

Use real or realistic runtime dependencies:

- fake WebSocket server controlled by the test
- Redis test container when TTL ownership is used
- fake listen-key/session-key HTTP API when listen keys are used
- the service's production WebSocket manager where practical
- health endpoint or health registry assertions

Avoid mocking the WebSocket lifecycle itself. The point is to prove real close,
heartbeat, reconnect, and ownership behavior.

## Required Scenarios

### Server-Side Close

```text
1. Start fake WebSocket server.
2. Start service stream.
3. Assert stream is ready.
4. Close the socket from the server.
5. Assert readiness becomes false for required streams.
6. Restart or reopen the fake server.
7. Assert reconnect starts.
8. Assert subscriptions/session state are restored.
9. Assert readiness becomes true.
```

### Missed Pong

```text
1. Start fake WebSocket server.
2. Allow initial ping/pong.
3. Stop responding to ping.
4. Assert missed pong threshold is reached.
5. Assert the stale socket is terminated.
6. Assert reconnect starts.
```

### Subscription Restore

```text
1. Connect and subscribe to required channels.
2. Force server-side close.
3. Allow reconnect.
4. Assert required subscriptions are sent again.
5. Assert stream is not ready until restore completes.
```

### Market-Data Freshness

```text
1. Start stream that expects frequent market data.
2. Send messages until ready.
3. Stop sending messages but keep socket open.
4. Assert readiness becomes false after freshness window.
5. Send messages again.
6. Assert readiness recovers.
```

### Quiet Account Stream Freshness

```text
1. Start account/user-data stream.
2. Keep socket open and pong healthy.
3. Do not send business messages.
4. Keep listen key fresh.
5. Assert readiness remains true.
6. Stop pong or listen-key refresh.
7. Assert readiness becomes false.
```

### Ownership Claim Before Open

```text
1. Start Redis test container.
2. Attempt to start stream.
3. Assert ownership key is claimed before WebSocket open.
4. Assert socket does not open if ownership claim fails.
```

### Ownership TTL Expiry And Reclaim

```text
1. Start instance A and claim ownership.
2. Start instance B for the same ownership key.
3. Assert only instance A opens the WebSocket.
4. Stop A ownership renewal or kill A.
5. Wait for TTL expiry.
6. Assert B claims ownership.
7. Assert B opens exactly one WebSocket.
```

### Ownership Lost While Running

```text
1. Start stream with ownership.
2. Replace Redis ownership value with another owner.
3. Assert old owner detects ownership loss.
4. Assert old owner stops WebSocket and reconnect loop.
5. Assert readiness becomes false for the old owner.
```

### Listen-Key Refresh Failure

```text
1. Start stream with fake listen-key API.
2. Assert listen key is refreshed before expiry.
3. Make keepalive fail repeatedly.
4. Assert stream becomes unhealthy.
5. Assert old socket is stopped.
6. Assert new listen key is created.
7. Assert replacement socket opens.
8. Assert readiness recovers only after replacement stream is healthy.
```

### Duplicate Listener Race

```text
1. Start two service instances at the same time.
2. Both attempt to claim the same ownership key.
3. Assert only one ownership claim succeeds.
4. Assert only one WebSocket opens.
5. Assert duplicate listener prevention metric increments if applicable.
```

## Required Assertions

Tests should assert:

- stream status fields are updated
- readiness changes on outage and recovery
- reconnect attempts are bounded and observable
- ownership renew and release are atomic
- ownership loss stops sockets and reconnect loops
- listen-key refresh failures are tracked
- old sockets stop before replacement sockets open
- quiet account streams do not fail readiness only because no business event
  arrived
- market-data streams fail readiness when business data is stale

## Anti-Patterns

- using only mocked WebSocket client methods
- treating open socket as healthy without heartbeat or freshness
- using business-message freshness for quiet account streams
- opening WebSocket before ownership claim succeeds
- using `GET` then `EXPIRE` or `GET` then `DEL` for ownership
- allowing reconnect loops to continue after ownership is lost
- opening replacement sockets before old sockets stop

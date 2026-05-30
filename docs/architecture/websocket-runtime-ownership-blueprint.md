# WebSocket Runtime Ownership Blueprint

## Purpose

This blueprint defines how services should manage long-lived WebSocket streams
that require runtime ownership, heartbeats, reconnects, listen-key refresh, or
singleton listeners across multiple service instances.

It is intended for market-data streams, account/user-data streams, exchange
client streams, notification streams, and other long-lived WebSocket
connections.

## Core Rule

A WebSocket stream is ready only when the service owns the stream, the socket is
functionally alive, required session keys are fresh, and the stream can recover
without creating duplicate listeners.

An open socket is not enough.

## Required WebSocket State

Every managed WebSocket stream must expose state that can be used by readiness,
metrics, logs, and tests.

Required fields:

```ts
type WebSocketStreamStatus = {
  name: string;
  required: boolean;
  kind: 'market-data' | 'account-data' | 'user-data' | 'control' | 'other';
  status: 'starting' | 'connecting' | 'open' | 'stale' | 'reconnecting' | 'failed' | 'stopped';
  active: boolean;
  lastOpenAt?: number;
  lastMessageAt?: number;
  lastPongAt?: number;
  lastErrorAt?: number;
  lastError?: string;
  reconnectAttempts: number;
  ownerKey?: string;
  ownerId?: string;
  listenKeyExpiresAt?: number;
  listenKeyLastRefreshAt?: number;
  listenKeyRefreshFailures?: number;
};
```

Readiness must use this state directly instead of inferring health from process
uptime or object existence.

## Stream Classes

### Market-Data Streams

Market-data streams are expected to receive regular business messages.

Readiness should check:

- socket is open
- ownership is valid when ownership is required
- heartbeat pong is recent
- last business message is inside the freshness window
- subscriptions are restored after reconnect

If market data is stale beyond the service contract, readiness must become
false.

### Account/User-Data Streams

Account or user-data streams can be quiet for long periods when no account event
occurs.

Readiness should not require frequent business messages for quiet account
streams. Instead, use:

- socket open state
- ping/pong freshness
- listen-key or session-key freshness
- ownership TTL freshness
- subscription/session restore state
- explicit server keepalive signals when available

Business-message freshness may still be tracked as observability, but it should
not fail readiness for a stream that is expected to be quiet.

## Redis TTL Ownership

Use Redis TTL ownership when exactly one service instance should run a
WebSocket stream for a logical resource.

Examples:

- one account stream per account
- one exchange stream per user
- one notification stream per tenant
- one market shard listener per symbol group

Ownership key pattern:

```text
ws-owner:{provider}:{streamKind}:{resourceId}
```

Ownership value:

```text
{serviceName}:{podName}:{instanceId}:{streamId}
```

Rules:

- Claim ownership before opening the WebSocket.
- A process may run the stream only while it owns the Redis key.
- The ownership key must always have a TTL.
- The owner must renew the TTL periodically.
- If ownership renewal fails too many times, stop the WebSocket before another
  instance is likely to reclaim the key.
- If the stored owner value changes, the old owner must stop the WebSocket and
  reconnect loop.
- Releasing ownership must be compare-and-delete, not blind delete.

## TTL Ratios

Choose TTL, renew interval, and failure thresholds so the old owner stops before
another instance is likely to reclaim.

Baseline ratio:

```text
ttl >= renewInterval * (maxFailedRenewals + 1)
```

Recommended starting point:

```text
ttl: 30s
renewInterval: 10s
maxFailedRenewals: 2
```

This allows two failed renewals and still leaves a bounded window for the owner
to stop before the key expires.

Use a longer TTL when allowing more failed renewals:

```text
ttl: 60s
renewInterval: 10s
maxFailedRenewals: 3
```

Rules:

- Renew interval should usually be one third or less of the TTL.
- The owner should stop after `maxFailedRenewals`, not wait for TTL expiry.
- Reclaim delay must be longer than the old owner's stop threshold.
- Add jitter to reclaim attempts to reduce instance races.
- Do not use ownership TTLs so short that normal Redis latency causes false
  failover.

## Atomic Ownership Operations

Claim ownership with an atomic `SET key value NX PX ttl` operation or an
equivalent compare-and-set primitive.

Do not implement ownership renewal or release with separate `GET` then
`EXPIRE` or `GET` then `DEL` calls. They are race-prone.

Use compare-and-expire for renew:

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("PEXPIRE", KEYS[1], ARGV[2])
end

return 0
```

Use compare-and-delete for release:

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
end

return 0
```

The owner value must be unique per runtime instance so old processes cannot
renew or delete ownership after a newer process has claimed it.

## Exchange Listen-Key Lifecycle

Some exchange-style APIs require a temporary listen key, session key, or stream
token before opening a user-data WebSocket. This blueprint calls that value a
listen key, but the pattern applies to any expiring stream session token.

Rules:

- Create or load the listen key before opening the WebSocket.
- Refresh the listen key before expiry with a safety margin.
- Track refresh attempts, failures, last success time, and expiry time.
- If refresh repeatedly fails, mark the stream unhealthy.
- After repeated keepalive failures, recreate the listen key and replace the
  WebSocket.
- Stop the old socket before opening the replacement socket.
- Do not run two sockets for the same listen key or ownership key.
- Readiness for quiet account streams should use listen-key freshness and
  heartbeat freshness, not business-message freshness.

Example lifecycle:

```text
1. Claim Redis ownership.
2. Create or refresh listen key.
3. Open WebSocket with the listen key.
4. Mark stream ready after socket open and heartbeat/subscription state is valid.
5. Renew Redis ownership on schedule.
6. Refresh listen key before expiry.
7. If listen-key refresh fails repeatedly, stop old socket.
8. Recreate listen key.
9. Open replacement socket.
10. Mark readiness true only after the replacement stream is healthy.
```

## Exchange-Style Examples

Keep implementations provider-neutral by modeling the lifecycle, not the vendor.

Binance-style pattern:

```text
1. Create listen key with authenticated REST call.
2. Open private WebSocket with listen key.
3. Refresh listen key before expiry.
4. Recreate listen key and socket after repeated refresh failure.
```

BingX-style pattern:

```text
1. Create or authenticate a private stream session.
2. Open private WebSocket with session credentials.
3. Maintain heartbeat and session freshness.
4. Replace the socket/session after repeated keepalive failure.
```

The readiness contract should describe the required lifecycle fields and
freshness windows without hard-coding provider-specific endpoint behavior into
the service architecture.

## Duplicate Listener Prevention

The service must prevent duplicate WebSocket listeners for the same logical
resource.

Rules:

- Maintain an in-process registry keyed by ownership key or stream ID.
- Reject or reuse duplicate start requests inside the same process.
- Claim Redis ownership before opening a cross-process singleton stream.
- Stop the WebSocket and reconnect loop when ownership is lost.
- Reclaim only after TTL expiry or explicit compare-and-delete release.
- A reconnect loop must re-check ownership before opening a new socket.

## Readiness Semantics

Required streams fail readiness when:

- ownership is required and not currently owned
- socket is closed, failed, or stuck reconnecting beyond the recovery window
- heartbeat pong is stale
- market-data business messages are stale beyond the freshness window
- account/user-data listen key is expired or near expiry without successful
  refresh
- required subscriptions were not restored after reconnect
- duplicate listener conflict is detected

Required quiet account streams should remain ready when no business messages are
arriving if heartbeat, ownership, and listen-key state are healthy.

## Metrics

Expose metrics for:

- stream active state
- stream status
- last open timestamp
- last message timestamp
- last pong timestamp
- reconnect attempts and failures
- ownership renew success and failure
- ownership lost count
- duplicate listener prevented count
- listen-key refresh success and failure
- listen-key expiry timestamp
- stale stream count

## Shutdown

Shutdown must:

1. Mark readiness false.
2. Stop reconnect loops.
3. Stop WebSocket streams.
4. Stop listen-key refresh timers.
5. Stop ownership renewal timers.
6. Release ownership with compare-and-delete when safe.
7. Close sockets with bounded timeouts.

Shutdown must not leave a reconnect loop running after ownership has been
released.

## Required Tests

Services using this pattern should test:

- stream status exposes required fields
- market-data readiness fails on stale business messages
- quiet account-stream readiness uses heartbeat and listen-key freshness
- ownership claim happens before socket open
- ownership renew uses compare-and-expire
- ownership release uses compare-and-delete
- owner stops after max failed renewals
- another instance reclaims after TTL expiry
- two instances racing create only one active listener
- reconnect re-checks ownership before opening a socket
- listen key refresh happens before expiry
- repeated listen-key refresh failure recreates listen key and socket
- old socket is stopped before replacement socket opens

## Related Documents

- [Microservice Resilience Blueprint](microservice-resilience-blueprint.md)
- [WebSocket Integration Test Blueprint](../testing/websocket-integration-test-blueprint.md)
- [WebSocket Resilience Tests](../testing/websocket-resilience-tests.md)
- [Redis Resilience Tests](../testing/redis-resilience-tests.md)
- [Health Checks](../observability/health-checks.md)

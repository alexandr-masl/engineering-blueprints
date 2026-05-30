# Microservice Resilience Blueprint

This document defines a reusable architecture blueprint for building robust microservices that depend on long-lived external connections such as RabbitMQ, Redis, WebSockets, MongoDB, exchange APIs, or other infrastructure services.

The main goal is simple:

> A pod must not be considered healthy only because the Node.js process is alive. It is healthy only when all required runtime dependencies, consumers, publishers, subscriptions, timers, and background loops are functionally alive.

---

## 1. Problem Statement

Microservices that use brokers, pub/sub systems, WebSockets, or external APIs often fail in a misleading way:

* The Kubernetes pod is still `1/1 Running`.
* The Node.js process has not exited.
* Logs may look quiet.
* But RabbitMQ consumers are detached.
* Publishers are not initialized.
* Redis subscriptions are gone.
* WebSocket streams stopped receiving data.
* Background loops silently stopped.

This creates a dangerous state: the service is technically alive but functionally dead.

The architecture must therefore treat external connections as first-class runtime components with lifecycle, state, health checks, reconnection logic, metrics, and tests.

---

## 2. Core Design Principles

### 2.1 Do Not Start Runtime Work Before Dependencies Are Ready

Consumers, WebSocket listeners, schedulers, and handlers must not start until all required dependencies are initialized.

Correct startup order example:

```text
1. Load config
2. Connect MongoDB
3. Connect Redis
4. Connect RabbitMQ
5. Initialize publishers
6. Assert queues/exchanges/contracts
7. Start consumers
8. Start WebSocket listeners
9. Start background timers/heartbeats
10. Mark application ready
```

Bad startup order:

```text
1. Start consumers
2. Message arrives
3. Handler tries to publish
4. Publisher is undefined
5. Process crashes or message is stuck
```

Rule:

> No consumer should be able to receive a message before its required publisher, database, Redis client, and service dependencies are ready.

---

### 2.2 Separate Liveness From Readiness

A running process is not enough.

Use two different health concepts:

#### Liveness

The process is alive and not permanently stuck.

Example:

```http
GET /health/live
```

Returns healthy if:

* Node.js event loop is responsive.
* Process is not shutting down.
* Service has not entered a fatal state.

#### Readiness

The service is functionally able to do its job.

Example:

```http
GET /health/ready
```

Returns healthy only if required runtime components are healthy:

* RabbitMQ connection is open.
* RabbitMQ publishers are initialized.
* Required RabbitMQ consumers are registered.
* Redis is connected, if required.
* MongoDB is connected, if required.
* WebSocket streams are connected or intentionally reconnecting within an allowed window.
* Required timers and heartbeats are active.
* Queue contracts have been asserted successfully.

Rule:

> Kubernetes readiness should fail when the service cannot process work correctly.

---

### 2.3 Make Runtime Components Observable

Every long-running connection or loop should expose state.

Each component should report:

```ts
export type ComponentStatus = {
  name: string;
  required: boolean;
  status: 'starting' | 'ready' | 'degraded' | 'reconnecting' | 'failed' | 'stopped';
  lastOkAt?: number;
  lastErrorAt?: number;
  lastError?: string;
  reconnectAttempts?: number;
  details?: Record<string, unknown>;
};
```

Examples of runtime components:

* `rabbitmq.connection`
* `rabbitmq.publisher.exchangeStreamCommands`
* `rabbitmq.consumer.tradeStationNewTgTrade`
* `redis.publisher`
* `redis.subscriber.satoshiChannelCommand`
* `mongo.main`
* `ws.binance.accountStream`
* `ws.bingx.accountStream`
* `timer.listenKeyRefresh`
* `timer.redisOwnershipHeartbeat`

Rule:

> If a component can silently stop, it must expose state and be part of readiness or observability.

---

## 3. Recommended Architecture

Use explicit lifecycle managers instead of scattered connection logic.

```text
src/
  app.ts
  bootstrap.ts
  health/
    health-registry.ts
    health.controller.ts
  infrastructure/
    rabbitmq/
      rabbitmq-connection-manager.ts
      rabbitmq-publisher-manager.ts
      rabbitmq-consumer-manager.ts
      rabbitmq-contracts.ts
    redis/
      redis-connection-manager.ts
      redis-pubsub-manager.ts
      redis-streams-manager.ts
    websocket/
      websocket-manager.ts
      reconnect-policy.ts
    mongo/
      mongo-connection-manager.ts
  runtime/
    lifecycle-manager.ts
    component-status.ts
    shutdown-manager.ts
  metrics/
    metrics.ts
```

The app should have one top-level lifecycle coordinator:

```ts
class LifecycleManager {
  async start() {
    await this.mongo.connect();
    await this.redis.connect();
    await this.rabbitmq.connect();

    await this.rabbitmq.assertContracts();
    await this.rabbitmq.initPublishers();
    await this.rabbitmq.startConsumers();

    await this.websockets.start();
    await this.timers.start();

    this.health.markReady();
  }

  async stop() {
    this.health.markShuttingDown();

    await this.timers.stop();
    await this.websockets.stop();
    await this.rabbitmq.stopConsumers();
    await this.rabbitmq.closePublishers();
    await this.redis.close();
    await this.mongo.close();
  }
}
```

---

## 4. RabbitMQ Resilience Rules

### 4.1 Connection Management

RabbitMQ connections and channels can close unexpectedly after:

* Broker restart.
* Network partition.
* Kubernetes node disruption.
* Heartbeat timeout.
* Protocol error.
* Queue declaration mismatch.

Rules:

* Use a connection manager.
* Reconnect automatically with capped exponential backoff.
* Recreate channels after reconnect.
* Reassert exchanges, queues, bindings, and consumers after reconnect.
* Reinitialize publishers after reconnect.
* Required publishers must reconnect proactively after connection or channel
  close; they must not wait for the next publish attempt.
* Treat closed channels as unhealthy.
* Never assume a channel remains valid forever.

Example reconnect policy:

```ts
type ReconnectPolicy = {
  minDelayMs: number;
  maxDelayMs: number;
  factor: number;
  jitter: boolean;
};

const defaultReconnectPolicy: ReconnectPolicy = {
  minDelayMs: 500,
  maxDelayMs: 30_000,
  factor: 2,
  jitter: true,
};
```

---

### 4.2 Publisher Rules

Publishers must be initialized before consumers start.

Rules:

* Publisher initialization must be awaited during startup.
* Publisher state must be exposed in readiness.
* Publisher connection or channel close must mark the publisher unhealthy and
  schedule proactive reinitialization.
* Publisher close handlers must be attached to the underlying connection and
  channel, not only to publish failures.
* Publisher reconnect must retry with capped backoff while the broker is still
  booting.
* Publisher reconnect must not depend on the next call to `publish`.
* Publisher reconnect attempt counters must reflect real reconnect attempts and
  reset or stabilize after a successful reconnect.
* Publishing should fail fast if the publisher is not ready.
* Critical publish paths should use confirms when message durability matters.

Bad:

```ts
let publisher: Publisher | undefined;

consumer.onMessage(async msg => {
  await publisher!.publish(msg);
});
```

Good:

```ts
class PublisherManager {
  private publisher?: Publisher;
  private reconnectTimer?: NodeJS.Timeout;
  private shuttingDown = false;

  async init() {
    this.publisher = await createPublisher();
    this.bindPublisherCloseHandlers();
  }

  assertReady() {
    if (!this.publisher) {
      throw new InfrastructureNotReadyError('Publisher is not initialized');
    }
  }

  async publish(message: unknown) {
    this.assertReady();
    return this.publisher!.publish(message);
  }

  private bindPublisherCloseHandlers() {
    this.publisher!.on('close', () => {
      this.publisher = undefined;
      this.scheduleReconnect();
    });
  }

  private scheduleReconnect() {
    if (this.shuttingDown || this.reconnectTimer) return;

    this.reconnectTimer = setTimeout(async () => {
      this.reconnectTimer = undefined;

      try {
        await this.init();
      } catch {
        this.scheduleReconnect();
      }
    }, nextReconnectDelayMs());
  }
}
```

Implementation warning:

> A required publisher that reconnects only on the next publish can keep
> `/health/ready` stuck at `503` after RabbitMQ recovers. Readiness checks the
> publisher, so the publisher must recover without waiting for new traffic.

---

### 4.3 Consumer Rules

Consumers must be restartable and observable.

Rules:

* Track consumer tags returned by RabbitMQ.
* Store expected consumers in a registry.
* Readiness must fail if required consumers are missing.
* After reconnect, all consumers must be registered again.
* If a consumer channel closes, mark the consumer unhealthy.
* Use `prefetch` to control concurrency.
* Avoid unlimited parallel processing.

Example expected consumer state:

```ts
type ExpectedConsumer = {
  name: string;
  queue: string;
  required: boolean;
  consumerTag?: string;
  active: boolean;
  lastMessageAt?: number;
};
```

Readiness should verify:

```text
expected required consumers count === active required consumers count
```

This catches the dangerous state where the pod is running but RabbitMQ consumer count is `0`.

---

### 4.4 Ack/Nack Rules

Every handler must explicitly define what happens on failure.

Failure categories:

#### Permanent Business Error

Examples:

* Invalid payload.
* Unknown trade ID.
* Unsupported command.
* Validation failed.

Expected behavior:

* `ack` and log, or
* publish to dead-letter queue, then `ack`.

Do not requeue forever.

#### Transient Infrastructure Error

Examples:

* MongoDB temporarily unavailable.
* Redis unavailable.
* Exchange API timeout.
* Publisher temporarily unavailable.

Expected behavior:

* retry with cap, or
* `nack(requeue=true)` with dead-letter/retry policy, or
* move to retry queue with delay.

#### Unexpected Error

Examples:

* Null reference.
* Unknown exception.
* Serialization bug.

Expected behavior:

* log with full context,
* prevent process crash if possible,
* route to retry or DLQ according to policy,
* increment metrics.

Rule:

> No thrown handler error should leave a message unacked forever.

---

### 4.5 Poison Message Protection

A poison message is a message that always fails.

Rules:

* Never requeue infinitely.
* Add retry count metadata.
* Use retry queues or delayed retries.
* Send to DLQ after max attempts.
* Alert on DLQ growth.

Example metadata:

```json
{
  "attempt": 3,
  "maxAttempts": 5,
  "originalQueue": "trade_station_new_tg_trade",
  "lastError": "Validation failed: missing tradeId"
}
```

---

### 4.6 Queue Contract Rules

All producers and consumers must agree on queue and exchange declarations.

Validate:

* Queue name.
* Exchange name.
* Routing key.
* Durable flag.
* Auto-delete flag.
* Exclusive flag.
* Dead-letter exchange.
* Dead-letter routing key.
* Message TTL.
* Prefetch policy.

Important queues from current systems:

```text
satoshi_trade_station_update
trade_station_new_tg_trade
satoshi-channels-update
tg_bot_channel_update
exchange-stream-commands
```

Rule:

> Queue contracts should be centralized and imported by producers and consumers instead of duplicated manually.

Example:

```ts
export const RabbitContracts = {
  tradeStationNewTgTrade: {
    queue: 'trade_station_new_tg_trade',
    durable: true,
    exchange: 'trade-station',
    routingKey: 'new-tg-trade',
  },
} as const;
```

---

## 5. Redis Resilience Rules

### 5.1 Redis Client Separation

Use separate Redis clients for different roles:

* Commands/client operations.
* Pub/sub publisher.
* Pub/sub subscriber.
* Streams consumer.
* Distributed lock or ownership heartbeat.

Reason:

> A Redis connection in subscriber mode cannot safely be reused for normal commands.

---

### 5.2 Redis Pub/Sub Loss Behavior

Redis pub/sub is not durable.

If the subscriber is disconnected while a message is published, the message is lost.

Therefore:

* Use Redis pub/sub only for non-critical real-time notifications.
* Use RabbitMQ or Redis Streams for critical commands/events.
* Document all pub/sub channels and whether loss is acceptable.

Example classification:

```text
satoshiChannelCommand
- Transport: Redis pub/sub
- Durable: No
- Loss acceptable: Only if command can be recalculated or resent
- If not acceptable: migrate to RabbitMQ or Redis Streams
```

---

### 5.3 Redis Disconnect Behavior

Every Redis publish or command path must define behavior during outage.

Possible policies:

```text
Policy A: Fail message processing and retry RabbitMQ message
Policy B: Ack RabbitMQ message and log Redis failure
Policy C: Persist fallback record to MongoDB for later replay
Policy D: Publish to RabbitMQ instead of Redis
```

Rule:

> Redis failures must never be silent. The code must explicitly choose retry, drop-with-log, fallback, or fail-fast.

---

### 5.4 Redis Ownership and TTL Heartbeats

For services that claim ownership of a resource, such as one WebSocket listener per account/channel, use TTL-based ownership.

Rules:

* Ownership key must have TTL.
* Owner must renew TTL periodically.
* If renewal fails, the service should stop owning that resource.
* Another instance may reclaim ownership after TTL expiry.
* The system must prevent duplicate WebSocket listeners for the same Redis record.
* Ownership renewal and release must be atomic compare-and-expire or
  compare-and-delete operations.
* The owner must stop before another instance is likely to reclaim the key.

Example:

```text
Key: ws-owner:binance:account:{accountId}
Value: {podName}:{instanceId}
TTL: 30 seconds
Renew interval: 10 seconds
```

Ownership rule:

> A service may run a WebSocket listener only while it owns the corresponding Redis ownership key.

Detailed rules for TTL ratios, atomic Lua operations, ownership loss, and
duplicate listener prevention are defined in
[WebSocket Runtime Ownership Blueprint](websocket-runtime-ownership-blueprint.md).

---

## 6. WebSocket Resilience Rules

WebSockets require active lifecycle management because they can silently degrade.

### 6.1 Required WebSocket Features

Every WebSocket manager should support:

* Connect.
* Reconnect.
* Stop.
* Backoff with cap.
* Heartbeat ping/pong.
* Missed pong detection.
* Message timeout detection.
* Resume/re-subscribe after reconnect.
* Duplicate listener prevention.
* Explicit stream readiness state.
* Listen-key or session-key refresh when required.
* Metrics.

---

### 6.2 WebSocket Heartbeat

Use heartbeat to detect half-open connections.

Example behavior:

```text
1. Send ping every 20 seconds.
2. Expect pong within 10 seconds.
3. If pong is missed, terminate socket.
4. Start reconnect loop.
5. Re-subscribe after reconnect.
```

Rule:

> Do not rely only on the socket `close` event. Half-open sockets may not emit close quickly.

---

### 6.3 Reconnect Backoff

Reconnect loops must not hammer external APIs.

Use capped exponential backoff with jitter:

```text
500ms -> 1s -> 2s -> 4s -> 8s -> 16s -> 30s max
```

Rules:

* Cap the maximum delay.
* Add jitter.
* Stop reconnect loop when service shuts down.
* Track reconnect attempts.
* Alert on excessive reconnects.

---

### 6.4 Resume After Reconnect

After reconnect, the service must restore its previous runtime state.

Examples:

* Re-subscribe to exchange streams.
* Refresh listen key.
* Reload active account streams from Redis or MongoDB.
* Reclaim ownership if needed.
* Avoid duplicate listeners.

Rule:

> Reconnection is incomplete until subscriptions are restored and data is flowing again.

---

### 6.5 Listen Key Refresh

For exchange APIs that require listen keys:

* Refresh before expiry.
* Track refresh failures.
* Recreate stream if refresh fails repeatedly.
* Alert when refresh failure count exceeds threshold.
* Stop old socket before creating a new one.
* For quiet account/user-data streams, use heartbeat and listen-key freshness
  for readiness instead of requiring frequent business messages.

Detailed listen-key lifecycle, quiet stream readiness, and duplicate listener
rules are defined in
[WebSocket Runtime Ownership Blueprint](websocket-runtime-ownership-blueprint.md).

---

## 7. Heartbeats and Watchdogs

Heartbeats should verify that runtime components are still functional.

Types of heartbeats:

```text
RabbitMQ heartbeat        -> connection/channel alive
Consumer watchdog         -> required consumers registered
Redis heartbeat           -> ping succeeds
Redis ownership heartbeat -> TTL renewed
WebSocket heartbeat       -> ping/pong works
Timer watchdog            -> scheduled loops still running
Business heartbeat        -> messages/data received within expected window
```

### 7.1 Heartbeat Service Pattern

```ts
class HeartbeatService {
  constructor(private healthRegistry: HealthRegistry) {}

  start() {
    setInterval(async () => {
      await this.checkRabbitMQ();
      await this.checkRedis();
      await this.checkConsumers();
      await this.checkWebSockets();
      await this.checkTimers();
    }, 10_000);
  }
}
```

Important:

* Heartbeat checks should have timeouts.
* Heartbeats should not hang forever.
* Failed heartbeat should mark component degraded or failed.
* Critical failed components should make readiness fail.

---

## 8. Kubernetes Health Check Design

### 8.1 Liveness Probe

Liveness should be conservative. It should restart only when the process is broken beyond recovery.

Example:

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 3000
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

### 8.2 Readiness Probe

Readiness should reflect functional availability.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 2
```

Readiness should fail if:

* RabbitMQ required consumers are detached.
* Required publisher is not initialized.
* Redis required client is disconnected.
* MongoDB is disconnected.
* Required WebSocket ownership is lost.
* Required background loop is stopped.

### 8.3 Startup Probe

For services with slow boot or backlog recovery:

```yaml
startupProbe:
  httpGet:
    path: /health/live
    port: 3000
  failureThreshold: 30
  periodSeconds: 5
```

---

## 9. Graceful Shutdown

On SIGTERM:

```text
1. Mark readiness false.
2. Stop accepting new work.
3. Stop WebSocket reconnect loops.
4. Stop timers.
5. Cancel RabbitMQ consumers.
6. Finish in-flight messages within timeout.
7. Ack/nack in-flight messages according to policy.
8. Close RabbitMQ channels/connections.
9. Close Redis connections.
10. Close MongoDB connection.
11. Exit cleanly.
```

Example:

```ts
process.on('SIGTERM', async () => {
  await lifecycle.stop();
  process.exit(0);
});
```

Rule:

> Shutdown must stop reconnect loops. Otherwise the process may continue trying to reconnect while Kubernetes is terminating the pod.

---

## 10. Testing Blueprint

Robustness requires integration-style tests, not only unit tests.

Use testcontainers or docker-compose to start real RabbitMQ, Redis, and MongoDB where possible.

---

### 10.1 Startup Order Test

Goal:

> Consumers cannot start before required publishers and dependencies are initialized.

Test:

```text
1. Mock Mongo connection.
2. Mock publisher initialization.
3. Mock RabbitMQ consumer startup.
4. Start app bootstrap.
5. Assert order:
   - Mongo connected
   - Redis connected if required
   - RabbitMQ connected
   - publisher initialized
   - consumers launched
```

Expected result:

```text
No consumer starts before publisher init resolves.
```

---

### 10.2 RabbitMQ Backlog On Startup Test

Goal:

> Service must survive if RabbitMQ already has pending messages when it boots.

Test:

```text
1. Start RabbitMQ test container.
2. Publish message to trade_station_new_tg_trade.
3. Start app or consumer module.
4. Assert message is processed.
5. Assert no Publisher not initialized error.
6. Assert process does not exit.
```

---

### 10.3 Handler Error Ack/Nack Test

Goal:

> One bad message must not kill the process or block the queue forever.

Test cases:

```text
Permanent business error:
- Message is acked or dead-lettered.
- Message is not requeued forever.

Transient infrastructure error:
- Message is retried or nacked according to retry policy.
- Retry cap is respected.

Unexpected thrown error:
- Process stays alive.
- Error is logged.
- Message outcome is explicit.
```

---

### 10.4 RabbitMQ Reconnect Test

Goal:

> If RabbitMQ restarts, consumers reconnect automatically.

Test:

```text
1. Start app against RabbitMQ test container.
2. Confirm consumer count > 0.
3. Restart RabbitMQ.
4. Wait for reconnect.
5. Assert consumer count returns > 0.
6. Publish message.
7. Confirm message is consumed.
```

This catches the issue where a pod is `Running` but has no active consumers.

---

### 10.5 Publisher Reconnect Test

Goal:

> Required publishers reconnect proactively after broker restart and readiness
> can recover without waiting for a new publish.

Test:

```text
1. Initialize publisher.
2. Publish message successfully.
3. Stop RabbitMQ.
4. Assert readiness becomes false because the publisher is unhealthy.
5. Start RabbitMQ.
6. Wait for the publisher reconnect timer to reinitialize the channel.
7. Assert readiness becomes true before any new publish is attempted.
8. Publish again.
9. Assert publish succeeds.
```

Required variants:

```text
1. Close the publisher connection or channel directly.
2. Assert proactive reconnect is scheduled without calling publish.
3. Keep RabbitMQ unavailable during the first reconnect attempt.
4. Assert reconnect is retried with bounded backoff.
5. Assert reconnect attempt accounting is correct.
```

This catches the Stage 4 gap where consumers recover but the publisher remains
`reconnecting` because reconnect is lazy and only happens on the next publish.

---

### 10.6 RabbitMQ Queue Contract Test

Goal:

> Producer and consumer declarations must match.

Test:

```text
1. Load shared RabbitMQ contract definitions.
2. Assert expected queue names.
3. Assert durable settings.
4. Assert exchange names.
5. Assert routing keys.
6. Assert DLQ settings.
```

Important queues:

```text
satoshi_trade_station_update
trade_station_new_tg_trade
satoshi-channels-update
tg_bot_channel_update
exchange-stream-commands
```

---

### 10.7 Redis Pub/Sub Test

Goal:

> Confirm Redis publish path works and document loss behavior.

Test:

```text
1. Start Redis test container.
2. Subscribe to satoshiChannelCommand.
3. Trigger channel manager publish.
4. Assert subscriber receives expected payload.
```

Also document:

```text
Redis pub/sub is not durable. If subscriber is disconnected, messages are lost.
```

---

### 10.8 Redis Disconnect Test

Goal:

> Redis outage must not crash message processing unexpectedly.

Test:

```text
1. Make Redis unavailable.
2. Process RabbitMQ message that publishes to Redis.
3. Assert explicit behavior:
   - retry RabbitMQ message, or
   - ack and log Redis failure, or
   - fallback to durable storage.
4. Assert no silent hang.
```

---

### 10.9 WebSocket Reconnect Test

Goal:

> WebSocket reconnects after close and resumes subscriptions.

Test:

```text
1. Start fake WebSocket server.
2. Connect service.
3. Close socket from server side.
4. Assert reconnect backoff starts.
5. Bring server back.
6. Assert service reconnects.
7. Assert subscriptions are restored.
```

---

### 10.10 WebSocket Heartbeat Test

Goal:

> Missed pong terminates stale socket and starts reconnect.

Test:

```text
1. Start fake WebSocket server.
2. Allow initial ping/pong.
3. Stop responding to ping.
4. Assert socket is terminated after missed pong threshold.
5. Assert reconnect loop starts.
```

---

### 10.11 Duplicate WebSocket Listener Test

Goal:

> Prevent more than one WebSocket listener per Redis ownership record.

Test:

```text
1. Start two service instances.
2. Both try to claim same ownership key.
3. Assert only one owns the key.
4. Assert only one WebSocket listener starts.
5. Kill owner.
6. Wait for TTL expiry.
7. Assert second instance reclaims ownership.
```

---

### 10.12 Load And Soak Test

Goal:

> Verify long-running stability.

Run for 30–60 minutes with multiple accounts and streams.

Track:

* Memory growth.
* Reconnection rates.
* Log volume.
* RabbitMQ queue depth.
* Redis reconnects.
* WebSocket reconnects.
* Message processing latency.
* DLQ size.
* CPU usage.

---

## 11. Metrics And Alerts

Expose counters and gauges for every critical runtime component.

### 11.1 RabbitMQ Metrics

```text
rabbitmq_connection_status
rabbitmq_reconnect_attempts_total
rabbitmq_channel_closed_total
rabbitmq_publisher_status
rabbitmq_publisher_reconnect_attempts_total
rabbitmq_publisher_reconnect_failures_total
rabbitmq_publisher_last_reconnect_success_timestamp
rabbitmq_publish_failures_total
rabbitmq_consumer_active_count
rabbitmq_expected_consumer_count
rabbitmq_message_handler_errors_total
rabbitmq_message_retries_total
rabbitmq_dead_lettered_total
rabbitmq_queue_depth
```

Alert examples:

```text
required consumer count < expected consumer count for 1 minute
required publisher stuck reconnecting beyond recovery window
publish failures > 0 in 5 minutes
DLQ depth > 0
queue depth grows continuously for 10 minutes
```

---

### 11.2 Redis Metrics

```text
redis_connection_status
redis_reconnect_attempts_total
redis_publish_failures_total
redis_pubsub_messages_published_total
redis_pubsub_messages_received_total
redis_ownership_renew_failures_total
redis_ownership_lost_total
redis_ownership_reclaims_total
```

Alert examples:

```text
Redis required client disconnected for > 30 seconds
ownership renew failures > threshold
ownership reclaim loop failing
```

---

### 11.3 WebSocket Metrics

```text
websocket_connection_status
websocket_active
websocket_reconnect_attempts_total
websocket_missed_pongs_total
websocket_messages_received_total
websocket_last_message_timestamp
websocket_last_pong_timestamp
websocket_ownership_status
websocket_duplicate_listener_prevented_total
listen_key_expires_timestamp
listen_key_refresh_success_total
listen_key_refresh_failures_total
```

Alert examples:

```text
No messages received for expected active stream in N minutes
Required market-data stream stale beyond freshness window
Required quiet account stream heartbeat or listen key stale
listen key refresh failures exceed threshold
reconnect attempts exceed threshold
```

---

### 11.4 Application Metrics

```text
app_ready_status
app_lifecycle_state
background_timer_last_run_timestamp
background_timer_failures_total
handler_duration_ms
inflight_messages_count
```

---

## 12. Failure Mode Checklist

Every service should define behavior for these scenarios:

### RabbitMQ

* RabbitMQ unavailable at startup.
* RabbitMQ restarts after startup.
* Publisher channel closes.
* Consumer channel closes.
* Queue declaration mismatch.
* Message arrives before publisher init.
* Broker restarts while message is being processed.
* Handler throws before ack/nack.
* Handler timeout with prefetch `1`.
* Poison message retry loop.
* Dead-letter publish fails.
* Huge backlog on deploy.
* Multiple replicas consume same queue.

### Redis

* Redis unavailable at startup.
* Redis disconnects during message handling.
* Redis pub/sub subscriber is down.
* Redis ownership key expires.
* Redis ownership renew fails.
* Redis ownership value changes while WebSocket is running.
* Ownership release races with another instance claim.
* Redis reconnect causes duplicate subscriptions.
* Redis command hangs.

### WebSockets

* Socket closes from server side.
* Network drops but socket does not close.
* Pong is missed.
* Reconnect exceeds backoff cap.
* Listen key refresh fails.
* Quiet account stream receives no business messages but heartbeat is healthy.
* Replacement socket opens before old socket stops.
* Two instances race to open the same ownership-protected stream.
* Exchange rate-limits connections.
* Duplicate stream listener starts.
* Subscription restore fails after reconnect.

### Databases And APIs

* MongoDB disconnects during handler.
* MongoDB unavailable at startup.
* Exchange API timeout.
* Exchange API returns invalid payload.
* Exchange API returns empty precision data.
* Downstream service returns partial data.

### Kubernetes

* Pod receives SIGTERM while processing message.
* Pod is `Running` but readiness should be false.
* Pod is killed during in-flight work.
* Multiple replicas start simultaneously.
* Deployment creates backlog pressure.

---

## 13. Service Readiness Contract

Every microservice should document its readiness contract.

Example:

```md
## Readiness Contract

This service is ready only when:

- MongoDB main connection is connected.
- RabbitMQ connection is open.
- RabbitMQ publisher `exchangeStreamCommands` is initialized.
- Consumer `trade_station_new_tg_trade` is active.
- Consumer `satoshi_trade_station_update` is active.
- Redis publisher is connected.
- Required timers are running.
```

Example JSON readiness response:

```json
{
  "status": "not_ready",
  "components": [
    {
      "name": "rabbitmq.connection",
      "required": true,
      "status": "ready"
    },
    {
      "name": "rabbitmq.consumer.trade_station_new_tg_trade",
      "required": true,
      "status": "failed",
      "lastError": "Consumer tag missing after reconnect"
    },
    {
      "name": "redis.publisher",
      "required": true,
      "status": "ready"
    }
  ]
}
```

---

## 14. Implementation Rules For New Microservices

Every new microservice must follow these rules:

### Startup

* Use a single lifecycle manager.
* Initialize dependencies before starting consumers.
* Await publisher initialization.
* Assert broker contracts at startup.
* Mark readiness only after runtime work is active.

### RabbitMQ

* Use reconnectable connection manager.
* Recreate channels after reconnect.
* Re-register consumers after reconnect.
* Track expected vs active consumers.
* Define ack/nack behavior for every handler.
* Use DLQ or retry cap for poison messages.

### Redis

* Separate publisher/subscriber/command clients.
* Treat pub/sub as lossy.
* Use Streams or RabbitMQ for durable events.
* Define behavior when Redis publish fails.
* Use TTL ownership for singleton runtime resources.

### WebSockets

* Use heartbeat ping/pong.
* Use reconnect backoff with jitter.
* Restore subscriptions after reconnect.
* Prevent duplicate listeners.
* Stop reconnect loops during shutdown.

### Health

* Provide `/health/live`.
* Provide `/health/ready`.
* Include required runtime components in readiness.
* Fail readiness when consumers detach.
* Fail readiness when required publishers are not initialized.

### Testing

* Add integration tests for broker restarts.
* Add startup order tests.
* Add backlog-on-startup tests.
* Add handler error ack/nack tests.
* Add Redis disconnect tests.
* Add WebSocket reconnect and heartbeat tests.
* Add queue contract tests.

### Observability

* Expose metrics for reconnects, failures, queue depth, consumers, timers, and heartbeats.
* Alert on consumer count mismatch.
* Alert on DLQ growth.
* Alert on repeated reconnects.
* Alert on stale WebSocket streams.

---

## 15. Recommended Pull Request Checklist

Before merging a service that uses RabbitMQ, Redis, WebSockets, or long-running external connections, verify:

```text
[ ] Startup order prevents consumers from running before publishers are ready.
[ ] RabbitMQ connection reconnects automatically.
[ ] RabbitMQ channels are recreated after reconnect.
[ ] Consumers are re-registered after reconnect.
[ ] Expected consumers are tracked in health status.
[ ] Publisher readiness is tracked.
[ ] Required publishers proactively reconnect after channel/connection close.
[ ] Readiness can recover after RabbitMQ returns without waiting for new traffic.
[ ] Handler ack/nack behavior is explicit.
[ ] Poison messages cannot retry forever.
[ ] Queue contracts are centralized.
[ ] Redis pub/sub loss behavior is documented.
[ ] Redis disconnect behavior is explicit.
[ ] Redis ownership keys use TTL where needed.
[ ] WebSockets use heartbeat ping/pong.
[ ] WebSockets use capped reconnect backoff.
[ ] WebSocket subscriptions are restored after reconnect.
[ ] Duplicate WebSocket listeners are prevented.
[ ] Timers/background loops expose last-run status.
[ ] `/health/live` exists.
[ ] `/health/ready` validates functional dependencies.
[ ] Kubernetes probes use the correct endpoints.
[ ] Graceful shutdown stops consumers and reconnect loops.
[ ] Integration tests cover broker/service restarts.
[ ] Metrics and alerts exist for critical failure modes.
```

---

## 16. Minimal TypeScript Interfaces

### Component Health Registry

```ts
export type ComponentState =
  | 'starting'
  | 'ready'
  | 'degraded'
  | 'reconnecting'
  | 'failed'
  | 'stopped';

export type ComponentHealth = {
  name: string;
  required: boolean;
  state: ComponentState;
  lastOkAt?: number;
  lastErrorAt?: number;
  lastError?: string;
  details?: Record<string, unknown>;
};

export class HealthRegistry {
  private components = new Map<string, ComponentHealth>();

  set(component: ComponentHealth) {
    this.components.set(component.name, component);
  }

  markReady(name: string, details?: Record<string, unknown>) {
    this.components.set(name, {
      ...this.components.get(name),
      name,
      required: this.components.get(name)?.required ?? true,
      state: 'ready',
      lastOkAt: Date.now(),
      details,
    });
  }

  markFailed(name: string, error: unknown) {
    const previous = this.components.get(name);

    this.components.set(name, {
      name,
      required: previous?.required ?? true,
      state: 'failed',
      lastErrorAt: Date.now(),
      lastError: error instanceof Error ? error.message : String(error),
      details: previous?.details,
    });
  }

  isReady() {
    return [...this.components.values()].every(component => {
      if (!component.required) return true;
      return component.state === 'ready';
    });
  }

  snapshot() {
    return {
      status: this.isReady() ? 'ready' : 'not_ready',
      components: [...this.components.values()],
    };
  }
}
```

### Consumer Registry

```ts
export type ConsumerStatus = {
  name: string;
  queue: string;
  required: boolean;
  active: boolean;
  consumerTag?: string;
  lastMessageAt?: number;
  lastError?: string;
};

export class ConsumerRegistry {
  private consumers = new Map<string, ConsumerStatus>();

  registerExpected(name: string, queue: string, required = true) {
    this.consumers.set(name, {
      name,
      queue,
      required,
      active: false,
    });
  }

  markActive(name: string, consumerTag: string) {
    const current = this.consumers.get(name);
    if (!current) throw new Error(`Unknown consumer: ${name}`);

    this.consumers.set(name, {
      ...current,
      active: true,
      consumerTag,
    });
  }

  markInactive(name: string, error?: unknown) {
    const current = this.consumers.get(name);
    if (!current) return;

    this.consumers.set(name, {
      ...current,
      active: false,
      consumerTag: undefined,
      lastError: error instanceof Error ? error.message : String(error ?? ''),
    });
  }

  requiredConsumersHealthy() {
    return [...this.consumers.values()].every(consumer => {
      if (!consumer.required) return true;
      return consumer.active && Boolean(consumer.consumerTag);
    });
  }

  snapshot() {
    return [...this.consumers.values()];
  }
}
```

---

## 17. Final Rule

The service is robust only when this is true:

```text
External dependency disappears
-> service detects it
-> service marks itself degraded/not ready
-> service retries safely or fails explicitly
-> service recovers automatically when dependency returns
-> no messages are silently lost unless loss is documented and acceptable
-> no pod stays Running while functionally dead
```

That is the baseline resilience contract for every broker-connected or stream-connected microservice.

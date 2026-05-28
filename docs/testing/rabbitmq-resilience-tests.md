# RabbitMQ Resilience Tests

## Purpose

RabbitMQ resilience tests prove that services can consume, publish, detect
outages, and recover correctly when broker connections or channels fail.

## Required Scenarios

- broker unavailable at startup
- broker restart while service is running
- channel closed by broker
- queue or exchange missing when declarations are required
- consumer cancellation
- publisher connection or channel close without a following publish
- broker still booting during the first publisher reconnect attempt
- publisher confirm timeout or negative acknowledgement
- poison message routing
- dead-letter path behavior
- backpressure when processing is slow

## Assertions

Tests should assert:

- readiness is false while required broker behavior is unavailable
- consumers resume after broker or channel recovery
- required publishers reconnect proactively after connection or channel close
- readiness can recover after broker restart without waiting for new publish
- publisher reconnect attempt metrics increase on real attempts and stop
  increasing after successful recovery
- publishers retry, fail, or persist messages according to the service contract
- messages are acknowledged only after successful processing
- poison messages do not block the queue indefinitely
- metrics include reconnects, consumer state, publish failures, processing
  latency, retry count, and dead-letter count

## Recovery Rules

After RabbitMQ recovery, the service must re-establish:

- TCP connection
- AMQP connection
- channels
- queue declarations
- exchange declarations
- bindings
- consumers
- publisher confirms when required
- publisher channels and close handlers

Readiness may become true only after required consumers and publishers are
functionally restored.

## Publisher Reconnect Regression

Required publishers must not reconnect only as a side effect of `publish`.

Regression test:

```text
1. Start the service with RabbitMQ available.
2. Assert readiness is healthy.
3. Stop or restart RabbitMQ.
4. Assert readiness is unhealthy.
5. Start RabbitMQ again.
6. Do not publish any message.
7. Wait for the publisher reconnect loop.
8. Assert readiness becomes healthy.
```

This prevents the failure where consumers recover but `/health/ready` remains
`503` because the publisher is stuck in `reconnecting` until the next publish.

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
- publisher confirm timeout or negative acknowledgement
- poison message routing
- dead-letter path behavior
- backpressure when processing is slow

## Assertions

Tests should assert:

- readiness is false while required broker behavior is unavailable
- consumers resume after broker or channel recovery
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

Readiness may become true only after required consumers and publishers are
functionally restored.

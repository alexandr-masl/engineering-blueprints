# Coding Agent Rules

## Core Rules

- Read the service README and readiness contract before changing runtime
  lifecycle code.
- Classify each dependency as required or optional.
- Do not implement shallow readiness checks for services with runtime
  dependencies.
- Do not add unbounded in-memory queues for retry or backpressure.
- Preserve existing service patterns unless they conflict with documented
  resilience requirements.
- Add focused tests for behavior changed by the task.

## Required Reasoning

Before implementing dependency, consumer, publisher, WebSocket, or background
loop code, identify:

- startup behavior
- outage behavior
- recovery behavior
- shutdown behavior
- health check impact
- metrics and log impact

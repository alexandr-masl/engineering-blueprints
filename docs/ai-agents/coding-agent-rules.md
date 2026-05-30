# Coding Agent Rules

## Core Rules

- Read the service README and readiness contract before changing runtime
  lifecycle code.
- Classify each dependency as required or optional.
- Do not implement shallow readiness checks for services with runtime
  dependencies.
- For WebSocket streams, identify whether the stream is market-data or quiet
  account/user-data before choosing readiness freshness signals.
- Do not open singleton WebSocket streams before Redis ownership is claimed.
- Do not implement Redis ownership renewal or release with separate `GET` then
  `EXPIRE` or `GET` then `DEL` calls.
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
- ownership and duplicate-listener behavior
- listen-key or session-key lifecycle
- shutdown behavior
- health check impact
- metrics and log impact

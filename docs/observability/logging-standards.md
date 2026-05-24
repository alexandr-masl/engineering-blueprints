# Logging Standards

## Purpose

Logs should explain service behavior during startup, runtime dependency changes,
failure, recovery, and shutdown.

## Required Log Events

Log structured events for:

- configuration validation failure
- startup begin and complete
- readiness state changes
- dependency connection loss and recovery
- broker consumer start, stop, cancellation, and recovery
- publish failure after retry exhaustion
- Redis reconnect and subscription restoration
- WebSocket reconnect and stale data detection
- background loop failure and recovery
- graceful shutdown begin and complete

## Field Guidance

Use consistent fields:

- `service`
- `component`
- `dependency`
- `operation`
- `status`
- `reason`
- `duration_ms`
- `attempt`
- `correlation_id` or `trace_id`

Never log secrets, tokens, passwords, private keys, or full sensitive payloads.

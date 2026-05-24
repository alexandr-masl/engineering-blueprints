# Metrics and Alerts

## Purpose

Metrics and alerts make dependency failures, stale runtime behavior, and service
degradation visible before users report incidents.

## Required Metric Areas

- process uptime and restarts
- readiness and liveness state
- dependency connection state
- broker consumer state
- publish successes and failures
- message processing latency
- retry counts
- queue depth or lag
- Redis errors and latency
- WebSocket connection count and reconnect count
- WebSocket message freshness
- background loop success and failure timestamps
- request rate, error rate, and latency

## Alert Guidance

Alert on sustained user-impacting or service-contract failures:

- readiness false for longer than the recovery window
- repeated dependency reconnects
- dead required consumer
- stale required WebSocket feed
- queue depth or lag above threshold
- background loop not successful within expected interval
- publish failures above threshold
- error rate or latency breach

Avoid alerts that fire on single transient failures when retry and recovery are
working as designed.

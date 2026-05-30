# Redis Resilience Tests

## Purpose

Redis resilience tests verify safe behavior when Redis is used as required state,
optional cache, pub/sub transport, locking backend, or coordination store.

## Required Scenarios

- Redis unavailable at startup
- Redis restart during request handling
- Redis pub/sub connection interruption
- stale subscription after reconnect
- lock acquisition failure
- TTL ownership renew failure
- ownership value changed by another instance
- ownership release race
- cache miss during Redis outage
- timeout under high latency

## Assertions

Tests should assert:

- readiness reflects required Redis availability
- optional cache failure does not corrupt primary state
- pub/sub subscriptions are restored after reconnect
- stale subscription state is detected
- lock failures do not create duplicate unsafe work
- ownership renew uses atomic compare-and-expire
- ownership release uses atomic compare-and-delete
- ownership loss stops the owned runtime component
- Redis timeouts are bounded
- metrics and logs identify Redis state transitions

## Classification

Before implementing tests, classify Redis usage:

- required request-path dependency
- optional cache
- pub/sub transport
- lock provider
- TTL ownership provider
- rate limit backend
- coordination state

The classification determines whether Redis outages should fail readiness or
trigger graceful degradation.

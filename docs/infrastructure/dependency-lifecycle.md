# Dependency Lifecycle

## Purpose

Dependency lifecycle standards define how services connect, monitor, recover,
and shut down external dependencies.

## Required Lifecycle States

Track dependency state explicitly:

- uninitialized
- connecting
- connected
- degraded
- reconnecting
- unavailable
- closing
- closed

## Implementation Rules

- validate configuration before opening connections
- use bounded retry with backoff and jitter
- expose current state through readiness and metrics
- recreate dependent channels, subscriptions, and consumers after reconnect
- avoid unbounded queues during outages
- distinguish required and optional dependencies
- close connections during graceful shutdown

## Recovery Rule

After a dependency reconnects, the service must re-establish all runtime objects
created on top of that dependency before reporting ready.

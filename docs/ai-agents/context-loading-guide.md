# AI Agent Context Loading Guide

When working on a microservice that uses RabbitMQ, Redis, WebSockets, MongoDB,
PostgreSQL, external APIs, timers, or background loops, load these documents
first:

1. [Microservice Resilience Blueprint](../architecture/microservice-resilience-blueprint.md)
2. [Microservice Readiness Checklist](../checklists/microservice-readiness-checklist.md)
3. [Integration Testing Blueprint](../testing/integration-testing-blueprint.md)
4. Service-specific README or readiness contract

The agent must not implement broker consumers, publishers, WebSocket managers,
runtime subscriptions, background loops, or health checks without following
these standards.

## Context Loading Order

1. Read the service README.
2. Read the service readiness contract if present.
3. Load the architecture blueprint.
4. Load the relevant testing blueprint for each dependency.
5. Inspect existing health, lifecycle, and observability code.
6. Implement the smallest change that satisfies the contract.
7. Add or update tests that prove startup, outage, recovery, and shutdown
   behavior where applicable.

# Engineering Blueprints

This repository contains reusable engineering standards, architecture blueprints,
testing strategies, production-readiness checklists, and AI-agent guidance for
building robust applications and microservices.

The goal is to provide a shared knowledge base for developers and AI coding
agents so that critical application behavior is implemented consistently across
services.

## Purpose

This repo defines reusable technical standards for production-grade
applications.

It is intended for:

- human developers
- AI coding agents
- code reviewers
- architecture planning
- testing strategy design
- production readiness checks

## Main Documents

### Architecture

- [Microservice Resilience Blueprint](docs/architecture/microservice-resilience-blueprint.md)
- [WebSocket Runtime Ownership Blueprint](docs/architecture/websocket-runtime-ownership-blueprint.md)

### Testing

- [Integration Testing Blueprint](docs/testing/integration-testing-blueprint.md)
- [RabbitMQ Resilience Tests](docs/testing/rabbitmq-resilience-tests.md)
- [Redis Resilience Tests](docs/testing/redis-resilience-tests.md)
- [WebSocket Resilience Tests](docs/testing/websocket-resilience-tests.md)
- [WebSocket Integration Test Blueprint](docs/testing/websocket-integration-test-blueprint.md)

### Observability

- [Health Checks](docs/observability/health-checks.md)
- [Metrics and Alerts](docs/observability/metrics-and-alerts.md)
- [Logging Standards](docs/observability/logging-standards.md)

### Infrastructure

- [Kubernetes Probes](docs/infrastructure/kubernetes-probes.md)
- [Graceful Shutdown](docs/infrastructure/graceful-shutdown.md)
- [Dependency Lifecycle](docs/infrastructure/dependency-lifecycle.md)

### AI Agents

- [Coding Agent Rules](docs/ai-agents/coding-agent-rules.md)
- [Review Agent Rules](docs/ai-agents/review-agent-rules.md)
- [Testing Agent Rules](docs/ai-agents/testing-agent-rules.md)
- [Context Loading Guide](docs/ai-agents/context-loading-guide.md)

### Checklists

- [Microservice Readiness Checklist](docs/checklists/microservice-readiness-checklist.md)
- [Production Release Checklist](docs/checklists/production-release-checklist.md)
- [Pull Request Checklist](docs/checklists/pull-request-checklist.md)

## Core Rule

A service is healthy only when its required runtime dependencies, consumers,
publishers, subscriptions, timers, and background loops are functionally alive.

## Repository Map

- `docs/architecture/`: system-level design rules and blueprints
- `docs/testing/`: integration, resilience, and failure-mode test guidance
- `docs/observability/`: health, metrics, alerting, and logging standards
- `docs/infrastructure/`: runtime, orchestration, and lifecycle standards
- `docs/ai-agents/`: instructions for AI-assisted implementation and review
- `docs/checklists/`: readiness, release, and pull request checks
- `templates/`: reusable document templates
- `examples/`: applied examples for specific service types

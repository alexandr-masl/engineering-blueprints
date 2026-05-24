# Trading Station Resilience Test Plan

## Required Tests

- starts with RabbitMQ, Redis, and market data WebSocket available
- reports not ready when RabbitMQ is unavailable
- resumes consumers after RabbitMQ restart
- reports not ready when Redis pub/sub is unavailable
- restores Redis subscriptions after reconnect
- reports not ready when market data becomes stale
- resubscribes to market data after WebSocket reconnect
- drains in-flight command handling during shutdown

## Evidence

Tests should assert readiness endpoint state, consumed messages, published
events, Redis subscription effects, WebSocket freshness, metrics, and structured
logs.

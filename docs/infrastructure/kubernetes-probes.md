# Kubernetes Probes

## Purpose

Kubernetes probes must represent the correct service lifecycle signal.

## Probe Types

### Liveness Probe

Use liveness to detect a stuck or unrecoverable process.

Do not fail liveness for normal recoverable dependency outages.

### Readiness Probe

Use readiness to control traffic and work assignment.

Readiness should fail when required dependencies, consumers, subscriptions, or
background loops are not functionally available.

### Startup Probe

Use startup probes for services with slow initialization, dependency warmup, or
schema compatibility checks.

## Probe Design Rules

- readiness must become false before graceful shutdown closes dependencies
- liveness should be conservative to avoid restart loops
- startup probe should cover expected initialization time
- probe timeouts should be shorter than application-level request timeouts
- probe endpoints must not perform expensive operations

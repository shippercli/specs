# Spec: Queue Workers

**Issue:** MAR-24
**Date:** 2026-04-24
**Status:** In Review

## Overview

Queue workers run Laravel queue jobs on deployed servers. Shipper configures workers via `QueueConfig` and delegates to provider-specific implementations (Ploi, Forge). Workers are named entries in the project config, each defining connection, queue, timeout, and process settings.

## Current Implementation

### Key Classes / Files

| Class/File | Role |
|---|---|
| `App\Config\QueueConfig` | Value object holding worker config (connection, queue, timeouts, processes) |
| `App\Deployment\Contracts\QueueManagerInterface` | Interface defining `plan()` and `apply()` |
| `App\Deployment\Providers\Ploi\PloiQueueManager` | Creates workers via Ploi API |
| `App\Deployment\Providers\Forge\ForgeQueueManager` | Creates workers via Forge API |
| `tests/Unit/Config/QueueConfigTest.php` | Unit tests for config defaults and getters |

## Functional Requirements

**FR-001 — Queue Config Defaults**
`QueueConfig` must default to `database` connection, `default` queue, 60s maxSeconds, 30s sleep, 1 process, 1 maxTry when no arguments are provided.

**FR-002 — Named Queue Entries**
A project exposes multiple named queue workers via `ProjectConfig::queues()`, returning `array<string, QueueConfig>`. Each entry is a distinct worker with its own settings.

**FR-003 — Plan Output**
`plan()` returns a human-readable list of actions, one per queue worker, describing connection:queue and process count.

**FR-004 — Apply via Provider**
`apply()` iterates over the provided `array<string, QueueConfig>` and calls the provider API to create each worker. Failures abort with `OperationResult::fail()`.

**FR-005 — Daemon Mode**
Forge workers are created with `daemon: true` to run continuously.

## Configuration Interface

```yaml
queues:
  horizon:
    connection: redis
    queue: horizon
    max_seconds: 90
    sleep: 30
    processes: 3
    max_tries: 3
  emails:
    connection: redis
    queue: emails
    max_seconds: 60
    sleep: 10
    processes: 2
    max_tries: 5
```

## Data Contracts

```php
// QueueConfig
final class QueueConfig {
    public function __construct(
        private readonly string $connection = 'database',
        private readonly string $queue = 'default',
        private readonly int $maxSeconds = 60,
        private readonly int $sleep = 30,
        private readonly int $processes = 1,
        private readonly int $maxTries = 1,
    ) {}

    public function connection(): string { ... }
    public function queue(): string { ... }
    public function maxSeconds(): int { ... }
    public function sleep(): int { ... }
    public function processes(): int { ... }
    public function maxTries(): int { ... }
}

// QueueManagerInterface
interface QueueManagerInterface {
    /** @return array<string> */
    public function plan(DeploymentContext $context): array;

    /** @param array<string, QueueConfig> $queues */
    public function apply(SiteContext $site, array $queues): OperationResult;
}
```

## Edge Cases

- **Empty queues config:** `plan()` returns an empty list; `apply()` succeeds with no-op.
- **API failure on one worker:** Returns `OperationResult::fail()` immediately with the worker name in the message.
- **Missing optional args:** All config values have defaults; no special handling required.
- **processes = 0:** Not validated at config level; provider API may reject.

## Acceptance Criteria

- [ ] `QueueConfig` defaults produce the same values as the existing test `'QueueConfig uses defaults when no arguments provided'`
- [ ] `plan()` output format matches `"Create queue worker: {$queueName} ({$queue->connection()}:{$queue->queue()}, {$queue->processes()} processes)"`
- [ ] Forge `apply()` passes `daemon: true` to `createWorker()`
- [ ] Both providers handle API exceptions and return `OperationResult::fail()` with descriptive message
- [ ] Unit tests in `QueueConfigTest.php` continue to pass

## Open Questions / Potential Concerns

- Should there be a maximum cap on `processes` per worker?
- Is `database` as the default connection appropriate for all provider scenarios, or should it be provider-specific?

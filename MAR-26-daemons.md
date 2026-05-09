# Spec: Daemons

**Issue:** MAR-26
**Date:** 2026-04-24
**Status:** In Review

## Overview

Daemons run persistent background processes on deployed servers (e.g., `php artisan horizon`, websocket servers). Shipper configures them via `DaemonConfig` and delegates to provider-specific implementations (Ploi, Forge). Each daemon entry specifies a command, user, process count, and optional working directory.

## Current Implementation

### Key Classes / Files

| Class/File | Role |
|---|---|
| `App\Config\DaemonConfig` | Value object holding daemon config (command, user, processes, directory) |
| `App\Deployment\Contracts\DaemonManagerInterface` | Interface defining `plan()` and `apply()` |
| `App\Deployment\Providers\Ploi\PloiDaemonManager` | Creates daemons via Ploi API |
| `App\Deployment\Providers\Forge\ForgeDaemonManager` | Creates daemons via Forge API |
| `tests/Unit/Config/DaemonConfigTest.php` | Unit tests for config and getters |

## Functional Requirements

**FR-001 — Daemon Config Defaults**
`DaemonConfig` requires `command` as the only positional argument. `user` defaults to `'ploi'`, `processes` defaults to `1`, and `directory` defaults to an empty string.

**FR-002 — Named Daemon Entries**
A project exposes multiple named daemons via `ProjectConfig::daemons()`, returning `array<string, DaemonConfig>`. Each entry is a distinct persistent process.

**FR-003 — Plan Output**
`plan()` returns a list of actions, one per daemon, describing the command and process count.

**FR-004 — Apply via Provider**
`apply()` iterates over `array<string, DaemonConfig>` and calls the provider API to create each daemon. The `{site}` placeholder in `directory` is replaced with the site domain before sending.

**FR-005 — Directory Interpolation**
When `directory` is non-empty, `{site}` is interpolated with the actual domain. When empty, `null` is passed to the provider API.

## Configuration Interface

```yaml
daemons:
  horizon:
    command: php artisan horizon
    user: ploi
    processes: 1
    directory: ""
  websockets:
    command: php artisan websockets:serve
    user: www-data
    processes: 2
    directory: /home/ploi/mysite
```

## Data Contracts

```php
// DaemonConfig
final class DaemonConfig {
    public function __construct(
        private readonly string $command,
        private readonly string $user = 'ploi',
        private readonly int $processes = 1,
        private readonly string $directory = '',
    ) {}

    public function command(): string { ... }
    public function user(): string { ... }
    public function processes(): int { ... }
    public function directory(): string { ... }
}

// DaemonManagerInterface
interface DaemonManagerInterface {
    /** @return array<string> */
    public function plan(DeploymentContext $context): array;

    /** @param array<string, DaemonConfig> $daemons */
    public function apply(SiteContext $site, array $daemons): OperationResult;
}
```

## Edge Cases

- **Empty daemons config:** `plan()` returns an empty list; `apply()` succeeds with no-op.
- **API failure on one daemon:** Returns `OperationResult::fail()` immediately with the daemon name in the message.
- **Empty directory:** Passes `null` to provider API (Ploi: non-empty string required, Forge: nullable).
- **`{site}` in directory:** Interpolated with `$site->domain` before API call.
- **processes = 0:** Not validated; provider API may reject.

## Acceptance Criteria

- [ ] `DaemonConfig` defaults produce `user = 'ploi'`, `processes = 1`, `directory = ''` when only command is provided
- [ ] `plan()` output format matches `"Create daemon: {$daemonName} ({$daemon->command()}, {$daemon->processes()} processes)"`
- [ ] `{site}` in directory is replaced with `$site->domain` before API call
- [ ] Empty directory results in `null` being passed to provider (via conditional `??` or ternary)
- [ ] Both providers handle API exceptions and return `OperationResult::fail()` with descriptive message
- [ ] Unit tests in `DaemonConfigTest.php` continue to pass

## Open Questions / Potential Concerns

- Should there be a validation on `processes` to prevent accidentally spawning too many?
- Should daemons be restartable, or only created once at deploy time?
- Is the Forge `directory` nullable behavior intentional and consistent with Ploi?

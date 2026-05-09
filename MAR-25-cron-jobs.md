# Spec: Cron Jobs

**Issue:** MAR-25
**Date:** 2026-04-24
**Status:** In Review

## Overview

Cron jobs schedule recurring tasks on deployed servers. Shipper configures them via `CronConfig` and delegates to provider-specific implementations (Ploi, Forge). Each entry maps a human-readable name to a command, frequency expression, and system user.

## Current Implementation

### Key Classes / Files

| Class/File | Role |
|---|---|
| `App\Config\CronConfig` | Value object holding cron config (command, frequency, user) |
| `App\Deployment\Contracts\CronManagerInterface` | Interface defining `plan()` and `apply()` |
| `App\Deployment\Providers\Ploi\PloiCronManager` | Creates cron jobs via Ploi API |
| `App\Deployment\Providers\Forge\ForgeCronManager` | Creates cron jobs via Forge API |
| `tests/Unit/Config/CronConfigTest.php` | Unit tests for config and getters |

## Functional Requirements

**FR-001 — Cron Config with Defaults**
`CronConfig` requires `command` and `frequency` as positional arguments. The `user` defaults to `'ploi'`.

**FR-002 — Named Cron Entries**
A project exposes multiple named cron jobs via `ProjectConfig::cron()`, returning `array<string, CronConfig>`. Each entry is a distinct scheduled task.

**FR-003 — Plan Output**
`plan()` returns a list of actions, one per cron job, describing the frequency.

**FR-004 — Apply via Provider**
`apply()` iterates over `array<string, CronConfig>` and calls the provider API to create each cron job. The `{site}` placeholder in the command is replaced with the actual site domain before sending to the API.

**FR-005 — User Context**
Cron jobs run under a configurable system user (defaults to `ploi`). Both providers accept a `user` parameter.

## Configuration Interface

```yaml
cron:
  scheduler:
    command: php artisan schedule:run
    frequency: "* * * * *"
    user: ploi
  backup:
    command: php artisan backup:run
    frequency: "0 2 * * *"
    user: www-data
  reports:
    command: php artisan reports:generate
    frequency: "0 9 * * 1"
    user: ploi
```

## Data Contracts

```php
// CronConfig
final class CronConfig {
    public function __construct(
        private readonly string $command,
        private readonly string $frequency,
        private readonly string $user = 'ploi',
    ) {}

    public function command(): string { ... }
    public function frequency(): string { ... }
    public function user(): string { ... }
}

// CronManagerInterface
interface CronManagerInterface {
    /** @return array<string> */
    public function plan(DeploymentContext $context): array;

    /** @param array<string, CronData> $cron */
    public function apply(SiteContext $site, array $cron): OperationResult;
}
```

## Edge Cases

- **Empty cron config:** `plan()` returns an empty list; `apply()` succeeds with no-op.
- **API failure on one cron:** Returns `OperationResult::fail()` immediately with the cron name in the message.
- **Invalid frequency expression:** Not validated at config level; passed as-is to provider API.
- **`{site}` placeholder not present:** `str_replace` leaves the command unchanged.
- **Root user cron:** Valid but potentially risky; no guard in place.

## Acceptance Criteria

- [ ] `CronConfig` defaults produce `user = 'ploi'` when only command and frequency are provided
- [ ] `plan()` output format matches `"Create cron job: {$cronName} ({$cronConfig->frequency()})"`
- [ ] `{site}` in command is replaced with `$site->domain` before calling provider API
- [ ] Both providers pass `user` correctly to their respective API methods
- [ ] Unit tests in `CronConfigTest.php` continue to pass

## Open Questions / Potential Concerns

- Should there be validation for frequency expression format?
- Should `root` be allowed as a cron user, or should that be blocked?
- Is there a limit on how many cron jobs a single site can register?

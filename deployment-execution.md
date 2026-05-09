# Spec: Deployment Execution

**Issue:** MAR-35
**Date:** 2026-04-23
**Status:** In Review

## Overview

Deployment execution in Shipper is the process of taking a validated deployment plan and applying it to the target infrastructure provider. This layer is responsible for coordinating site creation/lookup, deployment triggering, and log retrieval through provider-specific implementations. It sits between the Flow layer (which handles planning and orchestration) and the provider-specific adapters.

## Current Implementation

### Key Classes / Files

| File | Class | Responsibility |
|------|-------|----------------|
| `app/Actions/ExecuteDeploymentAction.php` | `ExecuteDeploymentAction` | Calls `$provider->apply()` to trigger a deployment |
| `app/Actions/GetDeploymentLogsAction.php` | `GetDeploymentLogsAction` | Fetches deployment logs via the provider's `DeploymentLogReaderInterface` |
| `app/Actions/CreateDeploymentPlanAction.php` | `CreateDeploymentPlanAction` | Calls `$provider->plan()` to produce a deployment plan array |
| `app/Deployment/Contracts/DeploymentExecutorInterface.php` | `DeploymentExecutorInterface` | Interface for deployment executors with `plan()` and `apply()` |
| `app/Deployment/Contracts/DeploymentLogReaderInterface.php` | `DeploymentLogReaderInterface` | Interface for reading deployment logs |
| `app/Deployment/OperationResult.php` | `OperationResult` | Value object: success + error string |
| `app/Deployment/SiteResult.php` | `SiteResult` | Value object: site creation/lookup result with siteId, isNew flag, and error |

## Functional Requirements

**FR-001 — Execute Deployment**
**As** the ApplyDeploymentFlow, **I want** to trigger a deployment via the provider **so that** the site is provisioned and the application is deployed.
- Acceptance: `ExecuteDeploymentAction::handle()` calls `$provider->apply()` and returns a boolean indicating success or failure.

**FR-002 — Retrieve Deployment Logs**
**As** the ApplyDeploymentFlow, **I want** to fetch deployment logs after execution **so that** I can present the output to the user.
- Acceptance: `GetDeploymentLogsAction::handle()` returns an array of log strings from the provider's log reader. Returns an empty array if no log reader is available or if siteId is 0.

**FR-003 — Create Deployment Plan**
**As** a Flow, **I want** to generate a deployment plan from a provider **so that** I can display the planned actions and pass the plan to execution.
- Acceptance: `CreateDeploymentPlanAction::handle()` delegates to `$provider->plan($project, $profile)` and returns the raw plan array.

**FR-004 — Deployment Executor Contract**
**As** a provider implementer, **I want** a clear interface for deployment execution **so that** I can implement any provider consistently.
- Acceptance: `DeploymentExecutorInterface` defines `plan(DeploymentContext): array<string>` and `apply(SiteContext, DeploymentContext): OperationResult`.

**FR-005 — Log Reader Contract**
**As** a provider implementer, **I want** a clear interface for reading deployment logs **so that** I can provide log access regardless of the provider's underlying API.
- Acceptance: `DeploymentLogReaderInterface::getLogs(SiteContext): array<int, string>` returns log entries indexed by integer.

## Data Contracts

### `ExecuteDeploymentAction`

```php
final class ExecuteDeploymentAction
{
    public function handle(
        DeploymentProviderInterface $provider,
        ProjectConfig $project,
        ProfileConfig $profile,
    ): bool;
}
```

**Returns:** `true` on success, `false` on failure.

---

### `GetDeploymentLogsAction`

```php
final class GetDeploymentLogsAction
{
    /**
     * @return array<int, string>
     */
    public function handle(DeploymentProviderInterface $provider, int $serverId, int $siteId): array;
}
```

**Behavior:**
- Returns `[]` if `siteId === 0`
- Retrieves `DeploymentLogReaderInterface` via `$provider->getOperation()`
- Returns `[]` if the provider does not expose a log reader
- Otherwise returns the result of `$logReader->getLogs(new SiteContext($serverId, $siteId, ''))`

---

### `CreateDeploymentPlanAction`

```php
final class CreateDeploymentPlanAction
{
    /**
     * @return array<string, mixed>
     */
    public function handle(
        DeploymentProviderInterface $provider,
        ProjectConfig $project,
        ProfileConfig $profile,
    ): array;
}
```

**Returns:** The raw plan array from `$provider->plan($project, $profile)`.

---

### `DeploymentExecutorInterface`

```php
interface DeploymentExecutorInterface
{
    /**
     * @return array<string>
     */
    public function plan(DeploymentContext $context): array;

    public function apply(SiteContext $site, DeploymentContext $context): OperationResult;
}
```

---

### `DeploymentLogReaderInterface`

```php
interface DeploymentLogReaderInterface
{
    /**
     * @return array<int, string>
     */
    public function getLogs(SiteContext $site): array;
}
```

---

### `OperationResult`

```php
final class OperationResult
{
    private function __construct(
        public readonly bool $success,
        public readonly string $error,
    ) {}

    public static function ok(): self;
    public static function fail(string $error): self;
}
```

---

### `SiteResult`

```php
final class SiteResult
{
    private function __construct(
        public readonly bool $success,
        public readonly int $siteId,
        public readonly bool $isNew,
        public readonly string $error,
    ) {}

    public static function found(int $siteId): self;
    public static function created(int $siteId): self;
    public static function fail(string $error): self;
}
```

## Edge Cases

- **No log reader available:** `GetDeploymentLogsAction` returns an empty array. The caller handles this gracefully (logs array is always present in the result, even if empty).
- **siteId is 0:** `GetDeploymentLogsAction` returns an empty array immediately without calling the provider.
- **Provider `apply()` returns false:** `ExecuteDeploymentAction` returns `false`. The caller (ApplyDeploymentFlow) captures `$provider->getLastError()` to populate the error message.
- **Provider `plan()` returns unexpected structure:** The plan array is passed through as-is. No schema validation is performed at this layer — it is assumed to be validated at the Flow or Command layer.

## Acceptance Criteria

- [ ] `ExecuteDeploymentAction::handle()` delegates to `$provider->apply()` and returns its boolean result
- [ ] `GetDeploymentLogsAction::handle()` returns `[]` when siteId is 0 without calling the provider
- [ ] `GetDeploymentLogsAction::handle()` returns `[]` when provider has no `DeploymentLogReaderInterface` registered
- [ ] `GetDeploymentLogsAction::handle()` returns the log array from the provider's log reader when available
- [ ] `CreateDeploymentPlanAction::handle()` delegates to `$provider->plan()` and returns the plan array
- [ ] `OperationResult` has `ok()` and `fail()` factory methods and is used as the return type for `DeploymentExecutorInterface::apply()`
- [ ] `SiteResult` distinguishes between `found` (existing site) and `created` (new site) via `isNew` flag

## Open Questions / Potential Concerns

- **`DeploymentExecutorInterface` is defined but not used by the existing Actions.** The Actions use `DeploymentProviderInterface` directly rather than composing an executor. The executor interface may be provider-internal or planned for future extraction.
- **`GetDeploymentLogsAction` requires the log reader to be registered via `getOperation()`.** This is a dynamic resolution pattern. If a provider does not register a `DeploymentLogReaderInterface`, logs are silently unavailable — there is no warning or fallback behavior.
- **`SiteResult` is defined but not used in any Action.** It appears to be a planned return type for site creation/lookup operations that hasn't yet been wired into the execution flow.
- **No timeout or polling mechanism** is present in these Actions. `GetDeploymentLogsAction` fetches logs once; there is no support for streaming or tailing logs in real-time.
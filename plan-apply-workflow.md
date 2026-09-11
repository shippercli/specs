---
description: "**Issue:** MAR-36 **Date:** 2026-04-23 **Status:** In Review"
---

# Spec: Plan/Apply Workflow

**Issue:** MAR-36
**Date:** 2026-04-23
**Status:** In Review

## Overview

The Plan/Apply workflow is the core operational pattern in Shipper. It separates the intent (planning) from the action (execution), giving operators a safe dry-run phase before any infrastructure is modified. This pattern is implemented consistently across deployment, destruction, and configuration validation flows. Each flow follows a load -> validate -> plan -> execute pipeline with clear error reporting at each stage.

## Current Implementation

### Key Classes / Files

| File | Class | Responsibility |
|------|-------|----------------|
| `app/Flows/PlanDeploymentFlow.php` | `PlanDeploymentFlow` | Validates config and produces a deployment plan |
| `app/Flows/ApplyDeploymentFlow.php` | `ApplyDeploymentFlow` | Validates config, produces plan, and optionally executes deployment |
| `app/Flows/ValidateConfigurationFlow.php` | `ValidateConfigurationFlow` | Validates all projects and profiles in a config file |
| `app/Flows/DestroyDeploymentFlow.php` | `DestroyDeploymentFlow` | Validates config, produces destruction plan, and executes destruction |
| `app/Commands/Concerns/FormatsDeploymentPlan.php` | `FormatsDeploymentPlan` | Trait: safely renders plan values for command output |

## Functional Requirements

**FR-001 — Plan Deployment**
**As** an operator, **I want** to preview a deployment without executing it **so that** I can verify the plan is correct before committing.
- Acceptance: `PlanDeploymentFlow::handle()` returns a structured result containing the plan array, project and profile objects, success flag, errors list, and error message. It loads config, validates project/profile existence, validates configuration, and generates a plan.

**FR-002 — Apply Deployment**
**As** an operator, **I want** to execute a deployment after planning **so that** my site is actually provisioned.
- Acceptance: `ApplyDeploymentFlow` has a `handle()` method for planning (returns plan + objects) and an `execute()` method for running the deployment (returns success, logs, error message). The two-step design allows the command layer to show the plan and confirm before calling `execute()`.

**FR-003 — Destroy Deployment**
**As** an operator, **I want** to destroy a deployed site **so that** I can cleanly remove infrastructure.
- Acceptance: `DestroyDeploymentFlow` mirrors the apply pattern with `handle()` (plan) and `execute()` (destroy). It uses `DestroySiteAction` instead of `ExecuteDeploymentAction`.

**FR-004 — Validate Configuration**
**As** an operator, **I want** to validate an entire config file **so that** I can catch errors across all projects before attempting any deployment.
- Acceptance: `ValidateConfigurationFlow::handle()` iterates all projects and profiles, runs `ValidateProjectAction` for each combination, and returns a structured error map grouped by project and profile name.

**FR-005 — Confirm Before Execution**
**As** Shipper, **I want** to require operator confirmation before executing a destructive action **so that** accidental runs are prevented.
- Acceptance: Commands (`apply`, `destroy`) call `confirm()` unless `--force` is passed. Confirmation happens after the plan is displayed but before `execute()` is called.

## Flow Architecture

### Common Pipeline (PlanDeploymentFlow)

```
LoadConfiguration → ValidateProject → CreateDeploymentPlan → Return Result
```

### Apply Flow (ApplyDeploymentFlow)

```
handle(): LoadConfiguration → ValidateProject → CreateDeploymentPlan → Return (plan + objects for command layer)
execute(): ExecuteDeploymentAction → GetDeploymentLogs → Return (success, logs, error)
```

### Destroy Flow (DestroyDeploymentFlow)

```
handle(): LoadConfiguration → ValidateProject → CreateDeploymentPlan → Return (plan + objects for command layer)
execute(): DestroySiteAction → Return (success, error)
```

### Validate Flow (ValidateConfigurationFlow)

```
LoadConfiguration → For each project: For each profile: ValidateProject → Collect errors → Return (success, allErrors)
```

## Data Contracts

### PlanDeploymentFlow

```php
final class PlanDeploymentFlow
{
    /**
     * @return array{
     *   success: bool,
     *   project: ProjectConfig|null,
     *   profile: ProfileConfig|null,
     *   plan: array<string, mixed>,
     *   errors: array<int, string>,
     *   error_message: string
     * }
     */
    public function handle(string $configPath, string $projectName, string $profileName): array;
}
```

### ApplyDeploymentFlow

```php
final class ApplyDeploymentFlow
{
    public function __construct(
        private readonly ?\Closure $providerResolver = null,
    ) {}

    /**
     * @return array{
     *   success: bool,
     *   project: ProjectConfig|null,
     *   profile: ProfileConfig|null,
     *   plan: array<string, mixed>,
     *   errors: array<int, string>,
     *   error_message: string,
     *   provider: DeploymentProviderInterface|null
     * }
     */
    public function handle(string $configPath, string $projectName, string $profileName): array;

    /**
     * @param array<string, mixed> $plan
     * @return array{
     *   success: bool,
     *   logs: array<int, string>,
     *   error_message: string
     * }
     */
    public function execute(
        DeploymentProviderInterface $provider,
        ProjectConfig $project,
        ProfileConfig $profile,
        array $plan,
    ): array;
}
```

**Note:** The `providerResolver` closure allows test injection of a mock provider. When null, a `ProviderFactory` is used.

### DestroyDeploymentFlow

```php
final class DestroyDeploymentFlow
{
    public function __construct(
        private readonly ?\Closure $providerResolver = null,
    ) {}

    /**
     * @return array{
     *   success: bool,
     *   project: ProjectConfig|null,
     *   profile: ProfileConfig|null,
     *   plan: array<string, mixed>,
     *   errors: array<int, string>,
     *   error_message: string,
     *   provider: DeploymentProviderInterface|null
     * }
     */
    public function handle(string $configPath, string $projectName, string $profileName): array;

    /**
     * @return array{success: bool, error_message: string}
     */
    public function execute(
        DeploymentProviderInterface $provider,
        ProjectConfig $project,
        ProfileConfig $profile,
    ): array;
}
```

### ValidateConfigurationFlow

```php
final class ValidateConfigurationFlow
{
    /**
     * @return array{
     *   success: bool,
     *   errors: array<string, array<string, array<int, string>>>
     * }
     */
    public function handle(string $configPath): array;
}
```

**Return shape:** `errors[projectName][profileName][] = errorString`
Special key `_provider` is used for provider-level errors (e.g., missing API key).

### FormatsDeploymentPlan trait

```php
trait FormatsDeploymentPlan
{
    private function getPlanValue(array $plan, string $key, string $default = 'unknown'): string;
}
```

## Error Handling Patterns

All flows use a consistent result shape with `success: bool`, `errors: array`, and `error_message: string`. Missing project or profile returns `success: false` with a human-readable `error_message`. Validation failures return `success: false` with an `errors` array containing the validation messages.

## Edge Cases

- **Project not found:** Flows return `success: false` with `error_message: "Project not found: <name>"`. `project` is `null` in the result.
- **Profile not found:** Flows return `success: false` with `error_message: "Profile not found: <name>"`. `profile` is `null` in the result.
- **Configuration validation errors:** Flows return `success: false` with populated `errors` array and `error_message: "Configuration validation failed"`. `provider` may still be returned so the command layer can display provider-specific context.
- **Provider factory throws `InvalidArgumentException`:** Caught by the flow and stored under `projectErrors['_provider']`. This surfaces provider configuration issues (e.g., missing API key) at the flow level.
- **execute() called on a failed plan:** The command layer only calls `execute()` after confirming `success: true` from `handle()`. No explicit guard exists in the flow itself.
- **No log reader available:** `execute()` continues without logs; the result will have `logs: []`. No error is raised.

## Acceptance Criteria

- [ ] `PlanDeploymentFlow::handle()` returns a result for every code path (project not found, profile not found, validation error, success)
- [ ] `ApplyDeploymentFlow` separates `handle()` (planning) from `execute()` (execution)
- [ ] `ApplyDeploymentFlow::execute()` returns deployment logs from the provider's log reader if available
- [ ] `DestroyDeploymentFlow` mirrors the apply pattern with separate `handle()` and `execute()` methods
- [ ] `ValidateConfigurationFlow::handle()` validates all projects and profiles and returns a grouped error structure
- [ ] All flows use the same `ProviderFactory` to create providers from config
- [ ] `ApplyDeploymentFlow` and `DestroyDeploymentFlow` accept an optional `providerResolver` closure for testability
- [ ] Plan output includes provider, project, profile, branch, path, server_id, domain when available
- [ ] Commands use `FormatsDeploymentPlan::getPlanValue()` to safely render plan values in output

## Open Questions / Potential Concerns

- **`PlanDeploymentFlow` has no `providerResolver` injection** unlike `ApplyDeploymentFlow` and `DestroyDeploymentFlow`. This means it cannot be easily mocked in tests. This may be intentional (plan is read-only) but is worth noting.
- **The plan array has no formal schema** — it is provider-dependent. Commands display known keys but silently skip unknown ones. A JSON schema for the plan structure would improve type safety.
- **`execute()` in ApplyDeploymentFlow reads `plan['server_id']` using assertions** to convert to int. If `server_id` is a non-numeric string, this could throw. The assertion `\assert(\is_int($serverIdValue) || \is_string($serverIdValue) || \is_numeric($serverIdValue))` followed by `is_int() ? $serverIdValue : (int) $serverIdValue` is safe, but if `server_id` is absent and defaults to 0, the site context will have siteId=0 and logs will be empty.
- **`DestroySiteAction` source was not in the provided file list** for this spec. Verify it exists and has a compatible interface with `DestroyDeploymentFlow::execute()`.
# Spec: Shipper Test Coverage Initiative

**Date:** 2026-04-26
**Status:** Draft

## Overview

Shipper has zero meaningful test coverage for its core components: provider managers (34 files), actions (8 files), flows, and value objects. This spec defines the testing strategy and target coverage.

## Current State

| Component | Has Tests | Gap |
|-----------|-----------|-----|
| Config Classes | Partial (ProjectConfigTest.php only) | All other config classes uncovered |
| Config Loader | YES | — |
| Actions | NO | All 8 action files |
| Flows | NO | All deployment flows |
| Provider Managers | NO | All 34 manager files |
| Value Objects | NO | OperationResult, SiteResult, Contexts |

## Proposed Coverage Targets

### Tier 1 — Critical (must have before any deployment to production)
- `OperationResult` and `SiteResult` value object tests
- `DeploymentContext`, `ServerContext`, `SiteContext` tests
- `ProviderFactory` test — creates correct provider by name
- `ApplyDeploymentFlow` test — full happy path
- `DestroyDeploymentFlow` test — full happy path
- `CreateDeploymentPlanAction` test
- `ExecuteDeploymentAction` test
- `ValidateProjectAction` test

### Tier 2 — Important (regression protection)
- `SiteManagerInterface` implementations (Ploi + Forge)
- `DatabaseManagerInterface` implementations
- `DeployScriptManagerInterface` implementations
- `EnvironmentManagerInterface` implementations
- All 17 interface contract tests

### Tier 3 — Nice to have
- Config classes: ProfileConfig, ShipperConfig, SslConfig, EnvironmentConfig, DatabaseConfig, QueueConfig, CronConfig, RedirectConfig, DaemonConfig, NetworkRuleConfig
- `GetAllSitesAction`, `DeleteSiteAction`, `DestroySiteAction`
- `PlanDeploymentFlow`
- `CleanupOrphanedSitesFlow`

## Testing Approach

- Use PHPUnit (already configured)
- Mock provider SDK clients via interface injection
- Use data providers for multi-provider parity (Ploi + Forge)
- Test value objects for all combinations: ok/fail, found/created/new site

## Acceptance Criteria

- [ ] Tier 1 tests written and passing
- [ ] CI pipeline runs tests on every PR
- [ ] Code coverage report generated
- [ ] Tier 2 tests written (stretch goal)

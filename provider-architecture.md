# Spec: Provider Architecture

**Issue:** MAR-20
**Date:** 2026-04-23
**Status:** In Review

## Overview

The Deployment module is the core of Shipper's mission — provisioning Laravel sites on remote servers via provider APIs (Ploi, Forge). It uses a plugin-style architecture where each provider implements a set of contracts (interfaces) that drive a 15-step deployment pipeline. This spec documents the shared architecture, not any single provider.

## Current Implementation

### Key Classes / Files

| File | Class / Interface | Responsibility |
|------|------------------|----------------|
| `DeploymentProviderInterface.php` | `DeploymentProviderInterface` | Entry contract: validate, plan, apply, destroy |
| `ProviderFactory.php` | `ProviderFactory` | Creates provider instances by name (`ploi`, `forge`) |
| `AbstractProvider.php` | `AbstractProvider` | Base implementation with 15-step `apply()` sequence |
| `OperationResult.php` | `OperationResult` | Value object: `ok()` / `fail(string)` |
| `SiteResult.php` | `SiteResult` | Value object: `found(int)` / `created(int)` / `fail(string)` |
| `Context/DeploymentContext.php` | `DeploymentContext` | Value object: project + profile + server (+ optional site) |
| `Context/ServerContext.php` | `ServerContext` | Value object: `serverId` |
| `Context/SiteContext.php` | `SiteContext` | Value object: `serverId`, `siteId`, `domain` |
| `Concerns/InterpolatesNames.php` | trait | Replaces `${PROJECT_NAME}`, `${PROFILE}`, env vars in DB names |
| `Concerns/MergesEnvironment.php` | trait | Merges Shipper env vars into existing `.env` content |
| `Concerns/ResolvesDeployScript.php` | trait | Resolves profile/project script with `{site}` / `{branch}` interpolation |
| `Contracts/*.php` | 17 interfaces | Individual operation contracts |
| `Providers/Ploi/` | PloiProvider + 18 impls | Ploi SDK wrapper |
| `Providers/Forge/` | ForgeProvider + 18 impls | Forge SDK wrapper |

## Provider Contract Summary

| Interface | `plan()` return | `apply()` params |
|-----------|-----------------|------------------|
| `DeploymentProviderInterface` | `array<string,mixed>` (full plan) | `ProjectConfig, ProfileConfig` → `bool` |
| `SiteManagerInterface` | `array<string>` | `DeploymentContext` → `SiteResult`; `destroy(SiteContext)` → `OperationResult` |
| `RepositoryManagerInterface` | `array<string>` | `SiteContext, DeploymentContext` → `OperationResult` |
| `DatabaseManagerInterface` | `array<string>` | `SiteContext, DeploymentContext` → `OperationResult`; `destroy` → `OperationResult` |
| `DeployScriptManagerInterface` | `array<string>` | `SiteContext, DeploymentContext` → `OperationResult` |
| `DeploymentExecutorInterface` | `array<string>` | `SiteContext, DeploymentContext` → `OperationResult` |
| `DeploymentLogReaderInterface` | — | `SiteContext` → `array<int,string>` (logs) |
| `SslManagerInterface` | `array<string>` | `SiteContext, SslConfig` → `OperationResult` |
| `EnvironmentManagerInterface` | `array<string>` | `SiteContext, EnvironmentConfig` → `OperationResult` |
| `QueueManagerInterface` | `array<string>` | `SiteContext, array<QueueConfig>` → `OperationResult` |
| `CronManagerInterface` | `array<string>` | `SiteContext, array<CronConfig>` → `OperationResult` |
| `RedirectManagerInterface` | `array<string>` | `SiteContext, array<RedirectConfig>` → `OperationResult` |
| `DaemonManagerInterface` | `array<string>` | `SiteContext, array<DaemonConfig>` → `OperationResult` |
| `NetworkRuleManagerInterface` | `array<string>` | `SiteContext, array<NetworkRuleConfig>` → `OperationResult` |
| `PhpVersionManagerInterface` | `array<string>` | `SiteContext, string` (version) → `OperationResult` |
| `NginxConfigManagerInterface` | `array<string>` | `SiteContext, string` (config) → `OperationResult` |
| `AliasManagerInterface` | `array<string>` | `SiteContext, array<int,string>` → `OperationResult` |
| `ServerSiteListInterface` | — | `ServerContext` → `array<int, array{site_id:int, domain:string}>` |

## Data Contracts

### Value Objects

```php
final class OperationResult {
    private function __construct(
        public readonly bool $success,
        public readonly string $error,
    ) {}
    public static function ok(): self
    public static function fail(string $error): self
}

final class SiteResult {
    private function __construct(
        public readonly bool $success,
        public readonly int $siteId,
        public readonly bool $isNew,
        public readonly string $error,
    ) {}
    public static function found(int $siteId): self
    public static function created(int $siteId): self
    public static function fail(string $error): self
}

final class DeploymentContext {
    public function __construct(
        public readonly ProjectConfig $project,
        public readonly ProfileConfig $profile,
        public readonly ServerContext $server,
        public readonly ?SiteContext $site = null,
    ) {}
    public function withSite(SiteContext $site): self
}

final class ServerContext {
    public function __construct(public readonly int $serverId) {}
}

final class SiteContext {
    public function __construct(
        public readonly int $serverId,
        public readonly int $siteId,
        public readonly string $domain,
    ) {}
}
```

### Provider Entry Point

```php
interface DeploymentProviderInterface {
    public function validate(ProjectConfig $project, ProfileConfig $profile): array  // errors
    public function plan(ProjectConfig $project, ProfileConfig $profile): array        // full plan dict
    public function apply(ProjectConfig $project, ProfileConfig $profile): bool
    public function destroy(ProjectConfig $project, ProfileConfig $profile): bool
    public function getName(): string
    public function getLastError(): string
    public function getOperation(string $interface): ?object
    public function getLastSiteContext(): ?SiteContext
    public function getServerId(): string
}
```

### AbstractProvider apply() Sequence

1. Site find/create (SiteManagerInterface)
2. Repository install (RepositoryManagerInterface, new sites only)
3. Databases (DatabaseManagerInterface, if configured)
4. PHP version (PhpVersionManagerInterface, if set)
5. Nginx config (NginxConfigManagerInterface, if set)
6. Deploy script (DeployScriptManagerInterface)
7. Deployment execution (DeploymentExecutorInterface — trigger + poll)
8. SSL certificate (SslManagerInterface, if enabled)
9. Domain aliases (AliasManagerInterface, if configured)
10. Environment variables (EnvironmentManagerInterface, if not empty)
11. Queue workers (QueueManagerInterface, if configured)
12. Cron jobs (CronManagerInterface, if configured)
13. Redirects (RedirectManagerInterface, if configured)
14. Daemons (DaemonManagerInterface, if configured)
15. Network rules (NetworkRuleManagerInterface, if configured)

### Traits

```php
trait InterpolatesNames {
    // Replaces ${PROJECT_NAME}, ${PROFILE}, ${ENV_VAR} in DB names
    private function interpolateDatabaseName(string $name, string $projectName, string $profileName): string
}

trait MergesEnvironment {
    // Merges Shipper env vars into existing .env (override existing, append new)
    private function mergeEnvContent(string $existing, array $variables): string
}

trait ResolvesDeployScript {
    // Profile script overrides project script; replaces {site} and {branch}
    private function resolveDeployScript(ProjectConfig $project, ProfileConfig $profile): string
}
```

### Client Factory Pattern

Each provider has a `<Provider>ClientFactory` that:
- Lazily instantiates the SDK client (`Forge` / `Ploi`)
- Exposes `getServerId(): string`
- Exposes `getDeploymentTimeout(): int` (default 60s)
- Exposes `getConfig(): array`

```php
final class ProviderFactory {
    public function __construct(private readonly array $providersConfig = []) {}
    public function create(string $providerName): DeploymentProviderInterface
    // 'ploi' → PloiProvider, 'forge' → ForgeProvider, else throws InvalidArgumentException
}
```

## Edge Cases

- **Site already exists:** SiteManagerInterface `apply()` returns `SiteResult::found()` — deployment continues with existing site, skips repository install step.
- **Profile overrides deploy script:** `ResolvesDeployScript` always prefers profile-level `deploy_script` over project-level.
- **Empty domain:** Site creation fails fast with `"Domain is empty or invalid"`.
- **Deployment timeout:** Ploi's executor polls up to `deployment_timeout` seconds; Forge blocks with `wait=true`.
- **API exception:** Each provider maps SDK exceptions (`NotFound`, `Validation`, `Unauthenticated`) to human-readable strings via `handleException()`.
- **Partial env merge:** `MergesEnvironment` preserves existing comments and blank lines; shipper keys override, new keys appended.
- **Database name interpolation:** Env vars in `${DB_NAME}` are resolved at plan time via `InterpolatesNames`.
- **Destroy order:** `AbstractProvider::destroy()` deletes databases first, then the site.
- **Server ID validation:** Both providers validate server ID is numeric-only via `ctype_digit`.

## Acceptance Criteria

- [ ] 17 interfaces defined in `Contracts/`
- [ ] Both Ploi and Forge implement all 17 interfaces
- [ ] `ProviderFactory::create()` instantiates correct provider by name
- [ ] `AbstractProvider::plan()` returns full dict including all sub-actions
- [ ] `AbstractProvider::apply()` runs the 15-step sequence and stops on first failure
- [ ] Each provider maps SDK exceptions to user-friendly error strings
- [ ] Traits are reusable across any future provider
- [ ] All value objects are `final` with readonly props
- [ ] All manager `apply()` methods return `OperationResult::ok()` or `OperationResult::fail()`
- [ ] Site creation returns `SiteResult` with `isNew` flag

## Open Questions / Potential Concerns

- **Adding providers:** The SDK exceptions caught (`NotFound`, `Validation`, `Unauthenticated`) may differ per SDK — a new provider needs its own `handleException()` mapping. Consider a base exception mapper.
- **DeploymentLogReaderInterface** is used internally by `DeploymentExecutorInterface` (via constructor injection) but is also a public interface. Should it be a separate concern?
- **deployment_timeout** defaults to 60s in both client factories — is this sufficient for large Laravel apps with npm builds?
- **Databases planDatabases** is protected and returns `[]` in `AbstractProvider` — the interpolated names are only used by concrete providers. FR-005 in the functional spec should capture what happens when databases are not configured.
- **Site find logic** in both Ploi and Forge uses domain matching — what happens when a site exists but under a different internal ID? The find-by-domain approach is correct for the shipper workflow but relies on domain being globally unique per server.
# Spec: Forge Provider

**Issue:** MAR-39
**Date:** 2026-04-23
**Status:** In Review

## Overview

The Forge provider bridges Shipper's deployment pipeline to the Laravel Forge API. It extends `AbstractProvider`, registers all 17 operation managers, and uses the `laravel/forge-sdk` to provision sites, databases, SSL, queues, cron, and more on a Forge-managed server.

## Current Implementation

### Key Classes / Files

| File | Class | Responsibility |
|------|-------|----------------|
| `ForgeProvider.php` | `ForgeProvider` | Entry point; registers all 17 operations; extends `AbstractProvider` |
| `ForgeClientFactory.php` | `ForgeClientFactory` | Lazily creates `Forge` SDK client; exposes server ID + timeout |
| `ForgeSiteManager.php` | `ForgeSiteManager` | Find or create site by domain |
| `ForgeRepositoryManager.php` | `ForgeRepositoryManager` | Install git repository on site |
| `ForgeDatabaseManager.php` | `ForgeDatabaseManager` | Create databases with interpolated names |
| `ForgeDeploymentExecutor.php` | `ForgeDeploymentExecutor` | Trigger deployment with blocking wait + log inspection |
| `ForgeDeploymentLogReader.php` | `ForgeDeploymentLogReader` | Fetch deployment log lines |
| `ForgeSslManager.php` | `ForgeSslManager` | Provision SSL certificate |
| `ForgeEnvironmentManager.php` | `ForgeEnvironmentManager` | Write `.env` file via MergesEnvironment |
| `ForgeDeployScriptManager.php` | `ForgeDeployScriptManager` | Set deployment script |
| `ForgeQueueManager.php` | `ForgeQueueManager` | Configure queue workers |
| `ForgeCronManager.php` | `ForgeCronManager` | Configure scheduled jobs |
| `ForgeRedirectManager.php` | `ForgeRedirectManager` | HTTP redirect rules |
| `ForgeDaemonManager.php` | `ForgeDaemonManager` | Background daemon processes |
| `ForgeNetworkRuleManager.php` | `ForgeNetworkRuleManager` | Firewall rules |
| `ForgePhpVersionManager.php` | `ForgePhpVersionManager` | PHP version selection |
| `ForgeNginxConfigManager.php` | `ForgeNginxConfigManager` | Custom Nginx configuration |
| `ForgeAliasManager.php` | `ForgeAliasManager` | Domain aliases |
| `ForgeServerSiteList.php` | `ForgeServerSiteList` | List all sites on server |

### ForgeProvider: Additional Public API

```php
final class ForgeProvider extends AbstractProvider {
    use InterpolatesNames;

    private ForgeClientFactory $factory;

    public function getName(): string  // 'forge'
    public function getServerId(): string

    protected function registerOperations(): void   // registers all 17 managers
    protected function handleException(\Exception $e): string  // maps Forge SDK exceptions
    protected function planDatabases(array<DatabaseConfig> $databases, string $projectName, string $profileName): array  // InterpolatesNames
}
```

### Validation Rules (in addition to AbstractProvider base validation)

| Field | Rule |
|-------|------|
| `config['api_key']` | Required, non-empty string |
| `config['server_id']` | Required, numeric-only (digits) |
| `profile['domain']` | Required, non-empty |
| `project['repository']` | Must have `provider` (github/gitlab/bitbucket/custom) and `name` (username/repo) |

### Exception Mapping

| Exception class | Message prefix |
|----------------|---------------|
| `\Laravel\Forge\Exceptions\NotFoundException` | `"Resource not found: Server ID {id} may not exist or you don't have access."` |
| `\Laravel\Forge\Exceptions\ValidationException` | `"Validation error: {message}"` |
| Other `Exception` | `"Deployment error: {message} (Type: {class})"` (via parent) |

### SiteManager apply() Behavior

1. Fetch all sites on server via `$forge->sites($serverId)` — returns `Forge\Resources\Site[]`
2. Match by `site->name === domain`
3. If found: return `SiteResult::found(site->id)` — does NOT reinstall repository
4. If not found: create via `$forge->createSite($serverId, [...])` with `domain`, `project_type: 'php'`, `directory`
5. If `id` is 0 or null in response: return `SiteResult::fail('Invalid response')`

### DeploymentExecutor apply() Behavior (Forge)

1. Call `$forge->deploySite($serverId, $siteId, wait: true)` — **blocking**, returns when deployment completes
2. Fetch deployment logs via `ForgeDeploymentLogReader`
3. Scan log lines for keywords (`deployment failed`, `fatal error`, `critical error`, etc.)
4. If any keyword found: return `OperationResult::fail('Deployment failed on Forge server (detected in logs)')`
5. Else: return `OperationResult::ok()`
6. On exception: return `OperationResult::fail("Deployment failed: {message}")`

## Functional Requirements

**FR-001 — Provision site by domain**
**As** the CLI, **I want** to find or create a Forge site matching the profile's domain **so that** subsequent operations have a valid site ID.
- Acceptance: `SiteResult::found(id)` when domain already exists; `SiteResult::created(id)` on new site; `SiteResult::fail(...)` on API error.

**FR-002 — Trigger blocking deployment**
**As** the CLI, **I want** to deploy the site via Forge API with a blocking wait **so that** deployment status is known before the command returns.
- Acceptance: Returns `OperationResult::ok()` on success; `OperationResult::fail(msg)` if error keywords detected in logs.

**FR-003 — Read deployment logs**
**As** the CLI, **I want** to fetch deployment log lines from Forge **so that** I can surface failures and debug deployment issues.
- Acceptance: Returns `array<int, string>` of log lines; empty array if unavailable.

**FR-004 — Handle Forge API exceptions gracefully**
**As** the CLI, **I want** Forge-specific exceptions to produce user-friendly messages **so that** debugging is easier.
- Acceptance: NotFoundException maps to server ID error; ValidationException maps to validation error; all others use generic deployment error format.

## Edge Cases

- **Site create response missing id:** Returns `SiteResult::fail('Failed to create site: Invalid response from Forge API')` — does not throw.
- **Domain already exists:** Returns `SiteResult::found()` — `AbstractProvider` skips repository install because `isNew === false`.
- **Blocking deploy throws:** Caught and returned as `OperationResult::fail("Deployment failed: {message}")`.
- **Log scan finds error keyword:** Immediately returns `OperationResult::fail` with static message — no log excerpt returned (unlike Ploi which appends recent logs).
- **Delete site not found:** Returns `OperationResult::ok()` — idempotent destroy.
- **Server ID type coercion:** Accepts both `string` and `int` for `server_id` config; casts to `string` before `ctype_digit` validation.

## Acceptance Criteria

- [ ] ForgeProvider registers all 17 operation managers
- [ ] `getName()` returns `'forge'`
- [ ] `getServerId()` returns the configured numeric server ID
- [ ] Validation fails cleanly when `api_key` or `server_id` is missing/wrong
- [ ] Site find/create works for both existing and new domains
- [ ] Deployment blocks until completion and inspects logs for error keywords
- [ ] SDK exceptions are caught and mapped to user-friendly messages
- [ ] `plan()` includes all 15+ sub-actions with human-readable labels

## Open Questions / Potential Concerns

- **Forge's blocking `deploySite(..., true)` vs Ploi's polling approach:** Forge waits server-side; Ploi requires client-side polling. Forge's approach is simpler but provides no progress feedback during the wait. Is this intentional?
- **No deployment timeout configuration in Forge** — the blocking call could hang indefinitely if Forge's server-side wait has no timeout. `ForgeClientFactory` does expose `getDeploymentTimeout()` but it is not used by `ForgeDeploymentExecutor`. Should it be?
- **Log inspection on Forge** does not append log content to the error message (unlike Ploi which includes up to 10 log lines). This makes Forge errors harder to debug. Intentional?
- **Repository re-install on found site:** The current design correctly skips this, but there's no option to force a re-clone. Is this intentional?
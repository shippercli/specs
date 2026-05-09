# Spec: Ploi Provider

**Issue:** MAR-38
**Date:** 2026-04-23
**Status:** In Review

## Overview

The Ploi provider bridges Shipper's deployment pipeline to the Ploi API. It extends `AbstractProvider`, registers all 17 operation managers, and uses the `ploi/ploi-php-sdk` to provision sites, databases, SSL, queues, cron, and more on a Ploi-managed server.

## Current Implementation

### Key Classes / Files

| File | Class | Responsibility |
|------|-------|----------------|
| `PloiProvider.php` | `PloiProvider` | Entry point; registers all 17 operations; extends `AbstractProvider` |
| `PloiClientFactory.php` | `PloiClientFactory` | Lazily creates `Ploi` SDK client; exposes server ID + timeout |
| `PloiSiteManager.php` | `PloiSiteManager` | Find or create site by domain |
| `PloiRepositoryManager.php` | `PloiRepositoryManager` | Install git repository on site |
| `PloiDatabaseManager.php` | `PloiDatabaseManager` | Create databases with interpolated names |
| `PloiDeploymentExecutor.php` | `PloiDeploymentExecutor` | Trigger deployment + poll with log inspection |
| `PloiDeploymentLogReader.php` | `PloiDeploymentLogReader` | Fetch deployment log lines |
| `PloiSslManager.php` | `PloiSslManager` | Provision SSL certificate |
| `PloiEnvironmentManager.php` | `PloiEnvironmentManager` | Write `.env` file viaMergesEnvironment |
| `PloiDeployScriptManager.php` | `PloiDeployScriptManager` | Set deployment script |
| `PloiQueueManager.php` | `PloiQueueManager` | Configure queue workers |
| `PloiCronManager.php` | `PloiCronManager` | Configure scheduled jobs |
| `PloiRedirectManager.php` | `PloiRedirectManager` | HTTP redirect rules |
| `PloiDaemonManager.php` | `PloiDaemonManager` | Background daemon processes |
| `PloiNetworkRuleManager.php` | `PloiNetworkRuleManager` | Firewall rules |
| `PloiPhpVersionManager.php` | `PloiPhpVersionManager` | PHP version selection |
| `PloiNginxConfigManager.php` | `PloiNginxConfigManager` | Custom Nginx configuration |
| `PloiAliasManager.php` | `PloiAliasManager` | Domain aliases |
| `PloiServerSiteList.php` | `PloiServerSiteList` | List all sites on server |

### PloiProvider: Additional Public API

```php
final class PloiProvider extends AbstractProvider {
    use InterpolatesNames;

    private PloiClientFactory $factory;

    public function getName(): string  // 'ploi'
    public function getServerId(): string
    public function getClient(): Ploi  // exposes SDK directly
    public function getLastSiteId(): int
    public function getDeploymentLogs(int $serverId, int $siteId): array<string>  // backwards-compat helper

    protected function registerOperations(): void   // registers all 17 managers
    protected function handleException(\Exception $e): string  // maps Ploi SDK exceptions
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
| `\Ploi\Exceptions\Http\Unauthenticated` | `"Authentication failed: Invalid Ploi API key."` |
| `\Ploi\Exceptions\Http\NotFound` | `"Resource not found: Server ID {id} may not exist or you don't have access."` |
| `\Ploi\Exceptions\Http\NotValid` | `"Validation error: {message}"` |
| Other `Exception` | `"Deployment error: {message} (Type: {class})"` (via parent) |

### SiteManager apply() Behavior

1. Fetch all sites on server via `GET /servers/{id}/sites`
2. Match by `domain` property
3. If found: return `SiteResult::found(site->id)` — does NOT reinstall repository
4. If not found: create via `POST /servers/{id}/sites` with `web_directory` and `project_root`
5. If `id` is 0 or null in response: return `SiteResult::fail('Invalid response')`

### DeploymentExecutor apply() Behavior (Ploi)

1. Call `site->deployment()->deploy()` to trigger deployment
2. Poll `GET /servers/{id}/sites/{id}` every 5 seconds up to `deployment_timeout` (default 60s)
3. When `deploying === false`:
   - If `status === 'deploy-failed'`: fetch logs, return `OperationResult::fail` with log excerpt
   - Otherwise: scan log lines for keywords (`deployment failed`, `fatal error`, etc.)
   - If any keyword found: return `OperationResult::fail`
   - Else: return `OperationResult::ok()`
4. If timeout reached: return `OperationResult::fail("Deployment timeout after {n} seconds")`

## Functional Requirements

**FR-001 — Provision site by domain**
**As** the CLI, **I want** to find or create a Ploi site matching the profile's domain **so that** subsequent operations have a valid site ID.
- Acceptance: `SiteResult::found(id)` when domain already exists; `SiteResult::created(id)` on new site; `SiteResult::fail(...)` on API error.

**FR-002 — Trigger and poll deployment**
**As** the CLI, **I want** to deploy the site via Ploi API and wait for completion **so that** deployment status is known before the command returns.
- Acceptance: Returns `OperationResult::ok()` on success; `OperationResult::fail(msg)` on deploy-failed status or error keywords in logs; timeout after `deployment_timeout` seconds.

**FR-003 — Read deployment logs**
**As** the CLI, **I want** to fetch deployment log lines from Ploi **so that** I can surface failures and debug deployment issues.
- Acceptance: Returns `array<int, string>` of log lines; empty array if unavailable.

**FR-004 — Handle Ploi API exceptions gracefully**
**As** the CLI, **I want** Ploi-specific exceptions to produce user-friendly messages **so that** debugging is easier.
- Acceptance: Unauthenticated maps to auth error; NotFound maps to server ID error; ValidationException maps to validation error; all others use generic deployment error format.

## Edge Cases

- **Site create response missing id:** Returns `SiteResult::fail('Failed to create site: Invalid response from Ploi API')` — does not throw.
- **Domain already exists:** Returns `SiteResult::found()` — `AbstractProvider` skips repository install because `isNew === false`.
- **Deployment still running at timeout:** Returns `OperationResult::fail("Deployment timeout after {n} seconds. Deployment may still be running on Ploi.")`.
- **Poll response missing data:** Loop continues without incrementing elapsed — safe against null responses.
- **Empty site list from API:** Proceeds to create site — no false match against existing.
- **Delete site not found:** Returns `OperationResult::ok()` — idempotent destroy.

## Acceptance Criteria

- [ ] PloiProvider registers all 17 operation managers
- [ ] `getName()` returns `'ploi'`
- [ ] `getServerId()` returns the configured numeric server ID
- [ ] Validation fails cleanly when `api_key` or `server_id` is missing/wrong
- [ ] Site find/create works for both existing and new domains
- [ ] Deployment polling respects `deployment_timeout` config
- [ ] Log inspection catches `deployment failed`, `fatal error`, `critical error` in logs
- [ ] SDK exceptions are caught and mapped to user-friendly messages
- [ ] `plan()` includes all 15+ sub-actions with human-readable labels

## Open Questions / Potential Concerns

- **`deployment_timeout` default of 60s** may be too short for apps with npm asset builds. Should this be configurable per-profile or per-project?
- **`getDeploymentLogs` helper method** on `PloiProvider` is backwards-compatible but leaks `DeploymentLogReaderInterface` usage. Is this the right API surface, or should logs be retrieved through a command?
- **Repository re-install on found site:** The current design correctly skips this, but there's no option to force a re-clone. Is this intentional?
- **No site suspension support:** Ploi SDK supports it but Shipper doesn't expose it. Confirm this is out of scope.
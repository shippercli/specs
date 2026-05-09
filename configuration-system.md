# Spec: Configuration System

**Issue:** MAR-18
**Date:** 2026-04-23
**Status:** In Review

## Overview

The configuration system parses a single `shipper.yml` file into a strongly-typed PHP object graph that the rest of the application uses for all deployment decisions. It converts flat YAML into a hierarchy of immutable value objects (one per resource type), performs environment-variable interpolation at parse time, and exposes the result via a thin `LoadConfigurationAction` entry point. Validation of provider credentials and profile completeness happens in a separate `ValidateConfigurationFlow` that operates on the already-parsed config graph.

## Current Implementation

### Key Classes / Files

| Class / File | Role |
|---|---|
| `app/Config/ConfigLoader.php` | Reads YAML file, interpolates env vars, constructs the full object graph |
| `app/Config/ShipperConfig.php` | Root config object — holds projects and provider credentials |
| `app/Config/ProjectConfig.php` | Per-project settings: provider, path, profiles, databases, queues, etc. |
| `app/Config/ProfileConfig.php` | Per-environment settings: branch, domain, aliases, environment vars |
| `app/Config/DatabaseConfig.php` | Database name, user, and type |
| `app/Config/EnvironmentConfig.php` | Key-value env var map with merge support |
| `app/Config/QueueConfig.php` | Queue worker settings (connection, queue name, limits) |
| `app/Config/CronConfig.php` | Cron job: command, frequency, user |
| `app/Config/DaemonConfig.php` | Background process: command, user, processes, directory |
| `app/Config/NetworkRuleConfig.php` | Firewall rule: port, protocol, allow/deny, source IP |
| `app/Config/RedirectConfig.php` | HTTP redirect: from path, to path, type (redirect/permanent) |
| `app/Config/SslConfig.php` | SSL: enabled flag and type (letsencrypt) |
| `app/Actions/LoadConfigurationAction.php` | Thin action that instantiates `ConfigLoader` and calls `load()` |
| `app/Flows/ValidateConfigurationFlow.php` | Iterates all projects/profiles and runs `ValidateProjectAction` per profile |
| `shipper.yml` | Reference configuration with two example projects (api, frontend) |

## Functional Requirements

**FR-001 — File loading**
`ConfigLoader` MUST accept a file path (default: `shipper.yml`) and MUST throw `\RuntimeException` if the file does not exist or cannot be read.

**FR-002 — YAML parsing**
The loader MUST parse the file using `symfony/yaml`. The parsed result MUST be treated as `array<string, mixed>` at the top level.

**FR-003 — Provider block**
The `providers` top-level key MUST be treated as a freeform `array<string, mixed>`. Provider data MUST have environment variable interpolation applied before being passed to `ShipperConfig`. No schema validation is performed on provider values at parse time — this is deferred to `ProviderFactory`.

**FR-004 — Project enumeration**
Each key under `projects` MUST produce one `ProjectConfig`. Projects with non-string keys or non-array values MUST be silently skipped.

**FR-005 — Profile enumeration**
Each key under a project's `profiles` map MUST produce one `ProfileConfig`. Non-string or non-array entries MUST be silently skipped.

**FR-006 — Database parsing**
Each entry under `databases` MUST produce a `DatabaseConfig`. The `name` field defaults to the YAML key if absent; `user` defaults to the YAML key; `type` defaults to `'mysql'`. Invalid types are coerced to their defaults rather than rejected.

**FR-007 — Queue parsing**
Each entry under `queues` MUST produce a `QueueConfig`. Defaults: `connection='database'`, `queue='default'`, `max_seconds=60`, `sleep=30`, `processes=1`, `max_tries=1`. Non-integer numeric fields are coerced to their defaults.

**FR-008 — Cron parsing**
Each entry under `cron` MUST be skipped if either `command` or `frequency` is empty or missing. Default user is `'ploi'`.

**FR-009 — Daemon parsing**
Each entry under `daemons` MUST be skipped if `command` is empty or missing. Default user is `'ploi'`, processes defaults to `1`, directory defaults to `''`.

**FR-010 — Redirect parsing**
Each entry under `redirects` MUST be skipped if `from` or `to` is empty or missing. Default type is `'redirect'`.

**FR-011 — Network rule parsing**
Each entry under `network_rules` MUST be skipped if `port` is not a positive integer. Default `type` is `'tcp'`, default `rule_type` is `'allow'`. `from_ip` is optional and nullable.

**FR-012 — SSL parsing**
The `ssl` block is optional. If absent, `SslConfig` defaults to `enabled=false`, `type='letsencrypt'`. Boolean values for `enabled` that are not actual PHP booleans are coerced to `false`.

**FR-013 — Environment variable config**
The `environment` block at project level and at profile level MUST be parsed into an `EnvironmentConfig`. Keys MUST be strings; values MAY be string, int, float, or bool — all are cast to `string`. Non-conforming entries are silently dropped.

**FR-014 — Environment variable merging**
`EnvironmentConfig::mergeWith()` MUST return a new `EnvironmentConfig` where the other's values override matching keys from the base. The original instances MUST NOT be mutated.

**FR-015 — Load action**
`LoadConfigurationAction::handle(string $configPath): ShipperConfig` MUST be the public entry point used by commands. It creates a `ConfigLoader` and calls `load()`.

**FR-016 — Validation flow**
`ValidateConfigurationFlow::handle(string $configPath)` MUST return `array{success: bool, errors: array<string, array<string, array<int, string>>>}`. It loads config, instantiates `ProviderFactory`, iterates every project and profile, and accumulates errors per project and profile name. Provider instantiation errors are keyed as `'_provider'`.

**FR-017 — Immutability**
All config value objects MUST use `readonly` constructor promotion. No public setters exist. All classes MUST be marked `final`.

## Configuration Interface

```yaml
providers:
  ploi:
    api_key: "${PLOI_API_KEY}"
    api_url: "https://ploi.io/api"
    server_id: "105556"
    deployment_timeout: 60

projects:
  api:
    provider: ploi
    path: ./examples/api
    repository:
      provider: github
      name: org/repo
    php_version: "8.3"
    web_directory: /public
    project_root: /
    ssl:
      enabled: true
      type: letsencrypt
    deploy_script: |
      cd /home/ploi/{site}
      git pull origin {branch}
    environment:
      APP_NAME: "My App"
    databases:
      main:
        name: "app_${PROJECT_NAME}_${PROFILE}"
        user: "app_${PROJECT_NAME}_${PROFILE}"
        type: mysql
    queues:
      default:
        connection: database
        queue: default
        max_seconds: 60
        sleep: 30
        processes: 1
        max_tries: 1
    cron:
      scheduler:
        command: "php artisan schedule:run"
        frequency: "* * * * *"
        user: ploi
    daemons:
      horizon:
        command: "php artisan horizon"
        user: ploi
        processes: 1
        directory: ""
    redirects:
      old-to-new:
        from: /old
        to: /new
        type: redirect
    network_rules:
      allow-redis:
        name: "Allow Redis"
        port: 6379
        type: tcp
        rule_type: allow
        from_ip: "10.0.0.0/8"
    profiles:
      production:
        branch: main
        domain: api.example.com
        aliases:
          - www.api.example.com
        environment:
          APP_ENV: production
      preview:
        branch: "${GITHUB_HEAD_REF}"
        domain: "api-preview-${GITHUB_PR_NUMBER}.example.com"
        environment:
          APP_ENV: staging
```

## Data Contracts

```php
final class ShipperConfig
{
    public function __construct(
        private readonly array $projects,  // array<string, ProjectConfig>
        private readonly array $providers, // array<string, mixed>
    ) {}
    public function projects(): array {}
    public function providers(): array {}
    public function getProject(string $name): ?ProjectConfig {}
}

final class ProjectConfig
{
    public function __construct(
        private readonly string $name,
        private readonly string $provider,
        private readonly string $path,
        private readonly array $profiles,       // array<string, ProfileConfig>
        private readonly array $repository = [],
        private readonly string $webDirectory = '/public',
        private readonly string $projectRoot = '/',
        private readonly array $databases = [],  // array<string, DatabaseConfig>
        private readonly SslConfig $ssl = new SslConfig,
        private readonly EnvironmentConfig $environment = new EnvironmentConfig,
        private readonly string $deployScript = '',
        private readonly array $queues = [],     // array<string, QueueConfig>
        private readonly array $cron = [],       // array<string, CronConfig>
        private readonly array $redirects = [],  // array<string, RedirectConfig>
        private readonly string $phpVersion = '',
        private readonly string $nginxConfig = '',
        private readonly array $daemons = [],        // array<string, DaemonConfig>
        private readonly array $networkRules = [],   // array<string, NetworkRuleConfig>
    ) {}
    // Getters: name(), provider(), path(), profiles(), getProfile(string), repository(),
    //          webDirectory(), projectRoot(), databases(), getDatabase(string),
    //          ssl(), environment(), deployScript(), queues(), getQueue(string),
    //          cron(), getCron(string), redirects(), getRedirect(string),
    //          phpVersion(), nginxConfig(), daemons(), getDaemon(string),
    //          networkRules(), getNetworkRule(string)
}

final class ProfileConfig
{
    public function __construct(
        private readonly string $name,
        private readonly string $branch,
        private readonly array $config,           // array<string, mixed> (raw profile data)
        private readonly EnvironmentConfig $environment = new EnvironmentConfig,
    ) {}
    public function name(): string {}
    public function branch(): string {}
    public function config(): array {}
    public function get(string $key, mixed $default = null): mixed {}
    public function environment(): EnvironmentConfig {}
    public function aliases(): array {} // array<int, string>
}

final class DatabaseConfig
{
    public function __construct(
        private readonly string $name,
        private readonly string $user,
        private readonly string $type = 'mysql',
    ) {}
}

final class EnvironmentConfig
{
    public function __construct(private readonly array $variables = []) {} // array<string, string>
    public function variables(): array {}
    public function isEmpty(): bool {}
    public function mergeWith(self $other): self {}
}

final class QueueConfig
{
    public function __construct(
        private readonly string $connection = 'database',
        private readonly string $queue = 'default',
        private readonly int $maxSeconds = 60,
        private readonly int $sleep = 30,
        private readonly int $processes = 1,
        private readonly int $maxTries = 1,
    ) {}
}

final class CronConfig
{
    public function __construct(
        private readonly string $command,
        private readonly string $frequency,
        private readonly string $user = 'ploi',
    ) {}
}

final class DaemonConfig
{
    public function __construct(
        private readonly string $command,
        private readonly string $user = 'ploi',
        private readonly int $processes = 1,
        private readonly string $directory = '',
    ) {}
}

final class NetworkRuleConfig
{
    public function __construct(
        private readonly string $name,
        private readonly int $port,
        private readonly string $type = 'tcp',
        private readonly string $ruleType = 'allow',
        private readonly ?string $fromIp = null,
    ) {}
}

final class RedirectConfig
{
    public function __construct(
        private readonly string $from,
        private readonly string $to,
        private readonly string $type = 'redirect',
    ) {}
}

final class SslConfig
{
    public function __construct(
        private readonly bool $enabled = false,
        private readonly string $type = 'letsencrypt',
    ) {}
}
```

## Edge Cases

- **Missing config file:** `ConfigLoader` throws `\RuntimeException` immediately, no partial state.
- **Non-string project/profile keys:** Silently skipped — the YAML map key must be a string.
- **Unresolvable env var in YAML (ConfigLoader path):** The `${VAR}` token is left as-is in the string (not replaced with empty string). This is different from the `InterpolatesNames` trait used during deployment, which replaces unresolved vars with `''`.
- **Non-array `providers` block:** Treated as empty array, no error raised.
- **Invalid `enabled` type for SSL:** Non-boolean values coerce to `false`.
- **Cron/daemon with empty command:** Entry silently dropped from the parsed map.
- **Network rule with port 0 or negative:** Entry silently dropped (only `$rulePort > 0` passes).
- **Environment values that are booleans:** Cast to `string` — PHP's `(string) true` is `'1'` and `(string) false` is `''`. This may produce unexpected `.env` values.
- **Profile raw config array:** `ProfileConfig` stores the entire raw profile array including unrecognized keys, accessible via `get(key)`. There is no strict allowlist of profile fields.

## Acceptance Criteria

- [ ] Loading `shipper.yml` returns a `ShipperConfig` with the correct project and provider counts
- [ ] A missing config file causes `LoadConfigurationAction` to throw `\RuntimeException`
- [ ] `ProjectConfig` correctly exposes all typed sub-configs (databases, queues, cron, daemons, redirects, network rules, ssl, environment)
- [ ] `EnvironmentConfig::mergeWith()` merges keys with the other taking precedence and does not mutate the originals
- [ ] All config classes are `final` and use `readonly` constructor promotion
- [ ] Cron entries without a command or frequency are absent from the parsed map
- [ ] Daemon entries without a command are absent from the parsed map
- [ ] Redirect entries without from/to are absent from the parsed map
- [ ] Network rules with port <= 0 are absent from the parsed map
- [ ] `ValidateConfigurationFlow` returns `success: false` and populates `errors` when provider instantiation fails

## Open Questions / Potential Concerns

- **Default coercion vs. rejection:** When a YAML field has the wrong type (e.g., `max_seconds: "sixty"`), the current code silently falls back to the default value. This means misconfigured YAML produces no warning — the user may not notice. Consider adding a validation pass that surfaces type mismatches.
- **Boolean env values casting:** `(string) false` is `''` in PHP, which would set an env var to an empty string. This may be unexpected for values like `APP_DEBUG: false`. It is worth deciding whether `false` should become `'false'` explicitly.
- **`ProfileConfig` holds the entire raw array:** Consumers can call `$profile->get('any_key')`, meaning there is no schema enforcement on profiles beyond `branch` and `environment`. Undocumented keys are silently carried through.
- **`nginx_config` field:** Accepted in `ProjectConfig` but there is no test coverage for it and no documentation in `CONFIGURATION.md`. It is unclear whether this is a planned or partially-implemented feature.
- **`deploy_script` at profile level:** The reference `shipper.yml` shows a profile-level `deploy_script` (e.g., under `production`), but `ProfileConfig` stores the raw config array rather than a dedicated typed field. Consumers accessing the profile deploy script would need to call `$profile->get('deploy_script')`. This is inconsistent with how the project-level deploy script is exposed as a typed getter.
- **`aliases` in ProfileConfig:** Parsed from the raw config array via `aliases()` — not a constructor parameter. No test coverage for this method.

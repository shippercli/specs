# Spec: Implement DeployCommand

**Date:** 2026-04-26
**Status:** Draft

## Overview

`DeployCommand` currently exists as a stub/no-op. This spec defines what it should do.

## Current State

```php
public function handle(): int
{
    $this->info('Starting deployment...');
    // Deployment logic goes here  <-- EMPTY STUB
    $this->comment('Deployment completed successfully!');
    return self::SUCCESS;
}
```

## Proposed Behavior

`shipper deploy` should trigger a new deployment for an already-provisioned site without re-running the full 15-step apply sequence.

### Command Interface

```bash
shipper deploy <project> [--profile=<name>] [--config=<path>]
```

### Behavior

1. Load configuration for the project/profile
2. Validate project exists (same validation as `plan`/`apply`)
3. Identify the target site (domain → site ID mapping via `GetAllSitesAction`)
4. Trigger deployment via `DeploymentExecutorInterface` directly
5. Poll for deployment logs via `DeploymentLogReaderInterface`
6. Output deployment status

### Difference from `apply`

| Aspect | `apply` | `deploy` |
|--------|---------|----------|
| Creates site | Yes | No |
| Configures PHP/Nginx/SSL | Yes | No |
| Triggers deployment | Yes | Yes |
| Requires `--force` | Yes | Yes |
| Fails if site doesn't exist | N/A | Yes |

## Acceptance Criteria

- [ ] `shipper deploy <project>` triggers deployment for existing site
- [ ] Returns error if site does not exist
- [ ] Shows live deployment logs during polling
- [ ] Returns exit 0 on success, exit 1 on failure
- [ ] Tests cover: site exists, site not found, deployment success, deployment failure

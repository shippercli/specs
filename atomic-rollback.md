# Spec: Deployment Atomic Rollback

**Date:** 2026-04-26
**Status:** Draft

## Overview

When `AbstractProvider::apply()` fails mid-deployment (steps 2-15 after site creation succeeds in step 1), there is no automatic cleanup of created resources. This leaves orphaned sites, databases, and other provisioned resources.

## Problem Statement

The 15-step `apply()` sequence creates resources in order:
1. Site (site manager)
2. Repository (new sites only)
3. Databases
4. PHP version
5. Nginx config
6. Deploy script
7. **Deployment execution** ← most likely failure point
8. SSL certificate
9. Domain aliases
10. Environment variables
11. Queue workers
12. Cron jobs
13. Redirects
14. Daemons
15. Network rules

If step 7+ fails, steps 1-6 remain provisioned with no rollback.

## Proposed Solution

Add an `AtomicDeployment` wrapper that tracks created resources and cleans up on failure:

```php
class AtomicDeployment {
    private array $createdResources = [];

    public function trackSite(int $siteId): void { ... }
    public function trackDatabase(int $dbId): void { ... }
    public function rollback(): void { ... }
}
```

**Rollback behavior:**
- Site creation failure → no rollback needed (site not created)
- Repository failure after site creation → destroy site
- Database failure after site creation → destroy site + delete DBs created in this run
- Deployment failure → site remains but is marked failed; continue to allow re-apply
- Step 8-15 failure → site remains, user can fix and re-apply

## Alternative: Synchronous Rollback

On `apply()` failure at any step after step 1, call rollback in reverse order:
```
15 → 14 → ... → 2 → 1  (reverse destroy order)
```

## Acceptance Criteria

- [ ] `AtomicDeployment` class created
- [ ] `AbstractProvider::apply()` wraps execution in atomic block
- [ ] Failed deployment after site creation triggers cleanup
- [ ] Tests verify rollback behavior
- [ ] Rollback does NOT trigger on deployment failure alone (site is provisioned, just deployment failed)

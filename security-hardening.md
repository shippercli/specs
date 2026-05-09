# Spec: Security Hardening — Path Traversal, Env Validation, Rate Limit Retry

**Date:** 2026-04-26
**Status:** Draft

## Overview

Three security/robustness issues found during codebase analysis that require fixes before production deployment.

## Issue 1: Path Traversal in ConfigLoader (CRITICAL)

**File:** `ConfigLoader::load()`
**Problem:** `$this->configPath` is used directly with `file_exists()` and `file_get_contents()` without validation. An attacker could pass `../../etc/passwd` or similar paths.

**Fix:** Validate that `realpath($configPath)` resolves to within an expected directory. Reject paths that escape the project root.

```php
public function load(): ShipperConfig
{
    $resolved = realpath($this->configPath);
    if ($resolved === false) {
        throw new RuntimeException("Config file not found: {$this->configPath}");
    }
    // Optionally: ensure resolved path is within project directory
    $content = file_get_contents($resolved);
    // ...
}
```

## Issue 2: Missing GITHUB_REPOSITORY Format Validation (MEDIUM)

**File:** `CleanupOrphanedCommand::handle()`
**Problem:** Only checks `GITHUB_REPOSITORY` exists, not that it matches `owner/repo` format. Invalid formats like `invalidformat` would be passed to GitHub API.

**Fix:** Validate with regex `^[a-zA-Z0-9_.-]+/[a-zA-Z0-9_.-]+$` and provide clear error message.

## Issue 3: GitHub API Rate Limit Has No Retry Backoff (HIGH)

**File:** `GetOpenPullRequestsAction::handle()`
**Problem:** When GitHub returns 429, the action returns an error without retry. For large PR counts requiring pagination, this means the cleanup command can fail prematurely.

**Fix:** Implement exponential backoff retry:
- On 429: wait `min(60, 2^attempt * 10)` seconds, retry up to 3 times
- Track `X-RateLimit-Reset` header if available for precise sleep duration

## Acceptance Criteria

- [ ] `ConfigLoader` validates path traversal safety
- [ ] `CleanupOrphanedCommand` validates `GITHUB_REPOSITORY` format as `owner/repo`
- [ ] `GetOpenPullRequestsAction` retries on 429 with exponential backoff
- [ ] Tests cover all three fixes

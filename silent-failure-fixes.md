# Spec: Silent Failure Fixes — DB Deletion, Orphan Cleanup Errors

**Date:** 2026-04-26
**Status:** Draft

## Overview

Two issues where failures are silently ignored, making debugging difficult.

## Issue 1: Silent Database Deletion Failures (HIGH)

**File:** `PloiDatabaseManager::destroy()` (lines 87-91)

```php
try {
    $server->databases((int) $db->id)->delete();
} catch (\Exception $e) {
    // Continue even if database deletion fails
}
return OperationResult::ok();  // Always returns success
```

**Problem:** If DB deletion fails, it's silently ignored. Orphaned databases remain on the provider.

**Fix:** Log the failure and return `OperationResult::fail()` so the caller knows deletion failed:

```php
try {
    $server->databases((int) $db->id)->delete();
} catch (\Exception $e) {
    $this->logger?->error("Database deletion failed: {$e->getMessage()}");
    return OperationResult::fail("Failed to delete database: {$e->getMessage()}");
}
```

Same fix needed in `ForgeDatabaseManager::destroy()`.

## Issue 2: Orphan Cleanup Silently Swallows Errors (MEDIUM)

**File:** `CleanupOrphanedSitesFlow::execute()` (lines 102-113)

```php
foreach ($orphanedSites as $site) {
    try {
        if ($deleteSiteAction->handle($siteListProvider, $site['site_id'])) {
            $deleted++;
        }
    } catch (\Exception $e) {
        $failed++;
        // Error not logged to user
    }
}
$this->info("Deleted: $deleted, Failed: $failed");
```

**Problem:** User only sees counts. They don't know which site failed or why.

**Fix:** Collect error messages and display at end:

```php
$failures = [];
foreach ($orphanedSites as $site) {
    try {
        if ($deleteSiteAction->handle($siteListProvider, $site['site_id'])) {
            $deleted++;
        }
    } catch (\Exception $e) {
        $failed++;
        $failures[] = "  - {$site['domain']}: {$e->getMessage()}";
    }
}
if ($failures) {
    $this->error("Failed to delete {$failed} site(s):\n" . implode("\n", $failures));
}
```

## Acceptance Criteria

- [ ] `PloiDatabaseManager::destroy()` returns `OperationResult::fail()` on exception
- [ ] `ForgeDatabaseManager::destroy()` has same fix
- [ ] `CleanupOrphanedSitesFlow::execute()` shows per-site error messages on failure
- [ ] Tests cover both fixes

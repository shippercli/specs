---
description: "**Issue:** MAR-37 **Date:** 2026-04-24 **Status:** In Review"
---

# Spec: Orphan Cleanup — PR Preview Environments

**Issue:** MAR-37
**Date:** 2026-04-24
**Status:** In Review

## Overview

Shipper deploys preview environments for pull requests using dynamic domain names (e.g., `api-preview-123.example.com`). When a PR is closed or merged, the preview site must be cleaned up. The `cleanup-orphaned` command and `CleanupOrphanedSitesFlow` handle this by finding sites whose PRs are no longer open and deleting them.

## Current Implementation

### Key Classes / Files

| Class/File | Type | Responsibility |
|---|---|---|
| `CleanupOrphanedCommand` | Command | CLI entry point; reads env vars, runs flow with dry-run first |
| `CleanupOrphanedSitesFlow` | Flow | Orchestrates: load config → fetch sites → fetch PRs → find orphans → delete |
| `FindOrphanedSitesAction` | Action | Compares deployed sites against open PR list; extracts PR# from domain |
| `GetOpenPullRequestsAction` | Action | Calls GitHub API `/repos/{repo}/pulls?state=open`; paginates all results |
| `DeleteSiteAction` | Action | Deletes a site by ID via `SiteManagerInterface` |
| `GitHubHttpClient` | Client | Wraps Guzzle with GitHub API base URI, Bearer token, accept headers |
| `GetAllSitesAction` | Action | Lists all sites from provider via `ServerSiteListInterface` |
| `CleanupOrphanedCommandTest` | Test | Tests GITHUB_TOKEN and GITHUB_REPOSITORY env var requirements |
| `GitHubHttpClientTest` | Test | Verifies Guzzle client configuration (base_uri, headers, http_errors) |

### Flow Sequence

```
CleanupOrphanedCommand::handle()
  1. Read GITHUB_TOKEN, GITHUB_REPOSITORY env vars (required)
  2. Run CleanupOrphanedSitesFlow::handle(configPath, repo, token, dryRun=true)
     → returns orphaned list without deleting
  3. Display orphaned sites (or "none found")
  4. If --dry-run, exit; else confirm (unless --force)
  5. Run CleanupOrphanedSitesFlow::handle(configPath, repo, token, dryRun=false)
     → actually deletes each orphaned site
  6. Report deleted/failed counts
```

## Functional Requirements

**FR-001 — Detect Orphaned Preview Sites**
A preview site is "orphaned" when:
1. It matches a configured preview profile domain pattern (e.g., `api-preview-${GITHUB_PR_NUMBER}.example.com`)
2. The extracted PR number is NOT in the list of open PRs for the repository

**FR-002 — GitHub API Integration**
`GetOpenPullRequestsAction::handle()` must:
- Call `GET /repos/{owner}/{repo}/pulls?state=open&per_page=100`
- Handle pagination (iterate until `< 100` results received)
- Handle error status codes: 401 (auth failed), 403 (forbidden/rate limited), 429 (rate limited)
- Return `array{success: bool, prs: array<int>, error: string}`

**FR-003 — PR Number Extraction**
`FindOrphanedSitesAction` extracts PR numbers by matching site domain against the configured preview profile domain pattern. The pattern `api-preview-${GITHUB_PR_NUMBER}.example.com` becomes regex `api-preview-(\d+)\.example\.com`. Only profiles named `preview` are considered.

**FR-004 — Dry-Run Mode**
Running `cleanup-orphaned --dry-run` reports what would be deleted without making any changes. The command always runs in dry-run mode first to surface the orphaned list before requiring confirmation.

**FR-005 — Force Mode**
`--force` skips the confirmation prompt and proceeds directly to deletion.

**FR-006 — Site Deletion**
Each orphaned site is deleted via `DeleteSiteAction::handle($provider, $siteId)`. Failures are counted but do not stop the iteration.

## Configuration Interface / Command Interface

```bash
cleanup-orphaned
  {--config=shipper.yml : Path to config file}
  {--dry-run : Show what would be deleted without actually deleting}
  {--force : Skip confirmation prompt}
```

**Required environment variables:**
- `GITHUB_TOKEN` — GitHub personal access token with `repo` scope
- `GITHUB_REPOSITORY` — Repository in format `owner/repo` (e.g., `ulties/shipper`)

## Data Contracts

```php
// GetOpenPullRequestsAction
final class GetOpenPullRequestsAction {
    public function __construct(?LoggerInterface $logger = null) {}
    /** @return array{success: bool, prs: array<int>, error: string} */
    public function handle(string $repo, string $token): array;
}

// FindOrphanedSitesAction
final class FindOrphanedSitesAction {
    /** @return array<int, array{site_id: int, domain: string, pr_number: int}> */
    public function handle(array $sites, array $openPRs, array $projects): array;
}

// GitHubHttpClient
final class GitHubHttpClient {
    public function __construct(string $token);
    public function getClient(): Client;  // Guzzle Client
}

// CleanupOrphanedSitesFlow return type
/** @return array{success: bool, orphaned_sites: array<int, array{site_id: int, domain: string, pr_number: int}>, deleted: int, failed: int, error_message: string} */
```

## Edge Cases

- **No projects configured:** Returns `success: true` with empty orphaned list
- **No site list support in provider:** Returns `success: false` with error message
- **GitHub API failure:** Returns `success: false` with error; does not delete any sites
- **Rate limiting (429):** Handled as a non-retryable error; returns failure with message
- **Pagination exhaustion:** Loop exits when `count($prs) < $perPage` (last page)
- **Empty PR list:** `handle()` breaks out of pagination loop immediately
- **Invalid PR response body:** Non-array or empty response breaks loop
- **Delete failure:** Caught and counted in `$failed`; does not halt other deletions
- **Invalid server ID:** `DeleteSiteAction` throws `InvalidArgumentException` — caught at flow level as delete failure

## Acceptance Criteria

- [ ] `cleanup-orphaned --dry-run` shows orphaned sites without deleting
- [ ] `cleanup-orphaned` (with confirmation) deletes orphaned sites
- [ ] `GITHUB_TOKEN` missing causes command to fail with clear message
- [ ] `GITHUB_REPOSITORY` missing causes command to fail with clear message
- [ ] GitHub API errors (401, 403, 429) are surfaced with meaningful messages
- [ ] Pagination correctly fetches all open PRs (handles > 100 PRs)
- [ ] PR number extraction correctly matches domain pattern with numeric capture
- [ ] Only profiles named `preview` are considered for orphan detection
- [ ] Delete failures are counted and reported but do not halt other deletions
- [ ] `GetAllSitesAction` must be called to get current site list before comparison

## Open Questions / Potential Concerns

- **Rate limit backoff:** No retry logic for 429 or transient errors. Is retry intended, or is immediate failure acceptable?
- **GitHub token scopes:** The command requires `repo` scope for PR listing. Is there a minimal scope that would work for public repos only?
- **No projects using site list:** The flow finds the first project with a provider that exposes `ServerSiteListInterface`. If no project has this, cleanup cannot run. Should this be a fatal error or a silent no-op?
- **Concurrent cleanup race:** If a PR is closed and reopened before cleanup runs, the site would incorrectly be flagged as orphaned. Any mitigation intended?
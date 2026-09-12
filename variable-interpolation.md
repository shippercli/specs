---
description: "**Issue:** MAR-42 **Date:** 2026-04-23 **Status:** In Review"
---

# Spec: Variable Interpolation

**Issue:** MAR-42
**Date:** 2026-04-23
**Status:** In Review

## Overview

Variable interpolation allows database names and other identifiers to embed dynamic placeholders (`${PROJECT_NAME}`, `${PROFILE}`) and environment variables (`${ENV_VAR}`) that are resolved at runtime. The `InterpolatesNames` concern provides a reusable `interpolateDatabaseName()` method used by database config.

## Current Implementation

### Key Classes / Files

| File | Role |
|------|------|
| `app/Deployment/Concerns/InterpolatesNames.php` | Trait with `interpolateDatabaseName()` |
| `tests/Unit/Config/DatabaseConfigTest.php` | Unit tests (uses interpolation in test names) |

### InterpolatesNames trait

```php
trait InterpolatesNames
{
    private function interpolateDatabaseName(string $name, string $projectName, string $profileName): string
    {
        $name = \str_replace('${PROJECT_NAME}', $projectName, $name);
        $name = \str_replace('${PROFILE}', $profileName, $name);

        // Environment variable interpolation via regex
        $name = \preg_replace_callback(
            '/\$\{([A-Z_][A-Z0-9_]*)\}/',
            function (array $matches): string {
                $envVar = $matches[1];
                $envValue = \getenv($envVar);
                return $envValue !== false ? $envValue : '';
            },
            $name,
        ) ?? $name;

        // Cleanup: remove trailing underscores, collapse multiple underscores
        $name = \preg_replace('/_+$/', '', $name) ?? $name;
        $name = \preg_replace('/_+/', '_', $name) ?? $name;

        return $name;
    }
}
```

## Functional Requirements

**FR-001 — ${PROJECT_NAME} placeholder is replaced**
The literal string `${PROJECT_NAME}` in a name is replaced with the `$projectName` argument.

**FR-002 — ${PROFILE} placeholder is replaced**
The literal string `${PROFILE}` in a name is replaced with the `$profileName` argument.

**FR-003 — ${ENV_VAR} placeholders resolved from environment**
Any `${NAME}` pattern matching `[A-Z_][A-Z0-9_]*` is looked up via `getenv()`. If found, replaced with the env value; if not found, replaced with empty string.

**FR-004 — Trailing underscores cleaned up**
After interpolation, trailing underscores from missing env vars are removed via `preg_replace('/_+$/', '', $name)`.

**FR-005 — Multiple consecutive underscores collapsed**
`preg_replace('/_+/', '_', $name)` collapses runs of underscores into a single underscore.

**FR-006 — Null-safety via ?? fallback**
All `preg_replace` calls use `?? $name` to guard against null return from PCRE functions.

## Interpolation Examples

| Input name | PROJECT_NAME | PROFILE | Env vars | Result |
|------------|--------------|---------|----------|--------|
| `myapp` | `myproject` | `prod` | `{}` | `myapp` |
| `${PROJECT_NAME}_db` | `myproject` | `prod` | `{}` | `myproject_db` |
| `${PROJECT_NAME}_${PROFILE}` | `myproject` | `prod` | `{}` | `myproject_prod` |
| `${PROJECT_NAME}_${PROFILE}_${BUILD_ID}` | `myproject` | `prod` | `{BUILD_ID=42}` | `myproject_prod_42` |
| `${MISSING}_db` | `myproject` | `prod` | `{}` | `_db` → `db` (cleanup) |

## Edge Cases

- **Unknown env var:** Replaced with `''`, then cleanup collapses resulting `__` or trailing `_`
- **Empty projectName/profileName:** Valid — results in name starting with `_` then cleanup removes it
- **No placeholders:** Name returned unchanged
- **Mixed case env var name:** Pattern `[A-Z_][A-Z0-9_]*` requires uppercase — `$(my_var}` would not match

## Acceptance Criteria

- [ ] `${PROJECT_NAME}` replaced with project name argument
- [ ] `${PROFILE}` replaced with profile name argument
- [ ] `${ENV_VAR}` patterns resolve from environment via `getenv()`
- [ ] Unknown env vars replaced with empty string
- [ ] Trailing underscores removed after interpolation
- [ ] Multiple underscores collapsed to single
- [ ] Null returns from `preg_replace` do not cause errors
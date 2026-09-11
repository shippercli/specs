---
description: "**Issue:** MAR-43 **Date:** 2026-04-23 **Status:** In Review"
---

# Spec: Strict Type System

**Issue:** MAR-43
**Date:** 2026-04-23
**Status:** In Review

## Overview

Shipper enforces strict coding standards across the entire codebase. Every PHP file declares `strict_types=1`, all classes are `final` by default, return types are mandatory, no `mixed` types are permitted, and static analysis (PHPStan level 9) plus code style enforcement (Laravel Pint) run in CI.

## Current Implementation

### Key Files

| File | Role |
|------|------|
| `phpstan.neon` | PHPStan config, level 9, Larastan extension included |
| `pint.json` | Laravel Pint ruleset (strict, final classes, native functions) |
| `phpunit.xml` | Test suite config with `failOnRisky="true"` and `failOnWarning="true"` |
| `composer.json` | Scripts: `composer analyse`, `composer format`, `composer format:check`, `composer test` |
| `docs/STRICT_STANDARDS.md` | Full documentation of all strict rules |
| `docs/PHPSTAN_FIXES.md` | Historical fix log for phpstan configuration issues |

### phpstan.neon (critical settings)

```yaml
includes:
    - vendor/larastan/larastan/extension.neon
    - vendor/pestphp/pest/extension.neon

parameters:
    paths:
        - app
        - config
        - routes
        - bootstrap
        - tests
    level: 9
    checkExplicitMixed: true
    checkUninitializedProperties: true
    checkDynamicProperties: true
    phpVersion: 80300
    excludePaths:
        - vendor
        - tests/Pest.php
```

### pint.json (critical rules)

```json
{
    "preset": "laravel",
    "rules": {
        "declare_strict_types": true,
        "strict_comparison": true,
        "strict_param": true,
        "final_class": true,
        "void_return": true,
        "array_syntax": { "syntax": "short" },
        "native_function_invocation": { "include": ["@all"], "scope": "all", "strict": true },
        "no_unused_imports": true,
        "ordered_imports": { "sort_algorithm": "alpha" }
    }
}
```

## Functional Requirements

**FR-001 — All PHP files declare strict_types**
Every `.php` file must begin with `<?php\n\ndeclare(strict_types=1);` — enforced by Pint rule `declare_strict_types`.

**FR-002 — All concrete classes are final**
Pint rule `final_class: true` — all classes must be `final` unless explicitly designed for inheritance. No exceptions without team consensus.

**FR-003 — All method parameters have explicit types**
No parameter type may be omitted. `strict_param: true` in Pint. PHPStan `checkMissingCallableSignature: true`.

**FR-004 — All method return types are explicit**
Every method must declare a return type (`: void`, `: string`, `: array`, etc.). `void_return: true` in Pint. PHPStan level 9 enforces this.

**FR-005 — No mixed type anywhere**
PHPStan `checkExplicitMixed: true` — the type `mixed` is prohibited. All properties, parameters, and return values must have concrete types.

**FR-006 — Strict comparisons only**
Pint `strict_comparison: true` — `==` and `!=` are forbidden; `===` and `!==` must be used always.

**FR-007 — Native function calls use backslash**
Pint `native_function_invocation: { scope: "all", strict: true }` — all built-in PHP functions must be called with a leading backslash to avoid namespace resolution overhead.

**FR-008 — CI enforces all rules**
GitHub Actions workflow runs: Pint format check, PHPStan level 9 analysis, Pest tests. All three must pass for any PR to be merged.

**FR-009 — Tests fail on risky or warning**
phpunit.xml sets `failOnRisky="true"` and `failOnWarning="true"` — test suite fails if a test does not assert anything or emits a PHPUnit warning.

## PHP Version

PHP 8.3+ required. PHPStan `phpVersion: 80300`. Current `composer.json` requires `"php": "^8.3"`.

## Tool Usage

```bash
# Check code style (does not modify)
composer format:check

# Fix code style automatically
composer format

# Run static analysis
composer analyse

# Run tests
composer test
```

## Edge Cases

- **Abstract classes:** `final_class: true` does not apply — abstract classes cannot be final by PHP semantics. `final_internal_class: false` and `final_public_method_for_abstract_class: false` in Pint config allow non-final public methods on abstract classes.
- **Interfaces:** Not affected by `final_class` rule — interfaces remain interface types.
- **Exceptions:** Must still be `final` classes per the rule. Inner exception classes may extend `\Exception` but must be `final`.
- **Test files:** Excluded from PHPStan (`excludePaths: - tests/Pest.php`) but included in Pint analysis via `tests/` path in Pint config.

## Acceptance Criteria

- [ ] `composer format` produces no diff on clean codebase
- [ ] `composer analyse` exits with code 0 (no PHPStan errors at level 9)
- [ ] `composer test` exits with code 0 (all Pest tests pass, no risky/warning)
- [ ] No `mixed` types anywhere in `app/` or `config/`
- [ ] No `==` or `!=` comparisons in `app/` or `config/`
- [ ] All classes in `app/` are `final`
- [ ] All functions called with leading backslash in `app/` and `config/`
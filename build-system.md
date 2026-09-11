---
description: "**Issue:** MAR-41 **Date:** 2026-04-24 **Status:** In Review"
---

# Spec: Build System — PHAR Compilation and Distribution

**Issue:** MAR-41
**Date:** 2026-04-24
**Status:** In Review

## Overview

Shipper is built as a standalone PHAR binary using Box (box-project/box), distributed via GitHub Releases, and consumed either by direct download or via the reusable GitHub Action. The build system is defined in `box.json` and orchestrated via `composer build`.

## Current Implementation

### Key Files

| File | Purpose |
|---|---|
| `box.json` | Box configuration: entry point, output path, included dirs, compression |
| `composer.json` | Build script, dependency declarations, test/lint commands |
| `.github/workflows/build-release.yml` | CI workflow: triggers on `v*` tags, builds, tests, releases |
| `.github/actions/shipper/action.yml` | Reusable action: downloads binary from releases |
| `docs/BUILD_SYSTEM.md` | User-facing documentation of build/distribution |

### Build Process

**Local build:**
```bash
composer build
# → curl -LSso box.phar https://github.com/box-project/box/releases/latest/download/box.phar
# → php box.phar compile
# → output: builds/shipper
```

**CI build (triggered by tag push):**
1. `shivammathur/setup-php@v2` (PHP 8.3, extensions: mbstring, xml, ctype, json, zip)
2. `composer install --prefer-dist --no-dev --optimize-autoloader`
3. Download Box
4. `php box.phar compile`
5. Test: `./builds/shipper --version`
6. Create release via `softprops/action-gh-release@v1`

### Box Configuration (`box.json`)

```json
{
    "main": "shipper",
    "output": "builds/shipper",
    "directories": ["app", "bootstrap", "config", "routes", "vendor"],
    "files": ["composer.json"],
    "compression": "GZ",
    "compactors": [
        "KevinGH\\Box\\Compactor\\Php",
        "KevinGH\\Box\\Compactor\\Json"
    ]
}
```

**Included:** `app/`, `bootstrap/`, `config/`, `routes/`, `vendor/`, `composer.json`
**Excluded:** test files, dev dependencies, documentation

### Composer Scripts

```json
{
    "scripts": {
        "test": "vendor/bin/pest",
        "analyse": "php phpstan.phar analyse --memory-limit=1G",
        "format": "php pint.phar",
        "format:check": "php pint.phar --test",
        "build": [
            "[ -f box.phar ] || (curl -LSso box.phar ... && chmod +x box.phar)",
            "php box.phar compile"
        ]
    }
}
```

### Release URL Patterns

```
Latest:     https://github.com/ulties/shipper/releases/latest/download/shipper
Tagged v1:  https://github.com/ulties/shipper/releases/download/v1.0.0/shipper
```

## Functional Requirements

**FR-001 — PHAR Output**
The build produces a single executable PHAR at `builds/shipper` with GZ compression. The PHAR contains all application code, vendor dependencies, and configuration.

**FR-002 — Compactor Strategy**
Box uses `Php` and `Json` compactors to strip whitespace and reduce file size. This is lossless — PHP semantics are preserved.

**FR-003 — Semantic Version Tagging**
Releases follow semantic versioning. A tag like `v1.2.3` triggers the build workflow, creating a GitHub release with the binary attached.

**FR-004 — Binary Verification**
After building, the CI runs `./builds/shipper --version` to verify the binary is valid and executable before creating the release.

**FR-005 — No Dev Dependencies in Binary**
`composer install` uses `--no-dev`, ensuring dev-only packages (Pest, Pint, PHPStan, Mockery) are not included in the PHAR.

**FR-006 — GitHub Action Consumption**
The `.github/actions/shipper/action.yml` downloads the binary from the tagged release URL. It handles `latest` via a redirect and specific versions via explicit URLs.

## Data Contracts

### Build Output Structure

```
builds/
└── shipper    # executable PHAR, ~15-25MB (estimated)
```

### Release Asset

The `build-release.yml` attaches `builds/shipper` as a release asset with content-type inferred from filename.

## Edge Cases

- **Box not present:** `composer build` downloads Box via curl if `box.phar` does not exist; fails with message if curl fails
- **Box validation failure:** `php box.phar compile` fails if `box.json` is invalid; CI would catch this
- **Missing PHP extensions:** The CI step installs extensions: `mbstring, xml, ctype, json, zip`. If any are missing locally, the build fails locally but CI succeeds
- **Release already exists:** `softprops/action-gh-release` will overwrite an existing draft release but will not re-upload if assets are unchanged
- **Tag without `v` prefix:** `build-release.yml` triggers on `v*`; a tag `1.0.0` (no `v`) would not trigger the workflow

## Acceptance Criteria

- [ ] `composer build` produces a valid `builds/shipper` PHAR
- [ ] `./builds/shipper --version` runs without errors
- [ ] PHAR includes `app/`, `bootstrap/`, `config/`, `routes/`, `vendor/` directories
- [ ] PHAR does NOT include test files or dev vendor directories
- [ ] `v1.0.0` tag push triggers `build-release.yml` workflow
- [ ] Workflow creates a GitHub release with `builds/shipper` attached
- [ ] `softprops/action-gh-release` uses `GITHUB_TOKEN` from secrets (no manual token needed)
- [ ] Box compactor uses `KevinGH\Box\Compactor\Php` and `KevinGH\Box\Compactor\Json`
- [ ] Compression is `GZ` (gzip)

## Open Questions / Potential Concerns

- **No SHA256 checksums published:** The build does not generate or publish checksums. Users cannot verify binary integrity after download. Should `SHASUM256` be added as a release asset?
- **No multi-platform builds:** Currently only one PHAR is built (presumably Linux). No Darwin/macOS or Windows variants. Is cross-platform support intended?
- **Box version pinning:** `box.phar` is downloaded from `latest` — no version pin. A breaking Box release could break builds silently. Should Box be pinned to a known-good version?
- **Version embedding:** `--version` output shows Laravel Zero's version, not the git tag or commit hash. For debugging, embedding the git commit SHA would be valuable (mentioned as a future enhancement in docs)
- **Homebrew/Docker:** The docs list these as potential future enhancements but no current implementation exists. Any roadmap for these?
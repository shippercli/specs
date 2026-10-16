---
description: "Testing requirements, source-scoped evidence and remaining coverage and live-verification gaps."
---

# Spec: Shipper Test Coverage Initiative

**Date:** 2026-04-26
**Status:** Draft

## Current evidence and historical context

The original proposal described missing tests and a CLI-internal provider-manager
architecture. Its claim of zero meaningful coverage is not a current inventory.

Reviewed on 2026-10-16, CLI main revision
[`cb86caeef6704e14990e5e4f324d66e63df25d52`](https://github.com/shippercli/cli/tree/cb86caeef6704e14990e5e4f324d66e63df25d52)
contains feature tests for configuration and commands, provider-adapter tests,
action tests, and flow tests. Examples include:

- `tests/Feature/ConfigLoaderTest.php`
- `tests/Feature/ApplyCommandTest.php`
- `tests/Feature/CleanupOrphanedCommandTest.php`
- `tests/Unit/Actions/LoadConfigurationActionTest.php`
- `tests/Unit/Deployment/ContractDeploymentProviderAdapterTest.php`
- `tests/Unit/Flows/ApplyDeploymentFlowTest.php`
- `tests/Unit/Flows/ValidateConfigurationFlowTest.php`

The combined local integration of open CLI audit PRs at
`e16ba2b58b643e2de0272fe665d03f201db811ed` was rerun with
`php vendor/bin/pest --no-coverage --compact`: **213 tests / 585 assertions passed**.
This is a dated, unmerged integration snapshot, not a released-package claim or
code-coverage measurement. Test existence/counts do not prove every branch or
provider operation is tested.

Provider plugins have their own repositories, dependencies and test suites. Do
not infer plugin coverage from a passing CLI suite or target historical manager
classes that no longer implement the plugin operation being reviewed.

## Required test layers

| Layer | Evidence it provides | What it does not establish |
| --- | --- | --- |
| Configuration/value tests | Parsing, interpolation, precedence and invalid-input behavior | Remote provider behavior |
| Command/flow tests | Planning, confirmation, exit status and orchestration | Successful deployment on a live account |
| Provider unit tests | Scope/refusal rules and response handling under controlled inputs | Actual SDK serialization or account permissions |
| SDK HTTP fixtures | Serialized requests, response shapes, request ordering and redaction | Real database connectivity or provider consistency timing |
| Production package consumer | Archives, production autoloading, contracts and plugin discovery | Full CLI/provider workflows or release availability |
| Browser tests | Observed client interaction/layout under tested settings | Complete accessibility or all viewport/browser support |
| Live lifecycle tests | Actual account permissions, deployment and cleanup behavior | Untested accounts, providers, migrations or races |

Record source revisions, dependency versions, commands and resulting state for
each layer. Keep fixtures and generated reports free of real secrets.

## Priority regression requirements

### Configuration and CLI orchestration

- Verify literal secret preservation and exactly intended interpolation, including
  nested placeholder-looking text, quotes, empty values and profile precedence.
- Reject ambiguous/non-boolean control options rather than silently coercing them.
- Refuse blocked plans and invalid ownership before execution. Force skips
  confirmation only; cancellation must perform no mutation.
- Propagate provider failure, timeout and unreadable required evidence as failure.
- Exercise plugin discovery and adapters with installed Composer packages.

### Provider lifecycle and resource boundaries

- Cover unowned/cross-scope resources, ambiguous names, malformed inventories and
  changing identities before inferred mutation or deletion.
- Verify complete effective ownership documents. Explicit API failures invalidate
  matching marker content; unfinished trailing input must not authorize cleanup.
- Cover multiple database mappings, literal passwords, existing credential reuse,
  ignored writes, persistence readback and fresh retry after partial provisioning.
- Assert later configuration/deployment operations do not run after a prerequisite
  fails. Assert cleanup preserves unrelated resources and reports partial failure.
- Test supported preview/rollback behavior and explicit unsupported operations.

### Test correctness

- Assert the exact writes and refused writes, not just that an exception occurred.
- Keep assertions outside broadly caught operation scopes so assertion failures
  cannot be mistaken for expected provider exceptions.
- Demonstrate that critical refusal regressions fail against the faulty behavior
  or with the corresponding guard disabled, then restore and retest the source.
- Use valid controls alongside rejection cases to detect accidental blanket refusal.

## Running and measuring tests

Use each repository's documented scripts and dependency versions. The reviewed
CLI uses Pest through `composer test`; Pint and PHPStan are separate checks.
Provider repositories may use Pest, PHPUnit, Python HTTP fixtures or helper tests.
No framework is implied for every plugin by this specification.

Coverage percentages require an actual instrumented report with the included
source paths, exclusions, runtime/driver and revision recorded. No current overall
percentage or line/branch target is established by this review. A count of passing
tests must never be relabeled as a coverage percentage.

CI evidence must belong to the current PR head and required check context.
Historical green checks and local results are distinct evidence. A green build,
manifest validator or mocked suite alone cannot close live acceptance gaps.

## Acceptance checklist

- [ ] Maintain a requirement-to-test map for current CLI and plugin architecture.
- [ ] Verify current-head CI checks and supported runtime matrices.
- [ ] Generate and review instrumented coverage before making coverage claims.
- [ ] Demonstrate critical refusal regressions and valid controls.
- [ ] Verify production package installation and plugin discovery.
- [ ] Verify full create/re-apply/deploy/status/logs/cleanup flows on supported
      live accounts, including actual application database authentication.
- [ ] Verify partial-operation recovery, legacy migration and lifecycle concurrency.

Unverified checklist items remain open even when a particular source snapshot's
unit, HTTP or packaging checks pass.

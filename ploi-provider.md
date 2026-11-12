---
description: "Ploi plugin architecture, deployment operations, source baseline and audited lifecycle limitations."
---

# Spec: Ploi Provider

**Issue:** MAR-38
**Date:** 2026-04-23
**Status:** In Review

## Source scope

Reviewed on 2026-10-16 against provider main revision
[`fd60b25bfa4f3df58f0dc61639f3b4a4f6a05505`](https://github.com/shippercli/provider-ploi/tree/fd60b25bfa4f3df58f0dc61639f3b4a4f6a05505).
This is a source baseline, not a claim about the installed or latest released
package. Proposed ownership, response-validation and connection-delivery changes
are in [provider PR #26](https://github.com/shippercli/provider-ploi/pull/26), reviewed
at `3d7fc1418e089de372cdb514a6558b5d4bfc8455`; they remain unmerged.

## Package architecture

`shippercli/provider-ploi` is a Composer package of type `shipper-plugin`, requiring
PHP 8.3+, `ploi/ploi-php-sdk` 1.9-compatible releases and `shippercli/contracts` 1.2.
Its `extra.shipper-plugin` metadata identifies `ShipperCli\ProviderPloi\PloiPlugin`.
The plugin maps provider key `ploi` to `ShipperCli\ProviderPloi\PloiProvider`.

The provider implements shared deployment, capabilities, status and logs contracts.
The CLI discovers installed plugins through Composer and adapts their contract
implementations. The Ploi package does not extend a CLI `AbstractProvider` or
register a set of manager classes inside the CLI.

| Package file | Responsibility |
| --- | --- |
| `src/PloiPlugin.php` | Advertise the provider mapping. |
| `src/PloiProvider.php` | Validate, plan, provision/configure, deploy, inspect and clean up resources. |
| `src/ServerLifecycleClientInterface.php` | Server-lifecycle client boundary. |
| `src/PloiServerLifecycleClient.php` | Ploi server-lifecycle operations. |

The audit branch additionally introduces `src/PloiApiClient.php` for strict
transport/response behavior. Do not assume that file exists in the main baseline.

## Public contract and configuration

The provider exposes `getName()`, `capabilities()`, `validate()`, `plan()`,
`apply()`, `destroy()`, `getLastError()`, `postApply()`, `status()` and `logs()`.
Site listing and scoped deletion helpers support lifecycle orchestration.
`apply()` and `destroy()` return booleans; they do not return the historical
`SiteResult` or `OperationResult` objects described by earlier designs.

The SDK factory method `getClient()` is protected. Server/site state and the
internal deployment-log helper are not the public SDK access API.

Core validation requires a nonempty resolved API key, a server target, a profile
domain, and repository provider/name. A static server ID is used when no server
lifecycle configuration is supplied; lifecycle mode has its own validation.
Consult the provider README and installed version for the complete field schema.
The main baseline's `deployment_timeout` defaults to 60 seconds.

## Deployment flow

1. Validate local configuration and resolve the target server.
2. Inspect the target site or create it, then configure its repository as needed.
3. Provision declared databases and configure the site's deployment settings.
4. Trigger deployment through the Ploi SDK.
5. Poll site state and inspect deployment logs for failure until completion or
   timeout. Timeout is a failure and does not imply the remote deployment stopped.

The main source implements site PHP version, Nginx configuration, aliases,
redirects, firewall rules, queues, scheduled jobs and daemons as provider methods.
A capability declaration or method's presence alone does not prove successful
operation on a live account. API access and provider support remain prerequisites.

The audit proposal changes existing-resource checks and runs configuration steps
sequentially, refusing later mutations after a prerequisite fails. These are audit
changes, not guarantees supplied by the main baseline above.

## Ownership and database acceptance requirements

- Existing sites require successful, readable evidence matching the exact project
  and profile before ownership-inferred mutation or cleanup.
- Parse complete effective environment assignments; an unfinished trailing value
  or explicit failed API response must not establish ownership.
- Multiple databases require distinct connection-variable mappings. Preserve
  existing credentials on re-apply; do not rotate them implicitly.
- Deliver each confirmed connection before creating the next database and verify
  stored selected values before dependent configuration or deployment.
- Preserve unrelated environment content. Refuse unreadable or malformed content
  rather than replacing it as if it were empty.
- A fresh retry after a later creation fails must reuse earlier confirmed resources
  and credentials, creating only missing resources.
- Never expose generated/configured secrets in plans, logs or diagnostics.

These requirements are proposed in
[contracts PR #5](https://github.com/shippercli/contracts/pull/5). The audited
implementation has local unit and real-SDK simulated HTTP evidence, including
partial retry and environment readback. Live database authentication and complete
account lifecycle behavior remain unverified.

## Limitations and unresolved behavior

- Rollback is declared unsupported by the reviewed package; do not promise an
  automatic rollback operation.
- Server lifecycle and preview support are partial. Scope and cleanup permissions
  require verification on the intended account.
- Existing database association must already match the site. The audited path
  refuses missing/mismatched association; it does not implement attach-existing.
  [CLI issue #134](https://github.com/shippercli/cli/issues/134) remains open.
- SQL declaration labels do not select or prove the actual Ploi server engine.
- Creation and credential storage are not atomic. An ambiguous response or storage
  failure may leave a resource behind. There is no durable credential journal or
  automatic multi-database rollback in the reviewed audit branch.
- Immediate readback can fail when persistence is delayed. Inspect actual state
  before retrying; do not infer that creation failed from a readback failure.
- Local tests and package integration do not prove concurrent-operation safety,
  legacy migration, live credential connectivity or release readiness.

## Review acceptance checklist

- [ ] Verify installed plugin discovery and provider key `ploi`.
- [ ] Verify validation and blocked/read-only planning against actual inputs.
- [ ] Verify unowned and differently scoped existing resources are preserved.
- [ ] Verify create/re-apply, deployment polling, status and logs on a live account.
- [ ] Verify generated credentials authenticate from the intended application.
- [ ] Verify partial creation/storage failure and fresh-process recovery.
- [ ] Verify scoped cleanup, legacy migration and concurrent lifecycle behavior.

Keep unverified items open. The existence of a draft fix or a passing simulated
API test is not completion evidence for the live lifecycle checklist.

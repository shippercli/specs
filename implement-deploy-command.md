---
description: "Unimplemented proposal for redeploying an existing site without rerunning provisioning."
---

# Spec: Implement DeployCommand

**Date:** 2026-04-26
**Status:** Draft

## Current source and command availability

Reviewed on 2026-10-16 against CLI main revision
[`cb86caeef6704e14990e5e4f324d66e63df25d52`](https://github.com/shippercli/cli/tree/cb86caeef6704e14990e5e4f324d66e63df25d52).
That source contains `ApplyCommand`, `ApplyDeploymentFlow` and
`ExecuteDeploymentAction`. It does not contain `app/Commands/DeployCommand.php`.
The earlier claim that a current DeployCommand is a stub/no-op was incorrect.

A production-installed local integration of the open CLI audit PRs at
`e16ba2b58b643e2de0272fe665d03f201db811ed` also lists `apply`, not `deploy`, in
`shipper list --raw`. That fixture is not a published release.

Use the installed CLI's command help to determine available operations. The
reviewed deployment entry point is:

```bash
shipper apply <project> --profile=production --config=shipper.yml
```

`apply` validates and plans before execution and normally asks for confirmation.
`--force` skips that prompt; it is not mandatory for an interactive deployment.
It does not authorize bypassing provider validation or ownership checks. The
proposed blocked-plan enforcement is tracked in
[CLI PR #154](https://github.com/shippercli/cli/pull/154); do not infer that every
released version enforces that proposal.

The apply path can provision and update resources. It must not be presented as
an existing-site-only redeploy command.

## Proposed behavior, not an available command

A future `shipper deploy` could trigger deployment for an existing owned site
without rerunning provisioning. Its interface is a proposal:

```text
shipper deploy <project> [--profile=<name>] [--config=<path>] [--force]
```

Before implementation, define an explicit provider operation or contract for
redeploying an existing target. Calling the general provider `apply()` method is
insufficient because it can create resources or change configuration. Historical
manager interfaces such as `DeploymentExecutorInterface` are not a substitute
for verifying the current plugin API.

### Required sequence

1. Load and validate the selected project and profile.
2. Resolve a supported provider operation and inspect the target read-only.
3. Prove exact project/profile ownership and a single existing target. Refuse
   absence, collisions, ambiguous inventory and failed evidence reads.
4. Show the intended existing-target action and obtain confirmation unless
   `--force` was supplied. Force must not bypass the preceding checks.
5. Trigger only the reviewed redeployment operation and wait for its result.
6. Report provider status and available redacted logs. A timeout is failure, not
   proof that the remote deployment has stopped.

### Difference from apply

| Behavior | Reviewed apply command | Proposed deploy command |
| --- | --- | --- |
| May provision resources | Yes, according to provider behavior | Must not |
| May configure environment, workloads or runtime | Yes, according to provider support | Must not implicitly |
| Triggers deployment | Through provider apply | Through an explicit existing-target operation |
| Confirmation | Normally interactive; `--force` skips prompt | Same proposed semantics |
| Missing target | Provider may create it | Must refuse |
| Ownership checks | Required by provider acceptance rules | Required before triggering deployment |

## Acceptance criteria for a future implementation

- [ ] Define the provider API and advertise unsupported operations clearly.
- [ ] Register and document the command only when implemented.
- [ ] Refuse missing, unowned, cross-scope or ambiguous targets without mutation.
- [ ] Verify no site/database/workload creation or configuration writes occur.
- [ ] Verify confirmation cancellation performs no deployment.
- [ ] Verify `--force` cannot bypass ownership or validation failures.
- [ ] Return success only for a confirmed successful provider result; return failure
      for failed status, unreadable required evidence or timeout.
- [ ] Document whether logs are streamed or collected and redact secrets.
- [ ] Test success, missing target, provider rejection, failed deployment, timeout
      and remote deployment continuing after a local timeout.
- [ ] Verify the complete behavior against supported live provider accounts.

The current `apply` command and a passing command-list check do not satisfy this
future redeploy contract. This document remains a draft feature proposal.

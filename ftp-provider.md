---
description: "Released FTP/FTPS file deployment contract, ownership journal, transport security, and verification scope."
---

# Spec: FTP / FTPS Provider

**Date:** 2026-11-26
**Status:** Implemented in v1.0.0; hosting compatibility beta

## Package and scope

[`shippercli/provider-ftp` v1.0.0](https://github.com/shippercli/provider-ftp/releases/tag/v1.0.0)
uploads prebuilt local files to an existing FTP account. It requires PHP 8.3+,
`ext-curl`, and `shippercli/contracts:^1.2`. Composer discovers package type
`shipper-plugin` through `extra.shipper-plugin`, whose `FtpPlugin` registers
provider name `ftp`. No provider-specific dependency is added to the CLI.

Install the released package with the Composer CLI distribution:

```bash
composer config repositories.shipper-ftp git https://github.com/shippercli/provider-ftp.git
composer require 'shippercli/cli:^1.0.10' 'shippercli/provider-ftp:^1.0'
```

This installation uses Composer's Git driver and does not require Packagist
registration. Run the local installation with `vendor/bin/shipper`.

## Configuration contract

Provider settings are `host`, `port` (default 21), `username`, `password`,
`protocol` (default `ftps`), `timeout` (default 30 seconds per operation), and
optional `ca_file`. Unknown options, unresolved credentials, URL-shaped hosts,
control characters, invalid ports, and invalid timeouts fail validation.

Each project's `path` must resolve to an existing local directory. Its profile
contains `ftp.local_path` (default `.`), `ftp.remote_path`, and optional
`ftp.exclude`. The local artifact must resolve inside the project directory.
The remote path must be absolute within the FTP account's namespace and must
identify a dedicated non-root directory. Exclusions are exact relative paths
and directory prefixes, without glob syntax.

The source walker rejects symlinks, special files, unreadable files, unsafe
paths, and artifacts with no deployable files. It always excludes `.git`,
`.hg`, `.svn`, `.github`, `node_modules`, `.env*`, `auth.json`, `shipper.yml`,
`shipper.yaml`, `.phpunit.cache`, and reserved `.shipper-ftp*` entries. These
exclusions do not replace review of build artifacts for other secrets. Empty
directories and filesystem permission metadata are not uploaded.

## Transport requirements

- `ftps` requires explicit TLS on control and data connections. Certificate
  chain and hostname verification cannot be disabled. Private CAs use a
  readable PEM `ca_file`.
- `ftp` explicitly opts into plaintext transfer, including credentials.
- Implicit FTPS, SFTP, IPv6 host syntax, redirects, and proxies are unsupported.
- Passive data connections use the control connection's peer address rather
  than trusting a PASV-advertised alternate IP.
- The server must support MLSD with `file` and `dir` types, absolute paths,
  MKD, STOR, RETR, RNFR/RNTO, DELE, and replacement of existing files by rename.
  Malformed listings, unsupported types, and reported symlinks fail closed.
- Transport errors expose numeric cURL/FTP codes, without credentials, URLs,
  or raw server replies. Files are buffered individually in PHP memory.

## Lifecycle and ownership

`validate` and `plan` inspect configuration and the local artifact without
connecting to the server. A plan is not proof of remote permissions or
ownership. `apply` performs the following operations:

1. Walk and, when absent, create remote parent directories. A listing failure
   is a failure, never evidence that a directory is empty.
2. Require an empty first-deployment directory or a matching valid manifest.
   Never adopt an existing website. Check every desired destination before
   claiming ownership or uploading application files.
3. Save `.shipper-ftp.json` with state `deploying` and the union of previous
   and desired relative filenames. Its owner is a SHA-256 identity of project
   name, profile name, and remote path.
4. Upload each file to a deterministic temporary name under the deployment
   root, then rename it into its destination.
5. Remove stale tracked files and their upload temporary files. Save the
   desired file list, state `deployed`, and an application timestamp.

The manifest contains no passwords or local source paths. It is deployment
state, not cryptographic protection against someone with remote write access.
Restrict web access to reserved `.shipper-ftp*` files because their filenames
can reveal application structure.

`status` returns saved state, path, file count, and timestamp, or `not_deployed`.
It does not check application health or remote file integrity. `destroy`
requires matching ownership and deletes only tracked files, their temporary
uploads, and the deployment journal. Missing owned files are harmless;
unrelated files and all directories, including the root, are retained.
Tracked files remain deployment-owned even after manual edits.

## Failure and recovery contract

Failed application uploads retain the `deploying` journal so that another
apply can retry or destroy can clean tracked partial files. A valid matching
initial `.shipper-ftp-state.tmp` can be recovered when it is the directory's
only entry. A truncated or invalid initial journal requires manual inspection
and cleanup. Invalid JSON, unsafe manifest paths, mismatched owners,
untracked destination files, and directory/file conflicts stop mutation.

Operations are not atomic across multiple files and provide no portable
exclusive lock. Serialize apply/destroy per remote directory. An interrupted
deployment can expose a mixture of old and new files. File/directory layout
changes can require manual removal of retained empty directories or a new
deployment path. Server-hidden aliases cannot be detected; use a dedicated
restricted FTP account.

## Capability state

| Capability | State | Scope |
|---|---|---|
| `app_deploy` | Supported | Prebuilt file upload, repeat deployment, tracked cleanup |
| `profiles` | Supported | Separate artifact paths, remote paths, and exclusions |
| `observability` | Partial | Remote manifest state only |
| All other shared capabilities | Unsupported | No runtime or infrastructure management |

There are no builds, shell commands, migrations, environment generation,
website certificate provisioning, DNS/domain management, servers, databases,
workers, cron jobs, daemons, runtime logs, rollback, or automatic PR previews.
Unsupported declared resource sections and profile runtime settings are
rejected. Deploy an earlier prepared artifact to restore earlier content.
FTPS transport encryption does not provision the website's HTTPS certificate.

## Verification evidence

The [release CI run](https://github.com/shippercli/provider-ftp/actions/runs/45862668898)
passed on PHP 8.3, 8.4, and 8.5 for release source
`10d35bdec5f171652550624085f50108ff6b7f60`. The release includes 74 unit tests
with 319 assertions, disposable real FTP/explicit-FTPS lifecycle integration,
TLS/authentication rejection checks, dependency auditing, and distribution
checks. A fresh public tagged Composer installation with CLI v1.0.10 also
passed discovery, validate, plan, apply, status, and destroy against disposable
FTP and FTPS servers.

This evidence covers the tested servers and install path. It does not claim
compatibility with every hosting provider. See the package's
[operations and recovery guide](https://github.com/shippercli/provider-ftp/blob/v1.0.0/docs/operations.md)
for operational assumptions and limitations.

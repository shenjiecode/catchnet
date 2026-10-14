# Catchnet

English | [简体中文](README.zh.md)

Catchnet (罗网) is a self-hosted AI scenario testing harness. It reads project requirements, code changes, and existing scenarios, plans tests, executes browser or command-line actions, and asks an independent Reviewer to audit the evidence. It archives scenarios, reports, and confirmed bugs to GitHub.

One administrator can manage multiple projects. Different projects can run concurrently within a configured limit; requests for the same project run sequentially. Use trusted repositories and non-production environments. Catchnet does not test production data or automatically fix product code.

The package version is currently 0.7.1. Published versions and tags are listed in [GitHub Releases](https://github.com/shenjiecode/catchnet/releases); the development branch includes unreleased work.

- [How it works](#how-it-works)
- [Deployment](#deployment)
- [Getting started](#getting-started)
- [Existing instances](#existing-instances)
- [Development and validation](#development-and-validation)
- [Operations and security](#operations-and-security)
- [Versions](#versions)
- [Name and migration](#name-and-migration)
- [License](#license)

## How it works

A normal Run uses three Agent configurations and four isolated sessions: Main planning → Runner → Reviewer → Main finalization. Default thinking levels are low, off, low, and off. Roles exchange controlled artifacts; a Runner's narrative is not execution evidence.

- Each Run pins an immutable `target_commit`. Test code must first enter the target repository's `scenario-testing` branch.
- Main can maintain scenarios, but cannot change product code. Scenario changes requiring review go through pull requests.
- Scenarios and final reports live under `docs/scenario-testing/` in the target repository. Catchnet retains detailed Run records, while screenshots and other evidence use object storage.
- Model, browser, and object storage settings belong to the deployment. Repository tokens, test accounts, and cleanup settings belong to individual projects.
- Execution images are prepared from pinned commits, and Runs record immutable image IDs. Catchnet supports managed test applications, existing test environments, and repository-only testing. Images isolate toolchains; they are not a sandbox for malicious code. The built-in image provides Node; other toolchains require a project Dockerfile or a saved startup definition.

## Deployment

Prefer Docker Compose. Catchnet accesses the Docker Engine through its socket and has substantial host privileges. Deploy in a trusted environment and restrict access to both the service and socket. Use HTTPS or a trusted reverse proxy for deployed instances, and set `CATCHNET_ALLOWED_ORIGIN`.

### New empty instance

**These commands initialize a new, empty data volume only.** Read [Existing instances](#existing-instances) and [Name and migration](#name-and-migration) before handling an old database. The startup entry point does not create an empty database or migrate an old instance automatically.

```bash
# Replace these examples. Never commit real passwords or master keys.
export CATCHNET_ADMIN_PASSWORD='replace-with-a-long-random-password'
export CATCHNET_MASTER_KEY='replace-with-a-long-random-master-key'
# On Linux, set CATCHNET_DOCKER_GID if the Docker socket group is not root:
# stat -c '%g' /var/run/docker.sock

docker compose build
docker compose run --rm --no-deps catchnet npm run db:migrate
docker compose run --rm --no-deps catchnet npm run db:multi-project -- backup /data/upgrade-backup
docker compose run --rm --no-deps catchnet npm run db:multi-project -- upgrade-empty /data/upgrade-backup
docker compose run --rm --no-deps catchnet npm run db:multi-project -- verify
docker compose up -d --build
curl --fail http://127.0.0.1:3000/health
```

Open the console at <http://127.0.0.1:3000/>. Compose binds the host port to `127.0.0.1` and stores data in the `catchnet-data` volume. `docker compose down` stops services without deleting that volume.

The administrator password initializes an empty database; it does not replace an existing password hash. The master key derives Secret Store encryption keys. Neither plaintext value is stored in SQLite. Docker Secret files are supported through `CATCHNET_ADMIN_PASSWORD_FILE` and `CATCHNET_MASTER_KEY_FILE`; direct environment values take precedence. Keep the master key with your recovery materials.

The development version defaults to two concurrent projects. `CATCHNET_MAX_CONCURRENT_PROJECTS` accepts integers from 1 to 8; empty or invalid values prevent startup. The console shows the effective capacity. Before an upgrade, stop accepting requests, wait for active work, and back up the instance. Start at 1 if necessary, then raise the limit after checking machine, browser, and provider capacity. This setting does not roll back the database. Projects sharing test accounts or application databases still require data isolation. See the [concurrency plan](docs/changes/luowang-project-concurrency/plan.md) for implementation and validation status.

### Native development

Use Node.js 24 and npm. When `better-sqlite3` has no matching prebuilt binary, Linux requires `python3`, `make`, and `g++`. Windows requires Python 3 and Visual Studio 2022 Build Tools with the Desktop development with C++ workload, including MSVC and the Windows SDK. Re-run `npm ci` after installing them and verify SQLite with `npm run doctor`. An installation using `--ignore-scripts` is not a substitute for a working runtime. The Linux image includes the native build tools.

The following commands assume the database has already been initialized or upgraded:

```bash
npm ci
npm run doctor
npm run build
npm start
```

The default address is `127.0.0.1:3000`, and the default data directory is `/data`. Set `CATCHNET_DATA_DIR` to a writable directory for local development. `npm run dev` starts Vite and the development server. `doctor` checks the Node version and native database module; reinstall dependencies if it fails, or use the quality container.

## Getting started

1. Sign in and configure the provider and Main, Runner, and Reviewer models in global settings. Reviewer must accept image inputs. Save browser and object storage settings, then check connections in system status.
2. Connect a trusted GitHub repository. Private repositories require a token that can read repository identity. New projects remain paused.
3. Configure a non-production URL and, where needed, an execution Dockerfile. Configure test account/password and cleanup URL/token pairs together.
4. Prepare or rebuild the image, inspect readiness checks, and explicitly check and enable the project. Saving settings or building an image does not enable it. A changed target commit requires a suitable prepared image.
5. If the scenario branch does not exist, select and confirm a trusted source branch, tag, or commit on the testing page. Otherwise, incorporate new changes or start a normal Run.
6. Follow execution on the testing page and review conclusions, reports, evidence, and archive status in test history.

Start troubleshooting with Run readiness and system status. Inspect Dockerfile, Engine, and build logs for image failures; inspect the pinned commit and error for queue failures. A normal Run cannot replace first-time branch initialization. Settings are project-scoped, tokens are masked, and navigation cannot retarget queued requests.

The HTTP API requires an administrator session. Account endpoints use `/api/account`; deployment settings and secrets use `/api/deployment` and `/api/deployment/secrets/:key`; project endpoints use `/api/projects/:projectId/...`. Scenario, report, queue, Run, and evidence requests require a project ID. Legacy unscoped business APIs are unsupported. Secret APIs return configuration status and masked values only.

## Existing instances

The following describes historical single-project to multi-project upgrades. For an instance using the old English name, follow [Name and migration](#name-and-migration): these commands alone do not make its encrypted data compatible with the renamed development version.

For a v0.6.0 single-project instance, stop the service, back up the database, repository, reports, and master key, and manually verify historical repository ownership. Test upgrade and rollback on an isolated copy before touching a persistent instance.

The sequence is `npm run db:multi-project -- inspect` → `backup <new-directory>` → `upgrade-project <backup-directory> <reviewed-fingerprint>` → `verify`. Use `upgrade-empty` only for an empty instance. Upgrade commands neither create a missing database nor replace ownership review. Rollback must restore the database, repository, reports, and matching master key together.

Early v0.6.1 development instances may also need an offline `npm run db:multi-project -- upgrade-index <new-backup-directory>`, followed by `verify`. This backs up SQLite and fixes project-scoped report index uniqueness without rewriting Git reports, Runs, or object storage evidence. A fresh v0.6.0 upgrade includes it. The backup directory must not exist; an already completed migration returns `already_complete`. See the [historical upgrade plan](docs/changes/luowang-multi-project/plan.md).

## Development and validation

The Dockerfile's `quality` and `runtime` targets pin the Node image digest and use the lockfile's Playwright Chromium. CI and production use the same browser, without relying on a host installation.

```bash
docker build --target quality --tag catchnet:quality .
docker run --rm --init --ipc=host catchnet:quality npm run test:e2e
docker build --target runtime --tag catchnet:runtime .
```

Override mirror defaults with `NODE_IMAGE`, `NPM_REGISTRY`, `PLAYWRIGHT_DOWNLOAD_HOST`, `DEBIAN_MIRROR`, and `DEBIAN_SECURITY_MIRROR`. See the [Dockerfile](Dockerfile) and Compose configuration. Do not commit local `.env` files or substitute a host browser for the image's browser.

```bash
npm run test:acceptance:preflight # Readiness inspection; no model calls or acceptance claim
npm run test:acceptance:local     # Real Pi SDK with local controlled services; no external credentials
npm run test:acceptance:live      # Connectivity checks and verification of completed real Runs
npm run test:acceptance:release   # Public quality checks, local, and live
```

`local` uses temporary Git, SQLite, browser, and model-protocol services through production Pi sessions. Local test doubles can prove only `local.status=passed`. Missing live inputs produce a blocked status and a nonzero exit code. Until live passes, `release.status` remains blocked or failed. `npm run test:acceptance` is an alias for local acceptance; CI does not read live secrets.

Reports default to `.cynos/acceptance/<timestamp>-<mode>/`. Preflight reads existing readiness results, records inspection time, and produces a separate report without acceptance-criterion pass claims. Live checks retain model probes and temporary object storage checks but do not repeat completed Runs. Each invocation uses a new output directory. Past release validation does not establish readiness for a new deployment.

See the [release verification guide](docs/release-verification.md) for inputs, reruns, multi-project history checks, GitHub smoke tests, and browser preflight. The [0.7.1 plan](docs/changes/luowang-v071-release-verification/plan.md) records that version's evidence. Unrun live or release checks are not passes.

## Operations and security

- Use non-production environments, synthetic data, and dedicated accounts. Keep secrets out of Git, logs, and reports; use Secret Store or Docker Secrets.
- Project Run containers receive the current source copy, without the Docker socket, Catchnet database, master key, other project directories, test account credentials, or cleanup tokens.
- Startup reconciles leftover Run containers using the database's `instance_id` labels before recovering the queue. Image deletion requires matching instance, project, build, and target labels, with no ready/Run references. Images rejected by Docker for deletion remain in place.
- Verify ownership of unlabeled resources and build caches manually. Do not run a global `docker image prune` on a shared Engine.
- Measure capacity for actual workloads. A historical machine with about 4 GiB of memory only demonstrated isolated deployment and cleanup, not a supported minimum or a joint model/browser Run.

`npm run ops:inventory` provides a read-only inventory of instance containers, image references, and candidate space. See the [operations guide](docs/operations.md) for backup, recovery, and drills.

## Versions

See [GitHub Releases](https://github.com/shenjiecode/catchnet/releases) for published versions.

| Version                                                              | Main changes                                                                            | Evidence or limits                                                                                    |
| -------------------------------------------------------------------- | --------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| 0.7.1                                                                | Release-checker regression, inexpensive preflight, evidence verification, documentation | [Implementation and validation](docs/changes/luowang-v071-release-verification/plan.md)               |
| [0.7.0](https://github.com/shenjiecode/catchnet/releases/tag/v0.7.0) | Console redesign with project configuration, preparation, tests, history, and scenarios | Project isolation and production APIs; deployment-scoped system settings                              |
| 0.6.2                                                                | Onboarding, runtime diagnosis, Docker inventory, frozen model evaluation                | Deployment and troubleshooting improvements                                                           |
| 0.6.1                                                                | Multiple projects per administrator, execution images, offline upgrade                  | Database/API incompatibilities require offline upgrade despite the patch version                      |
| 0.6.0                                                                | Built-in code reading, pinned read receipts, plan references                            | Receipts prove access, not understanding; [evidence](docs/changes/luowang-code-understanding/plan.md) |

## Name and migration

The English name is **Catchnet**, the Chinese name remains **罗网**, and the repository is [`shenjiecode/catchnet`](https://github.com/shenjiecode/catchnet). The Chinese logo is unchanged.

The development version uses `CATCHNET_*` variables, `catchnet.db`, the `catchnet` Compose service, and the `catchnet-data` volume. Docker labels, report protocols, and encryption domain identifiers have also changed. This is a fresh-deployment transition before external adoption, without a legacy-name compatibility layer.

Stop and back up old development instances, including their database, master key, and Run artifacts, before rebuilding with new configuration and a separate volume. Renaming an old database file is insufficient: encrypted secrets are incompatible with the new encryption domain. There is no old-data import tool in this change. Reconfigure connections and credentials in the new instance, retaining backups for rollback.

Historical `docs/changes/luowang-*` paths, prototype filenames, and validation records retain their names for traceability. Independent fixture repositories keep their own addresses.

## License

GNU Affero General Public License v3.0-only (`AGPL-3.0-only`). Personal and commercial use, modification, distribution, and charging fees are allowed. Distribution of covered derivatives and network access to modified versions require compliance with AGPL's corresponding-source and notice obligations. See [LICENSE](LICENSE).

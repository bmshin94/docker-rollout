# docker-rollout

## What this is

A single POSIX `sh` script, [docker-rollout](docker-rollout), that installs as a Docker CLI plugin (`docker rollout <service>`) to do zero-downtime deploys of Docker Compose services. It scales a service to 2× its current instance count, waits for the new containers to become ready/healthy, runs an optional pre-stop hook on the old containers, then stops and removes them.

There is no build step and no runtime dependency beyond Docker. The entire program is that one file.

## Commands

Tasks are wrapped by the [run](run) script (wowu/run convention). `./run` lists them.

- **Lint**: `./run lint` (= `shellcheck docker-rollout`) — must stay clean; CI runs it via `ludeeus/action-shellcheck` on every push/PR (`.github/workflows/lint.yml`).
- **Test**: `./run test` — inits the Bats submodules and runs the integration suite (`bats test/`). **Requires a running Docker daemon**; CI runs it on `ubuntu-latest` via `.github/workflows/test.yml`.
- **Run locally**: symlink or copy the script into `~/.docker/cli-plugins/docker-rollout` (must be executable), then `docker rollout <service>`.
- **Docs site** (Jekyll, in [docs/](docs/)): `cd docs && bundle install && bundle exec jekyll serve`. Deployed to GitHub Pages from `main` via `.github/workflows/pages.yml`.

## Tests

Real-container integration tests using [Bats](https://github.com/bats-core/bats-core), in [test/rollout.bats](test/rollout.bats). Bats-core and bats-support are pinned git **submodules** under `test/` — a fresh clone needs `git submodule update --init --recursive` (which `./run test` does for you), and CI checks out with `submodules: recursive`.

- Fixtures in [test/fixtures/](test/fixtures/) are a tiny `busybox httpd` service (no build). `compose.yml` is the base (no healthcheck); `compose.healthcheck.yml` adds a fast file-toggle healthcheck; `compose.unhealthy.yml` adds an always-failing one to drive the rollback path. Tests combine them with multiple `-f` flags.
- The four scenarios: swap without healthcheck, swap after healthcheck passes, rollback when the new container never gets healthy, and a 2→4→2 roll that checks the container-ID diffing.
- Tests invoke the working-tree `./docker-rollout` **with the literal `rollout` token first** (`./docker-rollout rollout -f … web`) — the script consumes everything before that token as docker global args, so omitting it makes the script see no SERVICE. This mirrors how Docker invokes the plugin.
- Each test runs in its own Compose project via `COMPOSE_PROJECT_NAME` (inherited by the script's `docker compose` calls) and is torn down with `docker compose down -v` in `teardown`. Timeouts are kept short (`-t 5`, `-w 1`) so the suite stays fast.

## Working on the script

- Keep it POSIX `sh`, not bash — the shebang is `#!/bin/sh` and `set -e` is on. ShellCheck enforces this.
- `VERSION` at the top ([docker-rollout:4](docker-rollout#L4)) is the plugin version; bump it for releases. It surfaces via `docker-cli-plugin-metadata` and `-v`.
- Word-splitting is intentional in several places (`$DOCKER_ARGS`, `$COMPOSE_FILES`, `$ENV_FILES`, `$OLD_CONTAINER_IDS`) so that multi-value options expand into separate argv. These are guarded with `# shellcheck disable=SC2086` comments — preserve them when editing those lines.

### How the script is structured (top to bottom)

1. **Plugin metadata short-circuit**: if invoked as `docker-cli-plugin-metadata`, print JSON and exit. Docker calls this to discover the plugin.
2. **Docker arg capture**: arguments _before_ the literal `rollout` token are collected into `$DOCKER_ARGS` (global docker flags) and re-passed to every `docker`/`compose` invocation.
3. **Compose command detection**: prefers `docker compose` (v2), falls back to `docker-compose`. Stored in `$COMPOSE_COMMAND`.
4. **`main()`**: the rollout algorithm — start-if-not-running, record old container IDs, scale to 2×, diff to find new container IDs, wait on healthcheck (with per-second polling up to `--timeout`) or a fixed `--wait` sleep, run the pre-stop hook, then stop+remove old containers. Rolls back (stops/removes new containers) if healthchecks fail.
5. **Option parser**: the `while`/`case` loop at the bottom fills the globals used by `main`, then `main` runs last.

Key globals the option parser builds: `COMPOSE_FILES` (`-f`), `ENV_FILES` (`--env-file`), `HEALTHCHECK_TIMEOUT` (`-t`), `NO_HEALTHCHECK_TIMEOUT` (`-w`), `WAIT_AFTER_HEALTHY_DELAY` (`--wait-after-healthy`), `PRE_STOP_HOOK` (`--pre-stop-hook`), `SERVICE`. `--project-name` and `--profile` are appended onto `$COMPOSE_COMMAND` rather than kept separately.

- **Pre-stop hook precedence**: a `--pre-stop-hook` CLI value overrides the `docker-rollout.pre-stop-hook` container label; the label is read from the _old_ container ([docker-rollout:165](docker-rollout#L165)), so a label-based hook only takes effect on the _next_ deployment. Preserve this behavior.
- **Constraint enforced by design**: services cannot use `container_name` or fixed host `ports`, because two instances must coexist during rollout. This is a documented user caveat, not something the script checks.

## Docs

User-facing docs live in [docs/](docs/) and are the source of truth for option semantics and setup guides (getting started, CLI options, container draining, proxy examples for Traefik/nginx-proxy). When you change script behavior or options, update both the [README.md](README.md) and the relevant page under `docs/`.

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

`ntfy-ar` is an Ansible role that installs [ntfy](https://ntfy.sh/) as a Docker container wrapped in a systemd service. It is the SGC fork (`P3X-118/ntfy-ar`) of the upstream MASH role at `mother-of-all-self-hosting/ansible-role-ntfy`.

It is one piece of a three-part system — keep all three in mind when changing anything non-trivial:

| Component | Path / location | Role |
|---|---|---|
| **This role (`ntfy-ar`)** | `~/sgc/ansible/roles/ntfy-ar` (here) | Ansible glue that templates config, creates the network/volumes, and runs the container as a systemd unit. |
| **ntfy application source** | `~/sgc/apps/ntfy` | Upstream ntfy source. Used as the build context for our custom container image published to Docker Hub at **`legitservices/ntfy`**. |
| **SGC super-playbook** | `~/sgc/SGC` | Consumes this role via `requirements.yml` (pinned by tag) and wires it into `setup.yml` / `group_vars/mash_servers`. See `~/sgc/CLAUDE.md` for the 4-step pattern used to integrate roles. |

The role's own `defaults/main.yml` ships with `ntfy_container_image_registry_prefix_upstream_default: docker.io/` and the upstream image `binwiederhier/ntfy`. The SGC playbook is expected to override these to point at `legitservices/ntfy` for production. Do not hardcode `legitservices/ntfy` into this role's defaults — the override belongs in the consuming playbook.

## Branching model (3-branch — strict)

| Branch | Purpose | Push policy |
|---|---|---|
| `main` | **Upstream mirror only.** Tracks `mother-of-all-self-hosting/ansible-role-ntfy`. Renovate runs against this branch and the `.github/workflows` autotag job tags upstream Docker bumps here. | **Never push hand-written commits to `main`.** Only sync from upstream. |
| `sgc-dev` | Primary development branch for SGC customizations. All work — including merges from `main` — lands here first. | Push freely; this is where you work. |
| `sgc` | Production branch. Tagged releases (`vX.Y.Z-N`) are cut from here and consumed by `SGC/requirements.yml`. | Only fast-forward / merge from `sgc-dev` after review and testing. |

**Upstream sync flow** (security-sensitive — do not shortcut):

1. Fetch upstream into `main`.
2. Create a short-lived sub-branch off `sgc-dev` (e.g. `sgc-dev/upstream-sync-vX.Y.Z`).
3. **Cherry-pick** upstream commits into the sub-branch — do not bulk-merge `main` into `sgc-dev` without review. Each cherry-pick should be inspected for security implications (new env vars, new mounted paths, new network exposure, dependency bumps, build-arg changes, anything touching `tasks/setup_users.yml` or `tasks/install.yml`).
4. Open a PR / review locally, run `just lint`, then merge the sub-branch into `sgc-dev`.
5. Once stable, merge `sgc-dev` → `sgc` and tag.

**Tag scheme:** SGC releases append a monotonic `-N` suffix to the upstream version. If upstream is `v2.17.0`, our releases are `v2.17.0-0`, `v2.17.0-1`, etc. The `requirements.yml` entry in `SGC/` must reference the exact tag on `sgc`.

## Common commands (run from this directory)

```bash
# Lint (production profile, offline; see .ansible-lint)
just lint           # == ansible-lint .

# Pre-commit (runs on pre-push by default per .pre-commit-config.yaml)
pre-commit install  # one-time
pre-commit run --all-files
```

There are no unit tests — validation is via `ansible-lint`, REUSE/SPDX checks, codespell, markdownlint, and the renovate-config-validator, all wired through `.pre-commit-config.yaml`.

To exercise the role end-to-end, run it through the SGC super-playbook (`just install-service ntfy` / `just setup-service ntfy` from `~/sgc/SGC`) — there is no standalone test harness in this repo.

## Role internals — what to know before editing

- **Entrypoint**: `tasks/main.yml` dispatches to `validate_config.yml` → `install.yml` → `setup_users.yml` (on install), `uninstall.yml` (when `ntfy_enabled: false`), and `self_check.yml` (under the `self-check` tag). Tags exposed: `setup-all`, `setup-ntfy`, `install-all`, `install-ntfy`, `self-check`.
- **Implicit role dependencies** (not declared in `meta/main.yml` — must be present in the consuming playbook): `playbook_help` and `systemd_docker_base` (both `P3X-118/*`). The role uses `sysd_docker_service` and `sysd_docker_host_command` from `systemd_docker_base`.
- **Config rendering**: `defaults/main.yml` defines `ntfy_configuration_yaml` (rendered from `templates/ntfy/server.yml.j2`) and `ntfy_configuration_extension_yaml` (user override, deep-merged via `combine(..., recursive=True)`). To add a new ntfy setting, prefer extending the template and exposing a default; only touch `ntfy_configuration` directly for advanced overrides.
- **Container networking**: `ntfy_container_network` is auto-created by the role. `ntfy_container_additional_networks` is the place to attach to external networks (e.g. Traefik) — split into `_auto` (computed) and `_custom` (user) lists, following the pattern documented in `~/sgc/CLAUDE.md`. Network deletion on uninstall is gated by `ntfy_container_network_deletion_enabled` (added in this fork — see commit `054ed56`).
- **Rate-limit exempt-hosts trick**: `ntfy_visitor_request_limit_exempt_hosts` embeds runtime sub-shell `docker network inspect ...` commands generated by `vars/main.yml`. This means the rendered systemd unit contains live `$(docker ...)` invocations evaluated at service start — be careful when refactoring this; it is intentional, not a templating bug.
- **Healthcheck interval** flips between `5s` (Traefik enabled) and `60s` (no Traefik) because of the moby bug noted inline in `defaults/main.yml`. Do not "simplify" this to a single value.

## Licensing & SPDX (REUSE)

This repo is REUSE-compliant (AGPL-3.0-or-later). Every new file must carry an `SPDX-FileCopyrightText` and `SPDX-License-Identifier` header, or have a matching entry in `REUSE.toml` / `LICENSES/`. The `reuse` pre-commit hook will fail otherwise. When adding a file, copy the header style from a sibling file in the same directory.

## Renovate / autotag

`.github/workflows` runs on `main` only: it scans recent commits for Renovate's "Update docker.io/binwiederhier/ntfy Docker tag to vX.Y.Z" messages and creates an upstream-style tag automatically. Our `vX.Y.Z-N` tags are created **manually on `sgc`** after the cherry-pick / review cycle — the autotag workflow does not produce SGC tags.

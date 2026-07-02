# AGENTS.md

Guidance for AI agents working in `ansible-awx`.

## Obsolescence note

**The compose-based AWX install this role stages is obsolete upstream.** The AWX project has
moved to the [AWX Operator](https://github.com/ansible/awx-operator) running on Kubernetes;
`ansible/awx`'s `installer/install.yml` docker-compose path is no longer the supported deployment
method. This repo was converted from a flat playbook to a Galaxy role for structural consistency
with the rest of this account's Ansible repos — that is a mechanical/tooling change, not an
endorsement of the install method. Don't add new features to the compose path; if this role sees
real use, evaluate an AWX Operator rewrite instead (see README.md "Proposed follow-ups").

## Overview

Ansible role (Galaxy: `jahrik.awx`) that installs Docker (via the `geerlingguy.docker`
dependency), installs the Python Docker SDK packages the AWX installer's compose modules need,
clones a pinned tag of `ansible/awx`, and runs its docker-compose installer. `playbook.yml` at the
repo root is a thin wrapper that `include_role`s this role by directory, so
`ansible-playbook playbook.yml` keeps working without the repo being checked out under a
`roles/` path.

## Task Flow

`tasks/main.yml` dispatches to two files via `include_tasks` + `apply: tags:`, each independently
tag-addressable (`awx:docker`, `awx:install`):

- `tasks/docker.yml` — installs the `docker`/`docker-compose` Python packages. Docker Engine
  itself comes from the `geerlingguy.docker` role dependency in `meta/main.yml`, not from vendored
  tasks here.
- `tasks/awx.yml` — clones `awx_repo` at the pinned `awx_version` tag to `awx_dest`, checks
  whether AWX is already responding on port 80, and runs the installer only if it isn't.

## Conventions

- **Idempotency:** every active task must be safe to re-run; the installer command is guarded by
  the port-80 health check so it doesn't re-run against a live AWX instance.
- **Dependencies:** use `uv` for Python package management; do not use `pip` directly for
  tooling. `ansible-galaxy install -r requirements.yml` pulls `geerlingguy.docker`.
- **Facts:** use `ansible_facts['<fact>']`, never the bare `ansible_<fact>` magic vars;
  `ansible.cfg` sets `inject_facts_as_vars = False`.
- **FQCN:** all modules fully qualified (`ansible.builtin.*`).
- **Pinning:** `awx_version` must stay pinned to a real tag — never revert to `devel`.
- **General rules:** abide by the global `AGENTS.md` guidelines (no hardcoded secrets or IPs).

## Testing

```bash
uv sync
source .venv/bin/activate
ansible-galaxy install -r requirements.yml
yamllint .
ansible-lint
ansible-playbook playbook.yml --syntax-check
```

## CI/CD

- **lint** — `yamllint` + `ansible-lint` (profile `production`).
- **syntax** — `ansible-playbook playbook.yml --syntax-check`. No Molecule/converge job: a real
  convergence needs Docker-in-Docker plus a full docker-compose AWX stack, which is heavy and
  targets an install method that's obsolete upstream. Lint + syntax-check is the honest CI gate
  for this repo's current state.
- **release** — `needs: [lint, syntax]`, publishes to Ansible Galaxy on merge to `main`.

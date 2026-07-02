# jahrik.awx

[![CI/CD](https://github.com/jahrik/ansible-awx/actions/workflows/cicd.yml/badge.svg)](https://github.com/jahrik/ansible-awx/actions/workflows/cicd.yml)
[![Ansible Galaxy](https://img.shields.io/badge/ansible--galaxy-jahrik.awx-blue?logo=ansible)](https://galaxy.ansible.com/ui/standalone/roles/jahrik/awx/)

> [!WARNING]
> **This role is obsolete upstream.** It stages the legacy docker-compose-based AWX installer
> (`ansible/awx` -> `installer/install.yml`), which the AWX project has replaced with the
> [AWX Operator](https://github.com/ansible/awx-operator) running on Kubernetes. This role is
> converted to a Galaxy role for structural consistency with the rest of this account's Ansible
> repos, not as an endorsement of the compose-based install path. See **Proposed follow-ups**
> below.

Installs Docker, clones a pinned tag of [AWX](https://github.com/ansible/awx), and runs its
docker-compose installer on Ubuntu.

## Usage

Include the role in your playbook:

```yaml
- hosts: all
  become: true
  roles:
    - jahrik.awx
```

Or run the bundled thin wrapper directly:

```bash
ansible-galaxy install -r requirements.yml
ansible-playbook playbook.yml
```

AWX will be reachable at `http://<host>` once the installer finishes.

## Variables

Key variables (see `defaults/main.yml` for the full list):

| Variable | Default | Description |
|---|---|---|
| `awx_repo` | `https://github.com/ansible/awx.git` | Repo cloned to `awx_dest`. |
| `awx_version` | `9.3.0` | Pinned tag checked out (was unpinned `devel` before this conversion). |
| `awx_dest` | `/opt/awx` | Clone destination. |
| `awx_docker_python_packages` | `[docker, docker-compose]` | Python packages the installer's compose modules need. |

Docker Engine itself is provided by the `geerlingguy.docker` role dependency (see `meta/main.yml`
and `requirements.yml`), not vendored here.

## Testing

```bash
uv sync
source .venv/bin/activate
ansible-galaxy install -r requirements.yml
yamllint .
ansible-lint
ansible-playbook playbook.yml --syntax-check
```

CI runs lint and a syntax check only — see [AGENTS.md](AGENTS.md) for why.

## Proposed follow-ups

- Evaluate replacing this role's approach entirely with the
  [AWX Operator](https://github.com/ansible/awx-operator) on Kubernetes/k3s, which is the
  currently supported way to run AWX.
- Alternatively, archive this repo if there's no active use case for a compose-based AWX install.

## License

MIT

## Author

jahrik@gmail.com

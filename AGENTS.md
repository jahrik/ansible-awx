# AGENTS.md

This file provides guidance to AI coding agents when working with code in this repository.

## Overview

Vagrant lab + Ansible playbooks for deploying [Ansible AWX](https://github.com/ansible/awx) (open-source Ansible Tower) on Ubuntu 16.04. Spins up two VirtualBox VMs and provisions them with Ansible, Docker, and AWX.

No Molecule tests — this is a standalone lab, not an Ansible role.

## Lab Setup

```bash
vagrant up                          # start both VMs
vagrant ssh ansible-awx             # SSH into the AWX host
vagrant status                      # check VM state
```

| VM | IP | Purpose |
|---|---|---|
| `ansible-awx` | `192.168.33.11` | Hosts AWX |
| `ansible-lab` | `192.168.33.12` | Test target |

## Playbooks

Run individually with `-l ansible-awx` to target the AWX host, or omit to run against all:

```bash
# Install Ansible 2.4+ via PPA
ansible-playbook -i inventory.ini -l ansible-awx install_ansible.yml

# Install Docker + pip + docker-py
ansible-playbook -i inventory.ini -l ansible-awx install_docker.yml

# Clone ansible/awx and run its installer (skips if port 80 already responds 200)
ansible-playbook -i inventory.ini -l ansible-awx install_awx.yml

# Run all three in sequence
ansible-playbook -i inventory.ini playbook.yml
```

AWX UI: `http://192.168.33.11` — default credentials: `admin` / `password`

## Linting

```bash
yamllint .
```

## Notes

- Targets Ubuntu 16.04 / Docker Engine (not Docker CE) — these are old but intentional for the lab
- `install_docker.yml` uses the legacy `apt.dockerproject.org` repo — update if targeting a newer Ubuntu
- `install_awx.yml` checks port 80 before running the AWX installer to avoid reinstalling

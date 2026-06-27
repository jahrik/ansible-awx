# Ansible AWX Lab

Vagrant lab and Ansible playbooks for deploying [Ansible AWX](https://github.com/ansible/awx) in a local testing environment.

This repository spins up two Ubuntu 16.04 VirtualBox VMs using Vagrant, and configures them with Docker, Ansible, and AWX.

## Requirements

- Vagrant
- VirtualBox

## Lab Setup

Spin up the lab VMs:

```bash
vagrant up
vagrant status
```

| VM | IP | Purpose |
|---|---|---|
| `ansible-awx` | `192.168.33.11` | Hosts the AWX installation |
| `ansible-lab` | `192.168.33.12` | General lab testing target |

SSH into a node:

```bash
vagrant ssh ansible-awx
```

## Deployment

Playbooks are provided to provision the lab nodes.

Run the full stack deployment on all nodes:

```bash
ansible-playbook -i inventory.ini playbook.yml
```

Alternatively, run individual steps on a specific node (e.g., `ansible-awx`):

```bash
# 1. Install Ansible
ansible-playbook -i inventory.ini -l ansible-awx install_ansible.yml

# 2. Install Docker
ansible-playbook -i inventory.ini -l ansible-awx install_docker.yml

# 3. Install and start AWX
ansible-playbook -i inventory.ini -l ansible-awx install_awx.yml
```

### Accessing AWX

- URL: `http://192.168.33.11`
- Username: `admin`
- Password: `password`

## Testing

```bash
uv sync
source .venv/bin/activate
yamllint .
ansible-lint
molecule test
```

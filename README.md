# Ansible Ouroboros Leios Deployment

An automated, secure Infrastructure-as-Code (IaC) pipeline designed to provision, harden, and deploy an Ouroboros Leios node onto a remote VPS (such as IONOS) using Ansible and GitHub Actions.

---

## Project Structure

```text
ansible-leios-deployment/
├── .github/
│   └── workflows/
│       └── ci-leios.yml          # GitHub Actions CI/CD pipeline
├── inventory.ini                 # Target server inventory configuration
├── harden-node.yml               # OS security hardening & user provisioning playbook
├── install-docker.yml            # Docker engine and container runtime installation playbook
├── deploy-leios.yml              # Application deployment and container orchestration playbook
├── vars.empty.yml                # Fallback empty variables template for CI/CD runs
├── vars.local.yml                # Local override variables (Git-ignored for security)
└── README.md
```

## Prerequisites

* **Local Machine:** Ansible installed (via `pipx` or a Python virtual environment).
* **SSH Key Pair:** A dedicated SSH key pair (e.g., `leios_deploy_key`) generated for secure communication.
* **Remote VPS:** A clean Ubuntu/Debian server instance.

## Local Configuration

To test and run playbooks locally without exposing credentials or IP addresses in your public repository:

1. **Create your local variables file (`vars.local.yml`)**:
   ```yaml
   local_vps_ip: "YOUR_VPS_IP"
   local_vps_username: "root"
   local_ssh_public_key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIG..."
   ```

2. **Ensure local secrets are ignored** by verifying that `vars.local.yml` and your `venv/` are included in your `.gitignore`.

## Execution Guide

Run the modular playbooks sequentially from your local terminal:

### 1. Server Hardening
Provisions the secure `deployer` user, configures SSH keys, and sets up sudoers permissions:
```bash
ansible-playbook -i inventory.ini harden-node.yml --private-key ~/.ssh/leios_deploy_key --user root
```

### 2. Docker Installation
Installs Docker Engine, container plugins, and adds the `deployer` user to the `docker` group:
```bash
ansible-playbook -i inventory.ini install-docker.yml --private-key ~/.ssh/leios_deploy_key --user root
```

### 3. Application Deployment
Downloads configuration scripts, updates topology files, and spins up the Leios Docker container as the non-root `deployer` user:
```bash
ansible-playbook -i inventory.ini deploy-leios.yml --private-key ~/.ssh/leios_deploy_key --user deployer
```

---

## CI/CD Automation

The pipeline is automated via GitHub Actions (`.github/workflows/ci-leios.yml`). Ensure the following repository secrets are configured in your GitHub settings:
* `VPS_IP`
* `VPS_USERNAME`
* `VPS_SSH_KEY`
* `SSH_PUBLIC_KEY`
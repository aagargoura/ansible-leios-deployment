# Ansible Ouroboros-Leios Node Deployment

![Leios Prototype](https://img.shields.io/badge/Leios%20Prototype-2026w30-blue?style=flat-square&logo=cardano)
![Ansible Version](https://img.shields.io/badge/Ansible-%3E%3D2.15-red?style=flat-square&logo=ansible)
![Docker](https://img.shields.io/badge/Docker-Supported-2496ED?style=flat-square&logo=docker&logoColor=white)

An open-source, automated, secure Infrastructure-as-Code (IaC) pipeline designed to provision, harden, and deploy an Ouroboros Leios node onto a remote VPS (such as IONOS) using Ansible, Docker and GitHub Actions.

> **Note:** This project is currently a work in progress (WIP).

### My Node Specifications
Tested and verified on the following hardware profile:
* **CPU:** 6 vCores
* **RAM:** 8 GB
* **Storage:** 240 GB NVMe SSD
* **OS:** Ubuntu 26.04 (or compatible Linux distributions with Docker & Ansible support)
* **Location/Datacenter:**  Germany 🇩🇪


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

## CI/CD Automation

The pipeline is automated via GitHub Actions (`.github/workflows/ci-leios.yml`). Ensure the following repository secrets are configured in your GitHub settings:
* `VPS_IP`
* `VPS_USERNAME`
* `VPS_SSH_KEY`
* `SSH_PUBLIC_KEY`

## Verifying Node Status
Once your deployment finishes running, you can SSH into your remote server to check the health, version information, and synchronization status of your running node container.

### 1. SSH into your VPS
```bash
ssh deployer@YOUR_VPS_IP
```

### 2. Check Container Status
Verify that your container is actively running:
```bash
docker ps
```

### 3.Check Component Versions:
Inspect the versions of cardano-cli, cardano-node, or other components running inside the container:

```bash
docker exec -it leios-relay cardano-node --version
docker exec -it leios-relay cardano-cli --version
```

### 4. Query Node Tip:
To check the current synchronization slot, block, and epoch information via cardano-cli inside your running container, run:

```bash
docker exec -it leios-relay cardano-cli query tip --testnet-magic 164 --socket-path node.socket
```

> **Note:** Replace `leios-relay` with your specific container name if it differs

## Monitoring & Telemetry
The container exposes Prometheus metrics securely on 127.0.0.1:12798, mapped locally for internal Prometheus and Grafana stack consumption without public internet exposure.
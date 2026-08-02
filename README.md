# Ansible Ouroboros-Leios Node Deployment

![Leios Prototype](https://img.shields.io/badge/Leios%20Prototype-2026w30-blue?style=flat-square&logo=cardano)
![Prometheus](https://img.shields.io/badge/Prometheus-F05032?style=flat-square&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat-square&logo=grafana&logoColor=white)
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
│       ├── ci-leios.yml          # GitHub Actions CI/CD pipeline deployment
│       └── ci-test.yml           # GitHub Actions CI/CD pipeline tests
├── ansible.cfg                   # Global Ansible configuration and defaults
├── inventory.ini                 # Target server inventory configuration
├── group_vars/
│   ├── nodes.yml                 # Your actual secrets (Git-ignored)
│   └── nodes.yml.example         # Template committed to git
├── harden-node.yml               # OS security hardening & user provisioning playbook
├── install-docker.yml            # Docker engine and container runtime installation playbook
├── deploy-leios.yml              # Application deployment and container orchestration playbook
├── deploy-monitoring.yml         # Local observability stack (Prometheus & Grafana) playbook
├── vars.empty.yml                # Fallback empty variables template for CI/CD runs
├── .yamllint                     # Custom configuration for YAML linting rules
└── README.md
```

## Prerequisites

* **Local Machine:** Ansible installed (via `pipx` or a Python virtual environment).
* **SSH Key Pairs:** Two dedicated, separate key pairs:
  * `leios_deploy_key`: the ongoing key used by Ansible/CI to connect as the `deployer` user for all normal operations.
  * `leios_bootstrap_key`: a disposable key used **only once**, to authenticate as `root` during initial hardening. Root SSH access is disabled immediately after the first `harden-node.yml` run, so this key is never needed again unless the VPS is rebuilt from scratch.
* **Remote VPS:** A clean Ubuntu/Debian server instance from a cloud provider.

## Quick Start / Deployment

### 1. Clone the repository:
```bash
git clone git@github.com:aagargoura/ansible-leios-deployment.git
cd ansible-leios-deployment
```

### 2. Point Ansible at your server:

Update `inventory.ini` with your target server IP address. Connection parameters like your private key and remote user are handled automatically by `ansible.cfg` and `group_vars/nodes.yml`.

### 3. Set up local secrets:
To test and run playbooks locally without exposing credentials or IP addresses in your public repository:

1. **Set Up Your Local Variables**:
Navigate to the `group_vars/` directory and copy the template file to create your active configuration:
   ```bash
   cd group_vars
   cp nodes.yml.example nodes.yml
   ```

2. **Ensure local secrets are ignored** by verifying that `group_vars/nodes.yml` and your `venv/` are included in your `.gitignore`.

### 4. Generate your keys and establish initial SSH trust (new or reprovisioned VPS only)

Generate both key pairs locally, if you haven't already:
```bash
ssh-keygen -t ed25519 -f ~/.ssh/leios_deploy_key -C "leios deployer key"
ssh-keygen -t ed25519 -f ~/.ssh/leios_bootstrap_key -C "one-time root bootstrap"
```

A brand-new IONOS VPS trusts whatever credential the provider gave you at creation, typically an emailed root password, or a key selected in the Cloud Panel; **not** either key above. You must get `leios_bootstrap_key` onto the box once before Ansible can connect as root.

**Preferred:** when creating/rebuilding the VPS in the IONOS Cloud Panel, paste the contents of `~/.ssh/leios_bootstrap_key.pub` into the SSH key field. Root will trust it from first boot.

**If the VPS already exists with only a password:**
```bash
ssh-copy-id -i ~/.ssh/leios_bootstrap_key.pub root@YOUR_VPS_IP
```
Enter the root password when prompted.

> **Why two keys?** `harden-node.yml` uses `leios_bootstrap_key` only to authenticate as `root` for the first run, then adds `leios_deploy_key` to the new `deployer` user and disables root/password SSH login entirely. Keeping the two separate means a leaked `leios_deploy_key` (e.g. from a CI secret) never grants root access, and the bootstrap key can simply be discarded once hardening completes; it has no further use.

### 5. Run the deployment playbooks:
Run the modular playbooks sequentially from your local terminal:

1. **Server Hardening**

> ⚠️ **Run this as `root` only on the very first execution.** This playbook disables root SSH login and password authentication as part of hardening the box. Once it has run successfully once, **all future runs must use `--user deployer`** — running with `--user root` again will fail to connect, since root login is now disabled.

Provisions the secure `deployer` user, configures SSH keys, and sets up sudoers permissions:
 **First run (bootstrap):**
```bash
ansible-playbook harden-node.yml -e "ansible_user=root ansible_ssh_private_key_file=~/.ssh/leios_bootstrap_key"
```
**Every run after that:**
```bash
 ansible-playbook harden-node.yml
```

> After the bootstrap run completes successfully, `leios_bootstrap_key` is no longer needed; root login is disabled, and all subsequent connections use `deployer` with `leios_deploy_key` (as configured in `inventory.ini`). You can safely delete the bootstrap key pair at this point, or keep it aside only in case you need to rebuild the VPS later.

2. **Docker Installation**
Installs Docker Engine, container plugins, and adds the `deployer` user to the `docker` group:
```bash
ansible-playbook install-docker.yml --user deployer
```

3. **Application Deployment**
Downloads configuration scripts, updates topology files, and spins up the Leios Docker container as the non-root `deployer` user:
```bash
ansible-playbook deploy-leios.yml --user deployer
```

## CI/CD Automation

The pipeline is automated via GitHub Actions (`.github/workflows/ci-leios.yml`). Ensure the following repository secrets are configured in your GitHub settings:
* `VPS_IP`
* `VPS_USERNAME`
* `VPS_SSH_KEY`
* `LEIOS_PORT`
* `SSH_PUBLIC_KEY`

> **Prerequisite:** the automated pipeline only runs `deploy-leios.yml` and `deploy-monitoring.yml`. It assumes Docker is already installed and the `deployer` user already exists with SSH/sudo access. **Steps 1 (`harden-node.yml`) and 2 (`install-docker.yml`) from Quick Start are one-time manual operations and are intentionally excluded from CI** They provision the box itself, which shouldn't be re-run automatically on every push. If you fork this repo and only configure the secrets above without running steps 1–2 by hand first, the pipeline will fail with `docker: command not found`.

## Local Testing & CI Simulation

To ensure your local tests match the GitHub Actions CI pipeline, it is recommended to run the linters inside an isolated **Python 3.10** virtual environment.

**1. Create the Virtual Environment**
Ensure you have Python 3.10 installed on your system, then create a new virtual environment in the root of the repository:

```bash
python3.10 -m venv .venv
```
**2. Activate the Environment**
```bash
source .venv/bin/activate
```
**3. Install Testing Dependencies**
Install the exact linters and Ansible core used by the CI pipeline:
```bash
pip install --upgrade pip
pip install ansible ansible-lint yamllint
```

**4. Run Local Tests**
You can now run the same checks executed by the '.github/workflows/ci-test.yml' workflow:
```bash
# Check YAML formatting
yamllint .

# Verify Ansible syntax
ansible-playbook deploy-leios.yml --syntax-check
ansible-playbook deploy-monitoring.yml --syntax-check
ansible-playbook harden-node.yml --syntax-check
ansible-playbook install-docker.yml --syntax-check

# Run the Ansible Linter
ansible-lint deploy-leios.yml deploy-monitoring.yml install-docker.yml harden-node.yml
```

## Verifying Node Status
Once your deployment finishes running, you can SSH into your remote server to check the health, version information, and synchronization status of your running node container.

1. **SSH into your VPS**
```bash
ssh deployer@YOUR_VPS_IP
```

2. **Check Container Status**
Verify that your container is actively running:
```bash
docker ps
```

3. **Check Component Versions:**
Inspect the versions of cardano-cli, cardano-node, or other components running inside the container:

```bash
docker exec -it leios-relay cardano-node --version
docker exec -it leios-relay cardano-cli --version
```

4. **Query Node Tip:**
To check the current synchronization slot, block, and epoch information via cardano-cli inside your running container, run:

```bash
docker exec -it leios-relay cardano-cli query tip --testnet-magic 164 --socket-path node.socket
```

> **Note:** Replace `leios-relay` with your specific container name if it differs

## Monitoring & Telemetry
The container exposes Prometheus metrics securely on `127.0.0.1:12798`, mapped locally for internal Prometheus and Grafana stack consumption without public internet exposure.

**Test the local metrics endpoint:**
SSH into your VPS and run this quick check to confirm it worked:
```bash
curl -I http://127.0.0.1:12798/metrics
```

You should receive an `HTTP/1.1 200 OK` response

 --- 

As the container securely exposes Prometheus metrics internally on `127.0.0.1:12798`. You can deploy an automated local monitoring stack (Prometheus and Grafana) using the included observability playbook:
```bash
ansible-playbook deploy-monitoring.yml --user deployer
```

To view your dashboards locally without exposing ports to the public internet, use secure SSH local port forwarding:

- **Grafana Dashboard:** `ssh -L 3000:127.0.0.1:3000 deployer@YOUR_VPS_IP` (Navigate to `http://localhost:3000`)

- **Prometheus UI:** `ssh -L 9090:127.0.0.1:9090 deployer@YOUR_VPS_IP` (Navigate to `http://localhost:9090`)

## Contributing

Contributions, issues, and feature requests are welcome!

This repository enforces a Decoupled CI Pipeline for open-source contributions:

1. Fork the project and create your feature branch.

2. Open a Pull Request against main.

3. Our `ci-test.yml` workflow will automatically run linting and syntax checks on your code without accessing production secrets.

4. Once tests pass and code is reviewed, it will be merged and deployed.

## License

Distributed under the Apache 2.0 License. See `LICENSE` for more information.
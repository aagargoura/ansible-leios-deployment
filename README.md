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
├── inventory.ini                 # Target server inventory configuration
├── harden-node.yml               # OS security hardening & user provisioning playbook
├── install-docker.yml            # Docker engine and container runtime installation playbook
├── deploy-leios.yml              # Application deployment and container orchestration playbook
├── deploy-monitoring.yml         # Local observability stack (Prometheus & Grafana) playbook
├── vars.empty.yml                # Fallback empty variables template for CI/CD runs
├── vars.local.yml                # Local override variables (Git-ignored for security)
└── README.md
```

## Prerequisites

* **Local Machine:** Ansible installed (via `pipx` or a Python virtual environment).
* **SSH Key Pair:** A dedicated SSH key pair (e.g., `leios_deploy_key`) generated for secure communication.
* **Remote VPS:** A clean Ubuntu/Debian server instance from a cloud provider.

## Quick Start / Deployment

### 1. Clone the repository:
```bash
git clone git@github.com:aagargoura/ansible-leios-deployment.git
cd ansible-leios-deployment
```

### 2. Configure your inventory:
Update `inventory.ini` with your target server IP address and connection details.

To avoid hardcoding long command-line flags (like `--private-key` and `--user`) every time, bake those settings directly into your `inventory.ini` or use an SSH configuration alias (`~/.ssh/config`).

### 3. Local Configuration:
To test and run playbooks locally without exposing credentials or IP addresses in your public repository:

1. **Create your local variables file (`vars.local.yml`)**:
   ```yaml
   local_vps_ip: "YOUR_VPS_IP"
   local_vps_username: "root"
   local_ssh_public_key: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIG..."
   ```

2. **Ensure local secrets are ignored** by verifying that `vars.local.yml` and your `venv/` are included in your `.gitignore`.

### 4. Run the deployment playbooks:
Run the modular playbooks sequentially from your local terminal:

1. **Server Hardening**
Provisions the secure `deployer` user, configures SSH keys, and sets up sudoers permissions:
```bash
ansible-playbook -i inventory.ini harden-node.yml --private-key ~/.ssh/leios_deploy_key --user root
```

2. **Docker Installation**
Installs Docker Engine, container plugins, and adds the `deployer` user to the `docker` group:
```bash
ansible-playbook -i inventory.ini install-docker.yml --private-key ~/.ssh/leios_deploy_key --user root
```

3. **Application Deployment**
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
ansible-playbook -i inventory.ini deploy-leios.yml --syntax-check
ansible-playbook -i inventory.ini deploy-monitoring.yml --syntax-check
ansible-playbook -i inventory.ini harden-node.yml --syntax-check
ansible-playbook -i inventory.ini install-docker.yml --syntax-check

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
SSH into your VPS and run these quick check to confirm it worked:
```bash
curl -I http://127.0.0.1:12798/metrics
```

You should receive an `HTTP/1.1 200 OK` response

 --- 

As the container securely exposes Prometheus metrics internally on `127.0.0.1:12798`. You can deploy an automated local monitoring stack (Prometheus and Grafana) using the included observability playbook:
```bash
ansible-playbook -i inventory.ini deploy-monitoring.yml --private-key ~/.ssh/leios_deploy_key --user deployer
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
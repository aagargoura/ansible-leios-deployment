# Ansible Leios Node Deployment
Cardano Ouroboros Leios (Musashi Dōjō) Node Automation &amp; GitOps Pipeline

An automated infrastructure and continuous delivery pipeline designed to provision, harden, and manage a prototype relay node for **Cardano's Ouroboros Leios (Musashi Dōjō) testnet** on an IONOS VPS. 

This project combines Infrastructure as Code concepts, automated server hardening, strict container security isolation, and a GitOps deployment lifecycle for rapidly moving experimental blockchain builds.

## Architecture Overview

```text
[ SPO Push ] 
       │
       ▼
[ GitHub Actions ] ──(Ansible Lint & Syntax Check)
       │
       ▼ (Secure SSH / Key Injection)
[ IONOS VPS ]
  ├── OS Hardening (UFW Firewall, Fail2ban, Non-root user)
  ├── Docker Engine & Isolated Bridge Networks
  └── Ouroboros Leios Container (Musashi Testnet Prototype)
```

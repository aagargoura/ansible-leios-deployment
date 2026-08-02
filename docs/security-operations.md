# Security Operations & Audit Guide

This document outlines how to monitor, verify, and audit the defensive security stack running on the Ouroboros Leios VPS host.

## Architecture Overview

The node host relies on a multi-layered defense strategy:
* **OpenSSH Key-Only Auth:** Password authentication (`PasswordAuthentication no`) and direct root login (`PermitRootLogin no`) are strictly disabled.
* **UFW Rate-Limiting:** Restricts connection bursts on port `22/tcp` and opens `3010/tcp` for Leios P2P traffic.
* **Fail2ban (`sshd` jail):** Automatically blocks IPs exhibiting repeated pre-authentication failures.
* **Time Synchronization (Chrony):** Maintains high-precision NTP clock sync for consensus slot timing.
* **Unattended Security Upgrades:** Automatically applies OS security patches without manual intervention.
* **Swap Space:** 8G swapfile provides headroom against OOM kills during chain-sync memory spikes.

## Real-Time Log Inspection & Runtime Verification

1. **Check live SSH authentication events:** 

To inspect live SSH traffic and observe active connection attempts:

```bash
sudo journalctl -u ssh -n 50 --no-pager
```
2. **Fail2ban Jail Status & IP Management:**

To inspect active jail metrics and see banned IP addresses:

```bash
sudo fail2ban-client status sshd
```
**Example Terminal Output:**

<p align="center">
  <img src="images/fail2ban-status-screenshot.png" alt="Fail2ban Status" width="700"/>
  <br>
  <em>Figure 1: Active Fail2ban sshd jail status and banned IP list</em>
</p>

**Confirm fail2ban is banning repeat offenders, not just tracking them:**
```bash
sudo grep "Ban" /var/log/fail2ban.log | tail -20
```

**Unban an IP Address:** If an authorized IP is accidentally locked out, unban it via:

```bash
sudo fail2ban-client set sshd unbanip <IP_ADDRESS>
```

3. **Check UFW Firewall Rules:** 

Verify active packet rules and port bindings:

```bash
sudo ufw status verbose
```
**Example Terminal Output:**

<p align="center">
  <img src="images/ufw-status-screenshot.png" alt="UWF Status" width="700"/>
  <br>
  <em>Figure 2: Active UFW Status</em>
</p>

4. **Verify Active SSH Runtime Parameters:**

Check the evaluated SSH daemon settings (ensures cloud-init or default drop-in files aren't overriding `PasswordAuthentication`):

```bash
sudo sshd -T | grep -E "passwordauthentication|permitrootlogin|port"
```

 **Verify no secondary password-auth path is open (PAM/keyboard-interactive):**
```bash
sudo sshd -T | grep -E "passwordauthentication|kbdinteractiveauthentication|permitrootlogin"
```
All three must read `no`. 

**Check for unauthorized SSH drop-in overrides:**

```bash
sudo grep -r "PasswordAuthentication\|PermitRootLogin" /etc/ssh/sshd_config.d/
```

5. **Verify Unattended Upgrades is active and applying patches:**

Confirm the systemd timer/service is enabled and running:
```bash
systemctl status unattended-upgrades.service
```

6. **Verify NTP Time Synchronization:**
Ensure Chrony is actively syncing clock drift (critical for Cardano/Leios slot leader election):

```bash
chronyc tracking
```

7. **Disk usage check**

```bash
df -h /
```

8. **Verify swap is active and sized correctly**

```bash
free -h
swapon --show
```
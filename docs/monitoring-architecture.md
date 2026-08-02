# Monitoring Architecture

## Overview

The Ouroboros Leios deployment uses a private Docker network to isolate monitoring traffic between the Leios node, Prometheus, and Grafana.

Prometheus metrics are **not exposed publicly** and are only accessible through the internal Docker network.

Architecture:

```
                    leios-monitoring Docker Network

  +----------------+   :12798   +----------------+
  |  leios-relay   |----------->|   Prometheus   |
  +----------------+            +----------------+
                                         |
                                         v
                                 +----------------+
                                 |    Grafana     |
                                 +----------------+
```

## Docker Network

A dedicated Docker bridge network is created:

```
leios-monitoring
```

The network is managed by Ansible using:

```yaml
community.docker.docker_network
```

Example:

```yaml
- name: Ensure monitoring Docker network exists
  community.docker.docker_network:
    name: leios-monitoring
    state: present
```

Both the Leios node container and monitoring stack containers are attached to this network.

## Security Model

The previous implementation exposed the metrics endpoint using:

```
127.0.0.1:12798:12798
```

Although this prevented external access, it also prevented Prometheus running in another container from reaching the endpoint — loopback port bindings on the host aren't reachable from inside a separate container's network namespace.

The current design removes host port exposure for the metrics endpoint entirely:

```yaml
ports:
  - "3010:3010"
```

The metrics endpoint is now available only internally, over the Docker network:

```
leios-relay:12798
```

Benefits:

* No public Prometheus metrics endpoint
* No firewall exposure required for metrics traffic
* Container-to-container communication only
* Reduced attack surface

## Prometheus Configuration

Prometheus discovers the Leios metrics endpoint through Docker DNS:

```yaml
scrape_configs:
  - job_name: 'leios-relay'
    static_configs:
      - targets:
          - 'leios-relay:12798'
```

Docker automatically resolves the container name:

```
leios-relay
```

to the container's IP address inside the `leios-monitoring` network — no manual IP management needed, even across container restarts.

The Prometheus UI itself is bound to loopback only:

```yaml
ports:
  - "127.0.0.1:9090:9090"
```

Like Grafana, it requires an SSH tunnel to view remotely (see below).

## Grafana Access

Grafana is bound only to localhost:

```yaml
ports:
  - "127.0.0.1:3000:3000"
```

Access is provided through SSH tunneling:

```bash
ssh -L 3000:127.0.0.1:3000 deployer@YOUR_VPS_IP
```

Then open:

```
http://localhost:3000
```

## Troubleshooting

**Check Docker networks**

```bash
docker network ls
```

Expected:

```
leios-monitoring
```

**Inspect network members**

```bash
docker network inspect leios-monitoring
```

Expected containers:

```
leios-relay
prometheus
grafana
```

**Test metrics from the Prometheus container**

```bash
docker exec prometheus wget -qO- http://leios-relay:12798/metrics
```

Expected output:

```
# HELP ...
# TYPE ...
```

**Check Prometheus target status**

Tunnel to the Prometheus UI first:

```bash
ssh -L 9090:127.0.0.1:9090 deployer@YOUR_VPS_IP
```

Then open:

```
http://localhost:9090/targets
```

The target should show:

```
Endpoint: http://leios-relay:12798/metrics
State: UP
```
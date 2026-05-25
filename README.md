# 🏠 homelab-monitoring

A self-hosted monitoring stack running on Ubuntu 24.04 with Docker. Collects system metrics via Prometheus, visualises them in Grafana, and sends intelligent alerts to Telegram via [Hermes](https://github.com/your-hermes-repo) — an AI agent that acts as your personal Level 1 IT support.

All images are stored in **GitHub Container Registry (GHCR)** for easy portability across hosts.

---

## 📐 Architecture

```
┌─────────────────────────────────────────────┐
│               Docker Host (Ubuntu 24.04)     │
│                                             │
│  ┌─────────────┐     ┌─────────────────┐   │
│  │ Node        │────▶│   Prometheus    │   │
│  │ Exporter    │     │   :9090         │   │
│  │ :9100       │     └────────┬────────┘   │
│  └─────────────┘              │             │
│                        ┌──────▼──────┐      │
│                        │  Grafana    │      │
│                        │  :3000      │      │
│                        └─────────────┘      │
│                               │             │
│                        ┌──────▼──────┐      │
│                        │Alertmanager │      │
│                        │  :9093      │      │
│                        └──────┬──────┘      │
│                               │ webhook     │
│                        ┌──────▼──────┐      │
│                        │   Hermes    │      │
│                        │  (Docker)   │      │
│                        └──────┬──────┘      │
└───────────────────────────────┼─────────────┘
                                │
                         ┌──────▼──────┐
                         │  Telegram   │
                         │  (You/Boss) │
                         └─────────────┘
```

---

## 🧱 Stack

| Service | Image | Purpose |
|---|---|---|
| Prometheus | `prom/prometheus` | Metrics collection & alerting rules |
| Node Exporter | `prom/node-exporter` | Host system metrics (CPU, RAM, disk) |
| Grafana | `grafana/grafana` | Metrics visualisation dashboard |
| Alertmanager | `prom/alertmanager` | Routes alerts to Hermes webhook |
| Hermes | (your instance) | AI IT agent — notifies via Telegram |

> All images are tagged and pushed to `ghcr.io/YOUR_USERNAME/` for portability.

---

## 📁 Project Structure

```
homelab-monitoring/
├── docker-compose.yml
├── .env.example
├── prometheus/
│   ├── prometheus.yml        # Scrape config
│   └── alerts.yml            # Alert rules
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── prometheus.yml
├── alertmanager/
│   └── alertmanager.yml      # Webhook to Hermes
└── .github/
    └── workflows/
        └── publish.yml       # Auto-push images to GHCR
```

---

## ⚡ Quick Start

### 1. Clone the repo

```bash
git clone https://github.com/YOUR_USERNAME/homelab-monitoring.git
cd homelab-monitoring
```

### 2. Set environment variables

```bash
cp .env.example .env
# Edit .env and fill in your values
nano .env
```

`.env.example`:
```env
GRAFANA_PASSWORD=changeme
HERMES_WEBHOOK_URL=http://hermes:PORT/webhook
```

### 3. Start the stack

```bash
docker compose up -d
```

### 4. Access services

| Service | URL | Default credentials |
|---|---|---|
| Grafana | http://localhost:3000 | admin / (your GRAFANA_PASSWORD) |
| Prometheus | http://localhost:9090 | — |
| Alertmanager | http://localhost:9093 | — |

---

## 🔔 Alerts

Alerts are defined in `prometheus/alerts.yml` and routed through Alertmanager → Hermes → Telegram.

| Alert | Condition | Severity |
|---|---|---|
| HighCPUUsage | CPU > 85% for 2 min | warning |
| HighMemoryUsage | RAM > 90% for 2 min | critical |
| DiskRunningLow | Disk free < 15% | warning |

To add custom alerts, edit `prometheus/alerts.yml` and run:

```bash
docker compose restart prometheus
```

---

## 🤖 Hermes AI Agent

Hermes is an AI-powered IT agent running as a Docker container on the same host. It:

- Receives alerts from Alertmanager via webhook
- Communicates with you through Telegram as your personal Level 1 IT support
- Can query Prometheus directly for metric context

Make sure your Hermes container is on the same Docker network (`monitoring-net`) and that its webhook endpoint is set correctly in `alertmanager/alertmanager.yml`.

```yaml
# alertmanager/alertmanager.yml
receivers:
  - name: hermes
    webhook_configs:
      - url: "http://hermes:PORT/webhook"
        send_resolved: true
```

> **Note:** Replace `hermes:PORT` with your actual Hermes container name and port. Check your Hermes documentation for the exact webhook path.

---

## 📦 GitHub Container Registry (GHCR)

Images are stored in GHCR under your GitHub account for easy portability.

### First-time setup

```bash
# Create a GitHub Personal Access Token with write:packages scope
# Then login:
echo $GITHUB_TOKEN | docker login ghcr.io -u YOUR_USERNAME --password-stdin

# Tag and push images manually (first time only)
docker pull prom/prometheus:latest
docker tag prom/prometheus:latest ghcr.io/YOUR_USERNAME/prometheus:latest
docker push ghcr.io/YOUR_USERNAME/prometheus:latest

# Repeat for: node-exporter, grafana, alertmanager
```

After this, GitHub Actions handles all future pushes automatically on every commit to `main`.

### Pull on a new host

```bash
echo $GITHUB_TOKEN | docker login ghcr.io -u YOUR_USERNAME --password-stdin
docker compose pull
docker compose up -d
```

---

## 🚀 GitHub Actions — Auto Publish to GHCR

`.github/workflows/publish.yml`:

```yaml
name: Publish Images to GHCR

on:
  push:
    branches:
      - main

env:
  REGISTRY: ghcr.io
  OWNER: ${{ github.repository_owner }}

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    strategy:
      matrix:
        include:
          - source: prom/prometheus:latest
            target: prometheus
          - source: prom/node-exporter:latest
            target: node-exporter
          - source: grafana/grafana:latest
            target: grafana
          - source: prom/alertmanager:latest
            target: alertmanager

    steps:
      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Pull, retag, and push ${{ matrix.target }}
        run: |
          docker pull ${{ matrix.source }}
          docker tag ${{ matrix.source }} \
            ${{ env.REGISTRY }}/${{ env.OWNER }}/${{ matrix.target }}:latest
          docker push \
            ${{ env.REGISTRY }}/${{ env.OWNER }}/${{ matrix.target }}:latest
```

> `GITHUB_TOKEN` is automatically available in Actions — no manual secret needed for GHCR.

---

## 🔄 Moving to a New Host

```bash
# On the new host — login and pull
echo $GITHUB_TOKEN | docker login ghcr.io -u YOUR_USERNAME --password-stdin

git clone https://github.com/YOUR_USERNAME/homelab-monitoring.git
cd homelab-monitoring
cp .env.example .env && nano .env   # fill in your values

docker compose pull    # pulls all images from GHCR
docker compose up -d
```

That's it. No manual image transfers needed.

---

## 🔐 Security Notes

- **Never commit `.env`** — it is in `.gitignore` by default
- Change `GRAFANA_PASSWORD` before first use
- Prometheus and Alertmanager have no auth by default — do not expose ports 9090/9093 publicly without a reverse proxy (e.g. nginx + basic auth)
- Grafana on port 3000 should be behind a firewall or VPN for remote access

---

## 🛠 Useful Commands

```bash
# View all running services
docker compose ps

# Tail logs for a specific service
docker compose logs -f prometheus
docker compose logs -f hermes

# Restart a single service
docker compose restart alertmanager

# Stop the entire stack
docker compose down

# Pull latest images from GHCR and redeploy
docker compose pull && docker compose up -d
```

---

## 📄 License

MIT — feel free to fork and adapt for your own homelab.

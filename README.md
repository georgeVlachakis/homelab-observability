# Home-Lab Observability

A small, public, portfolio-quality observability stack for a home lab. The repository contains portable configuration only: private addresses, credentials, tokens, and deployment-specific home-lab details do not belong here.

## Stage-0 goal

Stage 0 establishes metrics collection for one Linux host, the GMKtec, without modifying any other hosts or introducing logs, traces, alerting, or dashboards.

| Component | Responsibility |
| --- | --- |
| Prometheus | Scrapes and stores metrics from itself and the two exporters. |
| Grafana | Queries Prometheus through an automatically provisioned datasource. |
| Node Exporter | Exposes Linux host metrics such as CPU, memory, filesystems, and networking. |
| cAdvisor | Exposes Docker container and resource-usage metrics. |

The two Stage-0 data paths are:

```text
GMKtec host metrics ──> Node Exporter ──> Prometheus ──> Grafana
Docker/container state ──> cAdvisor ────> Prometheus ──> Grafana
```

See [docs/architecture.md](docs/architecture.md) for the small architecture note.

## Repository structure

```text
.
├── .env.example
├── .gitignore
├── README.md
├── compose.yaml
├── docs/
│   └── architecture.md
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── prometheus.yml
└── prometheus/
    └── prometheus.yml
```

## Prerequisites

- A Linux host with Docker Engine and the Docker Compose plugin.
- Permission to run Docker containers and create named volumes.
- The standard Linux paths used by Node Exporter and cAdvisor: `/`, `/sys`, `/var/run`, `/var/lib/docker`, `/dev/disk`, and `/dev/kmsg`.

The bind mounts and cAdvisor permissions are designed for the target Linux host. Docker Desktop on Windows or macOS does not accurately represent the GMKtec runtime and is not a substitute for deployment acceptance.

## Development/test startup

1. Create a private environment file from the sanitized example:

   ```sh
   cp .env.example .env
   ```

2. Replace `GRAFANA_ADMIN_PASSWORD` in `.env` with a strong local password. Optionally change the loopback bind addresses or ports after considering the security implications.

3. Resolve and review the Compose model:

   ```sh
   docker compose --env-file .env config
   ```

4. Start the stack:

   ```sh
   docker compose --env-file .env up -d
   docker compose --env-file .env ps
   ```

Prometheus is then available at `http://127.0.0.1:9090` and Grafana at `http://127.0.0.1:3000` unless the local environment file changes those values.

## Verification

Open `http://127.0.0.1:9090/targets`. The following jobs should all report `UP` after startup:

- `prometheus` at `prometheus:9090`;
- `node-exporter` at `node-exporter:9100`;
- `cadvisor` at `cadvisor:8080`.

Sign in to Grafana with the administrator credentials from `.env`, then open **Connections → Data sources**. The `Prometheus` datasource should already exist, be marked as the default, and point to `http://prometheus:9090`; no manual datasource creation is required.

When finished with a development run, stop the containers while retaining metrics and Grafana state:

```sh
docker compose --env-file .env down
```

Adding `--volumes` to that command also deletes the named volumes and their stored data.

## Security notes

- Never commit `.env`; it is ignored by Git. `.env.example` contains placeholders only.
- Prometheus and Grafana bind to `127.0.0.1` by default. Do not expose them to an untrusted network without authentication, encryption, and an explicit access design.
- The Node Exporter host mount is read-only. cAdvisor still runs privileged and reads host runtime paths, so access to the Docker host remains security-sensitive.
- Container environment variables, including the Grafana password, are visible to users who can inspect Docker containers.
- This is a development-quality Stage-0 deployment, not a hardened production service. It currently has no TLS, reverse proxy, alerting, dashboards, log aggregation, or tracing.

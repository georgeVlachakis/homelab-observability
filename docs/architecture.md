# Stage-0 architecture

Stage 0 runs one Docker Compose project on the GMKtec Linux host. It collects host and container metrics, stores them in Prometheus, and makes them available to Grafana. It does not collect logs or traces.

## Data flow

```text
GMKtec host metrics ──> Node Exporter ──> Prometheus ──> Grafana
Docker/container state ──> cAdvisor ────> Prometheus ──> Grafana
```

Prometheus reaches both exporters by their Compose service names on the private `observability` network. Grafana uses the same network and receives a provisioned Prometheus datasource at `http://prometheus:9090`.

Prometheus and Grafana persist their state in named Docker volumes. Their web interfaces bind to the host loopback address by default; the exporter ports are not published to the host.

Node Exporter reads the host root filesystem through a read-only bind mount. cAdvisor requires privileged container access plus read-only mounts of Linux and Docker runtime paths. These permissions are intentionally limited to metrics collection but still make the stack unsuitable for unreviewed exposure or multi-tenant use.

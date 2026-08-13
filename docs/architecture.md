# Stage-0 and Stage-1 architecture

Stage 0 runs one Docker Compose project on the GMKtec Linux host. It collects host and container metrics, stores them in Prometheus, and makes them available to Grafana. It does not collect logs or traces.

Stage 1 adds a provisioned Grafana dashboard over the existing metrics path. It does not add another runtime component, collector, or network route.

## Data flow

```text
GMKtec host metrics ──> Node Exporter ──> Prometheus ──> Grafana
Docker/container state ──> cAdvisor ────> Prometheus ──> Grafana
```

Node Exporter uses the host network and PID namespaces, matching its host root filesystem bind mount. This lets its Linux collectors observe the GMKtec host rather than the container's network and process views. It listens only at `127.0.0.1:9100`, so its metrics are not bound to the GMKtec's home-network interface.

Prometheus also uses host networking and binds to `127.0.0.1:9090`. It scrapes itself and Node Exporter through host loopback. cAdvisor remains on the private `observability` bridge and publishes port `8080` only to `127.0.0.1`, providing the third loopback scrape target without home-network exposure.

Grafana uses host networking so its provisioned datasource can reach loopback-only Prometheus at `http://127.0.0.1:9090`. Grafana binds to host loopback by default, with an explicit environment-file setting if a different access design is required. Host-networked services are not addressed through Compose service DNS.

Grafana loads the **GMKtec Overview** dashboard from a read-only bind mount through its file dashboard provider. The dashboard uses the provisioned Prometheus datasource UID `prometheus`. Repository JSON is authoritative; UI edits are disabled so a later provisioning scan cannot silently overwrite unreviewed dashboard changes.

Prometheus and Grafana persist their state in named Docker volumes. There is no hard-coded GMKtec private address, and none of the metrics endpoints bind to the home network by default.

Node Exporter reads the host root filesystem through a read-only, recursively propagated bind mount. cAdvisor requires privileged container access plus read-only mounts of Linux and Docker runtime paths. These permissions and the host namespace sharing are intentionally limited to metrics collection but still make the stack unsuitable for unreviewed exposure or multi-tenant use.

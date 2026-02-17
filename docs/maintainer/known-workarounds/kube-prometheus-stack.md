# kube-prometheus-stack

kube-prometheus-stack is a Helm chart that deploys Prometheus, Grafana, and a constellation of exporters and operators. Most of it converts cleanly via the [servicemonitor extension](../../extensions.md#servicemonitor). Grafana does not.

The problem: Grafana in kube-prometheus-stack ships with **k8s-sidecar** containers that watch ConfigMaps/Secrets via the Kubernetes API at runtime. They provision dashboards and datasources by polling the apiserver for labeled ConfigMaps. This has no compose equivalent — there is no apiserver to poll.

> *The acolyte built a mirror-temple, faithful in every stone — yet the oracles within fell silent, for the gods they consulted resided in a firmament this world had never known. The prayers were correct; the heavens were absent.*
>
> — *Cultes des Goules, On Oracles Without Firmament (so they say)*

## What to exclude

The sidecar containers and the admission webhooks serve no purpose in compose:

```yaml
# helmfile2compose.yaml
exclude:
  - kube-prometheus-stack-grafana-sidecar-grafana*
  - kube-prometheus-stack-admission*
```

## Grafana override

The original Grafana service uses `kiwigrid/k8s-sidecar` as the main image (the sidecar container is first in the pod spec). Override it with the actual Grafana image and provide env vars directly:

```yaml
# helmfile2compose.yaml
overrides:
  kube-prometheus-stack-grafana:
    image: docker.io/grafana/grafana:12.3.1
    container_name: null
    environment:
      GF_SECURITY_ADMIN_USER: $secret:grafana-admin-secret:username
      GF_SECURITY_ADMIN_PASSWORD: $secret:grafana-admin-secret:password
      GF_PATHS_DATA: /var/lib/grafana/
      GF_PATHS_LOGS: /var/log/grafana
      GF_PATHS_PLUGINS: /var/lib/grafana/plugins
      GF_PATHS_PROVISIONING: /etc/grafana/provisioning
    volumes:
      - ./configmaps/kube-prometheus-stack-grafana/grafana.ini:/etc/grafana/grafana.ini:ro
      - ./data/kube-prometheus-stack-grafana:/var/lib/grafana
      - /var/lib/grafana-search
      - ./configmaps/kube-prometheus-stack-grafana-dashboards-custom/my-dashboard.json:/var/lib/grafana/dashboards/custom/my-dashboard.json:ro
      - ./configmaps/kube-prometheus-stack-grafana/dashboardproviders.yaml:/etc/grafana/provisioning/dashboards/dashboardproviders.yaml:ro
      - ./configmaps/kube-prometheus-stack-grafana-config-dashboards/provider.yaml:/etc/grafana/provisioning/dashboards/sc-dashboardproviders.yaml:ro
      - ./configmaps/kube-prometheus-stack-grafana-datasource/datasource.yaml:/etc/grafana/provisioning/datasources/datasource.yaml:ro
```

Adapt the secret name, image version, and dashboard paths to your setup.

## Datasource provisioning

In K8s, the k8s-sidecar populates `/etc/grafana/provisioning/datasources/` from a labeled ConfigMap. In compose, create the file statically.

The datasource ConfigMap is typically rendered by the Helm chart and already present in your manifests — h2c writes it to `configmaps/kube-prometheus-stack-grafana-datasource/`. Mount it as shown in the override above.

If the datasource references the Prometheus K8s Service by its FQDN (e.g. `kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090`), it resolves natively via network aliases — no replacement needed.

## Dashboard provisioning

Same pattern: the Helm chart renders dashboard JSON as ConfigMaps. h2c writes them to `configmaps/`. Mount each dashboard JSON into Grafana's dashboard directory and provide a `dashboardproviders.yaml` that points at it.

The chart's default `dashboardproviders.yaml` ConfigMap usually works as-is — mount it from `configmaps/kube-prometheus-stack-grafana/dashboardproviders.yaml`.

## What's excluded by default

kube-prometheus-stack also includes several components that h2c auto-excludes on first run (K8s-only):

- `kube-prometheus-stack-operator` — the Prometheus Operator itself (no CRDs to reconcile in compose)
- `kube-prometheus-stack-kube-state-metrics` — needs the Kubernetes API
- `kube-prometheus-stack-prometheus-node-exporter` — needs host-level access

These are already handled by first-run auto-exclude. You don't need to add them manually unless your auto-exclude list was generated before they were detected.

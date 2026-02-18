# Known workarounds

Tentacles everywhere? Yeah, what did you expect. You took an orchestrator designed for planet-scale infrastructure and squeezed it into `docker compose up`. Some things don't fit. Some things fight back. Some things grow new limbs when you're not looking.

Here are at least some known sushi recipes.

> *The fisherman who nets a kraken does not weep — he fetches a larger knife and a recipe book. The tentacles are already here; the only question is the sauce.*
>
> — *Voynich Manuscript, On Practical Cephalopod Cuisine (citation needed)*

## Workarounds

- **[Common charts](common-charts.md)** — Bitnami PostgreSQL, Bitnami Redis, MinIO: the repeat offenders
- **[kube-prometheus-stack](kube-prometheus-stack.md)** — Grafana sidecar containers, dashboard/datasource provisioning, what to exclude

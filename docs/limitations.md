# Limitations

*We replaced the perfectly sane orchestra conductor with a hideously mutated chimpanzee sprouting tentacle appendices, and you're surprised the audience needs earplugs.*

Kubernetes is an orchestrator. Docker Compose is a list of containers. This document covers what gets lost in translation.

## What changes behavior

Things you need to know for the output to work correctly.

### Network aliases (nerdctl) {#network-aliases-nerdctl}

!!! warning "This is a hard requirement"
    If you use nerdctl compose, **nothing will work**. This is not a cosmetic limitation — services will fail to reach each other.

h2c generates Docker Compose `networks.default.aliases` on each service so that Kubernetes FQDNs (`svc.ns.svc.cluster.local`, `svc.ns.svc`, `svc.ns`) resolve natively via compose DNS. This is how inter-service communication works without rewriting hostnames — the FQDNs match certificate SANs, Prometheus targets resolve, Grafana datasources work, everything behaves like it did in K8s.

**This requires Docker Compose.** nerdctl compose does not implement network aliases — it silently ignores the `aliases` key and does not support the `--network-alias` flag either. If you run containerd without Kubernetes (Rancher Desktop in containerd mode, Lima, etc.), FQDNs will not resolve and services that reference other services by their K8s DNS names will fail.

Workarounds:

- **Switch to Docker Compose** (recommended). Rancher Desktop can use dockerd (moby) as its backend instead of containerd. The switch takes one checkbox and a VM restart.
- **Use Podman Compose** — supports `networks.<name>.aliases` since v1.0.6.
- **Don't use this project.** You already run containerd. You already have a container runtime that speaks to Kubernetes natively. You are one `kubeadm init` away from having a real cluster. Why are you here? What offering did you bring to the altar of Yog Sa'rath that led you to this page? Go home. Deploy your helmfile on a real cluster. Be free.

> *The disciple forged a world without masters, without wards, without the sovereign's gaze — and found that the names he had given his servants no longer carried across the void. For in a realm stripped of all authority, even the act of calling out is an unanswered prayer.*
>
> — *Unaussprechlichen Kulten, On Worlds Without Shepherds (apocryphal)*

### Startup ordering

Kubernetes init containers block the main container until they complete. In compose, init containers become separate services with `restart: on-failure` — they retry until they succeed, but nothing prevents the main container from starting concurrently and crash-looping until its dependencies are ready.

This works in practice (everything eventually converges), but expect noisy logs on first boot.

Why not `depends_on`? nerdctl compose ignores it entirely. Docker compose supports `condition: service_completed_successfully`, but relying on it would break nerdctl compatibility. Brute force retry works everywhere.

Exception: sidecar containers use `depends_on` to ensure the main service's container exists before starting (required by `network_mode: container:<name>`). nerdctl compose respects the ordering even though it logs a warning about ignoring the directive.

### Sidecars and pod-level networking

Sidecar containers (`containers[1:]`) are converted to separate compose services sharing the main service's network namespace via `network_mode: container:<name>`. Both containers listen on the same hostname, each on its own port — same as a K8s pod.

Other compose services reach both the main container and its sidecars via the main service name, each on its own port.

Limitation: `emptyDir` volumes are not shared between the main container and its sidecars (same limitation as init containers — see [emptyDir volumes](#emptydir-volumes)).

### emptyDir volumes

K8s `emptyDir` volumes are shared between containers in the same pod. When init containers and the main container both mount the same `emptyDir` (e.g. to chmod a directory), compose converts them to anonymous volumes (`- /path`) which are NOT shared between services.

If an init container needs to prepare data for the main container via a shared volume, the `emptyDir` must be mapped to a named volume in `helmfile2compose.yaml` manually.

### Secrets

Kubernetes Secrets exist because serious people built a serious system for serious production workloads. RBAC-gated access. Base64 encoding (yes, it's encoding, not encryption — but at least it's *something*). Encryption at rest in etcd. Audit logs. Pod-level access control. A whole security model designed by people who think about threat vectors for a living.

We take all of that and dump it as plain-text environment variables into a YAML file on your laptop.

The generated `compose.yml` contains your database passwords, your API keys, your OAuth client secrets — everything — in clear text, because that's what happens when you devolve a production orchestrator into `docker compose up`. Do not commit it to version control. Not because we care about your security posture at this point (we lost that right several abstractions ago), but because your colleagues might see what you've done and ask questions you don't want to answer.

### TLS between services

In Kubernetes, services can use mTLS (via service mesh or cert-manager) for internal communication. In compose, inter-service traffic is plain HTTP on the shared bridge network by default. Only the Caddy reverse proxy terminates TLS for external access.

The [cert-manager extension](extensions.md#cert-manager) can generate real certificates at conversion time, enabling TLS between services when needed. Caddy backend SSL annotations (`haproxy.org/server-ssl`, `nginx.ingress.kubernetes.io/backend-protocol: HTTPS`) are translated to Caddy TLS transport configuration.

### Bind mount permissions (Linux / WSL)

Bitnami images (PostgreSQL, Redis, MongoDB) run as non-root (UID 1001) and expect Unix permissions on their data directories. The host directory is typically owned by your user (UID 1000), so the container can't write to it. This causes `mkdir: cannot create directory: Permission denied`.

This is handled automatically: helmfile2compose detects non-root containers (`securityContext.runAsUser`) with PVC bind mounts and generates a `fix-permissions` service that runs `chown -R <uid>` as root on first startup. No manual intervention needed.

### Hostname length

Linux hostnames are limited to 63 characters. Compose uses the service name as the container hostname. Services with names longer than 63 characters automatically get a truncated `hostname:` to avoid `sethostname: invalid argument` failures.

## What is ignored

Safe to skip — these affect the cluster's operational behavior, not what the application does.

### Scaling and replicas

Compose runs one instance of each service. HPA, replica counts (other than 0, which auto-skips the workload), and PodDisruptionBudgets are ignored. DaemonSets are converted as regular services (one instance, no node affinity or scheduling). This is a single-machine tool.

### Resource limits

CPU/memory requests and limits are ignored. Compose supports `mem_limit` / `cpus`, but translating K8s resource semantics (requests vs limits, burstable QoS) into compose constraints is more misleading than helpful.

### Probes and healthchecks

Liveness, readiness, and startup probes are not converted to compose `healthcheck`. The semantics differ enough that a blind translation would cause more problems than it solves (compose healthcheck only affects `depends_on` with `condition: service_healthy`, which we don't use anyway).

### Network isolation

Kubernetes namespaces and NetworkPolicies provide network isolation between services. In compose, all services share a single bridge network. Everything can talk to everything. Your database is one DNS lookup away from your frontend. The NetworkPolicies you carefully crafted? Gone. Welcome to the flat network — it's like 2005 again, but with more YAML.

### fieldRef (downward API)

Environment variables using `fieldRef` are partially supported:

- **`status.podIP`** — resolved to the compose service name (the container's DNS address in compose).
- Other fieldRef values (`metadata.name`, `metadata.namespace`, etc.) are skipped with a warning. There is no compose equivalent for most of these.

## What is not converted

Not supported, no workaround.

### CronJobs

Not converted. A CronJob would need an external scheduler or a `sleep`-loop wrapper, neither of which is a good idea.

### CRDs (Custom Resource Definitions)

Operator-managed resources (`Keycloak`, `KeycloakRealmImport`, Zalando `postgresql`, Strimzi `Kafka`, etc.) are skipped with a warning unless a loaded [extension](extensions.md) handles them.

Extensions can be loaded via `--extensions-dir` or installed with [h2c-manager](maintainer/h2c-manager.md). See the [extension catalogue](extensions.md) for available converters.

### Longhorn

Don't even think about it.

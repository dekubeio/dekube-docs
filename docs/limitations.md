# Limitations

*We replaced the perfectly sane orchestra conductor with a hideously mutated chimpanzee sprouting tentacle appendices, and you're surprised the audience needs earplugs.*

Kubernetes is an orchestrator. Docker Compose is a list of containers. This document covers what gets lost in translation.

## What changes behavior

Things you need to know for the output to work correctly.

### Network aliases (nerdctl) {#network-aliases-nerdctl}

!!! warning "nerdctl requires the flatten-internal-urls transform"
    Without it, nerdctl compose will silently ignore network aliases and services will fail to reach each other. Add `flatten-internal-urls` to your `depends` list — see workarounds below.

helmfile2compose generates Docker Compose `networks.default.aliases` on each service so that Kubernetes FQDNs (Fully Qualified Domain Names — e.g. `svc.ns.svc.cluster.local`, `svc.ns.svc`, `svc.ns`) resolve natively via compose DNS. This is how inter-service communication works without rewriting hostnames — the FQDNs match certificate SANs, Prometheus targets resolve, Grafana datasources work, everything behaves like it did in K8s.

nerdctl compose does not implement network aliases — it silently ignores the `aliases` key and does not support the `--network-alias` flag either. If you run containerd without Kubernetes (Rancher Desktop in containerd mode, Lima, etc.), FQDNs will not resolve unless you use the [`flatten-internal-urls`](catalogue.md#flatten-internal-urls) transform.

Workarounds:

- **Switch to Docker Compose** (recommended). Rancher Desktop can use dockerd (moby) as its backend instead of containerd. The switch takes one checkbox and a VM restart.
- **Use Podman Compose** — supports `networks.<name>.aliases` since v1.0.6.
- **Use the [`flatten-internal-urls`](catalogue.md#flatten-internal-urls) transform** — strips network aliases entirely and rewrites all FQDNs to short compose service names. nerdctl's built-in DNS resolves short names natively, so everything works. Trade-off: you lose FQDN preservation, which means no inter-service TLS with cert-manager SANs (`cert-manager` declares incompatibility with this transform). If you don't need backend SSL, this is the zero-friction option.
- **Don't use this project.** You already run containerd. You already have a container runtime that speaks to Kubernetes natively. You are one `kubeadm init` away from having a real cluster. Why are you here? What offering did you bring to the altar of Yog Sa'rath that led you to this page? Go home. Deploy your helmfile on a real cluster. Be free.

> *The disciple forged a world without masters, without wards, without the sovereign's gaze — and found that the names he had given his servants no longer carried across the void. For in a realm stripped of all authority, even the act of calling out is an unanswered prayer.*
>
> — *Unaussprechlichen Kulten, On Worlds Without Shepherds (one assumes)*

### Startup ordering

Kubernetes init containers block the main container until they complete. In compose, init containers become separate services with `restart: on-failure`, and the main service declares `depends_on` with `condition: service_completed_successfully` — so Docker Compose starts the main container only after its init containers exit successfully.

nerdctl compose ignores `depends_on` entirely, so on nerdctl the main container starts concurrently and crash-loops until its dependencies are ready. The `restart: on-failure` policy ensures everything converges eventually — expect noisy logs on first boot with nerdctl. On Docker Compose, startup ordering is correct out of the box.

Sidecar containers also use `depends_on` because `network_mode: container:<name>` needs the parent container to exist. Same story: Docker Compose respects it, nerdctl ignores it and relies on brute-force retry.

### Sidecars and pod-level networking

Sidecar containers (`containers[1:]`) are converted to separate compose services sharing the main service's network namespace via `network_mode: container:<name>`. Both containers listen on the same hostname, each on its own port — same as a K8s pod.

Other compose services reach both the main container and its sidecars via the main service name, each on its own port.

### PVC (PersistentVolumeClaim) conversion

Kubernetes PVCs request dynamic storage from a provisioner (Longhorn, Ceph, etc.). helmfile2compose converts them to bind mounts — host directories mapped into the container. Each PVC claim name is registered in `dekube.yaml` on first run with a default host path under `./data/`. `volumeClaimTemplates` (StatefulSets) are handled the same way.

### Secrets

Kubernetes Secrets exist because serious people built a serious system for serious production workloads. RBAC-gated access. Base64 encoding (yes, it's encoding, not encryption — but at least it's *something*). Encryption at rest in etcd. Audit logs. Pod-level access control. A whole security model designed by people who think about threat vectors for a living.

We take all of that and dump it as plain-text environment variables into a YAML file on your laptop.

The generated `compose.yml` contains your database passwords, your API keys, your OAuth client secrets — everything — in clear text, because that's what happens when you devolve a production orchestrator into `docker compose up`. Do not commit it to version control. Not because the heresy wasn't already committed several abstractions ago, but because your colleagues might see what you've done and ask questions you don't want to answer.

### TLS between services

In Kubernetes, services can use mTLS (via service mesh or cert-manager) for internal communication. You had encryption between every pod. You had mutual authentication. You had a trust model. In compose, inter-service traffic is plain HTTP on the shared bridge network by default. Only the reverse proxy (provided by the `IngressProvider`) terminates TLS for external access.

The [cert-manager extension](catalogue.md#cert-manager) can generate real certificates at conversion time, enabling TLS between services when needed. Backend SSL annotations (`haproxy.org/server-ssl`, `nginx.ingress.kubernetes.io/backend-protocol: HTTPS`) are translated to reverse proxy TLS transport configuration by the `IngressProvider`.

### Bind mount permissions (Linux / WSL)

Bitnami images (PostgreSQL, Redis, MongoDB) run as non-root (UID 1001) and expect Unix permissions on their data directories. The host directory is typically owned by your user (UID 1000), so the container can't write to it. This causes `mkdir: cannot create directory: Permission denied`.

This is handled automatically: the [fix-permissions](https://github.com/dekubeio/dekube-transform-fix-permissions) transform (bundled in helmfile2compose) detects non-root containers (`securityContext.runAsUser`) with bind-mounted volumes and generates a `fix-permissions` service that runs `chown -R <uid>` as root on first startup. No manual intervention needed in most cases.

**Caveat: image swaps.** fix-permissions reads the UID from the Kubernetes manifest. If another transform changes the image (e.g. bitnami replaces `bitnami/redis` with `redis:7-alpine`), the manifest UID is no longer reliable — fix-permissions detects the mismatch and skips the service with a warning. To restore the chown, set `user:` on the compose service (via `overrides:` in `dekube.yaml` or in the transform itself). fix-permissions will use the explicit `user:` value over the manifest UID.

### Resource limits

CPU/memory **limits** are translated to `deploy.resources.limits` (`memory` and `cpus`). **Requests** are ignored — compose has no concept of guaranteed vs burstable QoS classes; only hard limits exist.

nerdctl compose ignores `deploy.resources` entirely. Docker Compose enforces them.

### Probes and healthchecks

Readiness and liveness probes are converted to compose `healthcheck` (readiness preferred, fallback to liveness). Supported probe types: `exec` (→ `CMD`), `httpGet` (→ `wget`), `tcpSocket` (→ `/dev/tcp`). Timing fields (`periodSeconds`, `timeoutSeconds`, `failureThreshold`, `initialDelaySeconds`) are mapped to their compose equivalents.

Startup probes are not converted — compose has no equivalent concept (startup probes gate liveness checks, not readiness).

nerdctl compose ignores `healthcheck` entirely. Docker Compose uses it with `depends_on` `condition: service_healthy`.

### Hostname length

Linux hostnames are limited to 63 characters. Compose uses the service name as the container hostname. Services with names longer than 63 characters automatically get a truncated `hostname:` to avoid `sethostname: invalid argument` failures.

## What is ignored

Safe to skip — but only because you already abandoned the cluster that would enforce them. These affect operational behavior, not what the application does.

### Scaling and replicas

Compose runs one instance of each service. HPA, replica counts (other than 0, which auto-skips the workload), and PodDisruptionBudgets are ignored. DaemonSets are converted as regular services (one instance, no node affinity or scheduling). This is a single-machine tool.

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

Operator-managed resources (`Keycloak`, `KeycloakRealmImport`, Zalando `postgresql`, Strimzi `Kafka`, etc.) are skipped with a warning unless a loaded [extension](catalogue.md) handles them.

Extensions can be loaded via `--extensions-dir` or installed with [dekube-manager](https://manager.dekube.io/docs/). See the [extension catalogue](catalogue.md) for available extensions.

### Longhorn

Don't even think about it.

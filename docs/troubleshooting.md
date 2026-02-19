# Troubleshooting

Ah, mortal. Welcome to the most cursed lands of this documentation. If you are here, something broke — or rather, something *revealed itself*, because it was always broken. You merely summoned it into the light.

Before we begin: nothing on this page should surprise you. You converted a Kubernetes deployment into a flat list of containers. You stripped away the orchestrator, the service mesh, the DNS hierarchy, the namespace isolation, the scheduling, the self-healing. What remains is a YAML file and a prayer. The prayer sometimes works. When it doesn't, this page exists.

> *The disciple descended into the crypt seeking answers, and found only mirrors — each reflecting a different mistake, each older than the last.*
>
> — *Book of Eibon, On Descending (don't quote me on this)*

## Installing Helm and Helmfile

h2c needs `helm` and `helmfile` to render manifests.

!!! tip "Package manager"
    Some package managers already have both: `brew install helm helmfile` on macOS, `pacman -S helm helmfile` on Arch. If that works, skip the manual install below.
    
    Debian/Ubuntu don't package them — install manually:

**Helm:**

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash
```

Auto-detects OS/arch, downloads, verifies checksum, installs to `/usr/local/bin`. Verify: `helm version` should print v4.x.

**Helmfile:**

```bash
curl -sL https://github.com/helmfile/helmfile/releases/download/v1.2.3/helmfile_1.2.3_linux_amd64.tar.gz | tar -xzf - helmfile
sudo mv helmfile /usr/local/bin/
```

Verify: `helmfile --version` should print v1.x. Other platforms: check the [release assets](https://github.com/helmfile/helmfile/releases/latest) for your OS/arch.

## nerdctl compose does not work

nerdctl compose silently ignores `networks.*.aliases` — the key that makes K8s FQDNs resolve in compose. Without it, every service that references another by its K8s DNS name (`svc.ns.svc.cluster.local`) will fail to connect.

**The fix**: install the [`flatten-internal-urls`](catalogue.md#flatten-internal-urls) transform. It strips all network aliases and rewrites FQDNs to short compose service names, which nerdctl resolves natively. No runtime change needed.

```bash
python3 h2c-manager.py flatten-internal-urls
```

If you are running Rancher Desktop with containerd: switching to dockerd (moby) in Rancher Desktop settings avoids the problem entirely — one checkbox, one VM restart. See [limitations](limitations.md#network-aliases-nerdctl) for the full list of workarounds and caveats (including cert-manager compatibility).

## For maintainers: chart-specific issues

If the issue is specific to a Helm chart rather than h2c itself — a sidecar that needs the K8s API, a container that expects a CRD controller at runtime, an image that phones home to the apiserver on startup — check the [known workarounds](maintainer/known-workarounds/index.md). Those are sushi recipes for tentacles that don't fit, organized by chart.

## Network alias collisions (multi-project)

Your stack works half the time and breaks the other half? Services resolve to the wrong container? One request succeeds, the next returns someone else's login page?

Congratulations — you are experiencing the consequences of your own actions. You put two desecrated temples on the same street and expected the gods not to notice.

But alas, I am here. Not to save you — salvation left this project around v1.2 — but to teach you how to identify which curse is yours and how to keep the tentacles from strangling each other.

When multiple h2c projects share the same Docker network (`network: shared-infra` in `helmfile2compose.yaml`), every network alias from every service in every project lands on the same DNS namespace. Every FQDN, every short name, every cursed `.svc.cluster.local` suffix — all of them, cohabiting in a flat network that was never designed to hold the naming conventions of multiple fictional clusters.

The **FQDNs** are mostly safe — K8s namespaces are baked into the names, so `redis.stoatchat-redis.svc.cluster.local` and `redis.lasuite-redis.svc.cluster.local` resolve to different containers. The temple's naming conventions, for once, protect you.

The **short aliases** do not. When a K8s Service name differs from its compose service name (e.g. `keycloak-service` → compose service `keycloak`), the K8s name is added as a short alias. If two projects register the same short alias on the same network, Docker resolves it via round-robin between both containers. Silent. Random. One request hits your keycloak, the next hits the other project's keycloak. The kind of bug that surfaces at 3 AM and makes you question every architectural decision that led you here — which, to be fair, you should have been doing since page one of this documentation.

**How to diagnose:** inspect the network aliases on your running containers:

```bash
docker inspect <container> --format '{{ .NetworkSettings.Networks }}'
```

If two containers from different projects share the same alias on the same network, you found it.

In practice, short alias collisions are rare — they only appear when the K8s Service name differs from the compose service name, which is uncommon. But if you run multiple projects on a shared network and something resolves to the wrong service: you angered the gods by putting two desecrated temples on the same street. Now enjoy the tentacles. Actions, consequences.

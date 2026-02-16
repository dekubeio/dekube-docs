# Concepts

First of all, there is nothing to save in this project. The portals are opened, the flattening has begun, and the only question left is how far down the desecration goes before something pushes back.

This document describes the design philosophy behind h2c — what it *intends* to do, what it *refuses* to do, and where the line is drawn (arbitrarily, under duress, in the dark).

For the mechanical reality of how the conversion works, see [Architecture](architecture.md).

## Differences from Kubernetes

| Aspect | Kubernetes | Compose |
|--------|-----------|---------|
| Reverse proxy | Ingress controller (HAProxy, nginx, traefik) | Caddy (auto-TLS, path routing) |
| TLS | cert-manager (selfsigned or Let's Encrypt) | Caddy (internal CA or Let's Encrypt) |
| Service discovery | K8s DNS (`.svc.cluster.local`) | Compose DNS (service names) |
| Secrets | K8s Secrets (base64, RBAC-gated) | Inline env vars (derived from seed) |
| Volumes | PVCs (dynamic provisioning) | Bind mounts or named volumes |
| Port exposure | hostNetwork / NodePort / LoadBalancer | Explicit port mappings |
| Scaling | HPA / replicas | Single instance |
| Namespace isolation | Per-service namespaces | Single compose network |
| Secret replication | Reflector (cross-namespace) | Not needed (single network) |

## The emulation boundary

h2c is converging toward a K8s-to-compose emulator — taking declarative K8s representations and materializing them in a compose runtime. Not everything can cross that bridge.

### Three tiers

**Tier 1 — Flattened.** K8s as a declaration language. We consume the intent and materialize it in compose. Workloads, ConfigMaps, Secrets, Services, Ingress, PVCs. Operator CRDs fall here too — we emulate the *output* of the controller (the resources it would create), not the controller itself. cert-manager Certificates become PEM files. Keycloak CRs become compose services. The operator's job happens at conversion time.

**Tier 2 — Ignored.** K8s operational features that don't change what the application *does*, only how K8s manages it. NetworkPolicies, HPA, PDB, RBAC, resource limits/requests, ServiceAccounts. Safe to skip — they affect the cluster's security posture and scaling behavior, not the application's functionality on a single machine.

**Tier 3 — The wall.** Anything that talks to the kube-apiserver at runtime. This is the hard limit. Then [someone built a fake apiserver](https://github.com/baptisterajaut/h2c-api). Consult the [maritime police most wanted list](https://github.com/baptisterajaut/h2c-api#supported-endpoints) for details.

### What's behind the wall

- **Operators themselves** — they watch the API for CRDs, reconcile state. But we don't need to *run* them, just emulate their output (tier 1).
- **Apps that use the K8s API** — service discovery via API instead of DNS, leader election via Lease objects, dynamic config via watching ConfigMaps. [A suspect matching this description](https://github.com/baptisterajaut/h2c-api) was last seen near the border.
- **Downward API** — pod name, namespace, node name, labels injected as env vars or files. [The same suspect](https://github.com/baptisterajaut/h2c-api) forged these too. Annotations are still at large.
- **In-cluster auth** — ServiceAccount tokens, RBAC-gated API calls. [The documents have been falsified](https://github.com/baptisterajaut/h2c-api#leader-election). We don't talk about it.

> *He who flattens the world into files shall find that each file begets another, and each mount begets a service, until the flattening itself becomes a world — and the disciple realizes he has built not a bridge, but a second shore.*
> — *Necronomicon, On the Limits of Flattening (probably)*

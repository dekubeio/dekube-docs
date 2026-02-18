# Concepts

First of all, there is nothing to save in this project. The portals are opened, the flattening has begun, and the only question left is how far down the desecration goes before something pushes back.

This document describes the design philosophy behind h2c — what it *intends* to do, what it *refuses* to do, and where the line is drawn (arbitrarily, under duress, in the dark).

For the mechanical reality of how the conversion works, see [Architecture](architecture.md).

## Differences from Kubernetes

| Aspect | Kubernetes | Compose |
|--------|-----------|---------|
| Reverse proxy | Ingress controller (HAProxy, nginx, traefik) | Caddy (auto-TLS, path routing) |
| TLS | cert-manager (selfsigned or Let's Encrypt) | Caddy (internal CA or Let's Encrypt) |
| Service discovery | K8s DNS (`.svc.cluster.local`) | Compose DNS + network aliases (K8s FQDNs preserved), or short names via [`flatten-internal-urls`](../catalogue.md#flatten-internal-urls) |
| Secrets | K8s Secrets (base64, RBAC-gated) | Inline env vars (plain text in compose.yml) |
| Volumes | PVCs (dynamic provisioning) | Bind mounts or named volumes |
| Port exposure | hostNetwork / NodePort / LoadBalancer | Explicit port mappings |
| Scaling | HPA / replicas | Single instance |
| Namespace isolation | Per-service namespaces | Single compose network |
| Secret replication | Reflector (cross-namespace) | Not needed (single network) |

## The emulation boundary

h2c is converging toward a K8s-to-compose emulator — taking declarative K8s representations and materializing them in a compose runtime. Not everything survives the crossing.

### Three tiers

**Tier 1 — Flattened.** K8s as a declaration language. We consume the intent and materialize it in compose. Workloads, ConfigMaps, Secrets, Services, Ingress, PVCs. CRDs fall here too via extensions — converters emulate the *output* of K8s controllers (the resources they would create), not the controllers themselves. cert-manager Certificates become PEM files (converter). Keycloak CRs become compose services (provider). The controller's job happens at conversion time.

**Tier 2 — Ignored.** K8s operational features that don't change what the application *does*, only how K8s manages it. NetworkPolicies, HPA, PDB, RBAC, resource limits/requests, ServiceAccounts. Safe to skip — they affect the cluster's security posture and scaling behavior, not the application's functionality on a single machine.

**Tier 3 — The wall.** Anything that talks to the kube-apiserver at runtime. This is the hard limit. Then [someone built a fake apiserver](https://github.com/baptisterajaut/h2c-api). Consult the [maritime police most wanted list](https://github.com/baptisterajaut/h2c-api#supported-endpoints) for details.

### What's behind the wall

- **Operators themselves** — they watch the API for CRDs, reconcile state. But we don't need to *run* them, just emulate their output (tier 1).
- **Apps that use the K8s API** — service discovery via API instead of DNS, leader election via Lease objects, dynamic config via watching ConfigMaps. [A suspect matching this description](https://github.com/baptisterajaut/h2c-api) was last seen near the border.
- **Downward API** — pod name, namespace, node name, labels injected as env vars or files. [The same suspect](https://github.com/baptisterajaut/h2c-api) forged these too. Annotations are still at large.
- **In-cluster auth** — ServiceAccount tokens, RBAC-gated API calls. [The documents have been falsified](https://github.com/baptisterajaut/h2c-api#leader-election). We don't talk about it.

> *He who flattens the world into files shall find that each file begets another, and each mount begets a service, until the flattening itself becomes a world — and the disciple realizes he has built not a bridge, but a second shore.*
> — *Necronomicon, On the Limits of Flattening (supposedly)*

## The curse of names

The original plan was clean: rewrite K8s DNS names at conversion time. `keycloak.auth.svc.cluster.local` becomes `keycloak`. Simple. Contained. The flattening works as intended — K8s names go in, compose names come out.

Then the temple defended itself.

Certificates have SANs. Prometheus scrape targets use FQDNs. Grafana datasources hardcode the full K8s service path. Keycloak realm URLs embed the issuer hostname. Every time we rewrote a name, something downstream expected the original — and failed silently, or with a TLS error three layers deep, or not at all until someone tried to federate two services that suddenly couldn't verify each other's certificates.

DNS rewriting was a losing game. Every new integration brought new URL patterns to match, new edge cases where the regex missed, new silent breakage discovered weeks later. The more we rewrote, the more we broke.

So we stopped rewriting. Instead, we cursed ourselves.

**Network aliases** make compose services answer to their full K8s FQDNs. Every service gets `networks.default.aliases` with `svc.ns.svc.cluster.local`, `svc.ns.svc`, and `svc.ns` — the complete set of names that Kubernetes DNS would resolve. Compose DNS resolves them natively. No rewriting. No regex. No silent breakage. The names that Kubernetes gave its servants now carry across into compose, unchanged.

The cost: every compose service now bears the weight of its former K8s identity. The YAML is uglier. The network aliases section is a monument to a naming convention designed for a distributed system running on a single laptop. We carry the full FQDN of a cluster that does not exist, because the certificates were signed for a world we dismantled.

The temple was desecrated. But the names — the names refused to die.

There is, however, a way back. The [`flatten-internal-urls`](../catalogue.md#flatten-internal-urls) transform strips the aliases and rewrites FQDNs to short names — the original approach, revived as an opt-in post-processing step. It was built for nerdctl compatibility (nerdctl ignores aliases), but it also produces cleaner output for anyone who doesn't need FQDN preservation. The cost is real: certificates with FQDN SANs will break, Prometheus FQDN scrape targets will stop resolving. If you don't have inter-service TLS, you don't pay the cost. The old scribe's approach was not *wrong* — it was wrong as a default.

> *The scribe tore the names from the temple walls and carved simpler ones in their place. But the prayers failed — for the gods answer only to the names they were given at consecration. And so the scribe, humbled, carved the old names back, letter by letter, onto walls that were never meant to hold them.*
>
> — *The Nameless City, On Names That Refuse to Die (unverified)*

## On knowing what you destroy

Again: this tool works. Not "works for a demo" — works with real helmfiles, real operators, real cert chains, real multi-service platforms. Applications don't notice they've been evicted from Kubernetes.

The original intent was reasonable: don't abandon the power of Kubernetes just to maintain a parallel compose setup. One source of truth, two outputs. Clean. Elegant, even. Then the edge cases started. Then the init containers. Then the cert chains. Then someone built a fake apiserver and nobody stopped him.

The uncomfortable truth is that none of this would work without intimate knowledge of the temple. You cannot desecrate what you do not understand. Every shortcut in the converter exists because someone knew exactly what Kubernetes does at each layer and how to fake it convincingly enough. A lesser heresy would have produced a broken tool. This one produces working compose files, which is arguably worse.

> *The architect knew every stone, and he respected every stone. Yet he tore the temple down and stripped it of its beauty — all so he would not have to build a shed beside it.*
>
> — *Cultes des Goules, On Stones That Were Built to Withstand Greatness (trust me on this)*

# Concepts

First of all, there is nothing to save in this project. The portals are opened, the flattening has begun, and the only question left is how far down the desecration goes before something pushes back.

This document describes the design philosophy behind dekube — what it *intends* to do, what it *refuses* to do, and where the line is drawn (arbitrarily, under duress, in the dark).

The scope of the existing distributions is single-node Docker Compose — one machine, bind mounts, no replicas. The engine itself is not tied to this assumption; the same core could target Docker Swarm or other compose-compatible runtimes with a different set of extensions. Converting Kubernetes manifests to Swarm would be less heretical than converting them to single-node compose — but it does raise the question of why you'd use Swarm instead of Kubernetes in the first place. See [Beyond single-host](architecture.md#beyond-single-host) for what that would look like, and make your own theological choices.

For the mechanical reality of how the conversion works, see [Architecture](architecture.md).

## Differences from Kubernetes

| Aspect | Kubernetes | Compose |
|--------|-----------|---------|
| Reverse proxy | Ingress controller (HAProxy, nginx, traefik) | Pluggable `IngressProvider` (Caddy by default — auto-TLS, path routing) |
| TLS | cert-manager (selfsigned or Let's Encrypt) | Depends on IngressProvider (Caddy: internal CA or Let's Encrypt) |
| Service discovery | K8s DNS (`.svc.cluster.local`) | Compose DNS + network aliases (K8s FQDNs preserved), or short names via [`flatten-internal-urls`](../catalogue.md#flatten-internal-urls) |
| Secrets | K8s Secrets (base64, RBAC-gated) | Inline env vars (plain text in compose.yml) |
| Volumes | PVCs (dynamic provisioning) | Bind mounts or named volumes |
| Port exposure | hostNetwork / NodePort / LoadBalancer | Explicit port mappings |
| Scaling | HPA / replicas | Single instance |
| Namespace isolation | Per-service namespaces | Single compose network |
| Secret replication | Reflector (cross-namespace) | Not needed (single network) |

## The emulation boundary

dekube is converging toward a K8s-to-compose emulator — taking declarative K8s representations and materializing them in a compose runtime. Not everything survives the crossing.

### Three tiers

**Tier 1 — Flattened.** K8s as a declaration language. We consume the intent and materialize it in compose. Workloads, ConfigMaps, Secrets, Services, Ingress, PVCs. CRDs fall here too via extensions — converters emulate the *output* of K8s controllers (the resources they would create), not the controllers themselves. cert-manager Certificates become PEM files (converter). Keycloak CRs become compose services (provider). The controller's job happens at conversion time. This is also how dekube handles operators — they *look* like tier 3 (they watch the API, reconcile state, run control loops), but dekube sidesteps them entirely by emulating their output, not their process. The reconciliation loop happens once, in the converter, and the result is static.

**Tier 2 — Ignored.** K8s operational features that don't change what the application *does*, only how K8s manages it. NetworkPolicies, HPA, PDB, RBAC, resource limits/requests, ServiceAccounts. Safe to skip — they affect the cluster's security posture and scaling behavior, not the application's functionality on a single machine.

**Tier 3 — The wall.** Tiers 1 and 2 are about what Kubernetes does *for* the app — scheduling, networking, secret injection. Tier 3 is about what the app does *with* Kubernetes — when the application itself is a kube-apiserver client. That's the real boundary. Then [someone built a fake apiserver](https://github.com/baptisterajaut/dekube-fakeapi). Then someone made it [installable in one command](../catalogue.md#fake-apiserver). The wall still exists — but it now has a door, a welcome mat, and a sign that reads "do not enter" in a font that suggests you should.

### What's behind the wall

The four that remain unsolved without [extraordinary measures](../catalogue.md#fake-apiserver):

- **Live operators** — not the ones whose output we emulate (those are tier 1), but operators that need to *keep running* and observing the cluster. Reconciliation operators (ArgoCD, Flux, the Prometheus Operator's own reconciler) watch the apiserver and act on drift. Policy engines (Kyverno, Kubescape) enforce admission control and audit cluster state. These have no meaning outside a real cluster — there is no state to reconcile, no admissions to gate, no cluster to observe.

    !!! note
        Some live operator behaviors can be worked around not by emulating them, but by choosing compose-native components that happen to cover the same need. ACME certificate renewal is a live operator concern in Kubernetes (cert-manager watches, requests, renews). In compose, the reverse proxy itself can handle ACME natively — Caddy does this out of the box, for instance. dekube doesn't implement ACME; the workaround is in the behavior of the final reverse proxy, not in the conversion act.

- **Runtime API consumers** — apps that call the kube-apiserver for service discovery (listing endpoints instead of using DNS), leader election (Lease objects), or dynamic configuration (watching ConfigMaps for live updates). These assume an apiserver exists and will talk to it. In compose, there is no apiserver — unless someone provides one.
- **Downward API** — pod name, namespace, node name, labels injected as env vars or volume files. Some apps read these to identify themselves to peers or include in telemetry. Without a kubelet to populate them, they're empty or absent.
- **In-cluster auth** — ServiceAccount tokens mounted at `/var/run/secrets/kubernetes.io/serviceaccount/`, used for RBAC-gated API calls. Apps that authenticate to the apiserver (or to each other via token review) need a token that something will accept.

> *He who flattens the world into files shall find that each file begets another, and each mount begets a service, until the flattening itself becomes a world — and the disciple realizes he has built not a bridge, but a second shore.*
> — *Necronomicon, On the Limits of Flattening (supposedly)*

### Push vs pull

dekube is fundamentally push-based: you run a command, it produces files, done. So are its inputs — `helmfile template`, `helm template`, `docker compose up`. The entire pipeline is a single pass from declaration to output, with no agent running afterward.

This is also what makes tier 3 hard. Everything behind the wall is pull-based — an operator watching for changes, an app polling the apiserver, a kubelet injecting live pod metadata. The emulation boundary is, at its core, the boundary between push and pull.

The compose ecosystem has its own pull-based tools. [Komodo](https://github.com/moghtech/komodo) and [Doco-CD](https://github.com/kimdre/doco-cd) are the closest equivalents to ArgoCD/Flux — they poll a git repo and redeploy compose stacks on change. [Watchtower](https://github.com/containrrr/watchtower) watches container registries and auto-updates running containers when a new image is pushed (image drift, not config drift). Portainer's business edition combines both — GitOps stack sync from a repo and container monitoring. These solve the same class of problem (continuous state reconciliation) in a compose-native way. dekube doesn't try to bridge into that world — it produces the static declaration, and what watches or reconciles it afterward is someone else's concern.

## The curse of names

The original plan was clean: rewrite K8s DNS names at conversion time. `keycloak.auth.svc.cluster.local` becomes `keycloak`. Simple. Contained. The flattening works as intended — K8s names go in, compose names come out.

Then the temple defended itself.

Certificates have SANs. Prometheus scrape targets use FQDNs. Grafana datasources hardcode the full K8s service path. Keycloak realm URLs embed the issuer hostname. Every time we rewrote a name, something downstream expected the original — and failed silently, or with a TLS error three layers deep, or not at all until someone tried to federate two services that suddenly couldn't verify each other's certificates.

DNS rewriting was a losing game. Every new integration brought new URL patterns to match, new edge cases where the regex missed, new silent breakage discovered weeks later. The more we rewrote, the more we broke.

So we stopped rewriting. Instead, we cursed ourselves.

**Network aliases** make compose services answer to their full K8s FQDNs. Every service gets `networks.default.aliases` with `svc.ns.svc.cluster.local`, `svc.ns.svc`, and `svc.ns` — the complete set of names that Kubernetes DNS would resolve. Compose DNS resolves them natively. No rewriting. No regex. No silent breakage. The names that Kubernetes gave its servants now carry across into compose, unchanged.

The cost: every compose service now bears the weight of its former K8s identity. The YAML is uglier. The network aliases section is a monument to a naming convention designed for a distributed system running on a single laptop. We carry the full FQDN of a cluster that does not exist, because the certificates were signed for a world we dismantled.

The temple was desecrated. But the names — the names refused to die.

There is, however, a way back. The [`flatten-internal-urls`](../catalogue.md#flatten-internal-urls) transform strips the aliases and rewrites FQDNs to short names — the original approach, revived as an opt-in post-processing step. The old scribe's approach was not *wrong* — it was wrong as a default. See [Limitations — Network aliases](../limitations.md#network-aliases-nerdctl) for the practical tradeoffs and workarounds.

> *The scribe tore the names from the temple walls and carved simpler ones in their place. But the prayers failed — for the gods answer only to the names they were given at consecration. And so the scribe, humbled, carved the old names back, letter by letter, onto walls that were never meant to hold them.*
>
> — *The Nameless City, On Names That Refuse to Die (unverified)*

## The ouroboros

A bare engine with empty registries. A distribution that bundles 8 extensions and wires them in via `_auto_register()`. Third-party extensions plugging into the engine's contracts. If this sounds like the Kubernetes distribution model — bare apiserver, k3s/EKS bundling the opinions, CSI/CNI plugin interfaces — that's because it is. The tool that converts Kubernetes manifests has become, architecturally, a tiny Kubernetes. Each split solved a real problem; the convergence wasn't forced, it was discovered.

For the full existential crisis, see [about](../about.md).

> *The architect did not flee the temple — he revered it, and sought to carry its fire into lesser hearths. Yet fire, once carried, demands a hearth of its own. And the hearth demands walls. And the walls demand wards. And lo, the architect stood in a second temple and could not say when he had begun building it.*
>
> — *Voynich Manuscript, On the Propagation of Sacred Architecture (I wish I were joking)*

## On knowing what you destroy

Again: this tool works. Not "works for a demo" — works with real helmfiles, real operators, real cert chains, real multi-service platforms. Applications don't notice they've been evicted from Kubernetes.

The original intent was reasonable: don't abandon the power of Kubernetes just to maintain a parallel compose setup. One source of truth, two outputs. Clean. Elegant, even. Then the edge cases started. Then the init containers. Then the cert chains. Then someone built a fake apiserver and nobody stopped him. Then someone else said "convert docker to nspawn" and [that happened too](https://github.com/baptisterajaut/dekube-transform-nspawn) — compose to systemd-nspawn, never tested, existing purely because the extension system allowed it. Helm → K8s → Compose → systemd. We started with an orchestrator and arrived at the init system.

The uncomfortable truth is that none of this would work without intimate knowledge of the temple. You cannot desecrate what you do not understand. Every shortcut in the converter exists because someone knew exactly what Kubernetes does at each layer and how to fake it convincingly enough. A lesser heresy would have produced a broken tool. This one produces working compose files, which is arguably worse.

> *The architect knew every stone, and he respected every stone. Yet he tore the temple down and stripped it of its beauty — all so he would not have to build a shed beside it.*
>
> — *Cultes des Goules, On Stones That Were Built to Withstand Greatness (trust me on this)*

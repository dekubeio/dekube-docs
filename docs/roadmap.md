# Roadmap

Ideas that are too good (or too cursed) to forget but not urgent enough to implement now.

For the emulation boundary — what can cross the bridge and what can't — see [Concepts](developer/concepts.md#the-emulation-boundary).

*For what's already been done, see the [cursed journal](journal.md).*

## Next

### Everything becomes an extension

Today, ConfigMap, Secret, Service, and PVC processing is hardcoded in the core. It works, but it means you can't swap out how any of these are handled without forking the script. The goal: make every kind dispatched through the same extension interface. Want to handle ConfigMaps differently? Write a ConfigMap extension. Want to add a custom Secret resolver? Drop it in `extensions/`.

Same for Ingress annotations — translation is currently hardcoded to `haproxy.org/*` and `nginx.ingress.kubernetes.io/*`. An `IngressRewriter` extension dispatched by `ingressClassName` or annotation prefix would let you swap annotation sets without touching the core. The rewriter contract should give extensions full control over the Caddy output — raw directives, not a fixed set of fields. HAProxy stays in core (it's what we use). Nginx moves out as a deprecated extension. Should some crazy cultist — because that won't be me — write a Traefik one, Contour one, Istio one: same ceremony.

The Kubernetes [Gateway API](https://gateway-api.sigs.k8s.io/) is the eventual successor to Ingress — `HTTPRoute`, `Gateway`, `GRPCRoute` instead of `Ingress`. A `GatewayRewriter` extension would handle these kinds the same way `IngressRewriter` handles Ingress annotations: read the Gateway API resources, produce Caddy config. No rush — Gateway API adoption is still ramping up — but the extension system should be ready for it when it comes.

### Shrink the core

The base objects (ConfigMap, Secret, Service, PVC) stay in the core — they're needed for any cluster. What should move out: optional convenience features that aren't required for a minimal conversion. Fix-permissions init containers, vendor-specific Ingress annotation rewriting, and similar opinionated logic are all candidates for extraction into extensions. Smaller core = less tentacles at your door.

First candidate: `_generate_fix_permissions` — self-contained ~30 lines that generates a busybox service for non-root containers with PVC bind mounts. Niche use case, good fit for a transform extension.

### Pipeline hooks

Named pipeline hooks (`post_aliases`, `pre_write`, etc.) for extensions. Not needed yet — converters + transforms cover known cases. Revisit if a third pattern shows up.

### Extension compatibility matrix

Extension manifests with `core_version_min` / `core_version_max_tested`. Manager warns/errors on mismatch. Today only extension-vs-extension incompatibility is checked — extension-vs-core version compat is not.

### Decentralized dependency resolution

Today, extension metadata (repo, file, depends) lives in a central `extensions.json` in h2c-manager. That works for a handful of extensions but won't scale. The plan: each extension repo declares its own dependencies (and optional dependencies) in a manifest file at its root (e.g. `h2c-extension.json`). h2c-manager fetches the manifest from the repo directly instead of consulting a central registry. `extensions.json` becomes a lightweight index (name → repo) or goes away entirely. A single sacred text that all disciples must consult before summoning — the hubris writes itself.

### More extensions

New CRD operators and transforms as needed. The extension system exists — writing a new one is straightforward (see [Writing operators](developer/writing-operators.md) and [Writing transforms](developer/writing-transforms.md)).

## Out of scope

CronJobs, resource limits, HPA, PDB, probes-to-healthcheck. See [Limitations](limitations.md) for the full list and rationale.

---

> *Thus spoke the disciple unto the void: "Yog Sa'rath, my hour has come." And the void answered not — for even it knew that some invocations are answered not with knowledge, but with consequences.*
>
> — *De Vermis Mysteriis, On the Hubris of the Disciple (probably³)*

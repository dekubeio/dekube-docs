# Roadmap

Ideas that haven't earned implementation yet — or that the architect hasn't recovered enough bandwidth to face.

For the emulation boundary — what can cross the bridge and what can't — see [Concepts](understand/concepts.md#the-emulation-boundary).

## Next

*(Nothing here — the void is sated, for now.)*

## Someday

### Gateway API

The Kubernetes [Gateway API](https://gateway-api.sigs.k8s.io/) is the eventual successor to Ingress — `HTTPRoute`, `Gateway`, `GRPCRoute` instead of `Ingress`. A `GatewayRewriter` extension would handle these kinds the same way `IngressRewriter` handles Ingress annotations: read the Gateway API resources, produce reverse proxy config. No rush — Gateway API adoption is still ramping up — but the extension system should be ready for it when it comes.

### Pipeline hooks

Named pipeline hooks (`post_aliases`, `pre_write`, etc.) for extensions. Not needed yet — converters + transforms cover known cases. Revisit if a third pattern shows up. So far, two patterns have shown up. The third is watching.

### Extension compatibility matrix

Extension manifests with `core_version_min` / `core_version_max_tested`. Manager warns/errors on mismatch. Today only extension-vs-extension incompatibility is checked — extension-vs-core version compat is not.

### Additional structured ingress fields

Ingress entries currently support `response_headers` and `max_body_size` as structured, provider-agnostic fields. Other annotations exist in the wild — `auth-url` / `auth-signin` (external auth), `limit-rps` (rate limiting), `permanent-redirect`, `temporal-redirect`, etc. Each one would need a structured field in the entry contract, then implementation in every ingress provider.

This is inherently a losing game: Kubernetes ingress controllers have dozens of annotations, each controller invents its own, and we're flattening all of that into a compose reverse proxy that was never designed to speak this language. New structured fields will be added as real-world projects hit the gap — not preemptively. `extra_directives` remains as a deprecated, provider-specific escape hatch for anything not yet structured.

### helmfile2swarm distribution

A Swarm-oriented distribution with different monks — distributed volumes, `deploy.replicas`, Traefik in mesh mode. The engine supports it, the contracts are ready, the [architecture page](understand/architecture.md#beyond-single-host) has the blueprint. I've never used Swarm, and I hope I never will. But the door is open for anyone who wants to walk through it.

## Out of scope

CronJobs, resource limits, HPA, PDB, probes-to-healthcheck. These survived the flattening by virtue of not being worth flattening. See [Limitations](limitations.md) for the full list and rationale.

---

> *Thus spoke the disciple unto the void: "Yog Sa'rath, my hour has come." And the void answered not — for even it knew that some invocations are answered not with knowledge, but with consequences.*
>
> — *De Vermis Mysteriis, On the Hubris of the Disciple (probably³)*

*For what's already been done, see the [cursed journal](journal.md).*
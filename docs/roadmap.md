# Roadmap

Ideas that haven't earned implementation yet — or that the architect hasn't recovered enough bandwidth to face.

For the emulation boundary — what can cross the bridge and what can't — see [Concepts](understand/concepts.md#the-emulation-boundary).

## Next

### Nginx ingress provider

An `IngressProvider` that produces an Nginx reverse proxy service + `nginx.conf` instead of Caddy. For users who need Nginx specifically — corporate environments, existing Nginx expertise, or setups where Caddy's automatic TLS isn't wanted. The rewriter already exists ([dekube-rewriter-nginx](https://github.com/dekubeio/dekube-rewriter-nginx)); this would be the provider counterpart. See [Writing ingress providers](extend/extensions/writing-ingressproviders.md) for the contract.

### Overrides vs transforms ordering

`overrides:` currently runs *before* transforms, so services created by transforms (like `fix-permissions`) can't be overridden. Either overrides need to run after transforms, or transforms need to check `enabled` themselves.

## Someday

### Gateway API

The Kubernetes [Gateway API](https://gateway-api.sigs.k8s.io/) is the eventual successor to Ingress — `HTTPRoute`, `Gateway`, `GRPCRoute` instead of `Ingress`. A `GatewayRewriter` extension would handle these kinds the same way `IngressRewriter` handles Ingress annotations: read the Gateway API resources, produce reverse proxy config. No rush — Gateway API adoption is still ramping up — but the extension system should be ready for it when it comes.

### Pipeline hooks

Named pipeline hooks (`post_aliases`, `pre_write`, etc.) for extensions. Not needed yet — converters + transforms cover known cases. Revisit if a third pattern shows up. So far, two patterns have shown up. The third is watching.

### Promote `_`-prefixed helpers to public API

Several conversion primitives exported by dekube-engine (`_convert_command`, `_convert_volume_mounts`, `_build_alias_map`, `_build_service_port_map`, `_resolve_named_port`) are used by bundled extensions and useful to any third-party provider or indexer. They're currently `_`-prefixed (historical — they lived in the monolith before the split) but exported in `__all__`. Drop the underscore, document them in the pacts API, remove `_build_vol_map` (unused outside `_convert_volume_mounts`).

### Extension compatibility matrix

Extension manifests with `core_version_min` / `core_version_max_tested`. Manager warns/errors on mismatch. Today only extension-vs-extension incompatibility is checked — extension-vs-core version compat is not.

### helmfile2swarm distribution

A Swarm-oriented distribution with different monks — distributed volumes, `deploy.replicas`, Traefik in mesh mode. The engine supports it, the contracts are ready, the [architecture page](understand/architecture.md#beyond-single-host) has the blueprint. I've never used Swarm, and I hope I never will. But the door is open for anyone who wants to walk through it.

## Out of scope

CronJobs, resource limits, HPA, PDB, probes-to-healthcheck. These survived the flattening by virtue of not being worth flattening. See [Limitations](limitations.md) for the full list and rationale.

---

> *Thus spoke the disciple unto the void: "Yog Sa'rath, my hour has come." And the void answered not — for even it knew that some invocations are answered not with knowledge, but with consequences.*
>
> — *De Vermis Mysteriis, On the Hubris of the Disciple (probably³)*

*For what's already been done, see the [cursed journal](journal.md).*
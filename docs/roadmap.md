# Roadmap

Ideas that are too good (or too cursed) to forget but not urgent enough to implement now.

For the emulation boundary — what can cross the bridge and what can't — see [Concepts](developer/concepts.md#the-emulation-boundary).


## v3.1.0 — the deconstruction ✓ {#v31--the-deconstruction}

*Released 2026-02-23.* See the [journal entry](journal.md#v310--the-unlucky-seven) for details.

v3.0 split the engine from the distribution. v3.1 finished the job:

- **The distribution is a manifest.** Every built-in extension now lives in its own repo. The `helmfile2compose` repo is a `distribution.json` + CI workflow — a shopping list, not code.
- **`caddy_entries` → `ingress_entries`**, `disableCaddy` → `disable_ingress`, config keys normalized to snake_case with automatic migration.
- **Typed return contracts.** `ConvertResult` split into `ConverterResult` (no services) and `ProviderResult` (with services). `ConvertResult` kept as deprecated alias.
- **The Seven Bishops** — the founding extensions — each released as their own repo:

| Title | Repo | Heresy |
|-------|------|--------|
| **The Builder** | [h2c-converter-workload](https://github.com/helmfile2compose/h2c-converter-workload) | 7/10 |
| **The Gatekeeper** | [h2c-provider-caddy](https://github.com/helmfile2compose/h2c-provider-caddy) | 3/10 |
| **The Herald** | [h2c-rewriter-haproxy](https://github.com/helmfile2compose/h2c-rewriter-haproxy) | 2/10 |
| **The Weaver** | [h2c-indexer-simple-service](https://github.com/helmfile2compose/h2c-indexer-simple-service) | 2/10 |
| **The Binder** | [h2c-indexer-pvc](https://github.com/helmfile2compose/h2c-indexer-pvc) | 1/10 |
| **The Librarian** | [h2c-indexer-configmap](https://github.com/helmfile2compose/h2c-indexer-configmap) | 0/10 |
| **The Guardian** | [h2c-indexer-secret](https://github.com/helmfile2compose/h2c-indexer-secret) | 0/10 |

## Later

### Gateway API

The Kubernetes [Gateway API](https://gateway-api.sigs.k8s.io/) is the eventual successor to Ingress — `HTTPRoute`, `Gateway`, `GRPCRoute` instead of `Ingress`. A `GatewayRewriter` extension would handle these kinds the same way `IngressRewriter` handles Ingress annotations: read the Gateway API resources, produce reverse proxy config. No rush — Gateway API adoption is still ramping up — but the extension system should be ready for it when it comes.

### Pipeline hooks

Named pipeline hooks (`post_aliases`, `pre_write`, etc.) for extensions. Not needed yet — converters + transforms cover known cases. Revisit if a third pattern shows up.

### Extension compatibility matrix

Extension manifests with `core_version_min` / `core_version_max_tested`. Manager warns/errors on mismatch. Today only extension-vs-extension incompatibility is checked — extension-vs-core version compat is not.

### More extensions

New CRD extensions (providers and converters) and transforms as needed. The extension system exists — writing a new one is straightforward (see [Writing converters](developer/extensions/writing-converters.md) and [Writing transforms](developer/extensions/writing-transforms.md)).

## Out of scope

CronJobs, resource limits, HPA, PDB, probes-to-healthcheck. See [Limitations](limitations.md) for the full list and rationale.

---

> *Thus spoke the disciple unto the void: "Yog Sa'rath, my hour has come." And the void answered not — for even it knew that some invocations are answered not with knowledge, but with consequences.*
>
> — *De Vermis Mysteriis, On the Hubris of the Disciple (probably³)*

*For what's already been done, see the [cursed journal](journal.md).*
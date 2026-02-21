# Roadmap

Ideas that are too good (or too cursed) to forget but not urgent enough to implement now.

For the emulation boundary — what can cross the bridge and what can't — see [Concepts](developer/concepts.md#the-emulation-boundary).


## v3.1.0 — the deconstruction {#v31--the-deconstruction}

v3.0 split the engine from the distribution. v3.1 finishes the job: every built-in extension becomes its own repo, and the distribution becomes a manifest. Also the right time to clean up the API debt accumulated during the vibe-coded era.

### The distribution becomes a manifest

Every extension currently bundled in the helmfile2compose distribution (`workloads.py`, `caddy.py`, `haproxy.py`, indexers) becomes its own repo, its own extension referenced in `extensions.json`. The `helmfile2compose` repo shrinks to a single `distribution.yaml` — a manifest that `build-distribution.py` reads, calls h2c-manager with a new `--fetch-extensions-only` flag to download everything, and assembles the single-file script.

`helmfile2compose` stops being code and becomes a shopping list. It will forever carry the short but horrifying history of a project that started from a simple need and became far, *far* too complicated for what it was supposed to be.

### Rename `caddy_entries` to `ingress_entries`

`ConvertResult.caddy_entries`, `disableCaddy`, `caddy_email`, `caddy_tls_internal` — these names leak the default provider into the core API and config schema. Now that `IngressProvider` is an abstract base class and the system is provider-agnostic, rename to generic terms (`ingress_entries`, `disable_ingress`, `ingress_email`, `ingress_tls_internal` or similar). Breaking change for extensions that reference `caddy_entries` directly.

### Config key naming consistency

`helmfile2compose.yaml` mixes camelCase (`disableCaddy`, `ingressTypes`, `helmfile2ComposeVersion`) and snake_case (`volume_root`, `caddy_email`, `caddy_tls_internal`, `core_version`). Normalize to one convention (snake_case, with camelCase aliases for backwards compatibility during a transition period). Overlaps with the `caddy_entries` rename above — do both at once.

### Typed return contracts {#typed-return-contracts}

`ConvertResult` is shared by all extension types, but the dispatch silently discards `services` from non-`Provider` converters (with a stderr warning). The contract doesn't enforce this — a `Converter` can build services, return them, and only discover they were ignored by reading the warning. Split `ConvertResult` into typed variants (`ConverterResult` without `services`, `ProviderResult` with) or add runtime validation in the base classes so the contract itself guarantees what the dispatch already enforces.

## Later

### The distribution family

Two distributions stacked on top of each other. A bare engine without reverse proxy has no real use case outside extension development — h2c-core fills that role already. The intermediate layer ("helmfile2basic") was dropped.

```
h2c-core (bare engine — for extension devs, not a distribution)
  └─ helmfile2compose (the OG — core + indexers + workloads + Caddy + HAProxy)
       └─ kube2easy (turnkey — everything + bootstrap script)
```

| Distribution | What's in it | What it is |
|---|---|---|
| **helmfile2compose** | h2c-core + indexers + WorkloadConverter + Caddy + HAProxy rewriter | The OG. Patient zero, fully armed. Ingress → Caddy, h2c-manager for the rest. |
| **kube2easy** | helmfile2compose + all official extensions + bootstrap script | *Kubernetes made easy by removing Kubernetes. Regrets optional.* |

**kube2easy** drops "helmfile" from the name — it's the turnkey entry point, and naming it after a dependency contradicts the zero-friction promise. The bootstrap script checks the environment, installs helm and helmfile if needed, and runs. Zero external dependencies beyond Python. The `*2*` pattern echoes helmfile2compose; the double reading ("kube to easy" / "kube too easy") is intentional.

Name `dekube` is reserved (verified free on GitHub) — no immediate use, but too good to let go. Future alias, subproject, or rename candidate.

Requires: stacking support in `build-distribution.py` (build on top of a parent distribution, not just bare h2c-core). Blocked by v3.1 (extensions must be externalized first).

### Gateway API

The Kubernetes [Gateway API](https://gateway-api.sigs.k8s.io/) is the eventual successor to Ingress — `HTTPRoute`, `Gateway`, `GRPCRoute` instead of `Ingress`. A `GatewayRewriter` extension would handle these kinds the same way `IngressRewriter` handles Ingress annotations: read the Gateway API resources, produce reverse proxy config. No rush — Gateway API adoption is still ramping up — but the extension system should be ready for it when it comes.

### Pipeline hooks

Named pipeline hooks (`post_aliases`, `pre_write`, etc.) for extensions. Not needed yet — converters + transforms cover known cases. Revisit if a third pattern shows up.

### Extension compatibility matrix

Extension manifests with `core_version_min` / `core_version_max_tested`. Manager warns/errors on mismatch. Today only extension-vs-extension incompatibility is checked — extension-vs-core version compat is not.

### More extensions

New CRD extensions (providers and converters) and transforms as needed. The extension system exists — writing a new one is straightforward (see [Writing converters](developer/extensions/writing-converters.md) and [Writing transforms](developer/extensions/writing-transforms.md)).

## v3.0 — The Ouroboros ✓ {#v30--the-ouroboros}

**Shipped in v3.0.0.** The provider/converter distinction is now enforced via base classes: `Converter`, `Provider` (produces services), `IndexerConverter` (populates ctx), and `IngressProvider` (reverse proxy backends). The package was renamed from `helmfile2compose` to `h2c` internally, and `build-distribution.py` was introduced for building distributions. `_auto_register()` scans globals and registers all extension classes, with fatal duplicate-kind detection. See the [journal](journal.md#v300--the-ouroboros) for the full story.

## Out of scope

CronJobs, resource limits, HPA, PDB, probes-to-healthcheck. See [Limitations](limitations.md) for the full list and rationale.

---

> *Thus spoke the disciple unto the void: "Yog Sa'rath, my hour has come." And the void answered not — for even it knew that some invocations are answered not with knowledge, but with consequences.*
>
> — *De Vermis Mysteriis, On the Hubris of the Disciple (probably³)*

*For what's already been done, see the [cursed journal](journal.md).*
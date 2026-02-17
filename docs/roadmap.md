# Roadmap

Ideas that are too good (or too cursed) to forget but not urgent enough to implement now.

For the emulation boundary — what can cross the bridge and what can't — see [Concepts](developer/concepts.md#the-emulation-boundary).

## Done

The Moldavian Scam got its green card. ("Moldavian Scam" — *arnaque moldave* — is a French Hearthstone community reference to pro player Torlk, famous for pulling off improbably lucky plays in tournaments. Not a comment on Moldova.)

What started as a half-measure — CRD converters forging fake Deployments — is now a proper converter abstraction with external loading, a package manager, and its own GitHub org. The documents are no longer falsified. They are *official*.

- **Extension loading** (`--extensions-dir`) — CRD converters as external Python modules. Dispatch loop, `ConvertContext`/`ConvertResult`, dynamic loading all in place.
- **GitHub org** — `helmfile2compose/` org with separate repos for core, manager, docs, extensions.
- **h2c-manager** — lightweight package manager for installing h2c-core + extensions from GitHub releases.
- **Extension repos** — keycloak, cert-manager, trust-manager published as standalone repos with GitHub releases.
- **Deep merge for overrides** — nested dict merging instead of shallow `dict.update()`.
- **Hostname truncation** — services >63 chars get explicit `hostname:` to avoid sethostname failures.
- **Backend SSL** — Caddy TLS transport for HTTPS backends (server-ca, server-sni annotations).

## Next

### Everything becomes an extension

Today, ConfigMap, Secret, Service, and PVC processing is hardcoded in the core. It works, but it means you can't swap out how any of these are handled without forking the script. The goal: make every kind dispatched through the same extension interface. Want to handle ConfigMaps differently? Write a ConfigMap extension. Want to add a custom Secret resolver? Drop it in `extensions/`.

Same for Ingress annotations — translation is currently hardcoded to `haproxy.org/*` and `nginx.ingress.kubernetes.io/*`. An `IngressRewriter` extension dispatched by `ingressClassName` or annotation prefix would let you add Traefik, Contour, or whatever your cluster uses without touching the core.

### Shrink the core

The base objects (ConfigMap, Secret, Service, PVC) stay in the core — they're needed for any cluster. What should move out: optional convenience features that aren't required for a minimal conversion. Fix-permissions init containers, vendor-specific Ingress annotation rewriting, and similar opinionated logic are all candidates for extraction into extensions. Smaller core = less tentacles at your door.

### Decentralized dependency resolution

Today, extension metadata (repo, file, depends) lives in a central `extensions.json` in h2c-manager. That works for a handful of extensions but won't scale. The plan: each extension repo declares its own dependencies (and optional dependencies) in a manifest file at its root (e.g. `h2c-extension.json`). h2c-manager fetches the manifest from the repo directly instead of consulting a central registry. `extensions.json` becomes a lightweight index (name → repo) or goes away entirely. A single sacred text that all disciples must consult before summoning — the hubris writes itself.

### More extensions

New CRD operators as needed. The extension system exists — writing a new one is straightforward (see [Writing operators](developer/writing-operators.md)).

## Out of scope

CronJobs, resource limits, HPA, PDB, probes-to-healthcheck. See [Limitations](limitations.md) for the full list and rationale.

---

> *Thus spoke the disciple unto the void: "Yog Sa'rath, my hour has come." And the void answered not — for even it knew that some invocations are answered not with knowledge, but with consequences.*
>
> — *De Vermis Mysteriis, On the Hubris of the Disciple (probably³)*

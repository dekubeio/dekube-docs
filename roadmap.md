# Roadmap

Ideas that are too good (or too cursed) to forget but not urgent enough to implement now.

For the emulation boundary — what can cross the bridge and what can't — see [Concepts](developer/concepts.md#the-emulation-boundary).

## Done

The Moldavian Scam got its green card. [^1]

[^1]: "Moldavian Scam" (arnaque moldave) is a French Hearthstone community reference to pro player Torlk (a.k.a. "Jérémy Torlkany"), famous for pulling off improbably lucky plays in tournaments. Not a comment on Moldova.

What started as a half-measure — CRD converters forging fake Deployments — is now a proper converter abstraction with external loading, a package manager, and its own GitHub org. The documents are no longer falsified. They are *official*.

- **External operator loading** (`--extensions-dir`) — CRD converters as external Python modules. Dispatch loop, `ConvertContext`/`ConvertResult`, dynamic loading all in place.
- **GitHub org** — `helmfile2compose/` org with separate repos for core, manager, docs, operators.
- **h2c-manager** — lightweight package manager for installing h2c-core + operators from GitHub releases.
- **Operator repos** — keycloak, certmanager, trust-manager published as standalone repos with GitHub releases.
- **Deep merge for overrides** — nested dict merging instead of shallow `dict.update()`.
- **Hostname truncation** — services >63 chars get explicit `hostname:` to avoid sethostname failures.
- **Backend SSL** — Caddy TLS transport for HTTPS backends (server-ca, server-sni annotations).

## Next

### Everything becomes an extension

Today, ConfigMap, Secret, Service, and PVC processing is hardcoded in the core. It works, but it means you can't swap out how any of these are handled without forking the script. The goal: make every kind dispatched through the same extension interface. Want to handle ConfigMaps differently? Write a ConfigMap extension. Want to add a custom Secret resolver? Drop it in `extensions/`.

Same for Ingress annotations — translation is currently hardcoded to `haproxy.org/*` and `nginx.ingress.kubernetes.io/*`. An `IngressRewriter` extension dispatched by `ingressClassName` or annotation prefix would let you add Traefik, Contour, or whatever your cluster uses without touching the core.

### More operators

New CRD operators as needed. The extension system exists — writing a new one is straightforward (see [Writing operators](developer/writing-operators.md)).

## Out of scope

### CronJobs

Not converted. A CronJob would need an external scheduler or a `sleep`-loop wrapper, neither of which is a good idea.

### Resource limits / HPA / PDB

Compose supports `mem_limit` / `cpus`, but translating K8s resource semantics (requests vs limits, burstable QoS) is more misleading than helpful. Single-machine tool.

### Probes to healthcheck

The semantics differ enough that a blind translation causes more problems than it solves.

---

> *Thus spoke the disciple unto the void: "Yog Sa'rath, my hour has come." And the void answered not — for even it knew that some invocations are answered not with knowledge, but with consequences.*
>
> — *De Vermis Mysteriis, On the Hubris of the Disciple (don't quote me on this)*

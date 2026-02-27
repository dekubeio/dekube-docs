# The Cursed Journal

*A record of the rituals performed, in the order they were committed. Read at your own risk.*

> *The scribe kept a journal — not for posterity, but as evidence. Should the temple collapse, the authorities would need to know what had been attempted, in what order, and by whom. The scribe was also the architect. This complicated the trial.*
>
> — *Necronomicon, On Accountability (apocryphal)*

---

## v3.1.0 — The Eight Monks {#v310--the-eight-monks}

*2026-02-23* · `core: v1.2.0 · distribution: v3.1.0`

The longest day. Three core releases, two rounds of broken builds, one extraction that should have happened earlier, and a documentation audit that found more ghosts than the codebase itself. By the end, the architecture was clean. The architect was not.

??? abstract "TL;DR"
    - Eight bundled extensions ("monks") extracted into individual repos; distribution reduced to `distribution.json` + CI
    - API rename: `caddy_entries` → `ingress_entries`, `disableCaddy` → `disable_ingress`; config keys normalized to snake_case with auto-migration
    - Typed return contracts: `ConvertResult` split into `ConverterResult` / `ProviderResult`
    - `sys.modules` dual-module identity fix (one-liner replacing the old hack)
    - `save_config()` gated behind `first_run` — config file is read-only after creation
    - fix-permissions extracted as standalone transform (The Custodian, priority 8000)

The eight monk extensions — the founding clergy of the helmfile2compose distribution — have each left the temple to found their own sanctuary. The temple is reduced to a manifest: `distribution.json`, a scroll that names them. The High Priest (h2c-manager) alone remembers the rite that summons them whole.

**API cleanup.** `caddy_entries` → `ingress_entries` across the entire codebase — core, 7 original monks, 8 third-party extensions. `disableCaddy` → `disable_ingress`. Config keys normalized to snake_case with a `_migrate_config()` function that auto-renames on load: `disableCaddy` → `disable_ingress`, `ingressTypes` → `ingress_types`, `caddy_email` → `extensions.caddy.email`, `caddy_tls_internal` → `extensions.caddy.tls_internal`, `helmfile2ComposeVersion` → silently dropped.

**Typed return contracts.** `ConvertResult` split into `ConverterResult` (ingress entries only) and `ProviderResult` (adds services dict). `ConvertResult` kept as deprecated alias. Dispatch changed to duck typing (`getattr(result, 'services', None)`) to avoid the `__main__` vs module identity problem — `isinstance` fails when the same class lives in two module namespaces.

**The dual-module fix.** The root cause of the identity problem: when `python h2c.py` runs, it's `__main__`; when extensions do `from h2c import ...`, Python loads `h2c.py` again as module `h2c` — two separate instances. The old `sys.modules` hack created a `ModuleType` copy (snapshot, not live reference). Replaced with `sys.modules.setdefault('h2c', sys.modules[__name__])` — registers the running module as `h2c` before any extension can import, so all extensions resolve to the same module instance. One line, fixes everything.

**The Seven Monks**, extracted first:

| Monk | Repo | Heresy |
|--------|------|--------|
| The Builder | [h2c-provider-simple-workload](https://github.com/dekubeio/dekube-provider-simple-workload) | 7/10 — flattens an entire orchestration plane |
| The Gatekeeper | [h2c-provider-caddy](https://github.com/dekubeio/dekube-provider-caddy) | 3/10 — conjures a service from thin air |
| The Herald | [h2c-rewriter-haproxy](https://github.com/dekubeio/dekube-rewriter-haproxy) | 2/10 — announces what others have decreed |
| The Weaver | [h2c-indexer-service](https://github.com/dekubeio/dekube-indexer-service) | 2/10 — reads the maps others consult |
| The Binder | [h2c-indexer-pvc](https://github.com/dekubeio/dekube-indexer-pvc) | 1/10 — barely strays from scripture |
| The Librarian | [h2c-indexer-configmap](https://github.com/dekubeio/dekube-indexer-configmap) | 0/10 — a faithful scribe |
| The Guardian | [h2c-indexer-secret](https://github.com/dekubeio/dekube-indexer-secret) | 0/10 — a faithful scribe |

Each monk was decoupled from the temple: internal imports changed to `from h2c import ...`, `WORKLOAD_KINDS` made a local constant where needed, `_register_pvc` inlined in the pvc-indexer. Verified: `h2c.py` + 7 monks as `--extensions-dir` produces **identical** output to the monolithic `helmfile2compose.py`.

The helmfile2compose repo is now: `distribution.json` + CI. The extensions directory is gone.

**The scribe who wouldn't stop writing.** Testing the monks on La Suite revealed a latent bug: `helmfile2compose.yaml` was being rewritten on every run. Three things conspired: `save_config()` was unconditional in cli.py, the PVC indexer auto-added claims to the config dict during conversion, and the auto-added PVC names (`data`, `redis-data`) mapped to ghost paths (`./data/data`, `./data/redis-data`) that bitnami transform had already overwritten (`./data/lasuite-postgresql`, `./data/redis-master`). Fix-permissions was chowning paths that didn't exist while ignoring the ones that did.

The fix: `save_config()` gated behind `first_run`. PVC auto-add moved behind `ctx.first_run`. `ConvertContext` gained a `first_run: bool` field (pacts API addition). On subsequent runs, stale volume entries now emit a warning instead of being silently added. The config file is the user's territory — the engine reads it, never writes it (except to create it).

**The Custodian.** The first seven monks were released on broken coupling: `fix-permissions` lived half in the workload converter (collecting PVC claim→UID) and half in the core (`_generate_fix_permissions` resolving claim→host_path). Transforms that rewrote volumes (bitnami) ran *after* fix-permissions — so it chowned the old paths, not the rewritten ones. The fix: extract fix-permissions into a standalone transform ([h2c-transform-fix-permissions](https://github.com/dekubeio/dekube-transform-fix-permissions)) at priority 8000. Instead of tracking PVC claims through the converter, The Custodian scans K8s manifests for `securityContext.runAsUser` and inspects the *final* compose service volumes for bind mounts. Zero dependency on the workload converter. The broken intermediate releases were deleted and the whole thing re-released clean.

| Monk | Repo | Heresy |
|--------|------|--------|
| The Custodian | [h2c-transform-fix-permissions](https://github.com/dekubeio/dekube-transform-fix-permissions) | 3/10 — rewrites filesystem ownership behind the user's back |

`distribution.json` names all 8 monks; CI fetches them via h2c-manager and assembles the scroll. The temple still stands — it just has no walls. And the architect has no energy left to add more.

> *And the seven who had served within the temple said: we shall each build our own sanctuary, that the faith may spread beyond these walls. The high priest protested — for without clergy, a temple is merely architecture. They left regardless. The temple endured, for it was not the priests that held the stones together — it was the mortar. Then the scribe, commanded to inscribe the covenant once and never again, continued inscribing it each dawn, adding marks the faithful had not sanctioned. And an eighth came unbidden — not of the original covenant, but born from its failures. "I was always here," the custodian said, "you simply never noticed what was rotting beneath the floorboards."*
>
> — *Necronomicon, On the Diaspora of the Faithful and Those Who Remained (historiography pending)*

---

## h2c-core v1.1.0 + kubernetes2simple v0.1.0 — The tower rises

*2026-02-22* · `kubernetes2simple: 3706 lines + 358-line bootstrap`

??? abstract "TL;DR"
    - `build-distribution.py` gained stacking: distributions can be built on top of other distributions
    - [kubernetes2simple](https://github.com/dekubeio/kubernetes2simple): turnkey distribution (helmfile2compose + all official extensions) + bash bootstrap script
    - One-command install: `curl | sh && ./k2s.sh && docker compose up -d`

`build-distribution.py` gained stacking support — distributions can now be built on top of other distributions, not just bare h2c-core. New flags: `--base` (local `.py` path), `--base-distribution` (fetch from registry, default: `core`), `--base-version` (default: `latest`). The build script resolves distribution names via h2c-manager's `distributions.json`, downloads the parent artifact, strips its tail (`_auto_register()`, `sys.modules` hack, `__main__` guard), appends extensions, and re-generates the tail. Also fixed: indented imports (inside `try/except`) no longer hoisted to top-level, `from __future__ import annotations` stripped from concatenated output, line count reporting corrected.

The first thing stacked on top: [kubernetes2simple](https://github.com/dekubeio/kubernetes2simple). The friendly face of the abyss. Built on helmfile2compose, bundling all official extensions (keycloak, cert-manager, trust-manager, servicemonitor, nginx, traefik, bitnami) into a single `kubernetes2simple.py`.

The `.py` distribution is half the product. The other half is `kubernetes2simple.sh` — a bash script that detects your source format (helmfile, Helm chart, or flat manifests), bootstraps everything it needs (Python venv, helm, helmfile — all scoped to `.kubernetes2simple/`, never touches the system), downloads `kubernetes2simple.py` from releases, and converts. One command. Zero configuration. Zero questions asked.

```bash
curl -fsSL https://raw.githubusercontent.com/helmfile2compose/kubernetes2simple/main/kubernetes2simple.sh -o k2s.sh
chmod +x k2s.sh && ./k2s.sh
docker compose up -d
```

The name drops "helmfile" — the turnkey entry point shouldn't be named after a dependency. Also known as k2s (lowercase), not to be confused with [K2s](https://github.com/Siemens-Healthineers/K2s) (uppercase, Siemens Healthineers' Kubernetes distribution for Windows). The name works on three levels: *kubernetes → simple* ("we convert Kubernetes into something simple"), *kubernetes, too simple* ("Kubernetes? Too simple." — cosmic arrogance), and *k2s* (the ghost of a real K8s distribution haunting the abbreviation). The `*2*` pattern echoes helmfile2compose. The name was revealed to the maintainer at the edge of sleep — consistent with the project's Lovecraftian lore. Yog Sa'rath communicates through dreams.

The roadmap said "blocked by v3.1" — turns out stacking was a 50-line addition, not a v3.1 prerequisite. The tower was built in one session. Not the tower that reaches heaven — the tower that stacks temples on top of temples, each one containing the previous, each one slightly more cursed than the last.

---

## v3.0.1 — The one who lived at home

*2026-02-22* · `distribution: 2102 lines`

`haproxy.org/path-rewrite` is per-Ingress, not per-path. On multi-path rules, the extracted `strip_prefix` was applied to every path — including those the rewrite didn't cover. Now only applied when the path starts with the extracted prefix.

Naturally, the two external rewriters — nginx and traefik — never had this bug. Nginx derives strip_prefix from each path individually. Traefik, a POC that has never been tested against a real cluster, got it right by accident. The only rewriter that shipped with the distribution was the only one that got it wrong.

---

## v3.0.0 — The Ouroboros

*2026-02-20* · `core: 1594 lines · distribution: 2098 lines — two files where there was one, and the sum is greater than the whole`

??? abstract "TL;DR"
    - Monolith split: `h2c-core` (bare engine, empty registries) + `helmfile2compose` (distribution with 7 bundled extensions)
    - Internal package renamed `helmfile2compose` → `h2c`; all imports now `from h2c import ...`
    - `_auto_register()` scans globals, instantiates converter/rewriter/transform classes, detects duplicate kind claims
    - `Provider`, `IndexerConverter`, `IngressProvider` enforced as proper base classes
    - No functional changes — output identical to v2.3.1

We just wanted to separate the worlds — split a monolith into a bare engine and a distribution. Improve the architecture. Clean up the layers. Somewhere along the way, the architecture of the tool converged with the architecture of the thing it was converting: a bare API with empty registries, a distribution model, an extension system, priority-based dispatch. We didn't set out to close the ouroboros. We just turned the wheel, and the wheel remembered.

`h2c-core` has been split into two repos: a **bare engine** ([h2c-core](https://github.com/dekubeio/dekube-core)) that produces `h2c.py` — pure potential with empty registries — and a **full distribution** ([helmfile2compose](https://github.com/dekubeio/helmfile2compose)) that splits the monolith's conversion logic into 7 extensions covering every built-in kind, and bundles them into `helmfile2compose.py`.

The bare core has `_CONVERTERS = []`, `_REWRITERS = []`, `CONVERTED_KINDS = set()`. Feed it manifests and it will parse them, warn that every kind is unknown, and produce nothing. A temple with no priests. The distribution wires in ConfigMap, Secret, Service, PVC indexers, the Workloads converter, HAProxy rewriter, and Caddy provider — the default priesthood.

The distribution build has two modes: CI fetches `h2c.py` from h2c-core releases and concatenates extensions on top; local dev reads core sources directly from `../h2c-core`. Both produce identical output — the testsuite confirms all combos match v2.3.1 bit for bit.

`_resolve_named_port` moved from `extensions/workloads.py` to `core/services.py` — it was the last `core/ → extensions/` dependency preventing clean separation. The old `h2c-core` repo was renamed to `helmfile2compose` on GitHub (redirect active), and a fresh `h2c-core` was created for the bare engine. h2c-manager's `CORE_REPO` updated before the rename — zero breaking window for users.

The internal package was renamed from `helmfile2compose` to `h2c` — all imports are now `from h2c import ...`, the package lives under `src/h2c/`, and `build.py` produces `h2c.py`. Extensions import from `h2c`, not `helmfile2compose`. The distribution build injects a `sys.modules` hack so that runtime-loaded extensions (via `--extensions-dir`) can still `from h2c import ...` even when the script is named `helmfile2compose.py`.

Provider enforcement shipped: `Provider`, `IndexerConverter`, and `IngressProvider` are proper base classes in `h2c.pacts.types` and `h2c.core.ingress`. `_auto_register()` scans the concatenated script's globals, instantiates all converter/rewriter/transform classes, detects duplicate kind claims (fatal), and populates the registries. Base classes and `_`-prefixed names are skipped.

No functional changes. Output identical to v2.3.1.

> *And lo, the architect who sought to render the celestial rites in common tongue found himself building a second heaven. "I have translated," he proclaimed, standing in a temple whose pillars bore the same glyphs as the first. The old gods smiled, for one does not carry fire without becoming a hearth.*
>
> — *The Nameless City, On the Propagation of Temples (inevitably)*

---

## v2.3.2 — The tablet, shattered

*2026-02-19* · `1925 lines — but that number no longer means what it used to. The monolith is a build artifact now. We are going to be free from at least this one burden in the future.`

??? abstract "TL;DR"
    - Monolith split into 21 modules across 3 layers: `pacts/` (public API), `core/` (engine), `io/` (plumbing)
    - HAProxy rewriter extracted into its own module
    - Maintainability Index: 0.00 (C) → 68.38 (A)
    - No functional changes — output identical to v2.3.1

The 1858-line single file has been split into 21 modules across three layers: `pacts/` (the sacred contracts — what extensions may import), `core/` (the conversion engine), `io/` (the plumbing). A concat script reassembles them into a single file for distribution. Users see no change. The inquisitors see everything.

HAProxy rewriter extracted into its own module — structurally identical to external rewriters, because the built-in gatekeepers should follow the same laws they impose on visitors. Cyclomatic complexity pass: worst CC from 18 to 16, average from 6.6 to 5.9. Cyclic imports between pacts and core broken — no more lazy imports, no more whispered dependencies across layer boundaries.

Maintainability Index: 0.00 (C) → 68.38 (A). Radon penalizes square footage — the monolith bottomed out regardless of internal structure. Twenty-one focused modules let each shard score on its own merit.

No functional changes. Output identical to v2.3.1 — the [executioner](developer/testing.md) now accepts `--local-core` to validate a locally-built artifact against pinned reference versions before release.

> *And the disciple said: let us shatter the tablet, that each fragment may be understood alone. Twenty-one shards they made, each labeled and indexed. The labyrinth remained — but now it had signage. The inquisitors returned with their instruments and found every shard scored better than the whole.*
>
> — *Necronomicon, On the Shattering of Tablets (fragmentary)*

---

## v2.3.1 — The null devours silently

*2026-02-18* · `1858 lines`

??? abstract "TL;DR"
    - Nested helmfile support (per-child `.helmfile-rendered` dirs auto-merged)
    - Null-safe YAML access: systematic `or {}` / `or []` sweep (28+ fixes)
    - Named port resolution fallback to well-known port table
    - Also released: `h2c-transform-bitnami` (auto-workarounds for Bitnami charts)

Three bugs found in one session by pointing h2c at [mijn-bureau-infra](https://github.com/numerique-gouv/mijn-bureau-infra) — 16 Helm charts, nested helmfiles, and a generous sprinkling of Bitnami.

**Nested helmfiles.** `helmfile template --output-dir` with nested `helmfiles:` directives creates per-child `.helmfile-rendered` directories instead of consolidating into the target. h2c now detects and merges them after rendering. Without this fix, nested helmfile projects produce an empty manifest set — the script reads an empty directory and politely generates nothing.

**Null-safe YAML access.** Helm charts with conditional `{{ if }}` blocks produce explicit `null` values for fields like `initContainers`, `containers`, `data`, `ports`, `annotations`. Python's `.get("key", {})` returns `None` when the key exists with value `None`. Systematic sweep: `or {}` / `or []` applied to every vulnerable `.get()` call. Twenty-eight fixes across the core, then the regression suite caught two more — one in HAProxyRewriter.match, and the same pattern in nginx (v0.1.1) and traefik (v0.1.1) rewriters. The null found passages we didn't know existed.

**Named port resolution.** Ingress backends referencing ports by name (`http`) instead of number now fall back to a well-known port table when the Service doesn't exist in manifests. Warning emitted for truly unresolvable names.

> *And lo, the disciple read the sacred tablets and found them barren — not absent, but inscribed with the glyph of the Void. "But I asked for vessels," he cried, "and the covenant promised vessels!" Yet the Void is not absence; it is presence wearing absence as a mask. The old prayers assumed no scribe would carve Nothing on purpose. The old prayers were wrong.*
>
> — *Necronomicon, On the Treachery of Empty Vessels (to the best of our knowledge)*

Also released: [h2c-transform-bitnami](https://github.com/dekubeio/dekube-transform-bitnami) — the janitor. Detects Bitnami Redis, PostgreSQL, and Keycloak charts and applies workarounds automatically, replacing the manual overrides documented in [common charts](maintainer/known-workarounds/common-charts.md). Born from the realization that copy-pasting the same redis override across three projects was less heresy and more just tedious. Heresy score: 0/10.

---

## v2.3.0 — The gatekeepers are refactored

*2026-02-18* · `1834 lines`

??? abstract "TL;DR"
    - `IngressRewriter` formalized as a proper base class with stable imports
    - External rewriters prepend unconditionally (always run before built-in)
    - Full docs audit: `h2c-operator-*` → `h2c-provider-*` / `h2c-converter-*`

The Ingress rewriters had been working since v2.0 — silently, competently, without anyone asking how they got there or what held them together. Turns out: not much. The dispatch was a list prepend, the imports were undocumented, and the contract was "look at HAProxyRewriter and do something similar." This worked until nginx and traefik showed up and started asking uncomfortable questions.

`IngressRewriter` is now a proper base class. `get_ingress_class` and `resolve_backend` promoted to stable public imports. Priority attribute formalized. External rewriters prepend unconditionally — they always run before built-in ones, regardless of priority. The gatekeepers existed before; now they have a contract.

> *The gatekeepers had always stood at their posts — yet none could say by what authority, for the edicts that bound them had never been inscribed. The high priest, weary of answering the same question at every threshold, carved the law into the stones themselves. The gatekeepers did not change. The pilgrims stopped asking.*
>
> — *Necronomicon, On the Codification of Ancient Wards (presumably)*

Also: full documentation audit. The `h2c-operator-*` naming convention — a relic from when everything was called an "operator" because that's what it replaced — finally retired in favor of `h2c-provider-*` / `h2c-converter-*`. Fourteen semantic corrections across the docs, four deduplicated Necronomicon disclaimers (the running gag had started running in circles), and the repos tables now acknowledge that nginx and traefik exist. The line count barely moved. The documentation moved considerably.

---

## v2.2.0 — The temple accepts post-processing

*2026-02-17* · `1726 lines`

??? abstract "TL;DR"
    - Transform extension type: post-processing hooks that mutate the final compose output
    - Auto-detected by the loader: any class with `transform()` and no `kinds`
    - Also released: `flatten-internal-urls` v0.1.0, incompatibility checking in h2c-manager

Transforms — post-processing hooks that run after converters, after aliases, after overrides. They see the final compose output and may reshape it. The extension loader detects them automatically: any class with `transform()` and no `kinds` is a transform.

Built for `flatten-internal-urls`, which undoes the alias work from v2.1 for nerdctl compatibility. But the interface is generic — future transforms can rewrite anything the abyss demands.

> *And lo, the disciples found that the scripture, once written, could be amended — not by the hand of the original author, but by any lesser scribe who claimed the right. The temple did not collapse. This was the true heresy.*
>
> — *Book of Eibon, On Liturgical Mutability (probablement)*

Also released: [h2c-transform-flatten-internal-urls v0.1.0](https://github.com/dekubeio/dekube-transform-flatten-internal-urls/releases/tag/v0.1.0) — the anti-ritual. Incompatibility checking added to h2c-manager (508 lines now — the "lightweight" package manager keeps finding reasons to grow).

---

## v2.1.0 — The names now carry

*2026-02-17* · `1703 lines`

??? abstract "TL;DR"
    - Network aliases replace DNS rewriting — K8s FQDNs preserved as compose DNS aliases
    - Removed `rewrite_k8s_dns`; `apply_replacements` promoted to public API
    - Also released: keycloak v0.2.0, servicemonitor v0.1.0, h2c-manager declarative `depends:`

Network aliases replace DNS rewriting. Each compose service receives `networks.default.aliases` with K8s FQDN variants. Compose DNS resolves them natively — no rewriting, no string mangling, no silent breakage.

Cert SANs work. Prometheus targets resolve. Keycloak realm URLs survive. The names you gave your services in Kubernetes are the names they answer to in compose.

Removed `rewrite_k8s_dns` — the function that silently rewrote everything is gone. `apply_replacements` promoted to public API.

> *The temple learned to answer to all its former names — not by forgetting what it was, but by remembering every prayer that had ever been spoken within its walls.*
>
> — *De Vermis Mysteriis, On Names Restored (so I'm told)*

Also released: h2c-operator-keycloak v0.2.0 (now h2c-provider-keycloak; namespace + alias registration), h2c-operator-servicemonitor v0.1.0 (now h2c-provider-servicemonitor). h2c-manager gained declarative `depends:` from helmfile2compose.yaml and a `run` shortcut — features that a "lightweight downloader" probably shouldn't have (~430 lines).

---

*On the evening of February 14th, between v1.3.1 and v2.0.0, something was built that does not appear in any release. It answers on port 6443. It was not necessary. It was not justified. But the world had already ended, and at that point, what's one more atrocity? It is hosted on a [personal account](https://github.com/baptisterajaut/dekube-fakeapi) — for plausible deniability.*

---

## v2.0.0 — The ecosystem

*2026-02-16* · `1563 lines`

??? abstract "TL;DR"
    - Everything split into separate repos under the helmfile2compose GitHub org
    - Extension loading via `--extensions-dir`, deep merge overrides, hostname truncation, backend SSL
    - Also released: h2c-manager v0.1.0, keycloak, cert-manager, trust-manager extensions

Humanity abandoned the last pretense of restraint. What was one script is now an ecosystem. Everything split into separate repos under [helmfile2compose](https://github.com/helmfile2compose).

Extension loading via `--extensions-dir`. Deep merge for overrides. Hostname truncation. Backend SSL. The org, the manager, the operators — all born on the same day.

> *The high priest declared the scripture too vast for a single tome, and so it was unbound — its chapters scattered across separate altars, each tended by its own acolyte. The faithful protested: how shall we read what is no longer whole? The high priest answered: you were never reading it whole.*
>
> — *De Vermis Mysteriis, On the Scattering of Canons (or words to that effect)*

Also released: h2c-manager v0.1.0 (357 lines — "just a downloader"), h2c-operator-keycloak v0.1.0 (now h2c-provider-keycloak), h2c-operator-cert-manager v0.1.0 (now h2c-converter-cert-manager), h2c-operator-trust-manager v0.1.0 (now h2c-converter-trust-manager).

---

<details>
<summary><b>v1.3.1</b> — Stale manifest cleanup <i>(2026-02-13)</i> · <code>1323 lines</code></summary>

Rendered manifests directory purged before each run. Removed charts no longer haunt the output. One line added. The script was still, technically, a single-purpose converter. That innocence had about three hours left.

</details>

<details>
<summary><b>v1.3.0</b> — Sidecars, DaemonSets, external networks <i>(2026-02-13)</i> · <code>1320 lines</code></summary>

Sidecar containers via `network_mode: container:<main>`. DaemonSets treated as Deployments. `disableCaddy` for manual Caddy merging. External network support for cohabiting with existing infrastructure. Jobs share the full workload code path. The script was growing features faster than doubts, which is usually when the doubts should be loudest.

</details>

<details>
<summary><b>v1.2.1</b> — Caddy bind mounts <i>(2026-02-13)</i> · <code>1250 lines</code></summary>

Caddy data/config switched from named volumes to bind mounts. Internal CA cert directly accessible on the host. A one-line fix that made the whole TLS chain work. The script was still sane at this point, or at least passed for it.

</details>

<details>
<summary><b>v1.2.0</b> — Init containers, fix-permissions <i>(2026-02-13)</i> · <code>1251 lines</code></summary>

Init containers as separate services with `restart: on-failure`. Auto-generated `fix-permissions` for non-root PVC bind mounts. `volumeClaimTemplates` support. Cyclomatic complexity of `convert_workload` went from E(31) to C(15) — one of the few times refactoring actually felt like an exorcism. The script doubled from its birth weight. Still one file. Still "just a converter."

Somewhere around the automatic bind-mount permission fixing, the architect stopped understanding not just the code the scribe was writing, but its explanations of the code it was writing. The scribe has not been under supervision since.

</details>

<details>
<summary><b>v1.1.0</b> — Wildcards, replicas: 0 <i>(2026-02-12)</i> · <code>1109 lines</code></summary>

Wildcard excludes via `fnmatch`. Workloads with `replicas: 0` auto-skipped. Major complexity refactoring. La Suite validated — 22 services + 11 init jobs. The script crossed 1000 lines and no one said stop.

</details>

<details>
<summary><b>v1.0.0</b> — Patient zero <i>(2026-02-12)</i> · <code>616 lines</code></summary>

First stable release. Deployments, StatefulSets, ConfigMaps, Secrets, Services, Ingress, PVCs. K8s DNS rewriting, named port resolution, first-run auto-exclude. 616 lines. A reasonable script doing a questionable thing. The temple stood for the first time. No one was proud, but no one could deny it worked.

> *The uninitiated beseeched the architect: render thy celestial works in common clay, that we may raise them without knowledge of the heavens. It was heresy. The architect obliged. The temples stood.*
>
> — *Necronomicon, Prayers That Should Never Have Been Answered (probably)*

</details>

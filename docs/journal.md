# The Cursed Journal

*A record of the rituals performed, in the order they were committed. Read at your own risk.*

> *The scribe kept a journal — not for posterity, but as evidence. Should the temple collapse, the authorities would need to know what had been attempted, in what order, and by whom. The scribe was also the architect. This complicated the trial.*
>
> — *Necronomicon, On Accountability (apocryphal)*

---

## v2.2.0 — The temple accepts post-processing

*2026-02-17* · `1726 lines`

Transforms — post-processing hooks that run after converters, after aliases, after overrides. They see the final compose output and may reshape it. The extension loader detects them automatically: any class with `transform()` and no `kinds` is a transform.

Built for `flatten-internal-urls`, which undoes the alias work from v2.1 for nerdctl compatibility. But the interface is generic — future transforms can rewrite anything the abyss demands.

> *And lo, the disciples found that the scripture, once written, could be amended — not by the hand of the original author, but by any lesser scribe who claimed the right. The temple did not collapse. This was the true heresy.*
>
> — *Book of Eibon, On Liturgical Mutability (probablement)*

Also released: [h2c-transform-flatten-internal-urls v0.1.0](https://github.com/helmfile2compose/h2c-transform-flatten-internal-urls/releases/tag/v0.1.0) — the anti-ritual. Incompatibility checking added to h2c-manager (508 lines now — the "lightweight" package manager keeps finding reasons to grow).

---

## v2.1.0 — The names now carry

*2026-02-17* · `1703 lines`

Network aliases replace DNS rewriting. Each compose service receives `networks.default.aliases` with K8s FQDN variants. Compose DNS resolves them natively — no rewriting, no string mangling, no silent breakage.

Cert SANs work. Prometheus targets resolve. Keycloak realm URLs survive. The names you gave your services in Kubernetes are the names they answer to in compose.

Removed `rewrite_k8s_dns` — the function that silently rewrote everything is gone. `apply_replacements` promoted to public API.

> *The temple learned to answer to all its former names — not by forgetting what it was, but by remembering every prayer that had ever been spoken within its walls.*
>
> — *De Vermis Mysteriis, On Names Restored (so I'm told)*

Also released: h2c-operator-keycloak v0.2.0 (namespace + alias registration), h2c-operator-servicemonitor v0.1.0. h2c-manager gained declarative `depends:` from helmfile2compose.yaml and a `run` shortcut — features that a "lightweight downloader" probably shouldn't have (~430 lines).

---

*On the evening of February 14th, between v1.3.1 and v2.0.0, something was built that does not appear in any release. It answers on port 6443. It was not necessary. It was not justified. But the world had already ended, and at that point, what's one more atrocity? It is hosted on a [personal account](https://github.com/baptisterajaut/h2c-api) — for plausible deniability.*

---

## v2.0.0 — The ecosystem

*2026-02-16* · `1563 lines`

Humanity abandoned the last pretense of restraint. What was one script is now an ecosystem. Everything split into separate repos under [helmfile2compose](https://github.com/helmfile2compose).

Extension loading via `--extensions-dir`. Deep merge for overrides. Hostname truncation. Backend SSL. The org, the manager, the operators — all born on the same day.

> *The temple was not translated — it was dismantled, stone by stone, and rebuilt as a shed. The prayers still worked. The architect watched, powerless, as the faithful praised the shed.*
>
> — *De Vermis Mysteriis, On Unnecessary Simplifications (probably, again)*

Also released: h2c-manager v0.1.0 (357 lines — "just a downloader"), h2c-operator-keycloak v0.1.0, h2c-operator-cert-manager v0.1.0, h2c-operator-trust-manager v0.1.0.

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

> *The disciples beseeched the architect: render thy celestial works in common clay, that we may raise them without knowledge of the heavens. It was heresy. The architect obliged. The temples stood.*
>
> — *Necronomicon, Prayers That Should Never Have Been Answered (probably)*

</details>

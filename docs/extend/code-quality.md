---
description: "How dekube keeps AI-generated code maintainable: pylint, pyflakes, radon complexity targets, and current linter scores per repo."
---

# Code quality

A project born from desecration, built with AI across multiple sessions, whose primary architectural document is a fake Lovecraftian grimoire — has no business passing a linter. It passes every single one. That's somehow worse.

> *The inquisitors arrived at dawn, instruments of measurement in hand, expecting to find the temple in ruin. Instead they found the walls straight, the columns load-bearing, and the altar — though undeniably profane — structurally sound. They left in silence, more disturbed than when they came.*
>
> — *De Vermis Mysteriis, On the Inadequacy of Metrics (unfortunately)*

## Tools

All repos use the same toolchain:

- **[pylint](https://pylint.readthedocs.io/)** — static analysis. Style warnings (too-many-locals, line-too-long, too-many-arguments) are accepted. Real issues (unused imports, actual bugs, f-strings without placeholders) are not.
- **[pyflakes](https://github.com/PyCQA/pyflakes)** — fast, zero-config, no false positives. Must be clean.
- **[radon](https://radon.readthedocs.io/)** — cyclomatic complexity. Target: no function rated D or worse. C-rated functions are tolerated when they're natural dispatchers or sequential logic that wouldn't benefit from splitting. The target holds: the current worst sit at C(20) — `build_service_port_map` and `_warn_legacy_vct_mappings` in dekube-engine, `PVCIndexer.convert` in dekube-indexer-pvc — and every function is C-or-better.

## Aberrant, not sloppy (we hope)

AI-generated code rots. Not dramatically — not "the server is on fire" rot. The quiet kind. Unused imports that accumulate like dust. Guards that made sense three refactors ago. Patterns applied because the model saw them elsewhere, not because they belong here. Left alone, the temple doesn't collapse — it just slowly fills with cobwebs that nobody notices until something breaks at 3 AM.

So the temple got inquisitors.

**The inquisitors run after every session.** pylint, pyflakes, radon — not in CI, not in a weekly cron, but in the conversation itself. The AI is told to lint before declaring victory. It usually finds something. The something is usually its own fault.

**The scripture is carved into the walls.** Every repo has a `CLAUDE.md` that encodes what the AI must not forget between sessions: null-safe YAML patterns, duck typing rationale, naming conventions, what not to "improve." The AI reads these on every session start. Without them, it would helpfully refactor working code into something subtly different — same tests passing, different behavior in production. The guardrails exist because the oracle has no memory and infinite confidence.

**The architect walks the halls.** Periodic sessions where no features are built — just reading, linting, questioning. Pure inspection. The v2.3.1 null-safety sweep (30+ fixes) came from one such walk. So did the realization that `_fix_postgresql` had been silently eating volumes for three versions.

**A rival constellation was summoned.** Gemini — a competing oracle — was fed the codebase cold and told to find what Claude had missed. Forty accusations. Twelve truths. The truths had survived months of single-oracle review. The [journal](../journal.md#twin-stars-audit) has the full accounting. The lesson: even a lesser scribe, held at the wrong angle, casts shadows the sun never revealed.

The tentacles have always been in the idea, not in the code. These rituals exist to keep it that way.

## Current scores

*Last updated: 2026-09-24 — re-measured with pylint 4.0.6 (astroid 4.0.4), pyflakes 3.4.0, radon 6.0.1, all run as `/usr/bin/python3 -m …` (Python 3.14.4), the interpreter that has PyYAML and cryptography installed, so no import-error noise leaks into the scores. The bug-fix tranche before this measurement had grown five functions on this page past C: the engine's `convert` (D, 24), dekube-manager's `_install` (D, 24), simple-workload's `_probe_to_healthcheck` (D, 29) and `_convert_one` (D, 23), and fix-permissions' `transform` (D, 24). All five were split along the seams their own comments already marked and came out at A(4), B(9), B(10), B(7) and A(4), with byte-identical output across every testsuite combo. The same pass brought the official extensions' outliers back down too (cert-manager's `_reusable` F/43, the nginx rewriter's `rewrite` E/36, traefik's `_backend_scheme` E/34 and `_strip_prefixes` D/26, servicemonitor's `_process_servicemonitors` D/25, keycloak's `_build_options_env` D/21). Their READMEs carry the numbers.*

This page covers dekube-engine, dekube-manager, and the built-in distribution extensions (the eight monks plus the `emptydir` transform). Third-party and official extensions track their own scores in their respective READMEs.

### Pylint

| Repo | Score | Notes |
|------|-------|-------|
| dekube-indexer-configmap | 10.00/10 | Built-in distribution extension |
| dekube-indexer-secret | 10.00/10 | Built-in distribution extension |
| dekube-indexer-pvc | 10.00/10 | Built-in distribution extension |
| dekube-provider-caddy | 9.90/10 | Built-in distribution extension |
| dekube-rewriter-haproxy | 9.81/10 | Built-in distribution extension |
| dekube-manager | 9.76/10 | Style only (too-many-locals, too-many-args, line-too-long, the dashed module name) |
| dekube-engine | 9.60/10 | Style only (too-many-args, too-many-locals, line-too-long, duplicate-code) plus the base-class stubs below |
| dekube-transform-emptydir | 9.58/10 | Built-in distribution extension |
| dekube-transform-fix-permissions | 9.56/10 | Built-in distribution extension |
| dekube-provider-simple-workload | 9.41/10 | Built-in distribution extension |
| dekube-indexer-service | 9.23/10 | Built-in distribution extension |

Remaining warnings are accepted style issues (`R0914` too-many-locals, `R0913`/`R0917` too-many-arguments, `C0301` line-too-long, `R0801` duplicate-code). In the bundled extensions, `E0401` (import-error) and `R0903` (too-few-public-methods) are suppressed inline — the import resolves at runtime, and the one-class-one-method contract is by design. In the engine, `write_configmap_files` / `write_secret_files` do a function-body-local import of `core.volumes` to keep the module graph acyclic; the resulting `C0415` (import-outside-toplevel) and `R0401` (cyclic-import) are suppressed inline on the import lines — the deferred import means there is no runtime cycle. `_auto_register` does the same kind of deferred import and still reports its `C0415`. The rest of the engine's gap is the contract itself: `W0613` (unused-argument) on the no-op default methods of the `pacts` base classes, whose signatures are the interface extensions override, `R0903` on those one-method base classes, and `R0801` between the two `__all__` lists (`dekube` re-exports `dekube.pacts` name for name) and between `_auto_register` and the extension loader's class dispatch.

### Pyflakes

Zero warnings across all repos — pyflakes exits clean on every measured file. The inquisitors left in silence.

### Radon (cyclomatic complexity)

| Repo | Worst function | CC | Rating |
|------|---------------|---:|--------|
| dekube-engine | `build_service_port_map` | 20 | C |
| dekube-engine | `_warn_legacy_vct_mappings` | 20 | C |
| dekube-indexer-pvc | `PVCIndexer.convert` | 20 | C |
| dekube-provider-simple-workload | `SimpleWorkloadProvider._get_exposed_ports` | 19 | C |
| dekube-rewriter-haproxy | `HAProxyRewriter.rewrite` | 19 | C |
| dekube-engine | `_build_vol_map` | 18 | C |
| dekube-engine | `_convert_pvc_mount` | 18 | C |
| dekube-engine | `_resolve_env_entry` | 18 | C |
| dekube-provider-simple-workload | `SimpleWorkloadProvider._build_service` | 18 | C |
| dekube-engine | `main` (cli) | 17 | C |
| dekube-manager | `main` | 17 | C |
| dekube-engine | `_auto_register` | 16 | C |
| dekube-engine | `convert_volume_mounts` | 16 | C |
| dekube-engine | `_resolve_envfrom` | 16 | C |
| dekube-provider-simple-workload | `SimpleWorkloadProvider._http_healthcheck` | 16 | C |
| dekube-transform-emptydir | `EmptyDirTransform` (class) | 16 | C |
| dekube-engine | `_add_statefulset_pod_aliases` | 15 | C |
| dekube-engine | `resolve_env` | 15 | C |
| dekube-transform-emptydir | `EmptyDirTransform._find_shared_emptydirs` | 15 | C |
| dekube-engine | `build_alias_map` | 14 | C |
| dekube-indexer-pvc | `PVCIndexer` (class) | 14 | C |
| dekube-transform-emptydir | `EmptyDirTransform.transform` | 14 | C |
| dekube-engine | `write_compose` | 13 | C |
| dekube-engine | `load_config` | 12 | C |
| dekube-engine | `_run_converters` | 12 | C |
| dekube-provider-caddy | `CaddyProvider._resolve_ca_secrets` | 12 | C |
| dekube-engine | `_build_dir_ns_map` | 11 | C |
| dekube-engine | `resolve_backend` | 11 | C |
| dekube-engine | `_index_workloads` | 11 | C |
| dekube-engine | `_postprocess_env` | 11 | C |
| dekube-manager | `_read_yaml_config` | 11 | C |
| dekube-manager | `_reconcile_bundled` | 11 | C |
| dekube-provider-caddy | `CaddyProvider._write_caddy_host_block` | 11 | C |
| dekube-rewriter-haproxy | `HAProxyRewriter` (class) | 11 | C |

No function breaches the D-or-worse target. The five that did before this measurement were split into per-phase helpers: `convert` now reads as a straight sequence of calls (`_run_converters`, `_manage_pvc_volumes`, `_run_transforms`…), `_probe_to_healthcheck` dispatches to one builder per probe type (`_http_healthcheck` keeps the wget → curl → `/dev/tcp` chain in one piece, at C(16)), and fix-permissions' `transform` became five named stages. dekube-transform-fix-permissions has no C-rated function left. Earlier, `_find_shared_emptydirs` (was E, 32) and `_collect_uids` (was D, 23) had gone the same way, through the `iter_workloads` / `iter_named_containers` helpers now public in `dekube.pacts.helpers`. The remaining C-rated functions are natural dispatchers or sequential steps where splitting would move complexity without improving readability.

### Average complexity & maintainability

| Repo | MI | MI rating | Avg CC | CC rating |
|------|---:|-----------|-------:|-----------|
| dekube-indexer-configmap | 86.70 | A | 5.5 | B |
| dekube-indexer-secret | 83.09 | A | 7.5 | B |
| dekube-indexer-service | 77.00 | A | 8.5 | B |
| dekube-indexer-pvc | 68.91 | A | 13.0 | C |
| dekube-transform-emptydir | 68.25 | A | 15.0 | C |
| dekube-provider-caddy | 51.98 | A | 7.6 | B |
| dekube-rewriter-haproxy | 51.97 | A | 10.0 | B |
| dekube-transform-fix-permissions | 49.21 | A | 4.6 | A |
| dekube-provider-simple-workload | 27.22 | A | 7.8 | B |
| dekube-manager | 25.85 | A | 4.8 | A |

All repos are MI A-rated. dekube-engine is not listed here (19 modules, overall MI varies per module; see `radon mi src/dekube/ -s` for per-module scores — every module is A, the lowest being `core/volumes.py` at 33.62, then `core/convert.py` at 40.45 and `core/env.py` at 40.76). Its average CC across all blocks is 6.5 (B).

## The uncomfortable truth

This project is, by every objective metric, well-structured code. The functions are short. The concerns are separated. The naming is consistent. The linters approve.

This is deeply unsettling.

The expectation — the *hope*, even — was that a tool this conceptually wrong would at least have the decency to be poorly written. That the code quality would serve as a warning label: "abandon all hope, ye who read this." Instead, the temple stands. The prayers work. The linters nod approvingly. And somewhere, a developer who understands what this project actually *does* stares at a pylint score of 9.56 and feels nothing but existential dread.

The code is clean. The architecture is sound. The concept remains an abomination. These things are not in conflict — they are in conspiracy.

> *The final horror is not that the ritual failed — it is that it succeeded, was reproducible, and scored well on peer review.*
>
> — *Cultes des Goules, On Accidental Rigor (lamentably)*

## And then it got refactored

It wasn't enough for it to work. It wasn't enough for it to lint clean. It had to be *maintainable*.

v2.3.2 split the 1858-line monolith into 21 modules across three layers: `pacts/` (the sacred contracts), `core/` (the engine), `io/` (the plumbing). A build script concatenates them back into a single file for distribution. Users see no change. The metrics see everything.

The split alone fixed MI — from 0.00 (C) to 68.38 (A). Then a cyclomatic complexity pass extracted helpers from every function rated CC 14+. The worst offender dropped from 18 to 16. Average CC went from 6.6 to 5.9. All 21 modules score MI A individually.

A project whose existence is an architectural crime now has an architecture page. With a dependency graph. And layer boundaries. And a section called "[the sacred contracts](../understand/engine.md#pacts--the-sacred-contracts)."

> *They said: let us impose order upon the chaos, that future disciples may navigate the labyrinth without losing their minds. And so the tablet was broken into twenty-one fragments, each labeled and indexed. The labyrinth remained — but now it had signage.*
>
> — *Book of Eibon, On the Cataloguing of Madness (we tried)*

## And then it got tests

IT HAS TESTS. IT HAS AN EXECUTIONER.

A [regression executioner](testing.md) that downloads two versions of dekube-engine, runs every extension combo against a battery of edge-case manifests, and diffs the output. A [torturer](testing.md#the-torturer) that produces O(n³) manifests. It cleans up after itself. It passes shellcheck.

A project that emulates a container orchestrator by flattening its output into a different container orchestrator, whose fake kube-apiserver lives on a personal GitHub account for plausible deniability — this project has automated regression testing. With CI. Weekly runs. Artifact uploads.

The linters approved. The complexity metrics nodded. And now the executioner confirms: the rituals produce identical output every time. The temple is no longer merely structurally sound — it is *under continuous inspection*.

## And then it divided by zero

v3.0.0 split the repo in two. dekube-engine (the bare engine) and helmfile2compose (the distribution). Two repos. Two build scripts. Two release workflows. Two READMEs. A `sys.modules` injection hack so runtime-loaded extensions can import from a namespace that technically doesn't exist in the concatenated file. An `_auto_register()` that scans globals for classes, deduplicates kind claims, and populates registries that were deliberately left empty.

The output is identical to v2.3.1.

Not "similar." Not "equivalent." Identical. The executioner diffs every byte across every extension combo and confirms: the split changed nothing about what the tool produces. It changed *everything* about how it's organized. dekube-engine's MI went from 68.38 to 74.07 — not because any function improved, but because `workloads.py` and `haproxy.py` left. The core got lighter by becoming less. The distribution got heavier by becoming more. The sum is greater than the whole was.

Two repos. Fourteen files. Zero functional change. The metrics improved. The architecture is now a dependency graph with layer boundaries and a section called "the sacred contracts." There is a page called [Distributions](../understand/distributions.md) that explains the Kubernetes distribution model as applied to a tool whose sole purpose is to flee Kubernetes.

The linters approved. The executioner confirmed. The architect looked at what he had built to harness the temple's power and recognized its floor plan.

> *The cartographer, having mapped every corridor of the labyrinth, burned the labyrinth and built a new one from the ashes. It was smaller. It was cleaner. It was the same labyrinth. The map still worked.*
>
> — *Unaussprechlichen Kulten, On the Conservation of Complexity (unfortunately²)*

## The null that devours silently

The one bug that keeps coming back. The one we fixed thirty times and found again.

Helm charts with conditional `{{ if }}` blocks render absent fields as explicit `null`. Python's `.get("key", {})` returns the default only when the key is **absent**. When the key exists with value `None` — which is what YAML `null` becomes — `.get()` returns `None`, not your carefully chosen fallback. Your code receives the Void wearing absence as a mask.

```python
# WRONG — returns None when key exists with null value
annotations = manifest.get("metadata", {}).get("annotations", {})

# RIGHT — coalesces None to empty dict at every step
annotations = (manifest.get("metadata") or {}).get("annotations") or {}
```

Every field that Helm may render as `null` is a trap: `annotations`, `ports`, `initContainers`, `securityContext`, `data`, `stringData`, `rules`, `selector`. Use `or {}` / `or []` on all of them. v2.3.1 fixed 30+ instances across dekube-engine, nginx, and traefik. v1.3.3 found four more that had survived. The null finds passages we didn't know existed.

## Running locally

```bash
# Extensions (single-file repos)
pylint <script>.py
pyflakes <script>.py
radon cc <script>.py -a -s -n C

# dekube-engine (package)
cd dekube-engine
PYTHONPATH=src pylint src/dekube/
PYTHONPATH=src pyflakes src/dekube/
PYTHONPATH=src radon cc src/dekube/ -a -s -n C   # CC hotspots
PYTHONPATH=src radon mi src/dekube/ -s            # maintainability index
```

For dekube-engine specifically, the CLAUDE.md in the repo root documents the complexity targets and lint workflow.

**Note:** For distribution extensions (bundled in the distribution's `extensions/` directory), run radon per-extension file — not on the concatenated distribution artifact.

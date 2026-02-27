# Code quality

A project born from desecration, vibe-coded across multiple sessions, whose primary architectural document is a fake Lovecraftian grimoire — has no business passing a linter. It passes every single one. That's somehow worse.

> *The inquisitors arrived at dawn, instruments of measurement in hand, expecting to find the temple in ruin. Instead they found the walls straight, the columns load-bearing, and the altar — though undeniably profane — structurally sound. They left in silence, more disturbed than when they came.*
>
> — *De Vermis Mysteriis, On the Inadequacy of Metrics (unfortunately)*

## Tools

All repos use the same toolchain:

- **[pylint](https://pylint.readthedocs.io/)** — static analysis. Style warnings (too-many-locals, line-too-long, too-many-arguments) are accepted. Real issues (unused imports, actual bugs, f-strings without placeholders) are not.
- **[pyflakes](https://github.com/PyCQA/pyflakes)** — fast, zero-config, no false positives. Must be clean.
- **[radon](https://radon.readthedocs.io/)** — cyclomatic complexity. Target: no function rated D or worse. C-rated functions are tolerated when they're natural dispatchers or sequential logic that wouldn't benefit from splitting. Current worst: `_read_yaml_config` (20) and `_collect_uids` (18), both C.

## Current scores

*Last updated: 2026-02-23 — the monolith shattered, and each shard scored better alone.*

This page covers dekube-engine, dekube-manager, and the built-in distribution extensions (the Eight Monks). Third-party extensions track their own scores in their respective READMEs.

### Pylint

| Repo | Score | Notes |
|------|-------|-------|
| dekube-indexer-configmap | 10.00/10 | Built-in distribution extension |
| dekube-indexer-secret | 10.00/10 | Built-in distribution extension |
| dekube-indexer-pvc | 10.00/10 | Built-in distribution extension |
| dekube-indexer-service | 10.00/10 | Built-in distribution extension |
| dekube-rewriter-haproxy | 10.00/10 | Built-in distribution extension |
| dekube-transform-fix-permissions | 9.86/10 | Built-in distribution extension |
| dekube-provider-caddy | 9.87/10 | Built-in distribution extension |
| dekube-manager | 9.78/10 | Style only (too-many-locals) |
| dekube-provider-simple-workload | 9.66/10 | Built-in distribution extension |
| dekube-engine | 9.55/10 | Style only (too-many-args, too-many-locals) |

Remaining warnings are accepted style issues (`R0914` too-many-locals, `R0913` too-many-arguments). `E0401` (import-error) and `R0903` (too-few-public-methods) are suppressed inline — the import resolves at runtime, and the one-class-one-method contract is by design.

### Pyflakes

Zero warnings across all repos. The inquisitors left in silence.

### Radon (cyclomatic complexity)

| Repo | Worst function | CC | Rating |
|------|---------------|---:|--------|
| dekube-manager | `_read_yaml_config` | 20 | C |
| dekube-transform-fix-permissions | `_collect_uids` | 18 | C |
| dekube-manager | `main` | 17 | C |
| dekube-engine | `_build_service_port_map` | 16 | C |
| dekube-provider-caddy | `_write_caddy_host_block` | 13 | C |
| dekube-provider-simple-workload | `SimpleWorkloadProvider._build_service` | 13 | C |
| dekube-engine | `_auto_register` | 13 | C |
| dekube-indexer-pvc | `PVCIndexer.convert` | 12 | C |
| dekube-rewriter-haproxy | `HAProxyRewriter.rewrite` | 12 | C |
| dekube-transform-fix-permissions | `FixPermissions` (class) | 12 | C |
| dekube-engine | `main` (cli) | 12 | C |
| dekube-engine | `_resolve_env_entry` | 11 | C |
| dekube-provider-simple-workload | `_get_exposed_ports` | 11 | C |
| dekube-provider-simple-workload | `SimpleWorkloadProvider._convert_one` | 11 | C |
| dekube-engine | `_resolve_secret_keys` | 11 | C |
| dekube-engine | `_convert_volume_mounts` | 11 | C |
| dekube-engine | `convert` | 11 | C |
| dekube-transform-fix-permissions | `FixPermissions.transform` | 11 | C |
| dekube-manager | `_install` | 11 | C |

No D/E/F rated functions. The v2.3.2 refactor reduced dekube-engine's worst CC from 18 to 16; v3.0 moved several C-rated functions out of the core into distribution extensions (`SimpleWorkloadProvider`, `_write_caddy_host_block`, `HAProxyRewriter`). The remaining C-rated functions are natural dispatchers (if/elif chains, type switches) or sequential steps — splitting them would move complexity without improving readability.

### Average complexity & maintainability

| Repo | MI | MI rating | Avg CC | CC rating |
|------|---:|-----------|-------:|-----------|
| dekube-engine | 74.14 | A | 5.2 | B |
| dekube-indexer-configmap | 70.29 | A | 4.5 | A |
| dekube-indexer-secret | 70.29 | A | 4.5 | A |
| dekube-indexer-pvc | 67.76 | A | 9.7 | B |
| dekube-indexer-service | 61.73 | A | 6.5 | B |
| dekube-transform-fix-permissions | 60.33 | A | 8.0 | B |
| dekube-provider-caddy | 51.64 | A | 7.6 | B |
| dekube-rewriter-haproxy | 41.75 | A | 7.2 | B |
| dekube-provider-simple-workload | 39.31 | A | 7.6 | B |
| dekube-manager | 31.51 | A | 4.8 | A |

All repos and bundled extensions are MI A-rated. dekube-engine's MI improved from 68.38 to 74.14 after the v3.0 split — moving `workloads.py` and `haproxy.py` to the distribution left the core with smaller, more focused modules. The lowest individual core module (`extensions.py`, 37.56) still rates A.

## The uncomfortable truth

This project is, by every objective metric, well-structured code. The functions are short. The concerns are separated. The naming is consistent. The linters approve.

This is deeply unsettling.

The expectation — the *hope*, even — was that a tool this conceptually wrong would at least have the decency to be poorly written. That the code quality would serve as a warning label: "abandon all hope, ye who read this." Instead, the temple stands. The prayers work. The linters nod approvingly. And somewhere, a developer who understands what this project actually *does* stares at a pylint score of 9.55 and feels nothing but existential dread.

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

## Code gotchas

### Null-safe YAML access

Python's `dict.get("key", default)` returns `default` only when the key is **absent**. When the key exists with an explicit `None` value (common in Helm charts with conditional `{{ if }}` blocks), `.get()` returns `None`, not the default.

```python
# WRONG — returns None when key exists with null value
annotations = manifest.get("metadata", {}).get("annotations", {})

# RIGHT — coalesces None to empty dict
annotations = manifest.get("metadata", {}).get("annotations") or {}
```

This applies to any field that Helm may render as `null`: `annotations`, `ports`, `initContainers`, `securityContext`, `data`, `stringData`, `rules`, `selector`. Use `or {}` / `or []` for any `.get()` on a YAML field that could be explicitly null. v2.3.1 fixed 30+ instances of this across dekube-engine, nginx, and traefik.

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

# Code quality

This should not score well. A project born from desecration, vibe-coded across multiple sessions of questionable mental health, whose primary architectural document is a fake Lovecraftian grimoire — this project has no business passing a linter. And yet.

> *The inquisitors arrived at dawn, instruments of measurement in hand, expecting to find the temple in ruin. Instead they found the walls straight, the columns load-bearing, and the altar — though undeniably profane — structurally sound. They left in silence, more disturbed than when they came.*
>
> — *De Vermis Mysteriis, On the Inadequacy of Metrics (unfortunately)*

## Tools

All repos use the same toolchain:

- **[pylint](https://pylint.readthedocs.io/)** — static analysis. Style warnings (too-many-locals, line-too-long, too-many-arguments) are accepted. Real issues (unused imports, actual bugs, f-strings without placeholders) are not.
- **[pyflakes](https://github.com/PyCQA/pyflakes)** — fast, zero-config, no false positives. Must be clean.
- **[radon](https://radon.readthedocs.io/)** — cyclomatic complexity. Target: no function rated D or worse. C-rated functions are tolerated when they're natural dispatchers or sequential logic that wouldn't benefit from splitting. Current worst: `_build_service_port_map` (16) and `NginxRewriter.rewrite` (16), both C.

## Current scores

*Last updated: 2026-02-21 — the monolith shattered, and each shard scored better alone.*

### Pylint

| Repo | Score | Notes |
|------|-------|-------|
| h2c-rewriter-haproxy | 10.00/10 | Built-in distribution extension |
| h2c-manager | 9.91/10 | Style only (too-many-locals) |
| h2c-provider-caddy | 9.87/10 | Built-in distribution extension |
| h2c-transform-flatten-internal-urls | 9.86/10 | Style only |
| h2c-provider-keycloak | 9.75/10 | Style + expected `import-error` (h2c not in path) |
| h2c-transform-bitnami | 9.68/10 | Style only |
| h2c-converter-workload | 9.64/10 | Built-in distribution extension |
| h2c-core | 9.59/10 | Style only (too-many-args, too-many-locals, duplicate-code) |
| h2c-indexer-pvc | 9.50/10 | Built-in distribution extension |
| h2c-converter-cert-manager | 9.45/10 | Style + expected `import-error` |
| h2c-provider-servicemonitor | 9.36/10 | Style + expected `import-error` |
| h2c-indexer-service | 9.29/10 | Built-in distribution extension |
| h2c-converter-trust-manager | 9.19/10 | Style + expected `import-error` |
| h2c-indexer-configmap | 9.09/10 | Built-in distribution extension |
| h2c-indexer-secret | 9.09/10 | Built-in distribution extension |
| h2c-rewriter-nginx | 8.50/10 | Style + expected `import-error` (h2c not in path) |
| h2c-rewriter-traefik | 7.33/10 | Style + expected `import-error` |

The `E0401: Unable to import 'h2c'` on extensions is expected — they import from h2c-core at runtime, not at lint time. The `R0903: Too few public methods` on converter classes is by design — the contract is one class, one method, that's it.

### Pyflakes

Zero warnings across all repos. We don't know how. We don't ask.

### Radon (cyclomatic complexity)

| Repo | Worst function | CC | Rating |
|------|---------------|---:|--------|
| h2c-core | `_build_service_port_map` | 16 | C |
| h2c-rewriter-nginx | `NginxRewriter.rewrite` | 16 | C |
| h2c-manager | `main` | 15 | C |
| h2c-provider-caddy | `_write_caddy_host_block` | 13 | C |
| h2c-converter-workload | `WorkloadConverter._build_service` | 13 | C |
| h2c-indexer-pvc | `PVCIndexer.convert` | 13 | C |
| h2c-core | `_auto_register` | 13 | C |
| h2c-manager | `_read_yaml_config` | 13 | C |
| h2c-provider-servicemonitor | `ServiceMonitorProvider._resolve_port` | 13 | C |
| h2c-core | `main` (cli) | 12 | C |
| h2c-rewriter-traefik | `TraefikRewriter.rewrite` | 12 | C |
| h2c-provider-keycloak | `_build_pod_template_volumes` | 12 | C |
| h2c-converter-trust-manager | `_collect_source` | 12 | C |
| h2c-provider-servicemonitor | `_process_servicemonitors` | 12 | C |
| h2c-core | `_resolve_env_entry` | 11 | C |
| h2c-converter-workload | `_get_exposed_ports` | 11 | C |
| h2c-converter-workload | `WorkloadConverter._convert_one` | 11 | C |
| h2c-core | `_resolve_secret_keys` | 11 | C |
| h2c-core | `_convert_volume_mounts` | 11 | C |
| h2c-provider-keycloak | `_build_options_env` | 11 | C |
| h2c-rewriter-nginx | `NginxRewriter` (class) | 11 | C |

`h2c-converter-workload`, `h2c-provider-caddy`, `h2c-indexer-pvc` are the future repo names for extensions currently bundled in `helmfile2compose/extensions/` as `workloads.py`, `caddy.py`, and `pvc_indexer.py` respectively. No D/E/F rated functions. The v2.3.2 refactor reduced h2c-core's worst CC from 18 to 16; v3.0 moved several C-rated functions out of the core into distribution extensions (`WorkloadConverter`, `_write_caddy_host_block`, `HAProxyRewriter`). The remaining C-rated functions are natural dispatchers (if/elif chains, type switches) or sequential steps — splitting them would move complexity without improving readability.

### Average complexity & maintainability

| Repo | MI | MI rating | Avg CC | CC rating |
|------|---:|-----------|-------:|-----------|
| h2c-rewriter-traefik | 78.60 | A | 8.3 | B |
| h2c-core | 74.07 | A | 5.2 | B |
| h2c-indexer-pvc | 73.65 | A | 12.5 | C |
| h2c-indexer-configmap | 70.29 | A | 4.5 | A |
| h2c-indexer-secret | 70.29 | A | 4.5 | A |
| h2c-converter-trust-manager | 64.45 | A | 7.8 | B |
| h2c-transform-flatten-internal-urls | 63.89 | A | 3.7 | A |
| h2c-transform-bitnami | 62.14 | A | 4.1 | A |
| h2c-indexer-service | 61.15 | A | 6.5 | B |
| h2c-rewriter-nginx | 56.75 | A | 8.8 | B |
| h2c-provider-caddy | 50.30 | A | 8.2 | B |
| h2c-converter-cert-manager | 47.61 | A | 4.0 | A |
| h2c-rewriter-haproxy | 43.51 | A | 6.4 | B |
| h2c-provider-servicemonitor | 40.78 | A | 5.3 | B |
| h2c-manager | 35.84 | A | 4.5 | A |
| h2c-converter-workload | 34.46 | A | 7.4 | B |
| h2c-provider-keycloak | 32.23 | A | 4.6 | A |

All repos and built-in extensions are MI A-rated. h2c-core's MI improved from 68.38 to 74.07 after the v3.0 split — moving `workloads.py` and `haproxy.py` to the distribution left the core with smaller, more focused modules. The lowest individual core module (`extensions.py`, 37.75) still rates A. Built-in distribution extensions use their future repo names (e.g. `h2c-converter-workload` for `workloads.py`, `h2c-provider-caddy` for `caddy.py`) — these will become the actual repo names when the distribution is externalized in v3.1.

## The uncomfortable truth

This project is, by every objective metric, well-structured code. The functions are short. The concerns are separated. The naming is consistent. The linters approve.

This is deeply unsettling.

The expectation — the *hope*, even — was that a tool this conceptually wrong would at least have the decency to be poorly written. That the code quality would serve as a warning label: "abandon all hope, ye who read this." Instead, the temple stands. The prayers work. The linters nod approvingly. And somewhere, a developer who understands what this project actually *does* stares at a pylint score of 9.68 and feels nothing but existential dread.

The code is clean. The architecture is sound. The concept remains an abomination. These things are not in conflict — they are in conspiracy.

> *The final horror is not that the ritual failed — it is that it succeeded, was reproducible, and scored well on peer review.*
>
> — *Cultes des Goules, On Accidental Rigor (regrettably)*

## And then it got refactored

It wasn't enough for it to work. It wasn't enough for it to lint clean. It had to be *maintainable*.

v2.3.2 split the 1858-line monolith into 21 modules across three layers: `pacts/` (the sacred contracts), `core/` (the engine), `io/` (the plumbing). A build script concatenates them back into a single file for distribution. Users see no change. The metrics see everything.

The split alone fixed MI — from 0.00 (C) to 68.38 (A). Then a cyclomatic complexity pass extracted helpers from every function rated CC 14+. The worst offender dropped from 18 to 16. Average CC went from 6.6 to 5.9. All 21 modules score MI A individually.

A project whose existence is an architectural crime now has an architecture page. With a dependency graph. And layer boundaries. And a section called "[the sacred contracts](h2c-core.md#pacts--the-sacred-contracts)."

> *They said: let us impose order upon the chaos, that future disciples may navigate the labyrinth without losing their minds. And so the tablet was broken into twenty-one fragments, each labeled and indexed. The labyrinth remained — but now it had signage.*
>
> — *Book of Eibon, On the Cataloguing of Madness (we tried)*

## And then it got tests

IT HAS TESTS. IT HAS AN EXECUTIONER.

A [regression executioner](testing.md) that downloads two versions of h2c-core, runs every extension combo against a battery of edge-case manifests, and diffs the output. A [torturer](testing.md#the-torturer) that produces O(n³) manifests. It cleans up after itself. It passes shellcheck.

A project that emulates a container orchestrator by flattening its output into a different container orchestrator, whose fake kube-apiserver lives on a personal GitHub account for plausible deniability — this project has automated regression testing. With CI. Weekly runs. Artifact uploads.

The linters approved. The complexity metrics nodded. And now the executioner confirms: the rituals produce identical output every time. The temple is no longer merely structurally sound — it is *under continuous inspection*.

## And then it divided by zero

v3.0.0 split the repo in two. h2c-core (the bare engine) and helmfile2compose (the distribution). Two repos. Two build scripts. Two release workflows. Two READMEs. A `sys.modules` injection hack so runtime-loaded extensions can import from a namespace that technically doesn't exist in the concatenated file. An `_auto_register()` that scans globals for classes, deduplicates kind claims, and populates registries that were deliberately left empty.

The output is identical to v2.3.1.

Not "similar." Not "equivalent." Identical. The executioner diffs every byte across every extension combo and confirms: the split changed nothing about what the tool produces. It changed *everything* about how it's organized. h2c-core's MI went from 68.38 to 74.07 — not because any function improved, but because `workloads.py` and `haproxy.py` left. The core got lighter by becoming less. The distribution got heavier by becoming more. The sum is greater than the whole was.

Two repos. Fourteen files. Zero functional change. The metrics improved. The architecture is now a dependency graph with layer boundaries and a section called "the sacred contracts." There is a page called [Distributions](distributions.md) that explains the Kubernetes distribution model as applied to a tool whose sole purpose is to flee Kubernetes.

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

This applies to any field that Helm may render as `null`: `annotations`, `ports`, `initContainers`, `securityContext`, `data`, `stringData`, `rules`, `selector`. Use `or {}` / `or []` for any `.get()` on a YAML field that could be explicitly null. v2.3.1 fixed 30+ instances of this across h2c-core, nginx, and traefik.

## Running locally

```bash
# Extensions (single-file repos)
pylint <script>.py
pyflakes <script>.py
radon cc <script>.py -a -s -n C

# h2c-core (package)
cd h2c-core
PYTHONPATH=src pylint src/h2c/
PYTHONPATH=src pyflakes src/h2c/
PYTHONPATH=src radon cc src/h2c/ -a -s -n C   # CC hotspots
PYTHONPATH=src radon mi src/h2c/ -s            # maintainability index
```

For h2c-core specifically, the CLAUDE.md in the repo root documents the complexity targets and lint workflow.

**Note:** For distribution extensions (bundled in the distribution's `extensions/` directory), run radon per-extension file — not on the concatenated distribution artifact.

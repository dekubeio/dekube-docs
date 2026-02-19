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

*Last updated: 2026-02-19 — the monolith shattered, and each shard scored better alone.*

### Pylint

| Repo | Score | Notes |
|------|-------|-------|
| h2c-manager | 9.90/10 | Style only (too-many-locals) |
| h2c-transform-flatten-internal-urls | 9.85/10 | Style only |
| h2c-provider-keycloak | 9.75/10 | Style + expected `import-error` (helmfile2compose not in path) |
| h2c-core | 9.68/10 | Style only (too-many-args, too-many-locals) |
| h2c-transform-bitnami | 9.68/10 | Style only |
| h2c-converter-cert-manager | 9.45/10 | Style + expected `import-error` |
| h2c-provider-servicemonitor | 9.35/10 | Style + expected `import-error` |
| h2c-converter-trust-manager | 9.18/10 | Style + expected `import-error` |
| h2c-rewriter-nginx | 8.50/10 | Style + expected `import-error` (helmfile2compose not in path) |
| h2c-rewriter-traefik | 7.33/10 | Style + expected `import-error` |

The `E0401: Unable to import 'helmfile2compose'` on extensions is expected — they import from h2c-core at runtime, not at lint time. The `R0903: Too few public methods` on converter classes is by design — the contract is one class, one method, that's it.

### Pyflakes

Zero warnings across all repos. We don't know how. We don't ask.

### Radon (cyclomatic complexity)

| Repo | Worst function | CC | Rating |
|------|---------------|---:|--------|
| h2c-core | `_build_service_port_map` | 16 | C |
| h2c-rewriter-nginx | `NginxRewriter.rewrite` | 16 | C |
| h2c-manager | `main` | 15 | C |
| h2c-core | `_write_caddy_host_block` | 13 | C |
| h2c-core | `WorkloadConverter._build_service` | 13 | C |
| h2c-manager | `_read_yaml_config` | 13 | C |
| h2c-provider-servicemonitor | `_resolve_port` | 13 | C |
| h2c-rewriter-traefik | `TraefikRewriter.rewrite` | 12 | C |
| h2c-provider-keycloak | `_build_pod_template_volumes` | 12 | C |
| h2c-converter-trust-manager | `_collect_source` | 12 | C |
| h2c-provider-servicemonitor | `_process_servicemonitors` | 12 | C |
| h2c-core | `_resolve_env_entry` | 11 | C |
| h2c-core | `_get_exposed_ports` | 11 | C |
| h2c-core | `WorkloadConverter._convert_one` | 11 | C |
| h2c-core | `_resolve_secret_keys` | 11 | C |
| h2c-core | `_convert_volume_mounts` | 11 | C |
| h2c-provider-keycloak | `_build_options_env` | 11 | C |
| h2c-rewriter-nginx | `NginxRewriter` (class) | 11 | C |

No D/E/F rated functions. The v2.3.2 refactor reduced h2c-core's worst CC from 18 to 16 by extracting helpers from `convert()`, `_register_extensions()`, `_load_extensions()`, `HAProxyRewriter.rewrite()`, `_infer_namespaces()`, and `_build_service_port_map()`. The remaining C-rated functions are natural dispatchers (if/elif chains, type switches) or sequential steps — splitting them would move complexity without improving readability.

### Average complexity & maintainability

| Repo | MI | MI rating | Avg CC | CC rating |
|------|---:|-----------|-------:|-----------|
| h2c-rewriter-traefik | 78.60 | A | 8.3 | B |
| h2c-core | 68.38 | A | 5.9 | B |
| h2c-converter-trust-manager | 64.66 | A | 7.8 | B |
| h2c-transform-flatten-internal-urls | 64.11 | A | 3.7 | A |
| h2c-transform-bitnami | 62.31 | A | 4.1 | A |
| h2c-rewriter-nginx | 56.75 | A | 8.8 | B |
| h2c-converter-cert-manager | 47.70 | A | 4.0 | A |
| h2c-provider-servicemonitor | 40.86 | A | 5.3 | B |
| h2c-manager | 35.84 | A | 4.5 | A |
| h2c-provider-keycloak | 32.28 | A | 4.6 | A |

All repos are MI A-rated. h2c-core went from MI 0.00 (C) to 68.38 (A) after the package split — radon penalizes file size heavily, so the 1858-line monolith bottomed out regardless of internal structure. Splitting into 21 focused modules let each one score on its own merit. The lowest individual module (workloads.py, 34.58) still rates A.

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

A project whose existence is an architectural crime now has an architecture page. With a dependency graph. And layer boundaries. And a section called "[the sacred contracts](core-architecture.md#pacts--the-sacred-contracts)."

> *They said: let us impose order upon the chaos, that future disciples may navigate the labyrinth without losing their minds. And so the tablet was broken into twenty-one fragments, each labeled and indexed. The labyrinth remained — but now it had signage.*
>
> — *Book of Eibon, On the Cataloguing of Madness (we tried)*

## And then it got tests

IT HAS TESTS. IT HAS AN EXECUTIONER.

A [regression executioner](testing.md) that downloads two versions of h2c-core, runs every extension combo against a battery of edge-case manifests, and diffs the output. A [torturer](testing.md#the-torturer) that produces O(n³) manifests. It cleans up after itself. It passes shellcheck.

A project that emulates a container orchestrator by flattening its output into a different container orchestrator, whose fake kube-apiserver lives on a personal GitHub account for plausible deniability — this project has automated regression testing. With CI. Weekly runs. Artifact uploads.

The linters approved. The complexity metrics nodded. And now the executioner confirms: the rituals produce identical output every time. The temple is no longer merely structurally sound — it is *under continuous inspection*.

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
PYTHONPATH=src pylint src/helmfile2compose/
PYTHONPATH=src pyflakes src/helmfile2compose/
PYTHONPATH=src radon cc src/helmfile2compose/ -a -s -n C   # CC hotspots
PYTHONPATH=src radon mi src/helmfile2compose/ -s            # maintainability index
```

For h2c-core specifically, the CLAUDE.md in the repo root documents the complexity targets and lint workflow.

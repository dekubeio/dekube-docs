# Code quality

This should not score well. A project born from desecration, vibe-coded across multiple sessions of questionable mental health, whose primary architectural document is a fake Lovecraftian grimoire — this project has no business passing a linter. And yet.

> *The inquisitors arrived at dawn, instruments of measurement in hand, expecting to find the temple in ruin. Instead they found the walls straight, the columns load-bearing, and the altar — though undeniably profane — structurally sound. They left in silence, more disturbed than when they came.*
>
> — *De Vermis Mysteriis, On the Inadequacy of Metrics (so I'm told)*

## Tools

All repos use the same toolchain:

- **[pylint](https://pylint.readthedocs.io/)** — static analysis. Style warnings (too-many-locals, line-too-long, too-many-arguments) are accepted. Real issues (unused imports, actual bugs, f-strings without placeholders) are not.
- **[pyflakes](https://github.com/PyCQA/pyflakes)** — fast, zero-config, no false positives. Must be clean.
- **[radon](https://radon.readthedocs.io/)** — cyclomatic complexity. Target: no function rated D or worse. C-rated functions are tolerated. Current worst offender: `main` in h2c-core at 18 (C).

## Current scores

*Last updated: 2026-02-17 — the day we released network aliases, accepted that the names would never die, then refactored the survivors.*

### Pylint

| Repo | Score | Notes |
|------|-------|-------|
| h2c-core | 9.71/10 | Style only (too-many-args on converter internals) |
| h2c-manager | 10.00/10 | — |
| h2c-operator-keycloak | 9.75/10 | Style + expected `import-error` (helmfile2compose not in path) |
| h2c-operator-cert-manager | 9.45/10 | Style + expected `import-error` |
| h2c-operator-servicemonitor | 9.35/10 | Style + expected `import-error` |
| h2c-operator-trust-manager | 9.18/10 | Style + expected `import-error` |

The `E0401: Unable to import 'helmfile2compose'` on operators is expected — they import from h2c-core at runtime, not at lint time. The `R0903: Too few public methods` on converter classes is by design — the contract is one class, one method, that's it.

### Pyflakes

Zero warnings across all repos. We don't know how. We don't ask.

### Radon (cyclomatic complexity)

| Repo | Worst function | CC | Rating |
|------|---------------|---:|--------|
| h2c-core | `main` | 18 | C |
| h2c-core | `_convert_one_ingress` | 17 | C |
| h2c-core | `convert` | 17 | C |
| h2c-core | `_infer_namespaces` | 14 | C |
| h2c-core | `_build_service_port_map` | 13 | C |
| h2c-manager | `_read_yaml_config` | 13 | C |
| h2c-operator-servicemonitor | `_resolve_port` | 13 | C |
| h2c-operator-keycloak | `_build_pod_template_volumes` | 12 | C |
| h2c-operator-trust-manager | `_collect_source` | 12 | C |
| h2c-operator-servicemonitor | `_process_servicemonitors` | 12 | C |
| h2c-core | `_write_caddy_host_block` | 11 | C |
| h2c-core | `WorkloadConverter._build_service` | 11 | C |
| h2c-operator-keycloak | `_build_options_env` | 11 | C |

No D/E/F rated functions.

### Average complexity & maintainability

| Repo | Avg CC | CC rating | MI | MI rating |
|------|-------:|-----------|---:|-----------|
| h2c-operator-trust-manager | 7.8 | B | 64.66 | A |
| h2c-core | 6.6 | B | 0.00 | C |
| h2c-operator-servicemonitor | 5.3 | B | 40.86 | A |
| h2c-operator-keycloak | 4.6 | A | 32.28 | A |
| h2c-manager | 3.95 | A | 43.55 | A |
| h2c-operator-cert-manager | 4.0 | A | 47.70 | A |

h2c-core MI is 0.00. MI is the only metric that sees through the cloud of desecration — alas, it is for the wrong reasons. Radon penalizes file size and volume of code heavily, so a 1700-line single-file converter with 50+ functions will bottom out regardless of internal structure. It condemned the temple not for the rituals performed within, but for the square footage.

## The uncomfortable truth

This project is, by every objective metric, well-structured code. The functions are short. The concerns are separated. The naming is consistent. The linters approve.

This is deeply unsettling.

The expectation — the *hope*, even — was that a tool this conceptually wrong would at least have the decency to be poorly written. That the code quality would serve as a warning label: "abandon all hope, ye who read this." Instead, the temple stands. The prayers work. The linters nod approvingly. And somewhere, a developer who understands what this project actually *does* stares at a pylint score of 9.70 and feels nothing but existential dread.

The code is clean. The architecture is sound. The concept remains an abomination. These things are not in conflict — they are in conspiracy.

> *The final horror is not that the ritual failed — it is that it succeeded, was reproducible, and scored well on peer review.*
>
> — *Cultes des Goules, On Accidental Rigor (regrettably)*

## Running locally

```bash
# From any repo root
pylint <script>.py
pyflakes <script>.py
radon cc <script>.py -a -s -n C
```

For h2c-core specifically, the CLAUDE.md in the repo root documents the complexity targets and lint workflow.

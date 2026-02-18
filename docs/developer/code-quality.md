# Code quality

This should not score well. A project born from desecration, vibe-coded across multiple sessions of questionable mental health, whose primary architectural document is a fake Lovecraftian grimoire — this project has no business passing a linter. And yet.

> *The inquisitors arrived at dawn, instruments of measurement in hand, expecting to find the temple in ruin. Instead they found the walls straight, the columns load-bearing, and the altar — though undeniably profane — structurally sound. They left in silence, more disturbed than when they came.*
>
> — *De Vermis Mysteriis, On the Inadequacy of Metrics (unfortunately)*

## Tools

All repos use the same toolchain:

- **[pylint](https://pylint.readthedocs.io/)** — static analysis. Style warnings (too-many-locals, line-too-long, too-many-arguments) are accepted. Real issues (unused imports, actual bugs, f-strings without placeholders) are not.
- **[pyflakes](https://github.com/PyCQA/pyflakes)** — fast, zero-config, no false positives. Must be clean.
- **[radon](https://radon.readthedocs.io/)** — cyclomatic complexity. Target: no function rated D or worse. C-rated functions are tolerated. Current worst offenders: `convert` (18), `_register_extensions` (17), and `_load_extensions` (16) in h2c-core, all C.

## Current scores

*Last updated: 2026-02-18 — the day we taught the gateway to delegate, and the annotations learned new dialects.*

### Pylint

| Repo | Score | Notes |
|------|-------|-------|
| h2c-manager | 9.90/10 | Style only (too-many-locals) |
| h2c-transform-flatten-internal-urls | 9.85/10 | Style only |
| h2c-provider-keycloak | 9.75/10 | Style + expected `import-error` (helmfile2compose not in path) |
| h2c-rewriter-nginx | 8.50/10 | Style + expected `import-error` (helmfile2compose not in path) |
| h2c-rewriter-traefik | 7.33/10 | Style + expected `import-error` |
| h2c-core | 9.70/10 | Style only (too-many-args on converter internals) |
| h2c-converter-cert-manager | 9.45/10 | Style + expected `import-error` |
| h2c-provider-servicemonitor | 9.35/10 | Style + expected `import-error` |
| h2c-converter-trust-manager | 9.18/10 | Style + expected `import-error` |

The `E0401: Unable to import 'helmfile2compose'` on extensions is expected — they import from h2c-core at runtime, not at lint time. The `R0903: Too few public methods` on converter classes is by design — the contract is one class, one method, that's it.

### Pyflakes

Zero warnings across all repos. We don't know how. We don't ask.

### Radon (cyclomatic complexity)

| Repo | Worst function | CC | Rating |
|------|---------------|---:|--------|
| h2c-core | `convert` | 18 | C |
| h2c-core | `_register_extensions` | 17 | C |
| h2c-core | `_load_extensions` | 16 | C |
| h2c-manager | `main` | 14 | C |
| h2c-core | `_infer_namespaces` | 14 | C |
| h2c-core | `_build_service_port_map` | 13 | C |
| h2c-core | `_write_caddy_host_block` | 13 | C |
| h2c-manager | `_read_yaml_config` | 13 | C |
| h2c-rewriter-nginx | `NginxRewriter.rewrite` | 13 | C |
| h2c-provider-servicemonitor | `_resolve_port` | 13 | C |
| h2c-provider-keycloak | `_build_pod_template_volumes` | 12 | C |
| h2c-converter-trust-manager | `_collect_source` | 12 | C |
| h2c-provider-servicemonitor | `_process_servicemonitors` | 12 | C |
| h2c-core | `WorkloadConverter._build_service` | 11 | C |
| h2c-provider-keycloak | `_build_options_env` | 11 | C |

No D/E/F rated functions.

### Average complexity & maintainability

| Repo | Avg CC | CC rating | MI | MI rating |
|------|-------:|-----------|---:|-----------|
| h2c-converter-trust-manager | 7.8 | B | 64.66 | A |
| h2c-rewriter-nginx | 7.3 | B | 58.54 | A |
| h2c-rewriter-traefik | 6.3 | B | 81.08 | A |
| h2c-core | 6.1 | B | 0.00 | C |
| h2c-provider-servicemonitor | 5.3 | B | 40.86 | A |
| h2c-manager | 4.6 | A | 36.29 | A |
| h2c-provider-keycloak | 4.6 | A | 32.28 | A |
| h2c-converter-cert-manager | 4.0 | A | 47.70 | A |
| h2c-transform-flatten-internal-urls | 3.6 | A | 65.52 | A |

h2c-core MI is 0.00. MI is the only metric that sees through the cloud of desecration — alas, it is for the wrong reasons. Radon penalizes file size and volume of code heavily, so an 1800-line single-file converter with 50+ functions will bottom out regardless of internal structure. It condemned the temple not for the rituals performed within, but for the square footage.

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

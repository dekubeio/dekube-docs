# dekube-engine — the bare engine

> *The disciple gazed upon the monolith and saw that it was vast — a single tablet bearing every law, every rite, every contradiction. And he said: let us shatter the tablet, that each fragment may be understood alone. But when the pieces lay scattered, each fragment still whispered the name of the whole.*
>
> — *Necronomicon, On the Shattering of Tablets (partial)*

## What it is

dekube-engine is the conversion engine — the pipeline, the extension loader, the CLI, and nothing else. No built-in converters. No built-in rewriters. All registries empty. Feed it manifests and it will parse them, warn that every kind is unknown, and produce nothing. A temple with no priests.

It exists so that [distributions](distributions.md) can bundle different sets of extensions on top of the same engine. The [helmfile2compose](https://github.com/dekubeio/helmfile2compose) distribution is the default — dekube-engine + 8 bundled extensions, concatenated into a single `helmfile2compose.py`. But dekube-engine can also be used standalone with `--extensions-dir`, or as the foundation for custom distributions.

**Users never interact with dekube-engine directly.** They use a distribution. dekube-engine is for extension developers, distribution builders, and people who want to understand how the engine works.

## Package structure

```
dekube-engine/
├── src/dekube/
│   ├── __init__.py          # re-exports pacts API
│   ├── __main__.py          # python -m dekube entry point
│   ├── cli.py               # argument parsing, orchestration
│   ├── pacts/               # public contracts for extensions
│   │   ├── types.py         # ConvertContext, ConverterResult, ProviderResult, Converter, Provider
│   │   ├── ingress.py       # IngressRewriter, get_ingress_class(), resolve_backend()
│   │   └── helpers.py       # apply_replacements(), _secret_value()
│   ├── core/                # internal conversion engine
│   │   ├── constants.py     # regexes, kind lists, well-known ports
│   │   ├── env.py           # resolve_env(), port remapping, command conversion
│   │   ├── volumes.py       # volume mount conversion (PVC, ConfigMap, Secret)
│   │   ├── services.py      # K8s Service indexing, alias maps, port maps
│   │   ├── ingress.py       # IngressProvider, rewriter dispatch, _NullRewriter
│   │   ├── extensions.py    # extension discovery and loading
│   │   └── convert.py       # convert() orchestration, overrides, _auto_register()
│   └── io/                  # input/output
│       ├── parsing.py       # helmfile template, YAML loading, namespace inference
│       ├── config.py        # dekube.yaml load/save
│       └── output.py        # compose.yml, warnings
├── build.py                 # concat → single-file h2c.py (bare engine)
├── build-distribution.py    # concat core + extensions → distribution
└── .github/workflows/
    └── release.yml          # CI: build + release assets on tag push
```

## The three layers

### `pacts/` — the sacred contracts {#pacts--the-sacred-contracts}

Everything extensions can import. This is the stable API:

```python
from dekube import ConvertContext, ConverterResult, ProviderResult
from dekube import ConvertResult  # deprecated alias for ProviderResult
from dekube import Converter, IndexerConverter, Provider
from dekube import IngressRewriter, get_ingress_class, resolve_backend
from dekube import apply_replacements, resolve_env, _secret_value
```

Or explicitly:

```python
from dekube.pacts import ConvertContext, IngressRewriter
from dekube.pacts.types import Provider
```

Both paths work — `__init__.py` re-exports the pacts API. These are the only imports extensions should use. If it's not in `pacts/`, it's internal and may change.

### `core/` — the conversion engine

Internal modules that do the actual work. Not part of the public API. The dependency graph is acyclic:

```
constants.py      ← standalone
env.py            ← pacts/helpers, constants
volumes.py        ← pacts/helpers, env
services.py       ← constants
ingress.py        ← pacts, IngressProvider base class
extensions.py     ← pacts/ingress, ingress
convert.py        ← pacts, all core modules, _auto_register()
```

Module-level mutable state lives in `core/convert.py` (`_CONVERTERS`, `_TRANSFORMS`, `CONVERTED_KINDS`) and `core/ingress.py` (`_REWRITERS`). In the distribution model, `_auto_register()` populates these registries from all classes found in the concatenated script's globals.

The conversion pipeline runs in a fixed order: converters → transforms → overrides. Overrides from `dekube.yaml` apply *after* transforms, so user config always wins over transform-produced output. This means a transform can create or modify a service, and the user can still override any field on it. Extension developers should not assume their transform output is final — it's not. This ordering was introduced in engine v1.3.0.

### `io/` — input/output

Parsing, config, and output writing. No conversion logic — just plumbing between the filesystem and the core.

#### Namespace inference

Helm charts don't always set `metadata.namespace` on every resource — many rely on Helm's `--namespace` flag at install time. When dekube parses rendered manifests, some resources end up without a namespace. This matters because compose network aliases use the namespace to build FQDN variants (`svc.ns.svc.cluster.local`). No namespace → no FQDNs → cross-service references by FQDN silently break.

`parsing.py` solves this with a three-phase inference strategy, where each phase fills gaps left by the previous:

1. **Sibling inference** — every manifest is tagged with its release directory (`_release_dir`, the first path component under the rendered output). If *any* manifest in the same release dir has a namespace, all siblings inherit it. This works because a single Helm release always targets one namespace.

2. **Release/namespace matching** — the release name is extracted from the directory name (format: `helmfile.yaml-<hash>-<release-name>`). If that name matches a known namespace (seen in a `Namespace` resource or another manifest's `metadata.namespace`), it's assigned. Covers the common case where the release and namespace share a name.

3. **`helmfile list` fallback** — when using `--helmfile-dir`, the engine runs `helmfile list --output json` before rendering. This gives a definitive release → namespace mapping straight from the helmfile. Only available in this mode — `--from-dir` skips it since there's no helmfile to query.

When inference fails entirely for a service, the engine emits a warning: *"service 'X' has no FQDN aliases (namespace unknown)"*. The fix is either adding `metadata.namespace` in the chart templates, or using `--helmfile-dir` so phase 3 can kick in.

To disable namespace inference entirely, set `infer_namespaces: false` in `dekube.yaml` (see [config reference](../reference/config.md#full-schema)).

I'd have liked to KISS this and apply GIGO — the engine is supposed to be transparent, dispatch manifests, and not have opinions. But even MinIO's main chart fails to set `metadata.namespace` on all its resources, and it's far from alone. This is probably the most "magical" and opinionated mechanism in a layer that was designed to do neither. The opt-out exists for when you'd rather fix your charts than have the engine guess.

Conceptually, this could have been an indexer — a ninth monk that populates context before the others run. It wasn't extracted because no distribution has a reason to remove it, no alternative implementation makes sense, and the mechanism is feature-complete. Adding indirection for a component that will never be swapped is the opposite of KISS.

## Build artifacts

Two build scripts produce two different outputs:

- **`build.py`** — concatenates `src/dekube/` into a single `dekube.py`. Bare engine, no extensions. This is the dekube-engine release artifact.
- **`build-distribution.py`** — builds a distribution from core + extensions dir. Used by distribution repos. Fetched directly from the repo (`main` branch) — not a release asset.

```bash
# Bare engine
python build.py
# → dekube.py (~1265 lines, not committed)

# Distribution (from a distribution repo)
python build-distribution.py helmfile2compose --extensions-dir extensions --core-dir ../dekube-engine
# → helmfile2compose.py
```

See [Build system](build-system.md) for the full deep dive on concatenation, import stripping, and the `sys.modules` hack.

## Dev workflow

```bash
# Run from the package (development)
PYTHONPATH=src python -m dekube --from-dir /tmp/rendered --output-dir /tmp/out

# Build and smoke test
python build.py
python dekube.py --from-dir /tmp/rendered --output-dir /tmp/out
```

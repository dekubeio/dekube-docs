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
│       ├── config.py        # helmfile2compose.yaml load/save
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

### `io/` — input/output

Parsing, config, and output writing. No conversion logic — just plumbing between the filesystem and the core.

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

# CLI reference

> *The ritual required precise incantation — not in the words themselves, which were few, but in the silences between them. Each flag unfurled was a choice; each flag omitted, a trust placed in the default.*
>
> — *Book of Eibon, On the Grammar of Summons (so they say)*

## Usage

```bash
# With a distribution (most common)
python3 helmfile2compose.py [flags]

# With the bare engine
python3 dekube.py [flags]
```

All flags are optional. Without `--from-dir`, the engine runs `helmfile template` automatically.

## Flags

| Flag | Type | Default | Description |
|------|------|---------|-------------|
| `--helmfile-dir` | `str` | `.` | Directory containing `helmfile.yaml`. Used as the working directory for `helmfile template`. |
| `-e`, `--environment` | `str` | *(none)* | Helmfile environment to use (e.g. `compose`, `local`). Passed to `helmfile template -e <env>`. |
| `--from-dir` | `str` | *(none)* | Skip `helmfile template` — read pre-rendered YAML from this directory instead. Use when you have already-rendered manifests (from `helm template`, `kustomize build`, or a previous `helmfile template` run). |
| `--output-dir` | `str` | `.` | Where to write `compose.yml`, reverse proxy config, `configmaps/`, `secrets/`, and `dekube.yaml` (on first run). |
| `--compose-file` | `str` | `compose.yml` | Name of the generated compose file. |
| `--extensions-dir` | `str` | *(none)* | Directory containing extension modules (converters, providers, transforms, rewriters). All `.py` files in this directory are scanned. See [Writing extensions](../extend/extensions/index.md). |

## Examples

### From a helmfile

```bash
# Render the "compose" environment and convert
python3 helmfile2compose.py --helmfile-dir ./infra -e compose --output-dir ./output

# Same, but the helmfile is in the current directory
python3 helmfile2compose.py -e compose
```

### From pre-rendered manifests

```bash
# You already ran helmfile template / helm template / kustomize build
python3 helmfile2compose.py --from-dir /tmp/rendered --output-dir ./output
```

### With extensions

```bash
# Load extensions from a directory
python3 helmfile2compose.py -e compose \
  --extensions-dir .dekube/extensions \
  --output-dir ./output
```

### Bare engine (no built-in converters)

```bash
# Only your extensions produce output — useful for testing a single extension in isolation
python3 dekube.py --from-dir /tmp/rendered \
  --extensions-dir ./my-extension --output-dir ./output
```

## First run vs subsequent runs

On **first run** (no `dekube.yaml` exists in `--output-dir`):

1. PVC claims are auto-registered in `dekube.yaml` with default host paths under `volume_root`
2. K8s-only workloads (cert-manager, reflector, ingress controllers…) are auto-added to `exclude`
3. `dekube.yaml` is written with the detected project name

On **subsequent runs** (`dekube.yaml` already exists):

- The config file is **read-only** — the engine never writes to it again
- New PVC claims not in `dekube.yaml` emit a warning (add them manually)
- Stale volume entries (config volumes not referenced by any PVC) emit a warning

See [Configuration reference](config.md) for the full `dekube.yaml` schema.

## Exit codes

| Code | Meaning |
|------|---------|
| `0` | Success — `compose.yml` and config files written |
| `1` | Fatal error — invalid config, extension conflict, helmfile failure, etc. |
| `2` | Empty output — no services generated. Not a crash, but nothing useful produced. Usually means all workloads were excluded or the input contained no convertible manifests. |

Useful for scripting (`generate-compose.sh`, CI pipelines):

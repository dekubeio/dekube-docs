# h2c-manager

h2c-manager is a lightweight package manager for helmfile2compose. It downloads the core script and CRD operator modules from GitHub releases. Python 3, stdlib only — no dependencies.

## Installation

Download from the h2c-manager repo (rolling release — always latest):

```bash
curl -fsSL https://raw.githubusercontent.com/helmfile2compose/h2c-manager/main/h2c-manager.py -o h2c-manager.py
```

## Usage

```bash
# Core only (no operators)
python3 h2c-manager.py

# Core + keycloak operator
python3 h2c-manager.py keycloak

# Core + multiple operators
python3 h2c-manager.py keycloak certmanager trust-manager

# Pin core version
python3 h2c-manager.py --core-version v2.0.0 keycloak

# Pin operator version
python3 h2c-manager.py keycloak==0.1.0

# Custom install directory (default: .h2c/)
python3 h2c-manager.py -d ./tools keycloak
```

## Output

h2c-manager creates the following directory structure:

```
.h2c/
├── helmfile2compose.py       # core script
└── extensions/
    ├── keycloak.py            # requested extension
    └── certmanager.py         # auto-resolved dependency
```

## Run mode

`run` is a shortcut to invoke helmfile2compose with sensible defaults:

```bash
python3 h2c-manager.py run -e compose
# equivalent to:
# python3 .h2c/helmfile2compose.py --helmfile-dir . --extensions-dir .h2c/extensions --output-dir . -e compose
```

Defaults: `--helmfile-dir .`, `--extensions-dir .h2c/extensions` (if it exists), `--output-dir .`. Any explicit flag overrides the default. All extra arguments are passed through to helmfile2compose.

## Version resolution

### Core

- No flag: latest GitHub release of `helmfile2compose/h2c-core`
- `--core-version v2.0.0`: exact tag

### Operators

- Bare name (`keycloak`): latest GitHub release
- Pinned (`keycloak==0.1.0`): exact version. The `v` prefix is added automatically if missing (`0.1.0` -> `v0.1.0`).

No version ranges. Just `latest` and `==exact`.

## Dependency resolution

Operators can declare dependencies in the registry. For example, `trust-manager` depends on `certmanager`:

```bash
python3 h2c-manager.py trust-manager
# Fetching h2c-core v2.0.0...
# Fetching extension certmanager v0.1.0 (dependency of trust-manager)...
# Fetching extension trust-manager v0.1.0...
```

Dependencies are resolved one level deep (no transitive chains). Duplicates are deduplicated.

## Python dependency checking

Some operators require Python packages (e.g. certmanager requires `cryptography`). h2c-manager checks if they're installed and prints an aggregated warning:

```
Missing Python dependencies for operators:
  certmanager: cryptography
Install with: pip install cryptography
```

This is a warning, not a fatal error — the operator file is still downloaded. Install the dependency before running helmfile2compose with that operator.

## Operator registry

The registry lives at `helmfile2compose/h2c-manager` as `extensions.json`. It maps extension names (operators, ingress rewriters, etc.) to GitHub repos:

```json
{
  "schema_version": 1,
  "extensions": {
    "keycloak": {
      "repo": "helmfile2compose/h2c-operator-keycloak",
      "description": "Keycloak and KeycloakRealmImport CRDs",
      "file": "keycloak.py",
      "depends": []
    }
  }
}
```

Available versions are whatever tags/releases exist on each repo. The registry doesn't list versions — GitHub is the source of truth.

### Adding an extension to the registry

Open a PR to `helmfile2compose/h2c-manager` adding your extension to `extensions.json`. Requirements:

- The operator repo must exist and have at least one GitHub Release
- The `file` field must point to the `.py` file in the repo root
- Optional: `requirements.txt` in the repo root (downloaded and checked by the manager)

## Declarative dependencies

If `helmfile2compose.yaml` exists, h2c-manager reads `core_version` and `depends` from it:

```yaml
# helmfile2compose.yaml
core_version: v2.0.0
depends:
  - keycloak
  - certmanager==0.1.0
  - trust-manager
```

```bash
python3 h2c-manager.py
# Core version from helmfile2compose.yaml: v2.0.0
# Fetching h2c-core v2.0.0...
# Reading extensions from helmfile2compose.yaml: keycloak, certmanager==0.1.0, trust-manager
# Fetching extension keycloak v0.1.0...
# ...
```

CLI flags override the yaml: `--core-version` overrides `core_version`, explicit extension args override `depends`. This keeps behavior predictable: either the yaml drives it, or you drive it.

h2c-manager also checks that `pyyaml` is installed (required by h2c-core) and warns alongside any extension-specific missing dependencies.

## Integration with generate-compose.sh

Projects that ship a `generate-compose.sh` wrapper can use h2c-manager as the bootstrap:

```bash
#!/bin/bash
set -euo pipefail

H2C_MANAGER_URL="https://raw.githubusercontent.com/helmfile2compose/h2c-manager/main/h2c-manager.py"
curl -fsSL "$H2C_MANAGER_URL" -o h2c-manager.py
python3 h2c-manager.py          # reads depends from helmfile2compose.yaml
python3 h2c-manager.py run -e compose
```

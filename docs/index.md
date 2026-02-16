# helmfile2compose

*For when you maintain a helmfile but people keep asking for a docker-compose. But it got REALLY out of hand*

Convert Kubernetes manifests to `compose.yml` + `Caddyfile`. Not Kubernetes-in-Docker (kind, k3d, minikube...) — no cluster, no kubelet, no shim. A devolution of the power of Kubernetes into the simplicity of compose: real `docker compose up`, real Caddy, plain stupid containers.

## How it works

The same Helm charts used for Kubernetes are rendered into standard K8s manifests, then converted to compose:

```
Helm charts (helmfile / helm / kustomize)
    ↓  helmfile template / helm template / kustomize build
K8s manifests (Deployments, Services, ConfigMaps, Secrets, Ingress...)
    ↓  helmfile2compose.py
compose.yml + Caddyfile + configmaps/ + secrets/
```

Despite the name, **helmfile is not required** — the core accepts any directory of K8s YAML files. Helmfile is just one way to produce them.

### The ecosystem

What started as a single script became an ecosystem of three components:

- **[h2c-core](https://github.com/helmfile2compose/h2c-core)** — *the mad scribe.* A single Python script (~1500 lines) that reads K8s manifests and writes compose. Handles Deployments, StatefulSets, Services, Ingress, ConfigMaps, Secrets, PVCs, init containers, sidecars, and more things than anyone asked for.
- **[Extensions](extensions.md)** — *the damned.* External modules that teach h2c new tricks. Today, all extensions are CRD operators (Keycloak, cert-manager, trust-manager) — but the system is open to anything that fits the interface. Each extension is a single `.py` file. For the glory of Yog Sa'rath.
- **[h2c-manager](https://github.com/helmfile2compose/h2c-manager)** — *the dark priest.* Downloads h2c-core and extensions from GitHub releases, resolves dependencies, and provides a `run` shortcut. Reads `helmfile2compose.yaml` for declarative dependency management. Stdlib only, no dependencies.

> *He who renders the celestial into the mundane does not ascend — he merely ensures that both realms now share his suffering equally.*
>
> — *Necronomicon, On the Folly of Downward Translation (probably)*

## But why?

There are dozens of tools that go from Compose to Kubernetes ([Kompose](https://github.com/kubernetes/kompose), [Compose Bridge](https://docs.docker.com/compose/bridge/), [Move2Kube](https://move2kube.konveyor.io/), etc.) — that's the "normal" direction. Almost nothing goes the other way, because who would design their deployment in K8s first and then downgrade?

Using Kubernetes manifests as an intermediate representation to generate a docker-compose is absolutely using an ICBM to kill flies — which is exactly why I find it satisfying.

The name is `helmfile2compose` because both helmfile and docker-compose share the same purpose: deploying an entire self-contained platform at once. If you're using this to convert something that isn't self-contained, you are further into the abyss than I ever ventured, and I am certain it will end terribly. Yog Sa'rath, stay away from me.

> *The disciples beseeched the architect: render thy celestial works in common clay, that we may raise them without knowledge of the heavens. It was heresy. The architect obliged. The temples stood.*
>
> — *Necronomicon, Prayers That Should Never Have Been Answered (probably²)*

## Documentation

### For users

*"I received a compose setup and want plug & play."*

- **[Operations](user/operations.md)** — day-to-day: updating, data management, troubleshooting
- **[Configuration](user/configuration.md)** — `helmfile2compose.yaml` deep dive: volumes, overrides, secrets, replacements
- **[Advanced](user/advanced.md)** — cohabiting with existing infrastructure, multiple projects, disabling Caddy

### For maintainers

*"I have a helmfile and need to provide a compose deployment."*

- **[Your project](maintainer/your-project.md)** — installation, first run, adapting h2c for your own helmfile
- **[h2c-manager](maintainer/h2c-manager.md)** — installing h2c and extensions via the package manager

### For developers

- **[Concepts](developer/concepts.md)** — design philosophy, emulation boundary, K8s vs Compose differences
- **[Architecture](developer/architecture.md)** — converter pipeline, what gets converted, dispatch loop
- **[Writing operators](developer/writing-operators.md)** — develop your own CRD converter

### Reference

- **[Extension catalogue](extensions.md)** — available extensions
- **[Limitations](limitations.md)** — what gets lost in translation
- **[Roadmap](roadmap.md)** — future plans

## Repositories

| Repo | Description |
|------|-------------|
| [h2c-core](https://github.com/helmfile2compose/h2c-core) | Core converter script (`helmfile2compose.py`) |
| [h2c-manager](https://github.com/helmfile2compose/h2c-manager) | Package manager + extension registry |
| [h2c-docs](https://github.com/helmfile2compose/helmfile2compose.github.io) | This documentation site |
| [h2c-operator-keycloak](https://github.com/helmfile2compose/h2c-operator-keycloak) | Keycloak and KeycloakRealmImport CRDs |
| [h2c-operator-certmanager](https://github.com/helmfile2compose/h2c-operator-certmanager) | Certificate, ClusterIssuer, Issuer CRDs |
| [h2c-operator-trust-manager](https://github.com/helmfile2compose/h2c-operator-trust-manager) | Bundle CRD (trust-manager) |

## Compatible projects

- **[stoatchat-platform](https://github.com/baptisterajaut/stoatchat-platform)** — 15 services. Chat platform (Revolt rebranded).
- **[lasuite-platform](https://github.com/baptisterajaut/lasuite-platform)** — 22 services + 11 init jobs. Collaborative suite (La Suite Num.).

## License

Public domain.

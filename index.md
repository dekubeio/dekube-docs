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

> *He who renders the celestial into the mundane does not ascend — he merely ensures that both realms now share his suffering equally.*
>
> — *Necronomicon, On the Folly of Downward Translation (probably)*

## Documentation

### For users

*"I received a compose setup and want plug & play."*

- **[Operations](user/operations.md)** — day-to-day: updating, data management, troubleshooting
- **[Configuration](user/configuration.md)** — `helmfile2compose.yaml` deep dive: volumes, overrides, secrets, replacements
- **[Advanced](user/advanced.md)** — cohabiting with existing infrastructure, multiple projects, disabling Caddy

### For maintainers

*"I have a helmfile and need to provide a compose deployment."*

- **[Your project](maintainer/your-project.md)** — installation, first run, adapting h2c for your own helmfile
- **[h2c-manager](maintainer/h2c-manager.md)** — installing h2c and operators via the package manager

### For developers

- **[Concepts](developer/concepts.md)** — design philosophy, emulation boundary, K8s vs Compose differences
- **[Architecture](developer/architecture.md)** — converter pipeline, what gets converted, dispatch loop
- **[Writing operators](developer/writing-operators.md)** — develop your own CRD converter

### Reference

- **[Operator catalogue](operators.md)** — available CRD converters
- **[Limitations](limitations.md)** — what gets lost in translation
- **[Roadmap](roadmap.md)** — future plans

## Repositories

| Repo | Description |
|------|-------------|
| [h2c-core](https://github.com/helmfile2compose/h2c-core) | Core converter script (`helmfile2compose.py`) |
| [h2c-manager](https://github.com/helmfile2compose/h2c-manager) | Package manager + extension registry |
| [h2c-docs](https://github.com/helmfile2compose/h2c-docs) | This documentation site |
| [h2c-operator-keycloak](https://github.com/helmfile2compose/h2c-operator-keycloak) | Keycloak and KeycloakRealmImport CRDs |
| [h2c-operator-certmanager](https://github.com/helmfile2compose/h2c-operator-certmanager) | Certificate, ClusterIssuer, Issuer CRDs |
| [h2c-operator-trust-manager](https://github.com/helmfile2compose/h2c-operator-trust-manager) | Bundle CRD (trust-manager) |

## Compatible projects

- **[stoatchat-platform](https://github.com/baptisterajaut/stoatchat-platform)** — 15 services. Chat platform (Revolt rebranded).
- **[lasuite-platform](https://github.com/baptisterajaut/lasuite-platform)** — 22 services + 11 init jobs. Collaborative suite (La Suite Num.).

## License

Public domain.

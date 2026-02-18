# helmfile2compose

*For when you just wanted to maintain a nice helmfile but people kept asking for a docker-compose — then it got REALLY out of hand*

Convert Kubernetes manifests to `compose.yml` + `Caddyfile`. Not Kubernetes-in-Docker (kind, k3d, minikube...) — no cluster, no kubelet, no shim. A devolution of the power of Kubernetes into the simplicity of compose: real `docker compose up`, real Caddy, plain stupid containers.

Here because this upmost abomination is causing issues? Here's the path to (partial) redemption: **[Troubleshooting](troubleshooting.md)**.

## But why?

There are dozens of tools that go from Compose to Kubernetes ([Kompose](https://github.com/kubernetes/kompose), [Compose Bridge](https://docs.docker.com/compose/bridge/), [Move2Kube](https://move2kube.konveyor.io/), etc.) — that's the "normal" direction. Almost nothing goes the other way, because who would design their deployment in K8s first and then downgrade?

Using Kubernetes manifests as an intermediate representation to generate a docker-compose is absolutely using an ICBM to kill flies — which is exactly why I find it satisfying.

The name is `helmfile2compose` because both helmfile and docker-compose share the same purpose: deploying an entire self-contained platform at once. If you're using this to convert something that isn't self-contained, you are further into the abyss than I ever ventured, and I am certain it will end terribly. Yog Sa'rath, stay away from me.

> *The disciples beseeched the architect: render thy celestial works in common clay, that we may raise them without knowledge of the heavens. It was heresy. The architect obliged. The temples stood.*
>
> — *Necronomicon, Prayers That Should Never Have Been Answered (probably)*

## Documentation

### For users

*"I received a compose setup and want plug & play."*

- **[Operations](user/operations.md)** — day-to-day: updating, data management, troubleshooting
- **[Advanced](user/advanced.md)** — cohabiting with existing infrastructure, multiple projects, disabling Caddy

### For maintainers

*"I have a helmfile and need to provide a compose deployment."*

- **[Your project](maintainer/your-project.md)** — installation, first run, adapting h2c for your own helmfile
- **[Configuration](maintainer/configuration.md)** — `helmfile2compose.yaml` deep dive: volumes, overrides, secrets, replacements
- **[Known workarounds](maintainer/known-workarounds/index.md)** — sushi recipes for the tentacles that don't fit
- **[h2c-manager](maintainer/h2c-manager.md)** — installing h2c and extensions via the package manager

### For developers

- **[Concepts](developer/concepts.md)** — design philosophy, emulation boundary, K8s vs Compose differences
- **[Architecture](developer/architecture.md)** — converter pipeline, what gets converted, dispatch loop
- **[Code quality](developer/code-quality.md)** — linter scores, complexity metrics, existential dread
- **[Testing](developer/testing.md)** — regression suite, torture generator, performance tracking
- **[Writing extensions](developer/extensions/index.md)** — converters, providers, transforms, rewriters

### Extensions

- **[Extension catalogue](catalogue.md)** — available providers, converters, transforms, rewriters

- **[Troubleshooting](troubleshooting.md)** — when the cursed lands fight back

### Reference
- **[Limitations](limitations.md)** — what gets lost in translation
- **[Roadmap](roadmap.md)** — future plans

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

- **[h2c-core](https://github.com/helmfile2compose/h2c-core)** — *the mad scribe.* A single Python script (~1860 lines) that reads K8s manifests and writes compose. Handles Deployments, StatefulSets, DaemonSets, Jobs, Services, Ingress, ConfigMaps, Secrets, PVCs, init containers, sidecars, and more things than anyone asked for.
- **[Extensions](catalogue.md)** — *the damned.* External modules that teach h2c new tricks. Four types: providers (CRD converters that produce compose services), converters (CRD converters that produce synthetic resources), transforms (post-processing hooks that reshape the final output), and ingress rewriters (translate controller-specific annotations to Caddy rules). Each extension is a single `.py` file. For the glory of Yog Sa'rath.
- **[h2c-manager](https://github.com/helmfile2compose/h2c-manager)** — *the dark priest.* Downloads h2c-core and extensions from GitHub releases, resolves dependencies, and provides a `run` shortcut. Reads `helmfile2compose.yaml` for declarative dependency management. Stdlib only, no dependencies.

## Repositories

| Repo | Description |
|------|-------------|
| [h2c-core](https://github.com/helmfile2compose/h2c-core) | Core converter script (`helmfile2compose.py`) |
| [h2c-manager](https://github.com/helmfile2compose/h2c-manager) | Package manager + extension registry |
| [helmfile2compose.github.io](https://github.com/helmfile2compose/helmfile2compose.github.io) | This documentation site |
| [h2c-provider-keycloak](https://github.com/helmfile2compose/h2c-provider-keycloak) | Keycloak and KeycloakRealmImport CRDs |
| [h2c-converter-cert-manager](https://github.com/helmfile2compose/h2c-converter-cert-manager) | Certificate, ClusterIssuer, Issuer CRDs |
| [h2c-converter-trust-manager](https://github.com/helmfile2compose/h2c-converter-trust-manager) | Bundle CRD (trust-manager) |
| [h2c-provider-servicemonitor](https://github.com/helmfile2compose/h2c-provider-servicemonitor) | Prometheus and ServiceMonitor CRDs |
| [h2c-transform-flatten-internal-urls](https://github.com/helmfile2compose/h2c-transform-flatten-internal-urls) | Strip aliases, rewrite FQDNs to short names |
| [h2c-transform-bitnami](https://github.com/helmfile2compose/h2c-transform-bitnami) | Bitnami Redis, PostgreSQL, Keycloak workarounds |
| [h2c-rewriter-nginx](https://github.com/helmfile2compose/h2c-rewriter-nginx) | Nginx ingress annotation rewriter |
| [h2c-rewriter-traefik](https://github.com/helmfile2compose/h2c-rewriter-traefik) | Traefik ingress annotation rewriter (POC) |
| [h2c-testsuite](https://github.com/helmfile2compose/h2c-testsuite) | Regression & performance test suite |

## Compatible projects

- **[stoatchat-platform](https://github.com/baptisterajaut/stoatchat-platform)** — 15 services. Chat platform (Revolt rebranded).
- **[lasuite-platform](https://github.com/baptisterajaut/lasuite-platform)** — 22 services + 11 init jobs. Collaborative suite (La Suite Num.).
- **[mijn-bureau-infra](https://github.com/numerique-gouv/mijn-bureau-infra)** — ~30 services. Dutch government digital workplace. Not tested extensively, but it starts and the apps respond. Requires `nginx` and `bitnami` extensions.
- **A proprietary, real production-grade helmfile** — Why do you think there are CRDs extension that aren't used in any public repo?

## License

Public domain.

---

*Looking for the full record of what was done, and in what order? The [cursed journal](journal.md) remembers.*

*Looking for why this exists, and why the documentation is complete? The [about page](about.md) has opinions.*

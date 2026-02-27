# helmfile2compose


Translates k8s manifests into a docker compose.yml. Kubernetes reinvented almost by accident.

**Just want it to work?** → **[kubernetes2simple](kubernetes2simple.md)** — one script, zero configuration, no questions asked.

> *And lo, the architect who sought to render the celestial rites in common tongue found himself building a second heaven. "I have translated," he proclaimed, standing in a temple whose pillars bore the same glyphs as the first. The old gods smiled, for one does not carry fire without becoming a hearth.*
>
> — *The Nameless City, On the Propagation of Temples (regrettably)*


## Documentation

### [Troubleshooting](troubleshooting.md)
When the cursed lands fight back

### For users

*"I received a compose setup and want plug & play."*

- **[Operations](user/operations.md)** — day-to-day: updating, data management, troubleshooting
- **[Advanced](user/advanced.md)** — cohabiting with existing infrastructure, multiple projects, disabling Caddy

### For maintainers

*"I have a helmfile and need to provide a compose deployment."*

- **[Your project](maintainer/your-project.md)** — installation, first run, adapting helmfile2compose for your own helmfile
- **[Configuration](maintainer/configuration.md)** — `helmfile2compose.yaml` deep dive: volumes, overrides, secrets, replacements
- **[Known workarounds](maintainer/known-workarounds/index.md)** — sushi recipes for the tentacles that don't fit
- **[dekube-manager](maintainer/h2c-manager.md)** — installing helmfile2compose and extensions via the package manager

### For developers

- **[Concepts](developer/concepts.md)** — design philosophy, emulation boundary, K8s vs Compose differences
- **[Architecture](developer/architecture.md)** — converter pipeline, what gets converted, dispatch loop
- **[dekube-engine](developer/h2c-core.md)** — the bare engine: package structure, three layers, pacts API (the stable public contracts for extensions)
- **[Distributions](developer/distributions.md)** — what a distribution is, how to build one, `_auto_register()`
- **[Build system](developer/build-system.md)** — concatenation, import stripping, `sys.modules` hack
- **[Code quality](developer/code-quality.md)** — linter scores, complexity metrics, existential dread
- **[Testing](developer/testing.md)** — regression suite, torture generator, performance tracking
- **[Writing extensions](developer/extensions/index.md)** — converters, providers, transforms, rewriters

### [Extension catalogue](catalogue.md)
The **Eight Monks** are the founding extensions bundled in the helmfile2compose distribution — everything needed to convert standard K8s manifests. Beyond them, the **disciples** — third-party extensions for CRDs, vendor annotations, and post-processing transforms — extend the conversion to whatever your helmfile throws at it.


### Reference
- **[Limitations](limitations.md)** — what gets lost in translation
- **[Roadmap](roadmap.md)** — future plans
- **[Glossary](glossary.md)** — terms, acronyms, and Lovecraftian vocabulary decoded

## Compatible projects

- **[stoatchat-platform](https://github.com/baptisterajaut/stoatchat-platform)** — 15 services. Chat platform (Revolt rebranded).
- **[lasuite-platform](https://github.com/baptisterajaut/lasuite-platform)** — 22 services + 11 init jobs. Collaborative suite (La Suite Num.).
- **[mijn-bureau-infra](https://github.com/numerique-gouv/mijn-bureau-infra)** — ~30 services. Dutch government digital workplace. Requires `nginx` and `bitnami` extensions.
- A proprietary, real production-grade helmfile — Why do you think there are CRD extensions that aren't used in any public repo?


## But why?

I maintained a helmfile. People wanted a docker-compose. The logical direction is Compose → K8s — dozens of tools do that ([Kompose](https://github.com/kubernetes/kompose), [Compose Bridge](https://docs.docker.com/compose/bridge/), etc.). Almost nothing goes the other way, because why would you. I did it anyway. Then it scope-crept into an extension system, a package manager, a distribution model, and a regression suite. Then the architecture converged with the thing it was converting. Then the author ran out of ideas, energy, and excuses.

```
K8s manifests  →  helmfile2compose.py  →  compose.yml + Caddyfile
```

Not Kubernetes-in-Docker (kind, k3d, minikube...) — no cluster, no kubelet, no shim. Real `docker compose up`, real Caddy, plain stupid containers. Despite the name, **helmfile is not required** — the core accepts any directory of K8s YAML files.

More details, ramblings, and a thorough post-mortem on the [about page](about.md).

## The ecosystem

| Repo | What it is |
|------|------------|
| [kubernetes2simple](https://github.com/dekubeio/kubernetes2simple) | Turnkey distribution — helmfile2compose + all extensions + automagic bootstrap script. |
| [dekube-engine](https://github.com/dekubeio/dekube-engine) | Bare conversion engine — empty registries, no opinions. Produces `dekube.py`. |
| [helmfile2compose](https://github.com/dekubeio/helmfile2compose) | The distribution — core + 8 bundled extensions → single `helmfile2compose.py`. |
| [dekube-manager](https://github.com/dekubeio/dekube-manager) | Package manager — downloads distribution + extensions, resolves dependencies. |
| [Extensions](catalogue.md) | Single-file modules: providers, converters, transforms, rewriters. |
| [dekube-testsuite](https://github.com/dekubeio/dekube-testsuite) | Regression suite + torture generator. |
| [dekube-docs](https://github.com/dekubeio/dekube-docs) | This documentation site. |

## License

Public domain; entirely vibe-coded.

---

*Looking for the full record of what was done, and in what order? The [cursed journal](journal.md) remembers.*

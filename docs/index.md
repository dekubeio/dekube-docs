# dekube

Convert Kubernetes manifests to `compose.yml` + whatever configfile your proxy server will use. Sanity may or may not be included.


If you just want it to work and don't care why, [kubernetes2simple](https://k2s.dekube.io/) will handle everything. If you want to understand what you're running and control what gets loaded, [helmfile2compose](https://helmfile2compose.dekube.io/docs/) is the distribution for you. If you're here, you want to understand the engine itself, write extensions, or build your own distribution.

> *And lo, the architect who sought to render the celestial rites in common tongue found himself building a second heaven. "I have translated," he proclaimed, standing in a temple whose pillars bore the same glyphs as the first. The old gods smiled, for one does not carry fire without becoming a hearth.*
>
> — *The Nameless City, On the Propagation of Temples (regrettably)*

## Understand dekube

How the conversion engine works — from concepts to internals.

- **[Concepts](understand/concepts.md)** — design philosophy, emulation boundary, K8s vs Compose differences
- **[Architecture](understand/architecture.md)** — converter pipeline, what gets converted, dispatch loop
- **[dekube-engine](understand/engine.md)** — the bare engine: package structure, three layers, pacts API
- **[Distributions](understand/distributions.md)** — what a distribution is, how to build one, `_auto_register()`
- **[Build system](understand/build-system.md)** — concatenation, import stripping, `sys.modules` hack

## Extend dekube

Write extensions, maintain code quality, run the test suite.

- **[Code quality](extend/code-quality.md)** — linter scores, complexity metrics, existential dread
- **[Testing](extend/testing.md)** — regression suite, torture generator, performance tracking
- **[Writing extensions](extend/extensions/index.md)** — converters, providers, transforms, rewriters, ingress providers

## Reference

- **[Extension catalogue](catalogue.md)** — the Eight Monks, third-party extensions, the full bestiary
- **[Troubleshooting](troubleshooting.md)** — when the cursed lands fight back
- **[Limitations](limitations.md)** — what gets lost in translation
- **[Roadmap](roadmap.md)** — future plans (and past hubris)
- **[Journal](journal.md)** — the full record of what was done, and in what order
- **[About](about.md)** — the complete and unhinged explanation

## The ecosystem

| Repo | What it is |
|------|------------|
| [helmfile2compose](https://helmfile2compose.dekube.io/docs/) | The distribution — core + 8 bundled extensions → single `helmfile2compose.py`. |
| [kubernetes2simple](https://k2s.dekube.io/) | Turnkey distribution — helmfile2compose + all extensions + automagic bootstrap. |
| [dekube-engine](https://github.com/dekubeio/dekube-engine) | Bare conversion engine — empty registries, no opinions. Produces `dekube.py`. |
| [dekube-manager](https://github.com/dekubeio/dekube-manager) | Package manager — downloads distribution + extensions, resolves dependencies. |
| [Extensions](catalogue.md) | Single-file modules: providers, converters, transforms, rewriters. |
| [dekube-testsuite](https://github.com/dekubeio/dekube-testsuite) | Regression suite + torture generator. |

## License

Public domain; entirely vibe-coded.

---

*Looking for the full record of what was done, and in what order? The [cursed journal](journal.md) remembers.*

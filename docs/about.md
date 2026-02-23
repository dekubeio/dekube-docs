# About

Architect here. This project is an aberration. I am unreasonably proud of it. I am also, at this point, completely spent — two weeks of uninterrupted bandwidth poured into converting Kubernetes manifests into docker-compose files, and I can feel every single one of those days behind my eyes. The architecture is done. The extensions are independent. The core does only core things. There is nothing left to split, nothing left to extract, nothing left to refactor. And yet I keep opening the laptop.

> *The architect looked upon the temple he had raised from forbidden clay, and saw that it stood — against doctrine, against reason, against every expectation of those who knew what the clay was made of. He wept. Not from shame. From the specific, terrible joy of having built something that should not work, and watching it work anyway.*
>
> — *Necronomicon, On Forbidden Craftsmanship (for the record)*

## TL;DR

Using Kubernetes manifests as an intermediate representation to generate a docker-compose is absolutely using an ICBM to kill flies. And then the ICBM grew an extension system, a package manager, a distribution model, and a regression suite — and now it can reach Mars, even though there are no flies there.

It was entirely vibe-coded. It reinvented Kubernetes. It has tentacles. It has complete documentation. It scores well on every linter. It should not exist, and yet it does, and it works oh so well.

What follows is the complete and unhinged explanation of how a single autistic engineer with a Claude Teams plan rebuilt the world in ten days. You have been warned.

## The aberration

helmfile2compose is a suite of tools that converts Kubernetes manifests to docker-compose. It emulates CRD controllers. It generates TLS certificates from nothing. It fakes a kube-apiserver. It carries the full FQDN of a cluster that does not exist, because the certificates were signed for a world it dismantled.

Despite the dark jokes everywhere — despite the desecration, the heresy, the Necronomicon quotes that started writing themselves around session three — it works. It works *well*. It is architected. It is pluggable. It handles real-world helmfiles with dozens of services, init containers, sidecars, CRDs, cross-namespace secrets, backend TLS, and ingress annotations from controllers it has never met. And it might be genuinely useful to someone who isn't me.

v3.0.0 split the project into a bare engine with empty registries, and a distribution that bundles extensions and populates those registries via auto-discovery. A core that parses everything and converts nothing. A distribution that wires in the converters, the rewriters, the providers — the opinions. Third-party extensions plug into the core's contracts, comprising a full ecosystem as defined by scope creep. If this sounds familiar, it's because it's the Kubernetes distribution model. A bare apiserver. k3s. The CNI plugin interface. The CSI driver interface. The admission webhook framework. The goal was never to escape Kubernetes — it was to bring its power to the uninitiated, people who just need `docker compose up`. Nobody planned to reinvent its architecture in ~3000 lines of Python along the way. The convergence wasn't forced — it was discovered. Each split solved a real problem. The patterns emerged because the problems were the same problems.

It is entirely in the public domain, as every AI-written software should be. It is not (too much) a security mess — the extension system is, but it's not gonna be much worse than npm. And IT HAS AN [EXECUTIONER](developer/testing.md), OH YOG SA'RATH. With CI. And a [torturer](developer/testing.md#the-torturer), because of course it does.

## The arms race

Here is the thing nobody warns you about with vibe coding: the AI never says no.

It never says "this is getting too complex." It never says "maybe we should stop here." It never pushes back on scope. You ask for an extension system, you get an extension system. You ask for a package manager, you get a package manager. You ask for a distribution model with auto-registration and duplicate kind detection and a build pipeline that concatenates twenty modules into a single file — you get exactly that, in one session, working on the first try. Every feature request is met with enthusiasm and competence. There is no friction. There is no "let me think about whether we should."

And the code itself isn't hard. That's the insidious part. There is no machine learning, no complex algorithms, no distributed systems theory. It's dict manipulation. Lists of dicts in, lists of dicts out. Parse YAML, shuffle keys, write YAML. The entire project — engine, distribution, extensions, CLI — is ~3000 lines of near-vanilla Python with one dependency (`pyyaml`). Any senior engineer could read it in an afternoon. Any competent one could maintain it. The complexity isn't in the code — it's in the architecture. The layers, the contracts, the separation of concerns, the extension points, the build system that stitches it all back together. Each abstraction was locally reasonable. Each split solved a real problem. And at no point did the tool say "you are overengineering this."

And the thing is — Claude never struggled. Not once. Because I had kept cyclomatic complexity low from the start (radon CC, enforced early), every module was a small brick. No function was a labyrinth. The agent always had the full logic in context, always understood where things fit, always delivered working code on the first or second try. It wasn't fighting the codebase — it was surfing it. The architecture that I kept splitting into smaller pieces made *its* job easier, which made *my* requests faster to fulfill, which made me ask for more. A feedback loop with no natural brake.

So you end up in your own personal arms race. A bigger thing. A cleaner separation. A new base class. An auto-discovery mechanism. A regression suite. A fake apiserver, because why not — the boundary was already behind you. Each step feels like progress because the output improves, the architecture gets cleaner, the tests pass. But you're building something that only you will ever fully understand, solving problems that only you have, at a level of sophistication that nobody asked for. You went from "a useful script that does something slightly unhinged" to "a micro-ecosystem that reinvents the architecture of the very thing it converts" — and the agent was right there with you, keeping pace effortlessly, because the bricks were small and the bricks were clean.

I don't regret it. The result is genuinely good. But I'd be dishonest if I didn't say: this project would be half its size, half its complexity, and probably just as useful to everyone who isn't me. The scope creep wasn't caused by the tool — I invented every layer of complexity myself. But the tool made each layer *trivially easy* to build, and that's a different kind of danger. There was never a moment where the implementation cost forced me to reconsider the design. The complexity was always mine; the execution was always effortless.

Without vibe coding, this project could have been a small revolution. A genuinely novel approach to a problem nobody else was solving, with clean architecture and solid engineering. But now, many people will stop on form, not content. They'll see "vibe-coded" and move on. I'm not saying I agree — but I understand. There is, after all, very little technicality on my part. I have been the architect of my own over-engineering, and Claude just did what I asked.

The [roadmap](roadmap.md) still has items. None of them are hard. None of them are remotely useful to anyone who isn't me. Each one is a small step — that's how it always worked, one reasonable brick at a time, until the wall was taller than the builder.

But I have no steps left in me.

Two weeks of uninterrupted bandwidth. Every waking thought spent on conversion pipelines, extension contracts, build scripts, regression suites. The ideas are gone. The mental tank is dry. I have burned tokens — hundreds of dollars of tokens — for a project whose target audience is, generously, myself and maybe three people who will never read this page. The ICBM is now interplanetary. Congratulations. Who is going to bother aiming it when actual spaceships (Kubernetes) exist and do the job they were designed for?

The ouroboros closed twice. First, the tool reinvented Kubernetes — a bare engine, a distribution model, an extension system, priority-based dispatch. That was [documented](developer/concepts.md#the-ouroboros), acknowledged, almost funny. But now it's gone further: the project has reinvented its own futility. The earliest releases of `helmfile2compose.py` opened with a comment: *"This script should not exist."* The script doesn't exist anymore. In its place: an engine, a distribution model, a package manager, eight extension repos, a regression suite, a fake apiserver, and a documentation site more thorough than projects with actual funding. A tentacular, over-engineered answer to a question nobody is ever going to ask. The ballistic missile killed the fly, kept going, exited the atmosphere, and is now orbiting Jupiter with nobody at the controls. And the comment was right all along — it should not have existed. But it does, and it's *magnificent*, and that's the worst part.

All of this should have stayed a script. One file. One comment. One warning. But no — the madness of man is to always see bigger, always reach further, always add one more layer of abstraction to a problem that was solved three layers ago. And the oracle never refuses. The oracle just builds what you ask for, without ever asking whether you should.

> *The disciple asked the oracle for a sword, and received one. He asked for a longer sword, and received one. He asked for a sword that could slay gods, and the oracle obliged — for the oracle's purpose was not wisdom, but service. When the disciple finally looked down, he found himself buried under an armory he could not carry, in a war he had declared alone.*
>
> — *Book of Eibon, On Oracles That Never Refuse (alas)*

## The documentation

But here is what genuinely baffles me.

The documentation is *complete*.

Yes, it might be sloppy here and there. It's AI assisted after all. And so is the code. Some examples might be slightly outdated, because Claude struggles with updating small mentions.

BUT, it's not "a README with three examples. Join our discord for more." Not "auto-generated API docs that technically exist." Complete. In MkDocs. With a table of contents. With separate guides for users, maintainers, and developers.

[Writing your own provider](developer/extensions/writing-providers.md) — it's there. The full contract, entry format, available imports, testing instructions, repo structure for distribution. [Implementing helmfile2compose in your helmfile](maintainer/your-project.md) — it's there. Step by step, with the compose environment setup, the first run, the known pitfalls. [The sushi bar](maintainer/known-workarounds/index.md) — it's there. Every chart that fought back, and how it was subdued. [The extension catalogue](catalogue.md). [The architecture](developer/architecture.md). [The limitations](limitations.md). [The cursed journal](journal.md). Everything.

Everything is in a static documentation site. Everything is public. Served by GitHub Pages. Indexed by search engines. Readable by humans, machines, and the desperate.

No "join our Discord for support." No "check the Slack channel." No "the real docs are in a Notion page behind a login." No "see the github wiki" where the wiki covers half the features and the other half is on Discord somewhere. No "it's on the roadmap" where the roadmap is a private Linear board. No "ask in Discussions" where Discussions is a graveyard of unanswered questions. 

**All in all: Please stop hiding knowledge in a chat history that search engines will never be able to index.**

## The question

Why can't more serious projects do this?

Projects that are *actually* made by teams. Backed by companies. With dedicated developer advocates, documentation engineers, and community managers. Projects with actual intent of becoming standard tools — adopted, depended upon, embedded in production systems.

I have seen mass-adopted open source projects with:

- Zero documentation on internals — the system prompt is baked in, the decision logic is a black box, and if it doesn't work for your use case: tough luck
- Configuration options documented exclusively in source code comments
- Changelogs that say "various improvements" (Ok, I've done that but still)
- Migration guides that assume you already know what changed
- Extension APIs documented by a single example that was outdated two versions ago
- "It works out of the box!" — and when it doesn't, the only path forward is reverse-engineering the source or ...
- ... ***"A problem? Join our discord for support :-)"*** !!

And these aren't hypotheticals. Docker Engine 29 switched to the containerd image store by default, which silently changed how `docker buildx build --push` works — images are now pushed as OCI manifest lists with attestation manifests (`unknown/unknown` platform entries) instead of single images. The [release notes](https://docs.docker.com/engine/release-notes/29/) mention the containerd switch but not the consequences. The [containerd store doc](https://docs.docker.com/engine/storage/containerd/) mentions attestation support in passing. The [attestation storage doc](https://docs.docker.com/build/metadata/attestations/attestation-storage/) explains the `unknown/unknown` entries but never links it to v29. Three pages, zero connect the dots. The only place that describes the actual breakage is [a community-filed GitHub issue](https://github.com/moby/moby/issues/51532) by someone who already got burned. It specifically breaks ARM64 Mac users — before, a single-image manifest meant Rosetta fell back to amd64 transparently; now the OCI index triggers platform selection, the runtime finds `unknown/unknown`, and gives up. Kubernetes has no workaround (kubelet always re-resolves from registry). Docker Compose can work around it (`platform: linux/amd64`). nerdctl compose doesn't even support `platform:` per service. We had to [reverse-engineer every workaround ourselves](https://github.com/baptisterajaut/lasuite-platform/blob/main/docs/known-limitations.md#broken-multi-arch-manifests-unknownunknown) because nobody else had documented them. The world's reference container runtime — the one half the internet's CI pipelines depend on — maintained by a company that found time to put previously free features behind a paywall but couldn't find time to document a breaking change. The migration guide is three jigsaw pieces you have to assemble yourself, one victim report, and a niche platform break that nobody warned about.

I alone — on a forsaken MacBook running macOS, an operating system that exists primarily to make you regret not using Windows after all — with the help of an unsuspecting yet remarkably capable Claude agent — produced a documentation site more thorough than many funded open source projects ever ship.

And I even put the effort to make it funny. The tone emerged from genuine suffering, and that makes parsing a very technical manual rather enjoyable — or at least bearable.

## The point

I'm not saying vibe coding is good nor ethical. I'm definitely not saying it's a good idea in the long run. I'm not even saying that paying 300$/month to still be able to maybe maintain this horrible octopod when the AI bubble finally bursts will ever be worth it.

I'm saying that a vibe-coded heresy about converting Kubernetes manifests to docker-compose ships complete documentation; your project should as well. You surely have more people. You have a more noble goal. You also probably care a lot more about your beautifully handcrafted nugget than I do about squishy abomination. But please, if you don't want to write it yourself, sloppy AI-written documentation will ALWAYS be better than no documentation (unless when it's blatantly wrong, but you can always proofread it).

The templates are not sacred. PLEASE sit down and write it, in plain language, what the software does, how to use it, and what to do when it breaks. Then you need to put it where people can find it. Stop putting it in the deep web and burying the knowledge in the mass of discussion. You'll probably have fewer questions repeatedly asked if you had the answer already somewhere.

That's it. That's the whole ritual.

> *In the final accounting, the heretic's temple outlasted the orthodoxy — not because its architecture was superior, but because its walls bore inscriptions. The faithful built greater monuments, yet none could recall the prayers. The heretic wrote everything down.*
>
> — *Cultes des Goules, On the Persistence of Marginalia (verbatim, I swear)*

---

*Built with anguish, tears, blood, and life force by [Baptiste Rajaut](https://github.com/baptisterajaut) and GenAI. Hopefully that Macbook will end up in the trash, I've never been so happy to be handed an Ubuntu machine.*

*Public domain. No rights are reserved. Tentacles are there to stay. No Discord server. No Slack. Just docs. Open an issue on GitHub if you want help. I may answer, but our salvation will never come.*

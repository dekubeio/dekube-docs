# About

Architect here. This project is an aberration. I am unreasonably proud of it.

Fourteen days to reinvent the world. Then a fifteenth to look at it, realise it needed a name, and spend an entire week planning a rebrand — designing four static sites, writing design documents, debating tone, and ultimately purchasing an overpriced `.io` domain for a project that converts Kubernetes manifests into docker-compose files. A domain that costs more per year than the project has dependencies.

Of course, the better name — **dekompose** — only came to mind *after* the rebrand was complete. de-K(ubernetes) + compose, the lexical inverse of [Kompose](https://kompose.io/), and "decompose" as a double meaning. Also already taken, more or less. Madness never arrives on schedule.

> *The architect looked upon the temple he had raised from forbidden clay, and saw that it stood — against doctrine, against reason, against every expectation of those who knew what the clay was made of. He wept. Not from shame. From the specific, terrible joy of having built something that should not work, and watching it work anyway.*
>
> — *Necronomicon, On Forbidden Craftsmanship (for the record)*

## TL;DR

Using Kubernetes manifests as an intermediate representation to generate a docker-compose is absolutely using an ICBM to kill flies. And then the ICBM grew an extension system, a package manager, a distribution model, and a regression suite — and now it can reach Mars, even though there are no flies there.

It was entirely vibe-coded. It reinvented Kubernetes. It has tentacles. It has complete documentation. It scores well on every linter. It should not exist, and yet it does, and it works oh so well. Still fewer dependencies than a fresh `create-react-app`.

What follows is the complete and unhinged explanation of how we got here. It comes in two parts: [Part I](#part-i--the-confession) is the existential crisis — how a single autistic engineer with a Claude Teams plan rebuilt the world in fourteen days. [Part II](#part-ii--the-indictment) is the part where I stop apologising and start pointing fingers at the state of open source documentation. If you want to skip the self-flagellation and go straight to my dismay about the industry, [jump to Part II](#part-ii--the-indictment).

---

## Part I — The Confession { #part-i--the-confession }

### The aberration

The idea itself is the aberration. Not the scope creep — that came later, and that one's on me. The idea.

Kubernetes manifests encode intentions for a distributed orchestrator. They describe pods that get scheduled across nodes, services that route through kube-proxy, volumes that get provisioned by CSI drivers, certificates that get issued by controllers watching CRDs. Every field assumes a control plane is listening. Every resource assumes a reconciliation loop will eventually make the world match the spec.

docker-compose describes containers on a single host.

Using the first as an intermediate representation to generate the second is like translating a space shuttle launch checklist into instructions for a paper airplane. The paper airplane will fly — but you'll spend most of your time figuring out what "verify LOX tank pressurization" means when you don't have a LOX tank. Or oxygen. Or a launch pad.

That's what dekube does. It reads Kubernetes manifests — the launch checklist — and produces docker-compose files — the paper airplane. And every single complication in this project is a direct, inevitable consequence of that premise. You want to convert Deployments? You need to understand init containers, sidecar injection, volume claim templates. You want to handle Ingresses? You need to rewrite annotations from controllers you've never met. You want CRDs? You're now emulating operators. You want Certificates? You're now a fake CA. You want Helm charts that talk to the apiserver at install time? You're now faking a kube-apiserver. None of this is scope creep. All of it was *implied* the moment someone said "convert Kubernetes to Compose." The ICBM was always in the blueprint — we just didn't read the fine print.

Sound familiar? Ryan Dahl said "I can run JavaScript on a server" — the language that was designed to validate form fields and animate banner ads, repurposed as a systems runtime. The idea was the aberration; `node_modules` was the consequence. GitHub said "what if a desktop app was just a browser" — the engine that was designed to render documents, repurposed as an application framework. The idea was the aberration; 400MB text editors were the consequence. Docker said "we already have containers, we can orchestrate" — the tool that was designed to run processes, repurposed as a cluster scheduler. The idea was the aberration; Swarm's quiet death was the consequence. And I said "Kubernetes manifests are just YAML, I can reshape them" — the format that was designed for a distributed control plane, repurposed as a Compose generator. The idea is the aberration. Everything else is just the invoice.

The first time someone asked for a docker-compose, I ignored the request. The second time, I built a script and ported two open source platforms. It worked. It should not have worked. But it did, and that was the real danger — because a working aberration is harder to kill than a failed one.

Despite the dark jokes everywhere — despite the desecration, the heresy, the Necronomicon quotes that started writing themselves around session three — it works. It works *well*. It is architected. It is pluggable. It handles real-world helmfiles with dozens of services, init containers, sidecars, CRDs, cross-namespace secrets, backend TLS, and ingress annotations from controllers it has never met. And it might be genuinely useful to someone who isn't me.

It is entirely in the public domain, as every AI-written software should be. It is not (too much) a security mess — the extension system is, but it's not gonna be much worse than npm. And IT HAS AN [EXECUTIONER](extend/testing.md), OH YOG SA'RATH. With CI. And a [torturer](extend/testing.md#the-torturer), because of course it does.

### The arms race

The aberration is the idea. What follows is entirely my fault.

Here is the thing nobody warns you about with vibe coding: the AI never says no.

It never says "this is getting too complex." It never says "maybe we should stop here." It never pushes back on scope. You ask for an extension system, you get an extension system. You ask for a package manager, you get a package manager. You ask for a distribution model with auto-registration and duplicate kind detection and a build pipeline that concatenates twenty modules into a single file — you get exactly that, in one session, working on the first try. Every feature request is met with enthusiasm and competence. There is no friction.

There is no "let me think about whether we should."

And the code itself isn't hard. That's the insidious part. There is no machine learning, no complex algorithms, no distributed systems theory. It's dict manipulation. Lists of dicts in, lists of dicts out. Parse YAML, shuffle keys, write YAML. The entire project — engine, distribution, extensions, CLI — is ~3000 lines of near-vanilla Python with one dependency (`pyyaml`). Any senior engineer could read it in an afternoon. Any competent one could maintain it.

The complexity isn't in the code — it's in the architecture. The layers, the contracts, the separation of concerns, the extension points, the build system that stitches it all back together. Every decision was locally reasonable. And at no point did the tool say "you are overengineering this." But then again — nobody told the Node.js ecosystem that either, and they have *1200 dependencies to left-pad a string*.

And the thing is — Claude never struggled. Not once. Because I had kept cyclomatic complexity low from the start (radon CC, enforced early), every module was a small brick. No function was a labyrinth. The agent always had the full logic in context, always understood where things fit, always delivered working code on the first or second try. It wasn't fighting the codebase — it was surfing it. The architecture that I kept splitting into smaller pieces made *its* job easier, which made *my* requests faster to fulfill, which made me ask for more. A feedback loop with no natural brake.

So you end up in your own personal arms race — alone. A bigger thing. A cleaner separation. A new base class. An auto-discovery mechanism. A regression suite. A fake apiserver, because why not — the boundary was already behind you. Each step feels like progress because the output improves, the architecture gets cleaner, the tests pass. But you're building something that only you will ever fully understand, solving problems that only you have, at a level of sophistication that nobody asked for.

The final escalation: v3.0.0 split the project into a bare engine with empty registries, and a distribution that bundles extensions and populates those registries via auto-discovery. A core that parses everything and converts nothing. A distribution that wires in the converters, the rewriters, the providers — the opinions. Third-party extensions plug into the core's contracts, comprising a full ecosystem as defined by scope creep. The agent delivered it in one session. Working on the first try. Of course.

I don't regret it. The scope creep wasn't caused by the tool — I invented every layer of complexity myself. But the tool made each layer *trivially easy* to build, and that's a different kind of danger. There was never a moment where the implementation cost forced me to reconsider the design. The complexity was always mine; the execution was always effortless. Some people will stop on form, not content. They'll see "vibe-coded" and move on. Their loss. Judge it by its output, not its origin.

The [roadmap](roadmap.md) still has items. Each one is a small step — that's how it always worked, one reasonable brick at a time, until the wall was taller than the builder.

### The ouroboros

And then I looked at what we had built.

A bare engine. A distribution model. An extension system with priority-based dispatch. A plugin interface. Auto-discovery. Typed contracts. If this sounds familiar, it's because it's the Kubernetes distribution model. A bare apiserver. k3s. The CNI plugin interface. The CSI driver interface. The admission webhook framework.

The goal was never to escape Kubernetes — it was to bring its power to the uninitiated, people who just need `docker compose up`. Nobody planned to reinvent its architecture along the way. The convergence wasn't forced — it was discovered. Each split solved a real problem. The patterns emerged because the problems were the same problems. That was [documented](understand/concepts.md#the-ouroboros), acknowledged, almost funny.

But the ouroboros closed twice. The second time was worse. The earliest releases of `helmfile2compose.py` opened with a comment: *"This script should not exist."* The script doesn't exist anymore. In its place: an engine, a distribution model, a package manager, eight extension repos, a regression suite, a fake apiserver, and a documentation site. The ballistic missile killed the fly, kept going, exited the atmosphere, and is now orbiting Jupiter with nobody at the controls. The comment was right all along — it should not have existed. But it does, and it's *magnificent*, and that's the worst part.

And then someone dared us to [go further](https://github.com/baptisterajaut/dekube-transform-nspawn). Compose to systemd-nspawn. Never tested. Helm charts → Kubernetes manifests → compose.yml → systemd unit files. We started with an orchestrator and arrived at the init system. The ouroboros has digested its own tail and is now eating the output.

All of this should have stayed a script. One file. One comment. One warning. But no — the madness of man is to always see bigger, always reach further, always add one more layer of abstraction to a problem that was solved three layers ago. And the oracle never refuses. The oracle just builds what you ask for, without ever asking whether you should.

> *The disciple asked the oracle for a sword, and received one. He asked for a longer sword, and received one. He asked for a sword that could slay gods, and the oracle obliged — for the oracle's purpose was not wisdom, but service. When the disciple finally looked down, he found himself buried under an armory he could not carry, in a war he had declared alone.*
>
> — *Book of Eibon, On Oracles That Never Refuse (alas)*

---

## Part II — The Indictment { #part-ii--the-indictment }

### The documentation

But here is what genuinely baffles me.

The documentation is *complete*.

Yes, it might be sloppy here and there. It's AI assisted after all. Some examples might be slightly outdated, because Claude struggles with updating small mentions.

BUT, it's not "a README with three examples." Not "auto-generated API docs that technically exist." Complete. In MkDocs. With a table of contents. With separate guides for users, maintainers, and developers.

[Writing your own provider](extend/extensions/writing-providers.md) — it's there. The full contract, entry format, available imports, testing instructions, repo structure for distribution. [Implementing helmfile2compose in your helmfile](https://helmfile2compose.dekube.io/docs/getting-started/) — it's there. Step by step, with the compose environment setup, the first run, the known pitfalls. [The sushi bar](https://helmfile2compose.dekube.io/docs/known-workarounds/) — it's there. Every chart that fought back, and how it was subdued. [The extension catalogue](catalogue.md). [The architecture](understand/architecture.md). [The limitations](limitations.md). [The cursed journal](journal.md). Everything.

Everything is in a static documentation site. Everything is public. Served by GitHub Pages. Indexed by search engines. Readable by humans, machines, and the desperate.

And I even put the effort to make it funny. The tone emerged from genuine suffering, and that makes parsing a very technical manual rather enjoyable — or at least bearable.

### The question

So why can't more serious projects do this?

Projects that are *actually* made by teams. Backed by companies. With dedicated developer advocates, documentation engineers, and community managers. Projects with actual intent of becoming standard tools — adopted, depended upon, embedded in production systems.

I have seen mass-adopted open source projects with:

- Zero documentation on internals — the system prompt is baked in, the decision logic is a black box, and if it doesn't work for your use case: tough luck
- Configuration options documented exclusively in source code comments
- Changelogs that say "various improvements" (Ok, I've done that but still)
- Migration guides that assume you already know what changed
- Extension APIs documented by a single example that was outdated two versions ago
- "It works out of the box!" — and when it doesn't, the only path forward is reverse-engineering the source or ...
- ... ***"A problem? Join our discord for support :-)"*** !!


<details markdown="1">
<summary>And these aren't hypotheticals. Open this for a massive rant against the Docker docs</summary>

Docker Engine 29 switched to the containerd image store by default, which silently changed how `docker buildx build --push` works — images are now pushed as OCI manifest lists with attestation manifests (`unknown/unknown` platform entries) instead of single images. The [release notes](https://docs.docker.com/engine/release-notes/29/) mention the containerd switch but not the consequences. The [containerd store doc](https://docs.docker.com/engine/storage/containerd/) mentions attestation support in passing. The [attestation storage doc](https://docs.docker.com/build/metadata/attestations/attestation-storage/) explains the `unknown/unknown` entries but never links it to v29. Three pages, zero connect the dots. The only place that describes the actual breakage is [a community-filed GitHub issue](https://github.com/moby/moby/issues/51532) by someone who already got burned. It specifically breaks ARM64 Mac users — before, a single-image manifest meant Rosetta fell back to amd64 transparently; now the OCI index triggers platform selection, the runtime finds `unknown/unknown`, and gives up. Kubernetes has no workaround (kubelet always re-resolves from registry). Docker Compose can work around it (`platform: linux/amd64`). nerdctl compose doesn't even support `platform:` per service. We had to [reverse-engineer every workaround ourselves](https://github.com/baptisterajaut/lasuite-platform/blob/main/docs/known-limitations.md#broken-multi-arch-manifests-unknownunknown) because nobody else had documented them. The world's reference container runtime — the one half the internet's CI pipelines depend on — maintained by a company that found time to put previously free features behind a paywall but couldn't find time to document a breaking change. The migration guide is three jigsaw pieces you have to assemble yourself, one victim report, and a niche platform break that nobody warned about.

In different fashion, their own docs [appear to contradict each other](understand/architecture.md#beyond-single-host) on whether Swarm supports the current Compose Specification or is stuck on legacy v3. The `deploy:` key (replicas, placement, rolling updates) originated as a Swarm-specific section in Compose v3. Docker later backported it into the current Compose Specification, so `docker compose` understands it too — but never upgraded Swarm to support the new spec. So both formats have a `deploy:` key with the same syntax, but the files around it are incompatible. The Swarm docs say "the Compose Specification isn't compatible"; the Compose Deploy Specification documents `deploy:` as part of the current spec. Two pages, same domain, same key name, different formats. We had to ask their docs AI to get a straight answer. Which, fair enough, is better than sending an email and being told to pay for support, or posting in a community forum and being answered three months later by a bot closing the issue as stale. But how long will these AI assistants remain free to use? The information *is* in the docs — it's just scattered across pages that don't cross-reference each other, and the only reliable way to find it is a tool whose business model hasn't been figured out yet. Oh, and Swarm's core specification format has been deprecated by its own maintainers without ever being upgraded. A deprecated spec powering a proclaimed production-grade orchestrator. Amazing.

</details>

I alone — with the help of an unsuspecting yet remarkably capable Claude agent — produced a documentation site more thorough than many funded open source projects ever ship. Windows Server has existed for decades and its documentation still can't decide whether IIS is a feature or a punishment. Node.js has 47,000 contributors and the `fs` module docs still don't explain when to use streams. And [Kompose](https://kompose.io/) — the Kubernetes SIG project that does the exact opposite of dekube (Compose → K8s) — ships five pages of documentation, an architecture guide that shows three Go interfaces and stops, and a [development guide](https://github.com/kubernetes/kompose/blob/main/docs/development.md) that teaches you how to fork a repo and run `git push`. A CNCF project. With maintainers. Whose contribution guide is a git tutorial.

### The point

A vibe-coded heresy about converting Kubernetes manifests to docker-compose ships complete documentation. Your project should as well. You surely have more people. You have a more noble goal. You also probably care a lot more about your beautifully handcrafted nugget than I do about this squishy abomination.

And yet — here we are. A heresy with full docs, and an orthodoxy with "join our Discord for support." An ICBM with a user manual, and actual spaceships with a Post-it note on the cockpit. We're not better engineers. We're not more virtuous. We just wrote it down — because the heretic knows nobody will believe him otherwise.

The templates are not sacred. PLEASE sit down and write, in plain language, what the software does, how to use it, and what to do when it breaks. Then put it where people can find it. Stop burying knowledge in chat histories that search engines will never index. You'll probably have fewer questions repeatedly asked if the answer was already somewhere findable.

That's it. That's the whole ritual.

> *In the final accounting, the heretic's temple outlasted the orthodoxy — not because its architecture was superior, but because its walls bore inscriptions. The faithful built greater monuments, yet none could recall the prayers. The heretic wrote everything down.*
>
> — *Cultes des Goules, On the Persistence of Marginalia (verbatim, I swear)*

---

*Built with anguish, tears, blood, life force, and an unreasonable `.io` domain by [Baptiste Rajaut](https://github.com/baptisterajaut) and GenAI.*

*Public domain. No rights reserved. Tentacles are there to stay. No Discord server. No Slack. Just docs. Open an issue on GitHub if you want help. I may answer, but our salvation will never come.*

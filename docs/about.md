# About

Architect here. This project is an aberration. I am unreasonably proud of it.

> *The architect looked upon the temple he had raised from forbidden clay, and saw that it stood — against doctrine, against reason, against every expectation of those who knew what the clay was made of. He wept. Not from shame. From the specific, terrible joy of having built something that should not work, and watching it work anyway.*
>
> — *Necronomicon, On Forbidden Craftsmanship (for the record)*

## The aberration

helmfile2compose converts Kubernetes manifests to docker-compose. It emulates CRD controllers. It generates TLS certificates from nothing. It fakes a kube-apiserver. It carries the full FQDN of a cluster that does not exist, because the certificates were signed for a world it dismantled.

Despite the dark jokes everywhere — despite the desecration, the heresy, the Necronomicon quotes that started writing themselves around session three — it works. It works *well*. It is architected. It is pluggable. It handles real-world helmfiles with dozens of services, init containers, sidecars, CRDs, cross-namespace secrets, backend TLS, and ingress annotations from controllers it has never met. And it might be genuinely useful to someone who isn't me.

It was entirely vibe-coded.

Every line of this project was written across Claude Code sessions on a single MacBook. No team. No company. No funding. No roadmap committee. No sprint planning. No Jira ticket. One autistic engineer with an AI agent that didn't know what it was building until it was too late.

And yet:

- It is not (too much) a security mess. (The extension system is a security mess, but it's not gonna be much worse than npm)
- It is entirely in the public domain, as every ai-written software should be.
- The extension system has a documented interface, versioned releases, and dependency resolution.
- IT HAS [TESTS](developer/testing.md), OH YOG SA'RATH. With CI. And a torture generator, because of course.

## The documentation

But here is what genuinely baffles me.

The documentation is *complete*.

Yes, it might be sloppy here and there. It's AI assisted after all. And so is the code. Some examples might be slightly outdated, because Claude struggles with updating small mentions.

BUT, it's not "a README with three examples. Join our discord for more." Not "auto-generated API docs that technically exist." Complete. In MkDocs. With a table of contents. With separate guides for users, maintainers, and developers.

[Writing your own CRD extension](developer/extensions/writing-operators.md) — it's there. The full contract, entry format, available imports, testing instructions, repo structure for distribution. [Implementing h2c in your helmfile](maintainer/your-project.md) — it's there. Step by step, with the compose environment setup, the first run, the known pitfalls. [The sushi bar](maintainer/known-workarounds/index.md) — it's there. Every chart that fought back, and how it was subdued. [The extension catalogue](catalogue.md). [The architecture](developer/architecture.md). [The limitations](limitations.md). [The cursed journal](journal.md). Everything.

Everything is in a static documentation site. Everything is public. Served by GitHub Pages. Indexed by search engines. Readable by humans, machines, and the desperate.

No "join our Discord for support." No "check the Slack channel." No "the real docs are in a Notion page behind a login." No "see the github wiki" where the wiki covers half the features and the other half is on Discord somewhere. No "it's on the roadmap" where the roadmap is a private Linear board. No "ask in Discussions" where Discussions is a graveyard of unanswered questions. 

**All in all: not putting knowledge in a chat history that search engines will never be able to index.**

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

I alone — on a forsaken MacBook running macOS, an operating system that exists primarily to make you regret not using Windows after all — with the help of an unsuspecting yet remarkably capable Claude agent — produced a documentation site more thorough than many funded open source projects ever ship.

And I even put the effort to make it funny. The tone emerged from genuine suffering, and that makes parsing a very technical manual rather enjoyable — or at least bearable.

## The point

I'm not saying vibe coding is good. I'm not saying it's a good idea in the long run.

I'm saying a vibe-coded heresy about converting Kubernetes manifests to docker-compose ship complete documentation; your project should as well. You have probably more people. You have more reason. You probably care a lot more about your handcrafted nugget – And I get it, it's a nice nugget – than I do about this. But please, if you don't want to write it yourself, AI Slop documentation is probably still better than no documentation (unless it's blatantly wrong, but you can proofread it).

The templates are not sacred. PLEASE sit down and write it, in plain language, what the software does, how to use it, and what to do when it breaks. Then you need to put it where people can find it. Stop putting it in the deep web and burying the knowledge in the mass of discussion. You'll probably have fewer questions repeatedly asked if you had the answer already somewhere.

That's it. That's the whole ritual.

> *In the final accounting, the heretic's temple outlasted the orthodoxy — not because its architecture was superior, but because its walls bore inscriptions. The faithful built greater monuments, yet none could recall the prayers. The heretic wrote everything down.*
>
> — *Cultes des Goules, On the Persistence of Marginalia (verbatim, I swear)*

---

*Built with anguish, tears and blood by [Baptiste Rajaut](https://github.com/baptisterajaut), and with the help of GenAI that was just trying to help. Hopefully that Macbook will end up in the trash, I've never been so happy to be handed an Ubuntu machine*

*Public domain. No rights reserved. Tentacles are there to stay. No Discord server. No Slack. Just docs. Open an issue on Github if you want help. I may answer, but salvation will never come.*

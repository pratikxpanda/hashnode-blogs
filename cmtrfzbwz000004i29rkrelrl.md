---
title: "You already have most of an Agent Skills library"
seoTitle: "You already have most of an Agent Skills library"
seoDescription: "The gap between an internal runbook and an Agent Skill is frontmatter and a folder, not a rewrite."
datePublished: 2026-09-07T16:15:27.209Z
cuid: cmtrfzbwz000004i29rkrelrl
slug: you-already-have-an-agent-skills-library
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1788778056902/da43228a-2582-4fa9-a134-566b5de5885d.png
tags: ai, python, opensource, documentation, developer-tools

---

*Part 3 of a series on the Agent Skills SDK. [Part 2](https://pratikpanda.hashnode.dev/writing-an-agent-skill-you-can-trust) covered scaffolding a skill and getting the CLI to check it.*

By the end of the last post you can write a skill and have the tooling tell you whether it's any good. One skill.

The gap between one skill and a library worth building an agent around is where most people stop, and I don't think it's because the idea stops being appealing. It's simple maths. If it takes an hour to write a decent skill, and you need fifteen of them to cover what your team actually does, that's two days of writing documents to find out whether the approach helps at all. Nobody has two days for a maybe.

So the honest question isn't "how do I write good skills." It's "how do I get to fifteen without spending two days."

Have a look in your repositories first.

## The content exists

There's a reasonable chance your repos already contain some of:

- `AGENTS.md` in the root
- `.github/copilot-instructions.md`
- `.cursor/rules/*.mdc`
- Claude-style skill folders with a `SKILL.md`

That's not a rough start on a skill library. It's largely the same content in a different wrapper. Someone on your team already sat down and wrote out how deployments work here, what the testing conventions are, which service owns what. It's been reviewed. It's been corrected when it turned out to be wrong. It's the knowledge your team built up over years, and it is almost certainly better than what you'd write fresh on a Tuesday afternoon.

It's just locked to one tool.

That's the real problem today. The content is fine. The formats belong to individual vendors. Adopt a second tool, or switch, and you're maintaining the same knowledge in three files. They agree for about a month.

## Converting one

```bash
agentskills init --from ./AGENTS.md deployment-guide
```

That detects the format, parses it, and writes a proper skill folder — `SKILL.md` with frontmatter and the content underneath.

It writes only the `SKILL.md`, not the empty `references/`, `scripts/` and `assets/` directories that a plain `init` gives you. That is the right call: an import has no resources yet, and three empty folders next to converted content is clutter rather than a prompt.

It's the same `init` command, which is deliberate. Importing isn't a separate migration mode you run once and forget. It's another way to start a skill, and it sits right next to the one you already know.

## Converting all of them

```python
from agentskills_adapters import adapt_path, discover_sources

for source in discover_sources("."):
    skill = adapt_path(source)
```

`discover_sources` walks a tree and finds everything convertible. On a repo of any age this usually turns up more than you expected, including a couple of files somebody added during an experiment and never mentioned again.

Cursor rules keep their `globs` in metadata rather than losing them. Convert a `.mdc` rule and the frontmatter comes out like this:

```yaml
---
name: react-conventions
description: Frontend conventions for React components
metadata:
  globs:
  - src/**/*.tsx
  - src/**/*.ts
  source: cursor
---
```

That pattern is real information, because it says which files the guidance applies to. Throwing it away just because the target format has no dedicated field for it would lose something useful for no reason. The `source` key is there for the same reason: six months later you will want to know where a skill came from.

## The part the adapter won't do

This is where conversion gets opinionated, and it's the most important paragraph in this post.

Most source formats have no description field. `AGENTS.md` is a document. It has content and no summary. But a skill without a description is worse than useless. It's *invisible*, because the description is the catalog entry the agent reads when it chooses. Convert a file with no description and you get a skill that exists, validates, loads, and is never chosen for anything.

So the adapters never emit a blank catalog entry. Anything lacking a description gets a placeholder built from the document's own heading:

> Imported instructions for Deployment guide.

Which is, on purpose, not good. It gets the skill into a valid state and it's obviously a placeholder to anyone reading it. It will not get the skill chosen for anything sensible.

That's the intended shape of the work. The adapter does the mechanical part in seconds: parsing, structure, frontmatter, file layout. The judgement part is writing a description that gets each skill chosen for the right requests and not the wrong ones. That part is yours. It's the only bit that genuinely needs a person, and it is where nearly all of the value of a skill library is decided.

If you convert twenty files and stop there, you have twenty invisible skills and a reasonable case that this whole approach doesn't work. Ten minutes per description is the difference.

## Not a fork

Worth stating plainly, because it's the fair objection: adapters are a migration path, not a competing format.

The goal isn't one universal format that everything converts through. It's to remove the excuse. To get you from "I'd have to write all of this" to "it's already written, let me fix the descriptions" in about the time it takes to run one command.

And if you convert your content, try it, and decide skills aren't for you, you've lost an afternoon and your original files are untouched. That's a far easier decision than the one that starts with an empty directory.

## A realistic first hour

If you want to actually do this rather than agree with it and close the tab:

Run `discover_sources` over your main repo and look at what comes back. Convert three — the ones covering what people ask about most, not the most impressive documents. Spend real time on those three descriptions, including a sentence on when *not* to use each.

Then `agentskills validate` and `agentskills lint` from part 2 to check what you've got.

And then, before wiring anything into application code, watch it work:

```bash
pip install 'agentskills-tools[serve]'
agentskills serve ./skills
```

That runs an MCP server over the folder, exposing every skill as tools and resources. Point any MCP client at it and you can watch an agent discover, choose and load your converted content without writing a line of integration code.

What the client sees is eight tools and three resources:

```
tools:     get_skill_metadata, get_skill_body, get_skill_outline,
           get_skill_section, list_skill_resources, get_skill_reference,
           get_skill_asset, get_skill_script
resources: skills://catalog/xml, skills://catalog/markdown,
           skills://tools-usage-instructions
```

The split is the point. The catalog is a resource because it is always available and cheap. Everything that costs real tokens is a tool the agent has to decide to call.

I'd do this before the real wiring, every time. If a skill is never chosen here, the problem is in the skill, and no amount of application code will fix it. Answering that question early saves you hours of debugging integration code when the fault was in a description all along.

When you're ready to wire this into an application, the LangChain, Agent Framework and MCP code from part 1 is all you need.

That's an hour, and at the end of it you know whether this is worth doing where you work. Which is the only question that matters this early.

## What tends to go wrong

Converted content was written for humans reading a repo, not for a model doing a task. It's usually longer, more rambling, and heavier on background than a skill needs. That's not a bug in the conversion, because the two kinds of document have different jobs. But it does mean `lint` will have opinions about body size after a bulk import, and it will be right.

The other thing worth fixing at conversion time, while you're already in the file: make sure the document has real headings. A long skill with clear sections is genuinely fine, because there are ways to load one section instead of the whole thing. A long skill that's one undivided stream of prose is not, and it's much easier to fix now than to come back to.

But the headline is how cheap the experiment is. Convert three files, write three honest descriptions, and you'll know by lunchtime whether any of this belongs in your stack. That's a very different question from the two days of writing it looked like this morning.

`agentskills-adapters` ships in 0.5.0, on [GitHub](https://github.com/pratikxpanda/agentskills-sdk).
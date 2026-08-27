---
title: "Writing an Agent Skill you can trust"
seoTitle: "Writing an Agent Skill you can trust"
seoDescription: "version: 1.0 is a YAML float, not a string. Validation catches that. It does not catch a skill that is merely useless."
datePublished: 2026-08-27T05:34:28.654Z
cuid: cmtb38nrt000009ksdrtt4u9h
slug: writing-an-agent-skill-you-can-trust
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1787750850544/a2320efd-3379-4300-b0f7-b2632acf30fd.png
tags: ai, productivity, python, opensource, developer-tools

---

*Part 2 of a series on the Agent Skills SDK.* [*Part 1*](https://pratikpanda.hashnode.dev/agent-skills-not-a-knowledge-base) *covered what the Agent Skills format is and how to get a folder of skills into a LangChain, Agent Framework or MCP agent.*

If you followed part 1, you have working plumbing. A `SkillRegistry`, a provider pointed at a folder, and an agent that can see whatever's in it.

Which raises the next question, and it's a harder one: what should actually go in that folder?

I wrote my first few skills by feel. Markdown file, some frontmatter, instructions underneath, looked fine. They worked, mostly. Then one didn't, and finding out why took an embarrassing amount of time. "The agent isn't behaving as expected" has roughly forty possible causes, and about six of them are things you got wrong in a file you were confident about.

That experience is most of why the SDK now has an opinion about what a skill should look like, and a CLI that will tell you when yours doesn't.

## Start from the scaffold

```bash
pip install agentskills-tools
agentskills init deployment-runbook
```

That prints `Created deployment-runbook` and gives you:

```plaintext
deployment-runbook/
  SKILL.md
  references/
  scripts/
  assets/
```

The three empty directories are deliberate. They're the resource types the format defines, and having them there changes how you write. When a section starts turning into three paragraphs of detail, there's an obvious place to put it. Without them, you just keep typing into the body.

The generated `SKILL.md` opens with this:

```yaml
---
name: deployment-runbook
description: One sentence on what this skill does and when an agent should reach for it.
version: 0.1.0
# Selection metadata: where this skill applies, and where it stops. Both are
# optional, capped at five entries of 200 characters, and charged on every
# turn — so keep them short, and say what the description cannot.
# when_to_use:
#   - The situation that should make an agent reach for this skill
# when_not_to_use:
#   - The nearby situation an agent might mistake for that one
---
```

Commented out, with the reasoning next to them. I went back and forth on whether to generate those fields as real values or as comments. Comments won. A scaffold that writes placeholder text people forget to replace is worse than one that writes nothing. And the comment explaining *why* the fields are capped does more good than the fields would.

`--description` sets it at scaffold time, and `--path` puts it somewhere other than the current directory. `--from` builds a skill out of content you've already written in another format, which is by far the fastest way to get a library started.

## What the validator actually enforces

Run `agentskills validate ./skills` and it checks the skill against the specification. Two fields are required and everything else is optional, but the required two have real rules:

`name` must be lowercase alphanumeric with hyphens, at most 64 characters, must not start or end with a hyphen, must not contain consecutive hyphens, and must match the directory name.

That last rule catches a nasty bug. The skill loads under one ID, while its own frontmatter says another. Every log line then points at a name that doesn't match anything, and you lose an hour doubting code that was fine.

`description` is required and capped at 1024 characters.

The optional fields are type-checked too: `license` must be a string, `compatibility` and `metadata` must be maps, `allowed-tools` must be a list. And `when_to_use` and `when_not_to_use` are limited to five entries of 200 characters each.

That last limit has an error message I'm quite attached to:

```plaintext
skills/deployment-runbook
  error   spec: field 'when_to_use' has 6 entries, over the limit of 5; it is charged on every turn, and a skill needing more conditions is probably two skills
```

Both halves matter. The first explains the cost: selection metadata is carried on every turn, exactly like the description, so it cannot be unbounded. The second is the actual advice. A skill that needs six separate activation conditions has usually been asked to do two unrelated jobs. Split it and you get two skills that each get chosen correctly, instead of one that gets chosen at random.

## The version field, and a bug I want you to avoid

`version` is optional, not part of the upstream spec, and supported here because pinning matters once skills are shared. It must be a **quoted** semver string.

The quoting rule is not me being fussy. This is valid YAML and completely wrong:

```yaml
version: 1.0
```

YAML parses that as the float `1.0`. Not the string `"1.0"`. And this:

```yaml
version: 2026-01-15
```

parses as a date object. Neither one is a version. Both look exactly like a version to a human reading the file. And both fail with an error that makes no sense unless you already know that YAML guesses types for you.

So the error says so:

```plaintext
skills/release-checklist
  error   spec: version must be a quoted string, got float (1.0). Unquoted YAML reads 1.0 as a number and 2024-01-15 as a date — write version: "1.0.0"
```

I lost a genuinely stupid amount of time to this before writing that message. An error that names the fix is worth ten that name the problem.

You may have noticed the scaffold above writes `version: 0.1.0` without quotes, which looks like it contradicts all of that. It doesn't. The rule the validator applies is that the *parsed* value has to be a string, and `0.1.0` has two dots, so YAML has no choice but to read it as one. The values that go wrong are the ones that are also valid numbers or dates. Quote it anyway — the habit costs nothing, and it is the version you type by hand, not the one the scaffold wrote, that catches you out.

## Valid and good are different questions

This is the distinction I think most skill tooling misses, and it's why there are two commands instead of one.

`validate` asks: **is this a legal skill?** Does it match the spec. Errors, exit code 1, no opinions.

`lint` asks: **is this a skill that will work well?** Warnings, exit 0 by default, entirely opinionated, and based on mistakes I've made.

```bash
agentskills lint ./skills
```

On a folder where I had deliberately planted one of each:

```plaintext
skills/database-migration
  warning missing-selection-metadata: description is 278 characters and the skill declares neither 'when_to_use' nor 'when_not_to_use'; state the boundary in those fields instead of folding it into the description
skills/deployment-runbook
  warning body-over-token-budget: body is roughly 7835 tokens, over the 5000 budget; move detail into references/ so it loads only when needed
skills/incident-response
  warning unreferenced-resource: references/severity-levels.md is never mentioned in the body, so an agent will not know to load it
skills/oncall-escalation
  warning description-too-long-for-catalog: description is 554 characters; catalog entries stay in context every turn, so keep it under 500
skills/release-checklist
  warning missing-version: no 'version' field, so consumers cannot pin this skill or detect drift

5 skills checked, 0 errors, 5 warnings
```

The rules:

`description-too-long-for-catalog` fires over 500 characters, even though `validate` allows 1024. Legal isn't the same as sensible. The description is sent on every turn, and 500 characters is where I start to worry. The warning tells you the count: `description is 554 characters; catalog entries stay in context every turn, so keep it under 500`.

`missing-selection-metadata` fires when a description is over 200 characters and the skill declares neither `when_to_use` nor `when_not_to_use`. Descriptions usually get long because someone is trying to describe a boundary in prose. There are fields for that.

`body-over-token-budget` fires over roughly 5,000 tokens, adjustable with `--max-body-tokens`. Suggests moving detail to `references/`, which loads only when needed.

`missing-version` warns that consumers can't pin the skill or detect drift.

And my favourite, `unreferenced-resource`: a file in `references/`, `scripts/` or `assets/` that the body never mentions. An agent that is never told the file exists will never fetch it. The file isn't broken and nothing throws an error. It just sits there, never used, and nothing at runtime will tell you.

`--strict` promotes warnings to failures if you want the stricter gate.

## Putting it in CI

None of this works on a real team unless it runs automatically. Nobody runs a linter by hand for more than two weeks.

```yaml
- name: Validate skills
  uses: pratikxpanda/agentskills-sdk/actions/validate@v1
  with:
    path: ./skills
    fail-on-lint: false
```

It installs the CLI if it isn't already on PATH, runs `validate`, and runs `lint` only if validate passed. Both use `--format github`, so every finding is annotated on the exact line in the pull request diff.

That last detail matters more than it sounds. A line in a CI log saying "description exceeds 1024 characters" gets skipped by someone who has already moved on to the next thing. The same message, attached to the line in the diff, while a reviewer is looking at it, gets fixed straight away.

I'd start with `fail-on-lint: false`. Let the warnings be visible for a few weeks and see which ones your team agrees with before making any of them blocking. A gate that fires on things people don't believe in gets switched off, and then you have neither the gate nor the credibility to add it back.

## What this doesn't tell you

Everything above checks the *shape* of a skill. Valid frontmatter, sensible sizes, no dead resources.

None of it has any idea whether your skill is any good. A skill can pass validate and lint cleanly, sit in your registry for months, and never once get chosen by the agent. Being well-formed and being chosen have nothing to do with each other. Whether a skill gets chosen comes down to a couple of sentences of description, and no linter can tell you those sentences are wrong.

So run the tooling and let it own the questions it can answer. It will keep you honest about shape, size and dead files, which is more than most teams have. Just don't read a clean run as a working skill.

There's a second problem, and it's the one that actually stops people. Writing skills from scratch is slow, and an empty folder is a discouraging place to start. It's also mostly unnecessary. The runbooks, the `AGENTS.md` files and the Copilot instructions already sitting in your repositories are skills that haven't been given frontmatter yet, which is what `--from` is for.

Everything here is in `agentskills-tools` 0.5.0. The SDK is on [GitHub](https://github.com/pratikxpanda/agentskills-sdk).
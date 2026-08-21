---
title: "Agent Skills: Your system prompt is not a knowledge base"
seoTitle: "Agent Skills: Your system prompt is not a knowledge base"
seoDescription: "A system prompt that grew into a knowledge base is charged on every request. Agent Skills is the open format that fixes it."
datePublished: 2026-08-21T13:01:51.685Z
cuid: cmt2ykvz6000009jmgied276n
slug: agent-skills-not-a-knowledge-base
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1787311281491/388d6dc0-e0f3-4dd2-b8c1-009439691c4f.png
tags: ai, python, opensource, developer-tools, llm

---

*Part 1 of a series on the Agent Skills SDK, an open-source project I have been working on.*

Every agent I have built went through the same phase.

It starts clean. A short system prompt, a few tools, and it works. Then someone asks it something it should have known, so you add a paragraph. Then a runbook. Then the deployment conventions, because it keeps suggesting the wrong thing. Six weeks later the system prompt is 4,000 tokens, nobody remembers why half of it is there, and deleting anything feels risky.

At that point the prompt is a knowledge base. It is just a bad one.

Everything in it is sent on every single request, whether or not the question has anything to do with it. None of it is versioned or tested in any real way. And in my experience, it gets less reliable as it grows: a model reading four thousand tokens of mixed instructions follows them less carefully than one reading four hundred.

The instinct is to reach for retrieval. Put the documents in a vector database, fetch the relevant chunk, paste it in. That works well for reference material and badly for procedures, because chunking cuts a runbook in half and hands the model step four without steps one to three. Retrieval was designed to find passages that answer a question. Instructions are not passages.

What I actually wanted was simpler. Let the agent see a short list of what it knows how to do, let it pick one, and only then load the detail.

## The format already exists

That idea has a name and an open specification. It is called [Agent Skills](https://agentskills.io/specification). Anthropic developed it originally and released it as an open standard, and it is now supported by Claude Code, GitHub Copilot, VS Code, Cursor, Gemini CLI and ChatGPT's Codex, among [many others](https://agentskills.io/clients).

I want to be clear about what is mine and what is not, because it matters for what follows. **The format is not mine.** It is an open spec maintained upstream. What I built is an SDK for using it in your own agents.

A skill is a folder. The only required file is `SKILL.md`, which is markdown with frontmatter:

```markdown
---
name: incident-response
description: Standard operating procedures for production incident management including severity classification, escalation paths, communication protocols, and postmortem processes.
version: "1.0.0"
when_to_use:
  - A production service is degraded, down, or paging
  - An incident needs a severity, a commander, or a status update
  - A resolved incident needs a postmortem
when_not_to_use:
  - Debugging a failing test or a local development environment
  - Planning capacity or reviewing architecture ahead of a launch
---

# Incident Response

## When to Declare an Incident

An incident should be declared when:

- A production service is degraded or unavailable for users
- Data integrity may be compromised
- A security breach is suspected
```

Only `name` and `description` are required. Next to `SKILL.md` you can put three optional folders: `references/` for long detail, `scripts/` for code, and `assets/` for everything else.

The important part is how it gets loaded. The agent is shown the name and description of every skill. That is all. When it decides a skill is relevant, it fetches the body. If the body points at a reference file, it fetches that separately.

This is called progressive disclosure, and it is the whole idea. You pay a little to advertise a skill, and you pay for the detail only when the detail is wanted.

## What the format does not give you

A specification tells you what a valid skill looks like. It does not read your skills off disk, or out of blob storage, or from an internal service. It does not check that the file someone just committed is valid. And it does not hand any of it to your agent, which is the part that actually matters, because every framework wants something different.

That gap is what the SDK fills.

```bash
pip install agentskills-fs agentskills-langchain
```

Two packages for a LangChain app reading skills from a folder. Then:

```python
import asyncio
from pathlib import Path
from agentskills_core import SkillRegistry
from agentskills_fs import LocalFileSystemSkillProvider

async def main():
    registry = SkillRegistry()

    # Registers every skill folder under my-skills/, no IDs to hard-code.
    await registry.register_all(LocalFileSystemSkillProvider(Path("my-skills")))

    # What the agent sees on every turn: names and descriptions. Nothing more.
    print(await registry.get_skills_catalog(format="xml"))

    # What it fetches only after deciding the skill is relevant.
    skill = registry.get_skill("incident-response")
    print(await skill.get_body())

asyncio.run(main())
```

That is the entire mental model. A registry holds skills. A provider knows where they live. The catalog is cheap and always visible. The body is expensive and loaded on demand.

The catalog that prints is what your agent actually sees every turn:

```xml
<available_skills>
  <skill>
    <name>incident-response</name>
    <description>Standard operating procedures for production incident management including severity classification, escalation paths, communication protocols, and postmortem processes.</description>
    <version>1.0.0</version>
    <when_to_use>
      <case>A production service is degraded, down, or paging</case>
      <case>An incident needs a severity, a commander, or a status update</case>
      <case>A resolved incident needs a postmortem</case>
    </when_to_use>
    <when_not_to_use>
      <case>Debugging a failing test or a local development environment</case>
      <case>Planning capacity or reviewing architecture ahead of a launch</case>
    </when_not_to_use>
  </skill>
</available_skills>
```

A few hundred characters to advertise a skill whose body is several thousand. That ratio is the whole argument.

## The same registry, three frameworks

This is the part I care most about, and the reason the project exists in the shape it does.

Your skills should not belong to your agent framework. If you write fifteen skills for a LangChain app and then move to something else, rewriting them is a bad outcome that nothing about the situation required.

So, the registry is the thing you build, and the integrations are thin. Each one below takes the same `registry` from above and produces something you can actually run.

**LangChain.** You get tools, and you assemble the system prompt yourself:

```python
from langchain.agents import create_agent
from agentskills_langchain import get_tools, get_tools_usage_instructions

tools = get_tools(registry)
catalog = await registry.get_skills_catalog(format="xml")

system_prompt = (
    "You are an SRE assistant.\n\n"
    f"{catalog}\n\n"
    f"{get_tools_usage_instructions()}"
)

agent = create_agent(llm, tools, system_prompt=system_prompt)
```

Two separate things are happening, and it is worth seeing them apart. `get_tools` gives the agent the ability to fetch skill content. The catalog string is what makes it aware there is anything to fetch. Forget the second and you get an agent with working tools it never thinks to call — which is the single most common way this goes wrong.

**Microsoft Agent Framework.** The same two things, assembled for you:

```python
from agent_framework import Agent
from agentskills_agentframework import AgentSkillsContextProvider

agent = Agent(
    client=client,
    name="SREAssistant",
    instructions="You are an SRE assistant.",
    context_providers=[AgentSkillsContextProvider(registry)],
)
```

No `tools=` list and no prompt concatenation. The context provider injects the catalog and registers the tools on `before_run()`. That is less code and less to get wrong, at the cost of the catalog no longer being a string you can print and inspect. Which of those you want depends on how much you are still debugging.

**MCP.** No agent at all — the skills become a server:

```python
from agentskills_mcp_server import create_mcp_server

server = create_mcp_server(registry, name="My Skills Server")
server.run()  # stdio; run(transport="streamable-http") to serve over HTTP
```

This one is different in kind rather than degree. The first two put skills inside your Python process. This puts a process boundary in the middle, so Claude Desktop, or an agent in another language, or three teams who share nothing but a network, all read the same skills without importing any of this.

Same `registry` object in all three. The skills do not change. Nothing about the folder on disk knows or cares which framework is reading it.

Providers work the same way. `LocalFileSystemSkillProvider` reads a directory. `HTTPStaticFileSkillProvider` reads an endpoint. If your skills live somewhere neither of those covers, the provider interface is five methods.

## What it deliberately does not do

Two things worth stating plainly, because they set expectations.

**The SDK never executes scripts.** A skill can carry code in `scripts/`, and the SDK will retrieve it for the agent. It will not run it. Deciding what is safe to execute is not something a library can do on your behalf.

**It does not invent extensions to the skill format.** The project is spec-first. There is exactly one field the SDK adds today, an optional `version`, because pinning a skill and detecting drift are impossible without one. It is documented as an addition rather than quietly presented as part of the spec, and the intent is to raise it upstream rather than keep it as a private extension.

## Where this actually is

Version 0.5.0, ten packages, Python 3.12 and up, on [GitHub](https://github.com/pratikxpanda/agentskills-sdk) under the MIT License. I maintain it.

It is also genuinely used, which is a lower bar than "production ready" and a higher one than "weekend experiment." The honest state is that the core is stable, the surface is still moving, and the thing I most want is people telling me what broke.

If you want to try it, the smallest useful experiment is this. Take one runbook that is currently a paragraph in your system prompt, put it in a folder as `SKILL.md`, register it, and delete the paragraph. Ten minutes, and your prompt gets shorter without the agent getting worse.

That is the easy half, and it really is ten minutes. The harder half is deciding what goes in the folder, which turns out to have surprisingly little to do with how well you write documentation.
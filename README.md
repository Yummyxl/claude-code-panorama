# Claude Code Local Runtime Walkthrough

Chinese overview: [README-zh.md](./README-zh.md)

## Start With The Right Question

If you look at Claude Code from the outside, it is easy to collapse it into one of these:

- a smart terminal chat app
- a tool-calling shell wrapper
- a pile of prompts, commands, and extensions

All three are partially true. None of them is the right mental model.

From the local installed package, the more useful description is:

**Claude Code is a local agent runtime built to keep a model working over time.**

That phrasing matters because it separates two jobs that people often blur together:

- the model decides
- the runtime gives the model a world in which those decisions can become actions

That world includes:

- tools for reading, editing, and writing code
- a shell execution layer
- prompt assembly instead of one giant fixed prompt
- context management for long sessions
- task and todo state
- subagents and team coordination
- worktree isolation
- MCP, plugins, and remote extensions
- product layers such as auth, voice, and telemetry

Claude Code is not trying to out-think the model.  
It is trying to make the model operational.

---

## What This Repository Tries To Teach

This repository is not mainly about feature usage.

It is about mechanism.

If you only learn that Claude Code can:

- read files
- run shell commands
- edit code
- spawn agents
- connect to MCP

then you learned the product surface, not the architecture.

The more valuable questions are:

- why prompt is assembled from sections instead of kept as one blob
- why tool schemas are not enough without tool policy
- why compaction and memory are different systems
- why teams need protocols instead of “just talk to each other”
- why worktree is infrastructure rather than convenience
- why tool results must flow back into message history

This repository is written around those questions.

---

## Scope And Evidence

This project is strict about evidence.

It analyzes the local installed package only. The main artifact is:

- `/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js`

Supporting evidence comes from the same local package:

- `/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/sdk-tools.d.ts`
- `/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/package.json`
- `/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/vendor/`

These are machine-local paths, so they are rendered as code references instead of repository links.

That means the working standard for every chapter is:

**either a claim is directly visible in local code, or it is a constrained inference from local code.**

This repository is not built by diffing public repos or copying product marketing language.

---

## The Smallest Core

If you strip away all advanced features, Claude Code still reduces to one loop:

```python
def agent_loop(messages):
    while True:
        response = model(messages, tools, system_prompt)
        messages.append(response.content)

        if response.stop_reason != "tool_use":
            return

        results = []
        for block in response.content:
            if block.type == "tool_use":
                results.append(run_tool(block))

        messages.append(tool_results_to_message(results))
```

This is the smallest machine.

Everything else exists because a real coding agent cannot survive on that loop alone.

Real work needs:

- prompt assembly
- structured tools
- permissions
- context control
- task state
- background execution
- subagents
- team protocols
- isolation
- extension runtimes

Claude Code is what that minimal loop becomes once you keep adding the machinery required for long, messy, real work.

---

## How To Study This Repository

Do not read it like API documentation.

Read it like you are disassembling a machine.

The best progression is:

### Layer 1: See The Shape Of The Machine

Start here:

1. [s00_runtime_map.md](./agents/s00_runtime_map.md)
2. [s01_agent_loop.md](./agents/s01_agent_loop.md)
3. [s02_tool_surface.md](./agents/s02_tool_surface.md)
4. [s03_system_prompt.md](./agents/s03_system_prompt.md)
5. [s_full.md](./agents/s_full.md)

After these chapters, you should be able to answer:

- what the main loop is
- why the tool surface is not just a bag of commands
- why prompt is runtime state
- why Claude Code is more than a CLI wrapper

### Layer 2: Learn How Long Work Survives

Then read:

6. [s04_compaction_and_memory.md](./agents/s04_compaction_and_memory.md)
7. [s05_task_todo_background.md](./agents/s05_task_todo_background.md)
8. [s11_memory_prompt_injection.md](./agents/s11_memory_prompt_injection.md)

These chapters explain:

- why compaction is not memory
- why task state is not UI decoration
- why background execution must be first-class

### Layer 3: Learn Delegation, Teams, And Isolation

Then read:

9. [s06_subagent_and_teams.md](./agents/s06_subagent_and_teams.md)
10. [s10_team_protocols.md](./agents/s10_team_protocols.md)
11. [s07_worktree_and_isolation.md](./agents/s07_worktree_and_isolation.md)

These chapters explain:

- the difference between delegation and organization
- why typed protocols matter
- why safe parallelism needs filesystem isolation

### Layer 4: Learn The Formal Interfaces

Finally read:

12. [s08_mcp_plugins_remote.md](./agents/s08_mcp_plugins_remote.md)
13. [s09_auth_voice_telemetry.md](./agents/s09_auth_voice_telemetry.md)
14. [s12_tools_catalog.md](./agents/s12_tools_catalog.md)
15. [s13_prompt_catalog.md](./agents/s13_prompt_catalog.md)
16. [s14_toolrunner_callgraph.md](./agents/s14_toolrunner_callgraph.md)
17. [s15_complete_rendered_prompts.md](./agents/s15_complete_rendered_prompts.md)

These answer:

- how the runtime exposes tools
- how prompt is assembled in detail
- how one turn actually flows from entry to result
- what the model really sees under different runtime conditions

---

## What Makes Claude Code Worth Studying

The strongest lesson in the local runtime is not that Claude Code has many features.

It is that those features are arranged as runtime mechanisms instead of product tricks.

For example:

- prompt is not treated as copywriting, but as state serialization
- tools are not random commands, but typed action interfaces
- memory is not the same thing as compaction
- task management is not only presentation, but control flow
- team collaboration is not left to vibe, but constrained by protocol
- worktree is not optional polish, but an execution boundary

That is why this repository keeps returning to the same claim:

**the most important thing to learn here is not what Claude Code can do, but how it keeps a model effective over long-running work.**

---

## Chapter Index

- [agents/s00_runtime_map.md](./agents/s00_runtime_map.md)
- [agents/s01_agent_loop.md](./agents/s01_agent_loop.md)
- [agents/s02_tool_surface.md](./agents/s02_tool_surface.md)
- [agents/s03_system_prompt.md](./agents/s03_system_prompt.md)
- [agents/s04_compaction_and_memory.md](./agents/s04_compaction_and_memory.md)
- [agents/s05_task_todo_background.md](./agents/s05_task_todo_background.md)
- [agents/s06_subagent_and_teams.md](./agents/s06_subagent_and_teams.md)
- [agents/s07_worktree_and_isolation.md](./agents/s07_worktree_and_isolation.md)
- [agents/s08_mcp_plugins_remote.md](./agents/s08_mcp_plugins_remote.md)
- [agents/s09_auth_voice_telemetry.md](./agents/s09_auth_voice_telemetry.md)
- [agents/s10_team_protocols.md](./agents/s10_team_protocols.md)
- [agents/s11_memory_prompt_injection.md](./agents/s11_memory_prompt_injection.md)
- [agents/s12_tools_catalog.md](./agents/s12_tools_catalog.md)
- [agents/s13_prompt_catalog.md](./agents/s13_prompt_catalog.md)
- [agents/s14_toolrunner_callgraph.md](./agents/s14_toolrunner_callgraph.md)
- [agents/s15_complete_rendered_prompts.md](./agents/s15_complete_rendered_prompts.md)
- [agents/s_full.md](./agents/s_full.md)

---

## Final Takeaway

If you learn Claude Code as “a product that can call tools,” you learn usage.

If you learn it as “a runtime that keeps a model working, coordinating, and surviving over time,” you learn system design.

This repository is trying to teach the second one.

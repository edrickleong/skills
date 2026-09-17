# Working in this repo

This repo is a Claude Code plugin that ships agent skills. It contains no
application code — the deliverable is prose that an agent reads at runtime.

A skill is a directory under `skills/` containing a `SKILL.md`. Claude Code
discovers them automatically; `.claude-plugin/plugin.json` carries metadata
only, and does not list the skills.

## Writing the frontmatter

```yaml
---
name: <matches the directory name>
description: <what it does, then when to use it>
allowed-tools: <optional, narrows what the skill may run>
---
```

The `description` is the only text the model sees when deciding whether to load
the skill. It has to carry both halves — what the skill does *and* the situation
that should trigger it — because a description that only describes capability
never fires. Write the trigger in the user's words ("the user has a big change
and wants it split"), not the implementation's.

## Writing the body

- Address the agent, not a human reader. Imperative, present tense.
- Put the judgement in `SKILL.md` and the lookup detail in `references/`. The
  body is loaded every time the skill fires; references are read on demand. A
  long `SKILL.md` is a tax on every invocation.
- Show the exact commands. An agent that has to infer flags will invent them.
- Say what to do when a step fails, not just when it succeeds.

## Versioning

Bump `version` in `.claude-plugin/plugin.json` when you change a skill's
behaviour. Installed plugins update from the marketplace, so the version is how
an install knows there is something new to take.

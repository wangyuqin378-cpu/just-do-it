# Just Do It

[简体中文](README.zh-CN.md) · [Workflow examples](EXAMPLES.md)

## What it is

An Agent Skill experiment that turns requests or meeting notes into a tracked work plan: identify what belongs to you, route each action, and keep a record of the outcome.

It contains workflow instructions and document templates, not a standalone app. **The original workflow relies on internal workplace services and identity configuration**, so this public repository cannot run it on its own.

## How to use it

Start by reading the workflow; no service access is needed for that:

1. Open [EXAMPLES.md](EXAMPLES.md) to see a request, its classification and the expected output.
2. Read [SKILL.md](SKILL.md) to check the required services, identity variables and routing rules.
3. Inspect the [task format](TODO_LIST_FORMAT.md) and [document templates](DOC_TEMPLATES.md) to understand the records it produces.
4. Before adapting it to another environment, review [fallback behavior](FALLBACKS.md) and supply equivalents for the unavailable integrations.

A simplified illustrative request:

```text
I need to finish the review document, arrange a follow-up meeting,
and ask a teammate for last week's data.
```

The workflow separates that into a document task, candidate meeting times and a message draft. Actions that need confirmation wait for it; each result or unresolved step goes back into the task record. This example describes the intended flow, not a completed external action.

There is no universal installation command: the repository does not provide the internal services or credentials. Use the examples as a workflow reference unless you have a compatible environment.

## Why this project exists

Meeting notes and scattered requests often mix personal actions, work owned by someone else and decisions that still need confirmation. A plain list can lose those distinctions.

This experiment explores how an agent can turn that input into follow-through: decide who owns an action, choose a suitable tool, and retain enough evidence to tell what actually happened.

[Routing extensions](ROUTE_EXTENSIONS.md) · [Shared task format](SHARE_LIST_FORMAT.md)

**License:** source is public; no open-source license has been granted.

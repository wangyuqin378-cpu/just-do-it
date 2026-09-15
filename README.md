# Just Do It

[简体中文](README.zh-CN.md) · [Workflow examples](EXAMPLES.md)

**An experiment in turning a request into a tracked, executable work plan.**

For exploring how an agent can capture a task, route it to the right workflow and keep a clear record of what happened. The repository contains skill instructions and document templates, not a standalone application.

**Stage:** workflow experiment with dependencies on internal workplace services and identity configuration. It is not a general-purpose, ready-to-install public tool. Reading the examples requires no service access; running the original workflow does.

## Start with the workflow

1. Read [EXAMPLES.md](EXAMPLES.md) for sample requests and the expected flow.
2. Read [SKILL.md](SKILL.md) for routing, prerequisites and execution behavior.
3. Use [TODO_LIST_FORMAT.md](TODO_LIST_FORMAT.md) and [DOC_TEMPLATES.md](DOC_TEMPLATES.md) to understand the resulting task records.
4. Review [FALLBACKS.md](FALLBACKS.md) before adapting unavailable integrations.

A typical flow is: **request → task record → confirmation where required → execution → result or follow-up**. It can depend on internal workspace CLIs, user identity variables and organization-specific services. This repository does not provide those services or credentials, so there is no generic installation command that completes the workflow.

## Contents

- [SKILL.md](SKILL.md): workflow entry point.
- [EXAMPLES.md](EXAMPLES.md): scenarios and examples.
- [ROUTE_EXTENSIONS.md](ROUTE_EXTENSIONS.md): additional routing rules.
- [SHARE_LIST_FORMAT.md](SHARE_LIST_FORMAT.md): shared task format.
- [TODO_LIST_FORMAT.md](TODO_LIST_FORMAT.md): personal task format.
- [DOC_TEMPLATES.md](DOC_TEMPLATES.md): output templates.
- [FALLBACKS.md](FALLBACKS.md): fallback behavior.

## License

The source is publicly visible. No open-source license has been granted in this repository.

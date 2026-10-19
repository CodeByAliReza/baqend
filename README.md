# baqend

A new Flutter project.

## Getting Started

This project is a starting point for a Flutter application.

A few resources to get you started if this is your first Flutter project:

- [Learn Flutter](https://docs.flutter.dev/get-started/learn-flutter)
- [Write your first Flutter app](https://docs.flutter.dev/get-started/codelab)
- [Flutter learning resources](https://docs.flutter.dev/reference/learning-resources)

For help getting started with Flutter development, view the
[online documentation](https://docs.flutter.dev/), which offers tutorials,
samples, guidance on mobile development, and a full API reference.

## Spec-Driven Development (OpenSpec + OpenCode)

This project uses the [intent-driven OpenSpec template](https://github.com/intent-driven-dev/intent-driven-template).

Changes move through these artifacts in order:

```text
proposal -> specs -> design -> adr -> tasks
```

- `proposal` captures why the change matters.
- `specs` describe observable behaviour with Gherkin-style scenarios.
- `design` explains the implementation approach and trade-offs.
- `adr` records durable architectural decisions.
- `tasks` turn the accepted intent, behaviour, design, and decisions into work.

The active schema is selected in `openspec/config.yaml` (`schema: intent-driven`), and the
schema itself is bundled at `openspec/schemas/intent-driven/`.

Commands from `.opencode/commands/`:

| Command | Purpose |
|---|---|
| `/opsx:new` | Start a new change, one artifact at a time |
| `/opsx:propose` | Generate all artifacts for a change in one step |
| `/opsx:continue` | Create the next artifact for the active change |
| `/opsx:explore` | Explore ideas before committing to a change |
| `/opsx:apply` | Implement the change's tasks |
| `/opsx:verify` | Verify implementation against the change artifacts |
| `/opsx:sync` | Sync delta specs into the main specs |
| `/opsx:archive` | Archive a completed change |
| `/opsx:bulk-apply` | Apply several active changes in isolated worktrees |
| `/opsx:bulk-archive` | Archive several completed changes |

For Flutter/Dart implementation work, follow `FLUTTER_GETX_AGENT_GUIDE.md`.

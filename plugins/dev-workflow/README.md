# dev-workflow-bundle

Development workflow bundle for Claude Code. Includes:

- **dev-workflow**: Guided development workflow skill
- **peer**: Peer engineer agent for plan review and code review
- **extract-rules**: Rule extraction skill for updating project rules

## Installation

### Bundle (all-in-one)

```bash
/plugin marketplace add hiroro-work/claude-plugins
/plugin install dev-workflow-bundle@hiropon-plugins
```

### Standalone (dev-workflow only)

Requires `peer` and `extract-rules` to be installed separately.

```bash
/plugin marketplace add hiroro-work/claude-plugins
/plugin install peer@hiropon-plugins
/plugin install extract-rules@hiropon-plugins
/plugin install dev-workflow@hiropon-plugins
```

## Usage

### Initial Setup

Run once per project to detect lint/format/test commands:

```bash
/dev-workflow --init
```

### Execute Workflow

```bash
/dev-workflow <task description>
```

## Workflow Phases

1. **Plan**: Create implementation plan in plan mode
2. **Plan Review**: Peer reviews the plan (with `.claude/rules/` reference)
3. **Plan Revision**: Fix valid concerns, re-review (max 3 iterations)
4. **Implement**: Execute the approved plan
5. **Quality Check**: Run lint/format/test commands
6. **Code Review**: Peer reviews the code changes (with `.claude/rules/` reference)
7. **Code Revision**: Fix valid concerns, re-test, re-review (max 3 iterations)
8. **Rules Update**: Extract rules from conversation

## Configuration

Settings are stored in `.claude/dev-workflow.local.md`:

```yaml
---
lint_command: "pnpm run lint:fix"
format_command: "pnpm run format"
test_command: "pnpm run test"
---
```

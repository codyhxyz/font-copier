# font-copier — Claude Code plugin

This repo is a **Claude Code plugin**. The rules below apply to every session that opens anywhere in this tree. For the full end-to-end workflow (scaffold → build → test → host → submit), invoke the `create-claude-plugin` skill.

## Directory invariants

- All component directories (`skills/`, `agents/`, `hooks/`, `monitors/`, `bin/`) live at the **plugin root**.
- **Only** `plugin.json` and `marketplace.json` live inside `.claude-plugin/`. Never put components there — installation will silently fail.

## Path rules

- Every hook command, MCP `command`/`args`/`env`, monitor command, and script reference uses `${CLAUDE_PLUGIN_ROOT}/...`.
- Never use absolute paths. Never use `..` traversal. Plugins are copied to `~/.claude/plugins/cache/` on install and both will break.
- For state that must survive plugin updates, use `${CLAUDE_PLUGIN_DATA}` instead.

## Naming rules

- Plugin `name` is **kebab-case** (lowercase + digits + hyphens only).
- Don't use unowned brand names (`anthropic-*`, `official-*`, `claude-*-official`).

## Manifest rules

- `version` lives in `plugin.json` only. Setting it in `marketplace.json` is silently ignored — `plugin.json` always wins.
- Bump `version` on every behavior change. Existing users won't see updates otherwise (plugins are cached).
- Keep `plugin.json` + `marketplace.json` in sync on `name`, `description`, `author`, `homepage`, `repository`, `license`, `keywords`.

## Skill description rule

Frontmatter `description` describes **when to use**, not **what it does**. Start with "Use when...". The skill body is where the workflow lives.

## Always-run commands

Before claiming the plugin works:

```bash
claude plugin validate .
```

## Pointer

Full walkthrough, reference docs, templates, and submission handoff live in the `create-claude-plugin` skill.

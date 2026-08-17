# Install this skill

**Skill:** `rook-<platform>-<data-source>` · **Author:** ROOK

> Replace `rook-org` below with the actual GitHub org once the `rook-skills` repo is created.

## Option A — `skills` CLI (recommended)

Installs into the correct directory for your AI tool automatically (Claude Code, Cursor, Codex, and 75+ others):

```bash
npx skills add rook-org/rook-skills --skill rook-<platform>-<data-source>
```

Target a specific agent and skip prompts:

```bash
npx skills add rook-org/rook-skills --skill rook-<platform>-<data-source> -a claude-code -y
```

## Option B — clone + copy (no Node.js required)

```bash
git clone --depth 1 https://github.com/rook-org/rook-skills

# Claude / Claude Code:
cp -r rook-skills/skills/rook-<platform>-<data-source> .claude/skills/

# Codex / Antigravity / Cursor:
cp -r rook-skills/skills/rook-<platform>-<data-source> .agents/skills/
```

Both options install the same files. Use global paths (`~/.claude/skills/`, `~/.cursor/skills/`, …) instead of the
project-local ones above if you want the skill available across all your projects.

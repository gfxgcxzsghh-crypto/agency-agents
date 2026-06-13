# CLAUDE.md

Guidance for AI assistants (Claude Code and others) working in this repository.

## What this repository is

**The Agency** (`agency-agents`) is a curated library of **specialized AI agent
personalities**. It is *not* a software application — there is no app to build,
serve, or deploy. The "source code" is a collection of Markdown agent
definitions, plus a set of Bash scripts that **convert** those definitions into
the formats required by various agentic coding tools (Claude Code, Cursor,
OpenCode, Aider, Windsurf, Gemini CLI, Qwen, Kimi, Codex, Antigravity,
OpenClaw, GitHub Copilot) and **install** them into the right config
directories.

The two things people do here:
1. **Add or improve an agent** — edit/create a `.md` file in a category directory.
2. **Regenerate/install integrations** — run `scripts/convert.sh` then `scripts/install.sh`.

## Repository layout

Agent definitions live in category directories at the repo root. Each `.md` file
(with YAML frontmatter) is one agent.

```
academic/  design/  engineering/  finance/  game-development/  gis/
marketing/  paid-media/  product/  project-management/  sales/  security/
spatial-computing/  specialized/  strategy/  support/  testing/
```

- `specialized/` is the largest catch-all bucket (~50+ agents that don't fit a named division).
- `game-development/` and `strategy/` contain **nested subdirectories** (e.g.
  `game-development/unity/`, `game-development/godot/`, `strategy/playbooks/`).
  Scripts discover agents with a recursive `find`, so nesting is fine.
- `strategy/` also holds **non-agent docs** (`QUICKSTART.md`, `EXECUTIVE-BRIEF.md`,
  `playbooks/`, `runbooks/`, `coordination/`). These have no frontmatter and are
  intentionally skipped by the conversion/lint tooling (it checks that line 1 is `---`).

Supporting files:

```
scripts/
  convert.sh                  # agent .md -> per-tool integration files
  install.sh                  # integration files -> local tool config dirs (interactive TUI)
  lib.sh                      # shared pure-bash helpers (frontmatter parse, slugify, TUI)
  lint-agents.sh              # validates frontmatter + structure (runs in CI)
  check-agent-originality.sh  # flags find-replace duplicate agents (runs in CI, needs python3)
  i18n/                       # Chinese localization tooling
integrations/<tool>/          # GENERATED output of convert.sh — do not hand-edit
examples/                     # multi-agent workflow walkthroughs
.github/workflows/lint-agents.yml  # CI: lint + originality on changed agent files
README.md                     # full agent roster (huge; the public catalog)
CONTRIBUTING.md               # contributor-facing agent design guide
```

## Agent file format (the core convention)

Every agent file is Markdown with a YAML frontmatter block. The linter
**requires** `name`, `description`, and `color`. The fuller template
(`emoji`, `vibe`, optional `services`, plus the recommended body sections) is in
`CONTRIBUTING.md`.

```markdown
---
name: Rapid Prototyper
description: One-line description of the agent's specialty and focus
color: green                 # named color or "#hexcode" (required)
emoji: ⚡                     # optional, used by some integrations
vibe: One-line personality hook
services:                    # optional — only if the agent needs external services
  - name: Service Name
    url: https://example.com
    tier: free               # free | freemium | paid
---

# Agent Name

## 🧠 Your Identity & Memory
## 🎯 Your Core Mission
## 🚨 Critical Rules You Must Follow
## 📋 Your Technical Deliverables
## 🔄 Your Workflow Process
```

Conventions to follow when authoring agents:

- **Frontmatter fields** are parsed by simple `awk` (`get_field` in `lib.sh`),
  which matches `^<field>: ` at the start of a line in the frontmatter block.
  Keep one field per line, `key: value`, no fancy YAML.
- **Section headers drive conversion.** The OpenClaw converter and the linter
  classify `##` headers into two buckets:
  - **SOUL** (persona) — headers matching `identity`, `learning & memory`,
    `communication`, `style`, `critical rule`, `rules you must follow`.
  - **AGENTS** (operations) — everything else (mission, deliverables, workflow…).
  A good agent has at least one header in each bucket (the linter warns otherwise).
- **The slug** is derived from `name` via `slugify` (lowercase, non-alphanumerics
  → `-`). This is the single source of truth for output filenames, so `name`
  must be unique across the whole roster.
- **Be genuinely original.** The originality check fails a PR if a new/changed
  agent is ≥40% shingle-similar to an existing one (proper nouns like
  country/platform names are neutralized first, so a find-replace re-skin still
  trips it). Existing-library baseline overlap is ~1.5%.

## Line endings & encoding

The repo is **LF-only** (`.gitattributes` enforces `eol=lf` for `.md/.yml/.yaml/.sh`).
The linter hard-fails on CRLF. If you somehow introduce `\r`, fix with
`perl -i -pe 's/\r$//' <file>`.

## Workflows

### Validate agents (do this before committing agent changes)

```bash
./scripts/lint-agents.sh                       # lint the whole roster
./scripts/lint-agents.sh path/to/agent.md ...  # lint specific files
./scripts/check-agent-originality.sh path/to/new-agent.md   # needs python3
```

`lint-agents.sh`: ERROR (exit 1) on missing frontmatter delimiters/fields or
CRLF; WARN (non-blocking) on missing recommended sections, short body, or a
header bucket with no matches. CI mirrors this on changed files only.

### Regenerate integrations after editing agents

`integrations/**` is **generated output** — never edit it by hand; regenerate it.

```bash
./scripts/convert.sh                 # all tools
./scripts/convert.sh --tool cursor   # one tool
./scripts/convert.sh --parallel      # faster (independent tools run concurrently)
```

Output goes to `integrations/<tool>/`. Single-file tools (Aider → `CONVENTIONS.md`,
Windsurf → `.windsurfrules`) accumulate all agents into one file.

### Install into local tool config

```bash
./scripts/install.sh                                  # interactive wizard (TTY)
./scripts/install.sh --tool claude-code               # Claude Code -> ~/.claude/agents/
./scripts/install.sh --tool cursor --agent frontend-developer,ui-designer
./scripts/install.sh --tool opencode --division engineering,security --dry-run
./scripts/install.sh --list teams
```

`install.sh` reads from `integrations/` (run `convert.sh` first if it's missing
or stale) and copies into per-tool config dirs. `convert.sh` never touches user
config dirs; `install.sh` is the only script that writes outside the repo.

### Keep the directory lists in sync

The list of agent directories is duplicated in several places and they are **not
all identical** — when adding a new category directory, update every relevant one:

- `scripts/convert.sh` → `AGENT_DIRS` (currently includes `strategy`)
- `scripts/lint-agents.sh` → `AGENT_DIRS`
- `scripts/check-agent-originality.sh` → `AGENT_DIRS` (Python list)
- `.github/workflows/lint-agents.yml` → both `paths:` and the `git diff` globs

Also add the agent to the roster table in `README.md` when contributing publicly.

## CI

`.github/workflows/lint-agents.yml` runs on pull requests that touch agent
directories. It diffs changed `.md` agent files against the base branch and runs
`lint-agents.sh` then `check-agent-originality.sh` on them. A PR with a frontmatter
error or a duplicate agent will fail.

## Conventions for an AI assistant operating here

- This is a **content + shell-tooling** repo. There is no build/test suite beyond
  the lint + originality scripts. "Tests pass" here means those two scripts pass.
- When you add or edit an agent: lint it, run the originality check, and (if the
  change is meant to ship to integrations) regenerate with `convert.sh`. Don't
  commit hand-edited files under `integrations/`.
- Match the surrounding style: emoji-prefixed `##` section headers, second-person
  voice ("You are…", "Your Core Mission"), concrete deliverables with code blocks.
- Bash scripts target **Bash 3.2+** (macOS ships 3.2) and run under
  `set -euo pipefail`. Use the helpers in `lib.sh` (`get_field`, `get_body`,
  `slugify`, `incr`, …) rather than re-implementing them. No non-portable bashisms.
- Keep new agents genuinely distinct — localization variants must differ in
  substance (platforms, tactics, examples), not just swapped proper nouns.

## Git / PR workflow for this environment

- Develop on the designated feature branch; never push to `main` without explicit
  permission.
- Push with `git push -u origin <branch>`, then open a **draft** PR if none exists.
- The upstream project is `msitarzewski/agency-agents`; this fork's scope is
  `gfxgcxzsghh-crypto/agency-agents`. Use the GitHub MCP tools for PR/issue work.

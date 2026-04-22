---
name: Sync Conductor Skills
description: Detect new conductor releases and convert TOML commands to agent-agnostic SKILL.md format

on:
  workflow_dispatch:

permissions:
  contents: read
  issues: read
  pull-requests: read

tracker-id: sync-conductor
engine: 
  id: copilot
  model: gpt-5.4
strict: true

tools:
  github:
    toolsets: [default]
  edit:
  bash: ["git", "gh", "curl", "jq", "cat", "ls", "find", "diff", "mkdir", "cp", "rm", "rsync"]

network:
  allowed:
    - defaults
    - github

safe-outputs:
  create-pull-request:
    title-prefix: "[sync] "
    labels: [automated-sync]
    max: 1
  noop:
    report-as-issue: false

timeout-minutes: 45
---

# Sync Conductor Upstream

You are an AI agent responsible for keeping this repository's skills in sync with the upstream [gemini-cli-extensions/conductor](https://github.com/gemini-cli-extensions/conductor) project. You must perform all work yourself using only the tools provided.

## Step 1: Check for Updates

1. Get the latest version from the upstream conductor repo by reading their `.release-please-manifest.json` file (use the GitHub API) — the version is at the `"."` key.
2. Resolve the exact upstream tag name for that version by listing tags in the upstream repo. Look for an exact match on `conductor-v<version>` first, then fall back to `v<version>`. If neither exists, fail loudly.
3. Get the latest version tag in THIS repo by running `git tag --sort=-v:refname | head -1`. If no tags exist, treat the current version as `0.0.0`.
4. Compare versions. If they match, report "Already up to date" and stop.

## Step 2: Discover Upstream Content

Fetch ALL content from the **resolved upstream tag ref** (not `main`):

1. Browse the upstream conductor repo's `commands/conductor/` directory and discover ALL `.toml` command files present. Do not assume a fixed list — new commands may be added or removed in any release.
2. Browse and fetch the upstream `templates/` directory and all its contents (workflow templates, code style guides).
3. Fetch `GEMINI.md` and the `skills/` directory contents as read-only reference material for conversion context.
4. Read ALL existing SKILL.md files in this repo's `skills/` directories to understand the current state.

## Step 3: Sync Templates

Copy the entire upstream `templates/` directory into this repo's root `templates/` directory. Replace any existing content. This is a direct mirror — no modifications. Copy templates exactly as a direct mirror of upstream and verify with a diff. Do not rewrite template content through any text processing that could alter encoding, whitespace, or special characters.

## Step 4: Convert Skills

For EVERY `.toml` command discovered upstream, rewrite a corresponding `skills/conductor-<name>/SKILL.md`. The name should be the TOML filename converted to kebab-case (e.g., `newTrack.toml` → `conductor-new-track`). If a skill directory already exists, overwrite it. If the command is new, create the directory.

If any commands were removed upstream, delete those skill directories from this repo.

### Conversion Rules

Each SKILL.md is the upstream TOML's `prompt` field, converted from a Gemini CLI extension to a portable Vercel skill.

**Faithful means preserving upstream Conductor as a near-exact replica in structure, sequencing, and behavioral intent.** Preserve step order, branching logic, approval gates, question text, answer options, MUST/HALT behavior, referenced files/artifacts, and user-visible intent. Do not add opinions, enhancements, safety defaults, or behavioral modifications that are not in the upstream source.

**CRITICAL — Item-by-Item Preservation:** Preserve the full structural outline of the upstream prompt, not just top-level phases. Every heading, numbered item, lettered sub-item, bullet, gate, branch, and conditional step in the upstream prompt must appear as a distinct corresponding item in the output, in the same order. Do not merge a source item into surrounding prose or demote it into a generic note inside another step. If the upstream TOML's `prompt` field has N phases each with K steps, the output has the same N phases with the same K steps. Phases that USE Gemini-specific tools are NOT Gemini-specific phases — they are workflow phases whose TOOL REFERENCES must be rewritten, but the phase and every item within it MUST remain. When in doubt, keep it.

**CRITICAL — Conductor Skill System Phases:** The upstream conductor has its own skill ecosystem where users install conductor skills (review checklists, style guides, specialized checks) that extend conductor's behavior. Multiple commands contain steps about discovering, activating, selecting, recommending, installing, and reloading these conductor skills. These are CONDUCTOR features, not Gemini features — they describe conductor's plugin architecture. They MUST be preserved as distinct items with agent-agnostic phrasing. This applies to any step mentioning skills, extensions, catalogs, installation, activation, selection, recommendation, reload, or skill-specific checks — including but not limited to: "Select Skills" phases, "Skill Reload Confirmation" gates, "Activate Relevant Skills" steps (with full skill-matching and prioritization logic), "Check for Installed Skills" steps, "Skill-Specific Checks" steps, and "Skill Recommendation" steps. Rewrite Gemini-specific paths within these steps but keep every step, gate, and branching decision intact.

**Example of what NOT to do:** Do NOT replace an "Activate Relevant Skills" step (which contains sub-items for scanning skill directories, matching skills to the current task, reading skill instructions, and prioritizing guidance) with a single generic sentence like "Discover available skills." Each sub-item must remain as a distinct instruction in the output.

Apply only these changes:

1. **Add YAML frontmatter** with `name` (kebab-case) and `description` (from the TOML `description` field).

2. **Replace Gemini-runtime-specific constructs with agent-agnostic equivalents.** The upstream TOMLs are written for Gemini CLI. Only the SYNTAX changes, never the BEHAVIOR. A phase that uses a Gemini tool is still a real workflow phase — rewrite the tool reference, keep the phase. Specific mappings:

    - **`ask_user` tool with structured `questions:` arrays** → Rewrite as a natural-language instruction to prompt the user. Preserve the exact question text, header, answer type (yes/no, free text, or choice), all option labels and descriptions, `multiSelect` semantics, and placeholder text. Example: `ask_user` with `type: "choice"` and 3 options → "Prompt the user to choose from these options: [Option A — description, Option B — description, Option C — description]."
    - **`enter_plan_mode` / `exit_plan_mode`** → Rewrite as an agent-agnostic restricted-mode gate: "Enter a restricted planning mode where only file creation and modification within the `conductor/` directory is permitted. No destructive or out-of-scope operations." / "Exit restricted planning mode." KEEP the gate — do NOT delete it.
    - **`run_shell_command`** → "Run a shell command" or describe the action directly. Preserve any restrictions (e.g., no redirection, relative paths only).
    - **`write_file` / `replace`** → "Create file" / "Edit file" or equivalent neutral phrasing.
    - **`{{args}}`** → "the user-provided arguments" or "the arguments passed to this skill."
    - **Gemini path conventions** (`~/.gemini/extensions/conductor/templates/`, `.geminiignore`) → Replace with this repo's equivalent paths (`templates/`). If no equivalent exists, describe the concept generically.
    - **Skill/extension discovery and activation** (e.g., "check if relevant skills are installed", "list installed extensions") → Rewrite as "Discover which conductor skills are available" or equivalent agent-agnostic phrasing. KEEP the discovery phase and any branching based on its result.
    - **Skill recommendation and reload** (e.g., "recommend installing X", "reload extensions") → Rewrite as "Recommend the user install the missing skill" / "Reload available skills." KEEP the recommendation step.

    Do NOT replace one runtime-specific tool or schema with another. Rewrite in capability-based language that can be followed by any AI coding agent.

3. **Update internal references.** Any `/conductor:commandName` references should become `conductor-<kebab-name>` to match the skill naming convention. Any hardcoded paths like `~/.gemini/extensions/conductor/templates/` should reference this repo's `templates/` directory instead.

4. **Self-check before writing each SKILL.md.** List every heading and every numbered, lettered, and bulleted instruction from the upstream `prompt`. List every heading and every numbered, lettered, and bulleted instruction from your converted output. Compare them one-by-one. If ANY source item is missing, merged into another item, or demoted into prose, you have violated the preservation rule — go back and restore it before writing the file. Pay special attention to skill-related instructions (discovery, activation, selection, recommendation, installation, reload, skill-specific checks) — these are the most commonly dropped.

## Step 5: Clean Up

Delete any skill directories in this repo that no longer have a corresponding upstream TOML command.

## Step 6: Create Pull Request

Push the branch and create a pull request targeting `master` using the `create-pull-request` safe output.

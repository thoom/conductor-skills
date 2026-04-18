# conductor-skills

Agent-agnostic port of [Conductor](https://github.com/gemini-cli-extensions/conductor) — a spec-driven development framework for AI coding agents — packaged for the [Vercel Skills CLI](https://github.com/vercel/skills).

Conductor gives your AI agent a structured methodology: define specs, generate plans, implement in tracked phases, review against style guides, and maintain project documentation — all through conversational commands.

## Install

```bash
npx skills add thoom/conductor-skills
```

## Skills

| Skill | Description |
|---|---|
| `conductor-setup` | Scaffolds the project and sets up the Conductor environment |
| `conductor-new-track` | Plans a track, generates track-specific spec documents and updates the tracks file |
| `conductor-implement` | Executes the tasks defined in the specified track's plan |
| `conductor-review` | Reviews the completed track work against guidelines and the plan |
| `conductor-revert` | Reverts previous work |
| `conductor-status` | Displays the current progress of the project |

## Typical Workflow

1. **`conductor-setup`** — Initialize the project: generate a product definition, tech stack, style guides, and workflow
2. **`conductor-new-track`** — Create a new track (feature, bug fix, chore) with an interactive spec and implementation plan
3. **`conductor-implement`** — Execute the track's plan task-by-task following the project workflow
4. **`conductor-review`** — Review the implementation against the spec, style guides, and project standards
5. **`conductor-status`** — Check progress at any time

## Auto-Sync

This repo is an autonomous mirror of upstream Conductor. A daily [GitHub Agentic Workflow](.github/workflows/sync-conductor.md) detects new upstream releases, converts the TOML commands to agent-agnostic SKILL.md files, and opens a pull request. A companion [workflow](.github/workflows/auto-merge-sync.yml) merges the PR and tags the release to match the upstream version exactly.

No manual intervention required — when Conductor ships a new version, this repo updates itself.

## Upstream

- **Source:** [gemini-cli-extensions/conductor](https://github.com/gemini-cli-extensions/conductor)
- **Current version:** v0.4.1

## License

[MIT](LICENSE)

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code plugin (and single-plugin marketplace) with skills for Go backend microservices. There is no Go code here: the content is skill instructions in Markdown plus JSON manifests. See `README.md` for how services install it.

## Layout

- `.claude-plugin/plugin.json` — plugin manifest. Bump `version` (semver) with every change to skills, otherwise installed copies won't update.
- `.claude-plugin/marketplace.json` — marketplace manifest; the plugin `source` is `./` (the repo root). Keep `name`/`description` in sync with `plugin.json`.
- `skills/<skill-name>/SKILL.md` — one directory per skill; the directory name equals the `name` in frontmatter (kebab-case).
- `commands/<command-name>.md` — slash commands; frontmatter `description`, body is the prompt run on invocation.
- `agents/<agent-name>.md` — subagents; frontmatter `name`, `description`, `tools`, `model`.

Skills, agents and commands reference each other by name, never by file path. When renaming or removing one, grep for its name and fix every reference.

## Attribution

Much of the content is adapted from ECC (MIT). Skills derived from it keep `metadata: origin: ECC` in frontmatter; keep that line when editing them and keep `NOTICE`, `LICENSE-ECC` and `LICENSE-SUPABASE` in the repo.

## Writing skills

- Frontmatter needs `name` and `description`. The description is what makes Claude pick the skill, so state both what it does and when to use it.
- Keep `SKILL.md` focused on the workflow. Put shared Go conventions (project layout, error handling, testing) in one reference file and link to it from skills instead of repeating them in each skill.
- Explain why a rule exists rather than only stating it.
- Keep skills project-agnostic: no service names or business rules of a specific product. Product-specific conventions belong in that product's repos.

## Checks

Run `claude plugin validate .` after changing manifests or adding a skill.

# dxloop

A Claude skill for building features and fixing bugs in an existing codebase, with a fixed loop that agrees on "done" before any code is written:

1. **Definition of done + brain dump**: turn the request into testable outcomes
2. **Context gathering**: read rules, docs, ADRs, related code and tests
3. **Reproduce** (bugs only): a failing test, or a best-guess hypothesis to confirm
4. **Clarifying questions**: checkpoint; numbered, each with a default
5. **Own words**: restate the problem, goal, constraints and non-goals
6. **Plan + trade-offs**: checkpoint; approach, options, tests, risks
7. **Make changes**: small increments that follow the codebase's patterns
8. **Verify and prove**: tests before and after, plus a DoD checklist with evidence

Subagents are used only above a size threshold: for broad code searches, independent implementation chunks, noisy test runs, and fresh-eyes review of risky changes.

## Install

### Claude Code

```shell
/plugin marketplace add T-D-X/dxloop
/plugin install dxloop@dxloop
```

Or from your shell:

```shell
claude plugin marketplace add T-D-X/dxloop
claude plugin install dxloop@dxloop
```

The skill triggers automatically on feature and bug requests. To call it by hand, use `/dxloop:dxloop`.

### Claude.ai and the Claude desktop app

1. Download `dxloop.zip` from the [latest release](https://github.com/T-D-X/dxloop/releases/latest).
2. In Claude, go to **Customize > Skills**, click **+**, then **+ Create skill > Upload a skill**.
3. Upload the zip.

### Manual (Claude Code)

Copy `skills/dxloop/` into `~/.claude/skills/` (for you everywhere) or `.claude/skills/` (for one project).

## Updating

Claude Code: `/plugin marketplace update dxloop`. Claude.ai: download the new zip from the latest release and upload it again.

## Repo layout

```
.claude-plugin/
  marketplace.json   # makes this repo an installable marketplace
  plugin.json        # plugin manifest (bump "version" on each release)
skills/dxloop/
  SKILL.md           # the skill
```

## Releasing a new version

1. Edit `skills/dxloop/SKILL.md`.
2. Bump `version` in `.claude-plugin/plugin.json`.
3. Rebuild the zip: `cd skills && zip -r ../dist/dxloop.zip dxloop && cd ..`
4. Commit, tag (`git tag v1.x.x`), push, and attach `dist/dxloop.zip` to a GitHub Release.

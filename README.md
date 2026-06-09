# Ralph Agent Skills

Reusable skills and Codex agent templates for structured implementation, multi-agent code review, review/fix loops, and workflow tuning.

[![skills.sh](https://skills.sh/b/krajcik/ralph-agent-skills)](https://skills.sh/krajcik/ralph-agent-skills)

## Skills

### `ralph-flow`

Runs a coordinated delivery workflow:

- structured plan review
- sequential plan-item implementation
- multi-agent review fanout
- review/fix loops
- language-specific review packs, currently Go
- project context from `.agents/ralph-flow.md`

### `ralph-tune`

Maintains the Ralph workflow:

- tunes `ralph-flow`
- tunes reviewer/fixer/plan agent templates
- adds language review packs
- creates and updates project-local `.agents/ralph-flow.md`
- learns from review comments and stores the lesson in the right place

## Install

Install both skills for Codex:

```bash
npx skills add krajcik/ralph-agent-skills --skill ralph-flow --skill ralph-tune -g -a codex
```

Install from a local checkout:

```bash
npx skills add . --skill ralph-flow --skill ralph-tune -g -a codex
```

List discovered skills before installing:

```bash
npx skills add . --list
```

## Codex Agents

`npx skills` installs skills, not Codex subagent TOML files. This repo bundles Codex agent templates in two places:

- `agent-templates/codex/*.toml`
- `skills/ralph-tune/references/codex-agents/*.toml`

After installing `ralph-tune`, ask Codex:

```text
Use ralph-tune install-agents for Codex.
```

`ralph-tune` will list target files under `~/.codex/agents/`, ask before overwriting, copy approved templates, and validate TOML syntax.

## Project Context

For each repository, `ralph-flow` can read project-local context from:

```text
.agents/ralph-flow.md
```

Create it by asking:

```text
Use ralph-tune init-project.
```

This file should contain only project-specific facts:

- project invariants
- architecture boundaries
- review priorities
- test commands
- environment constraints
- external API contracts
- known reviewer preferences
- do-not-change constraints

Do not store secrets, credentials, or global workflow instructions there.

## Package Layout

```text
skills/
  ralph-flow/
    SKILL.md
    references/languages/go-review.md
  ralph-tune/
    SKILL.md
    references/project-context-template.md
    references/codex-agents/*.toml
agent-templates/
  codex/*.toml
skills.sh.json
```

## Validate

```bash
npx skills add . --list
npx skills add . --skill ralph-flow --skill ralph-tune --copy -y
python3 -c 'import pathlib, tomllib; [tomllib.loads(p.read_text()) for p in pathlib.Path("agent-templates/codex").glob("*.toml")]'
rg -n -i 'BEGIN (RSA|OPENSSH|PRIVATE)|AKIA[0-9A-Z]{16}|xox[baprs]-|password\s*=|token\s*=|secret\s*=|credential\s*=' .
```

## Publishing

1. Push this repository to GitHub.
2. Install once with `npx skills add krajcik/ralph-agent-skills --list` so skills.sh can discover it.
3. Optionally customize the skills.sh page with `skills.sh.json`.

## Acknowledgements

Ralph Agent Skills was inspired by workflow and review-agent ideas from
[`umputun/ralphex`](https://github.com/umputun/ralphex) and
[`umputun/cc-thingz`](https://github.com/umputun/cc-thingz). Some review-agent
checklist structure and prompt ideas are adapted from those projects.

This project is independent and is not affiliated with or endorsed by Umputun.

## License

MIT

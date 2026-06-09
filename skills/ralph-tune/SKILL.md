---
name: ralph-tune
description: Tune ralph-flow, bundled agents, language packs, and project context.
---

# Ralph Tune

## Purpose
Maintain and improve the `ralph-flow` delivery system.

This skill is the control plane for:

- global `ralph-flow` workflow rules
- reviewer, fixer, and plan agents used by `ralph-flow`
- language review packs used by dynamic language reviewers
- per-project `.agents/ralph-flow.md` context files

Keep responsibilities separate:

- Global workflow behavior -> `ralph-flow/SKILL.md`
- Role/persona behavior -> global agent TOML files
- Language-specific review rules -> `ralph-flow/references/languages/<language>-review.md`
- Project-specific facts and preferences -> repository `.agents/ralph-flow.md`

## Language
Match the user's language by default.

Keep file paths, commands, logs, identifiers, and config snippets unchanged.

## Modes
Choose the mode from the user request.

- `audit`: inspect current `ralph-flow`, agents, language packs, and project context; report mismatches; do not edit.
- `init-project`: create `.agents/ralph-flow.md` for the current repository from the template.
- `update-project`: update only the current repository `.agents/ralph-flow.md`.
- `learn-from-review`: classify PR review comments into reusable lessons, then update the correct global or project artifact.
- `tune-flow`: update the global `ralph-flow/SKILL.md`.
- `tune-agent`: update one or more global agents used by `ralph-flow`.
- `add-language`: add or update a language review pack and language routing in `ralph-flow`.
- `tune-self`: update this `ralph-tune` skill.
- `install-agents`: copy bundled Codex agent templates into `~/.codex/agents/` after explicit user approval.

If unclear, use `audit` and ask before mutating.

## Canonical Paths
Resolve paths dynamically from the installed skill location and current repository.

- `ralph-flow` skill: `<installed-skill-root>/../ralph-flow/SKILL.md`
- language packs: `<installed-skill-root>/../ralph-flow/references/languages/`
- Codex agent templates: `<package-root>/agent-templates/codex/*.toml` or `references/codex-agents/*.toml`
- Codex global agents: `~/.codex/agents/*.toml`
- this skill: `<installed-skill-root>/SKILL.md`
- project context: `<repo>/.agents/ralph-flow.md`
- project context template: `references/project-context-template.md`

Known `ralph-flow` agents:

- `quality`
- `implementation`
- `testing`
- `simplification`
- `documentation`
- `smells`
- `fixer`
- `plan_review`
- `plan_implementer`

## Safety Rules
Before any non-trivial mutation, provide:

1. affected files
2. current behavior
3. intended behavior
4. invariant boundaries
5. minimal change plan
6. validation checks

Do not edit until the plan is clear, unless the user explicitly gave a narrow edit request.

Never put these into `.agents/ralph-flow.md` or global skills:

- secrets, tokens, credentials, private keys
- personal data
- production-only values that should not be persisted
- temporary debugging notes
- one-off opinions that are not reusable

Do not post to external systems. Do not commit unless explicitly asked.

Use small patches. Preserve unrelated content and manual edits.

## Artifact Ownership
Classify every requested change before editing.

| Change type | Target |
| --- | --- |
| Workflow orchestration, modes, fanout, lifecycle, output format | `ralph-flow/SKILL.md` |
| Reviewer/fixer persona, severity policy, role-specific checklist | `~/.codex/agents/<agent>.toml` or bundled `agent-templates/codex/<agent>.toml` |
| Language idioms, language checklist, language skill routing | `ralph-flow/references/languages/<language>-review.md` plus routing in `ralph-flow/SKILL.md` |
| Project invariants, repo commands, env constraints, local API contracts, reviewer preferences | `<repo>/.agents/ralph-flow.md` |
| Tuning workflow itself | `ralph-tune/SKILL.md` |

If a lesson belongs to multiple scopes, split it. Example:

- "Go external HTTP clients need timeout" -> Go language pack.
- "This repository treats build queue requests as asynchronous operations" -> project context.
- "Re-check must include language reviewers" -> `ralph-flow/SKILL.md`.
- "`quality` misses low-risk hygiene" -> `quality.toml` or `go-review` pack, depending on scope.

## Project Context Rules
`.agents/ralph-flow.md` must contain only project context.

Allowed:

- project invariants
- architecture boundaries
- review priorities
- test and validation commands
- environment constraints
- external API contracts used by this repo
- known reviewer preferences
- "do not change" constraints

Not allowed:

- generic `ralph-flow` workflow rules
- agent prompt instructions
- language-wide best practices
- instructions that belong in global skills
- secrets or credentials

When creating the file, use `references/project-context-template.md`.

When updating it:

1. Read the current file first.
2. Update only relevant sections.
3. Keep entries factual and scoped to the project.
4. Avoid duplicating global rules.
5. Add dates or PR references only when they help future maintainers.

## Learn From Review
Use this mode when the user gives PR comments or asks to learn from review feedback.

Flow:

1. Collect comments and linked code context if available.
2. For each comment, decide:
   - real bug
   - maintainability/hygiene
   - reviewer preference
   - false positive
   - project-specific convention
   - language-specific convention
   - workflow/agent gap
3. Classify target artifact using the ownership table.
4. Propose a compact tuning plan.
5. Apply approved updates.
6. Report what changed and what future run should catch.

Default classification:

- Reusable across Go repos -> Go review pack or targeted Go skill routing.
- Reusable across all languages -> `ralph-flow/SKILL.md` or role agent.
- Specific to the current repo -> `.agents/ralph-flow.md`.
- Specific to one reviewer but likely recurring in this repo -> `.agents/ralph-flow.md` under reviewer preferences.
- One-off or unsupported -> do not encode; mention as not persisted.

## Tuning Global Agents
Agent files are TOML. Read before editing.

Use `tune-agent` when the change affects a role:

- `quality`: correctness, security, error handling, reliability.
- `implementation`: requirement fit, data flow, contracts, integration.
- `testing`: coverage, test quality, missing regression tests.
- `simplification`: unnecessary complexity, abstraction, scope creep.
- `documentation`: doc drift, public docs, comments, changelogs.
- `smells`: code smells, maintainability, local hygiene.
- `fixer`: verification, safe fixes, validation discipline.
- `plan_review`: plan quality before implementation.
- `plan_implementer`: plan-item execution discipline.

Keep agent instructions focused. Do not paste whole language packs into role agents. Prefer language reviewers for language-specific details.

After editing agent TOML, validate syntax when practical:

```bash
python3 -c 'import pathlib, tomllib; [tomllib.loads(p.read_text()) for p in pathlib.Path.home().joinpath(".codex/agents").glob("*.toml")]'
```

## Installing Codex Agents
Use `install-agents` only when the user explicitly asks to install or sync bundled Codex agents.

Flow:

1. Locate bundled templates under `agent-templates/codex/` first, then `references/codex-agents/`.
2. List templates and target files under `~/.codex/agents/`.
3. If target files already exist, show which files would be overwritten.
4. Ask for explicit approval before overwriting existing files.
5. Copy only approved TOML files.
6. Validate TOML syntax.
7. Report exact files changed.

Do not install agents automatically during ordinary tuning or project-context updates.

## Adding A Language
Use `add-language` when a new language needs specific review.

Steps:

1. Add `ralph-flow/references/languages/<language>-review.md`.
2. Add detection rules to `ralph-flow/SKILL.md`.
3. Add dynamic language agent name, for example `<language>-review`.
4. Add targeted skill/library routing if available.
5. Keep the language pack compact and review-oriented.
6. Do not load the pack into every role agent; only the language reviewer gets it.

Language pack structure:

```md
# <Language> Review Pack

## Always Check
## Targeted Skill Routing
## Reporting
```

## Audit Checklist
In `audit` mode, inspect:

- `ralph-flow/SKILL.md` references `.agents/ralph-flow.md`.
- `ralph-flow/SKILL.md` defines language detection and language reviewers.
- Language packs exist for routed languages.
- Re-check rounds include selected language reviewers.
- Agent TOML files exist for known role agents and fixer/plan agents.
- Agent severity protocol matches `ralph-flow` output parsing.
- Project `.agents/ralph-flow.md` exists only if project context was initialized.
- Project context contains no global workflow rules or secrets.

Return findings grouped:

- `GLOBAL FLOW`
- `AGENTS`
- `LANGUAGE PACKS`
- `PROJECT CONTEXT`
- `RECOMMENDED CHANGES`

## Output Requirements
For mutation modes:

```text
Mode: <mode>
Affected files:
- ...

Plan:
- ...

Changes made:
- ...

Validation:
- command/result or "not run: reason"

Remaining tuning gaps:
- ...
```

For audit mode:

```text
Mode: audit
GLOBAL FLOW:
- ...
AGENTS:
- ...
LANGUAGE PACKS:
- ...
PROJECT CONTEXT:
- ...
RECOMMENDED CHANGES:
- ...
```

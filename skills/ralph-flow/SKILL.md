---
name: ralph-flow
description: Structured implementation, multi-agent code review, and fixer loops.
---

# Ralph Flow

## Purpose
Run a coordinated delivery workflow using Codex custom subagents:

- `quality`
- `implementation`
- `testing`
- `simplification`
- `documentation`
- `smells`

Default behavior for review requests: run all 6 role reviewer agents in parallel, add language-specific reviewer agents when matching language packs exist, wait for every selected agent, verify findings, deduplicate them, and return one consolidated review.

In fix mode, reviewers still stay read-only. The separate `fixer` agent verifies and fixes confirmed findings.

For plan implementation requests, review the plan first, implement plan items sequentially with fresh subagent contexts, then run the review/fix loop.

## Language
Match the user's language by default.

Keep code, file paths, commands, logs, error messages, identifiers, commit messages, and quoted findings in their original language.

## Tuning
Use the `ralph-tune` skill to update this workflow, reviewer/fixer agents, language review packs, and per-project `.agents/ralph-flow.md` context.

## Modes
Choose mode from the user request:

- `review-only`: default. Review and return consolidated findings. Do not edit files.
- `review-fix-loop`: if the user explicitly asks to fix, apply fixes, resolve findings, iterate until clean, or run a review/fix cycle.
- `plan-implement-review-fix`: if the user asks to implement code from a plan and then run the review/fix loop.

If unclear, use `review-only`.

## Plan Review Preflight
Use `plan_review` before implementation when:

- mode is `plan-implement-review-fix`
- user asks to implement by plan/spec/tasks/issue/checklist
- user provides a plan file and asks for fix-loop after implementation

Do not run `plan_review` for ordinary diff review unless the user explicitly asks.

Flow:

1. Spawn `plan_review` as read-only.
2. Wait for result and close it if `close_agent` is available.
3. If `plan_review` returns `CRITICAL` or `MAJOR`, stop before implementation and show plan findings.
4. If only `MINOR` findings exist, apply obvious plan clarifications only when the user asked to update the plan; otherwise continue.
5. If `NO FINDINGS`, implement the plan item-by-item using `plan_implementer`.
6. After all plan items are implemented, run `review-fix-loop`.

`plan_review` reviews any implementation plan, not only one specific plan format.

## Plan Item Implementation
In `plan-implement-review-fix` mode, implementation must be sequential by plan item and each item must use a fresh subagent context.

Flow:

1. Parse the plan into ordered implementation items.
2. For each item, spawn a new `plan_implementer` subagent.
3. Give that subagent only:
   - the plan path or relevant plan excerpt
   - the current item id/title
   - required constraints
   - known validation commands
4. Wait for the item result.
5. Close the subagent if `close_agent` is available.
6. Inspect the result and working tree.
7. If the item is blocked, stop and report the blocker.
8. If the item is done, continue to the next item with a new `plan_implementer`.

Never reuse one implementation subagent for multiple plan items.
Do not parallelize plan item implementation.
Do not let a `plan_implementer` implement future items.
After the final item, run the normal review/fix loop.

## Agent Selection
Default selected role agents:

`quality`, `implementation`, `testing`, `simplification`, `documentation`, `smells`

Skip an agent only when the user explicitly asks, for example:

- "skip docs" / "without documentation" -> skip `documentation`
- "skip tests" / "without testing review" -> skip `testing`
- "only quality and implementation" -> run only `quality`, `implementation`
- "no simplification" -> skip `simplification`
- "no smells" / "skip smells" -> skip `smells`

Do not infer skips from task type. Documentation, testing, simplification, and smells agents still run by default.

`simplification` and `smells` are mandatory hygiene reviewers in every review stage unless explicitly skipped. In `review-fix-loop` and `plan-implement-review-fix`, if they are selected, run them in the initial review, every post-fixer re-check, and every repeated review round. Do not limit them to Round 1.

Dynamic language agents:

- Detect changed languages before reviewer fanout.
- Add one language reviewer per detected language when this skill has a matching language pack.
- Currently supported: `go-review` for Go diffs.
- Do not replace role agents with language agents. Language agents are additive.
- Skip language agents only when the user explicitly asks, for example "skip language review", "skip go-review", or "without Go review".

## Review Target
Use the user's target if provided:

- branch vs branch
- PR
- staged changes
- working tree
- specific files
- plan/spec context

For PR targets, use a separate git worktree. Never switch the user's current checkout with `gh pr checkout`.

PR worktree flow:

1. Fetch without changing current branch: `git fetch origin pull/<number>/head:pr-<number>`.
2. Create review worktree: `git worktree add "/tmp/pr-review-<number>" pr-<number>`.
3. Run review/fix commands inside that worktree.
4. Clean up only after review is complete or cancelled:
   - `git worktree remove "/tmp/pr-review-<number>" --force`
   - `git branch -D pr-<number>`

If no target is provided, review the current branch against the repository default branch. Determine the default branch with local git information where possible:

1. `git symbolic-ref --short refs/remotes/origin/HEAD`
2. fallback to `main`
3. fallback to `master`

Use concise branch context:

- `git status --short`
- `git log <base>..HEAD --oneline`
- `git diff <base>...HEAD`

Do not paste large diffs into subagent prompts. Tell each subagent to inspect the diff itself.

## Review Context Preflight
Before spawning reviewer agents, build a compact context packet.

Changed files:

1. Use the resolved review target and base/scope.
2. Collect changed file names, preferably with `git diff --name-only <base>...HEAD` or the equivalent for the target.
3. Use file names plus focused diff inspection to detect languages and touched domains.

Project context:

1. Look for `.agents/ralph-flow.md` at the repository root.
2. If it exists, pass its path to every reviewer agent and to `fixer`.
3. Tell agents to read and apply it as project-specific review policy.
4. If it does not exist, omit the project-context block entirely. Do not create it.

Feature plan context:

1. If the initial user request supplied a plan/spec/tasks/issue/checklist path, or `plan-implement-review-fix` parsed one, preserve that path or concise excerpt.
2. Pass this feature plan context to every reviewer agent and to `fixer`.
3. Tell agents to verify the diff against the intended behavior from the plan.
4. If no plan context was provided or discovered during plan implementation, omit the block entirely.

Language detection:

- Go: `.go`, `go.mod`, `go.sum`, `go.work`, `vendor/modules.txt`.
- JavaScript/TypeScript: `.js`, `.jsx`, `.ts`, `.tsx`, `package.json`, lockfiles.
- Python: `.py`, `pyproject.toml`, `requirements*.txt`, `setup.py`, `setup.cfg`.
- Shell: `.sh`, executable scripts with shell shebangs.
- SQL: `.sql`, migration directories.
- YAML/JSON config: `.yaml`, `.yml`, `.json`, CI/deploy/config paths.

Language packs:

- Use language packs from `references/languages/<language>-review.md`.
- Add a language reviewer only when a pack exists.
- Pass full language-pack paths only to matching language reviewer agents, not to all role agents.
- Role agents receive only detected language names, project context, and feature plan context.

Go targeted skills:

- For `go-review`, always include `golang-code-style`, `golang-error-handling`, `golang-testing`, and `golang-safety` when available.
- Add targeted Go skills only when the diff touches matching code patterns:
  - `context.Context`, `WithTimeout`, `WithCancel`, `NewRequestWithContext`, HTTP clients -> `golang-context`
  - goroutines, channels, `select`, `sync`, atomics, `errgroup`, worker pools -> `golang-concurrency`
  - SQL, `database/sql`, `sqlx`, `pgx`, `pq`, migrations, transactions -> `golang-database`
  - auth, cookies, tokens, crypto, filesystem, `os/exec`, user input, network boundary -> `golang-security`
  - structs, interfaces, struct tags, type assertions, type switches -> `golang-structs-interfaces`
  - constants, enums, exported names, constructors, receiver names, error names -> `golang-naming`
  - `go.mod`, `go.sum`, `go.work`, `vendor/modules.txt` -> `golang-dependency-management`
  - `_test.go` changes or testify imports -> `golang-testing`, `golang-stretchr-testify`
  - logging, metrics, tracing, alerts -> `golang-observability`
  - CLI command code, `cmd/`, flags, Cobra, Viper -> `golang-cli`, plus `golang-spf13-cobra` or `golang-spf13-viper` when imported
  - gRPC, protobuf, `.proto` -> `golang-grpc`
  - `fx`, `dig`, `wire`, `samber/do` imports -> matching DI skill
  - benchmarks, pprof, performance claims -> `golang-benchmark`, `golang-performance`
- Do not load every `golang-*` skill for every Go diff. Keep language review targeted.

## Subagent Orchestration
Spawn all selected role agents and dynamic language agents in parallel in one turn. Do not serialize them.

## Subagent Lifecycle
Codex subagent slots can leak in persistent sessions if finished agents are only waited on.

Important lifecycle rule:

- `wait_agent` only observes final status.
- `wait_agent` does not guarantee that the subagent thread/slot is released.
- When the tool surface exposes `close_agent`, call `close_agent` for every spawned subagent after collecting its result.
- Close all agents from a review round before starting any fixer or next review round.
- If `close_agent` is unavailable in the current tool namespace, do not assume slots are released.

If `close_agent` is unavailable:

- `review-only` is still safe for one fanout round if selected agents fit under `agents.max_threads`.
- `review-fix-loop` must be conservative: after the first fanout round, avoid spawning another review fanout in the same persistent session unless enough thread capacity is clearly available.
- If a second round fails with `agent thread limit reached`, stop the loop, report the lifecycle limitation, and include the remaining findings and exact next command/prompt the user can run in a fresh session.
- Do not keep retrying spawn calls after `agent thread limit reached`; that burns time and tokens without freeing slots.

Each subagent prompt must include:

- review target
- base branch or diff scope
- project context path when `.agents/ralph-flow.md` exists
- feature plan context when one was supplied or parsed
- detected language names
- mode-specific instruction
- "Do not edit files" for review-only
- strict severity output format

Template:

```text
Review target: <target>
Base/scope: <base-or-scope>
Role: <agent-name>
Project context: <.agents/ralph-flow.md path or omitted>
Feature plan context: <plan path/excerpt or omitted>
Detected languages: <languages or "none">
Language pack: <matching pack path, language agents only>
Targeted language skills: <skill names/paths, language agents only>

Inspect the repository and diff yourself. Do not rely on pasted diffs.
If project context or feature plan context is present, read and apply it before reporting findings.
If you are a language reviewer, read and apply the language pack and targeted language skills before reporting findings.
Return problems only. No positive observations.
Every finding must be on its own line and start with CRITICAL:, MAJOR:, or MINOR:.
Format: SEVERITY: file:line — issue; impact/evidence; suggested fix.
If no findings, return NO FINDINGS.
Do not edit files.
```

For `review-fix-loop`, reviewer subagents still do not edit files. The `fixer` agent applies fixes after the main agent collects and verifies findings.

## Consolidation
After all selected agents finish:

1. Read every result.
2. Drop `NO FINDINGS`.
3. Drop findings without `CRITICAL:`, `MAJOR:`, or `MINOR:` unless the issue is obviously real; normalize only when severity is unambiguous.
4. Verify every reported finding against the actual code in review-only mode.
5. Remove false positives in review-only mode.
6. Deduplicate by same file/line/root cause.
7. Preserve source agent names for each confirmed finding.
8. Group by severity: `CRITICAL`, `MAJOR`, `MINOR`.

Review-only output order:

1. CRITICAL
2. MAJOR
3. MINOR
4. Agent summary: selected agents, skipped agents, clean agents

If no confirmed findings remain, say that clearly and mention selected agents.

## Review-Fix Loop
Use only when explicitly requested.

Loop:

1. Round 1: run selected reviewer subagents in parallel.
2. Consolidate findings without dropping possible false positives; preserve source agent, severity, file:line, and raw text.
3. Close every spawned review subagent if `close_agent` is available.
4. If no findings remain after parsing/deduplication, stop and report clean.
5. Spawn `fixer` with the full consolidated findings list.
6. Include project context and feature plan context in the fixer prompt when present.
7. Wait for `fixer`, collect `FIXES:` report, close `fixer` if `close_agent` is available.
8. Main agent inspects fixer changes and validation results.
9. Re-check rounds: run `quality`, `implementation`, `simplification`, `smells`, and selected language reviewers if they are selected, with instruction to report only `CRITICAL` and `MAJOR`. `simplification`, `smells`, and language reviewers must run in every re-check unless explicitly skipped.
10. If re-check is clean for critical/major findings, stop and report clean. Do not keep cycling on ordinary MINOR findings unless user explicitly asks. Treat MINOR findings about unnecessary production test seams, avoidable globals, needless factories, duplicate wrappers, or avoidable abstraction introduced by the fixer as diff-hygiene issues that the main agent should inspect and fix locally when safe.
11. If critical/major findings remain, send the consolidated list to `fixer` and repeat.

Fixer prompt requirements:

```text
You are the fixer agent.
Verify every finding against current code before fixing.
Classify each as CONFIRMED, FALSE POSITIVE, or UNSAFE TO FIX.
Fix confirmed CRITICAL findings.
Fix confirmed MAJOR findings when there is a clear safe fix.
Do not fix MINOR findings unless explicitly requested.
Run relevant formatters/tests/checks.
Do not commit.
Return a report starting with FIXES:.
```

Before repeating, confirm subagent lifecycle is safe:
   - If `close_agent` was available and all prior subagents were closed, run the re-check with `quality`, `implementation`, `simplification`, `smells`, and selected language reviewers if they are selected.
   - If `close_agent` was unavailable, prefer stopping after fixes and report that a fresh session should run the next review round.

Stop conditions:

- clean review: no confirmed findings
- `close_agent` unavailable after a fanout round in `review-fix-loop`
- `agent thread limit reached`
- same unresolved findings repeat twice without a safe fix
- tests/checks fail for a reason that cannot be fixed safely in scope
- user-specified iteration limit reached

Default iteration limit: 5 review/fix rounds.

When fixing:

- Keep diffs minimal.
- Do not change public APIs, storage formats, protocols, metrics, logs, or behavior unless required by a confirmed finding.
- Do not rewrite unrelated code.
- Do not commit unless the user explicitly asks.
- Preserve user changes in the working tree.
- Report exact commands run.

## Output Requirements
For review-only:

```text
Selected agents: ...
Selected language agents: ...
Skipped agents: ...
Project context: <path or none>
Feature plan context: <path/excerpt or none>

CRITICAL:
- file:line [agents] issue; impact/evidence; suggested fix

MAJOR:
- ...

MINOR:
- ...

Clean agents:
- ...
```

For review-fix-loop:

```text
Review/fix result: clean | stopped
Rounds: N
Selected agents: ...
Selected language agents: ...
Skipped agents: ...
Project context: <path or none>
Feature plan context: <path/excerpt or none>
Fixer runs: N
Changes made:
- ...
Verification:
- command -> result
Remaining findings:
- CRITICAL/MAJOR only unless user requested MINOR cleanup
```

For plan-implement-review-fix:

```text
Plan review: clean | stopped
Implementation: done | skipped
Review/fix result: clean | stopped
Rounds: N
Selected agents: ...
Selected language agents: ...
Skipped agents: ...
Project context: <path or none>
Feature plan context: <path/excerpt or none>
Fixer runs: N
Changes made:
- ...
Verification:
- command -> result
Remaining findings:
- ...
```

Keep the final answer concise. Do not include raw subagent transcripts unless the user asks.

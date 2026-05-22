# Codex + OpenCode/Kimi Bridge

A practical operating model for using Codex as the planner and reviewer while delegating bounded implementation work to OpenCode with Kimi K2.6.

The goal is to reduce expensive Codex token usage without giving up Codex's stronger planning and review judgment.

```text
Codex 5.5 high/xhigh: plan
OpenCode Kimi K2.6: implement bounded task
Codex 5.5 high/xhigh: review, verify, and tighten
```

## Status

This workflow has been smoke-tested with the OpenCode CLI.

Verified:

- OpenCode CLI can list `opencode-go/kimi-k2.6`.
- Kimi can implement a bounded file edit from a Codex-authored brief.
- Codex can review the resulting diff afterward.

Known issue:

- Some OpenCode/Kimi K2.6 runs can fail with `JSON Schema not supported: could not understand the instance {'default': 'latest'}`.
- This appears to be specific to the Kimi models behind `opencode-go`; other `opencode-go` models such as Qwen or DeepSeek may still work.
- Treat the smoke test below as the gate. If it fails, do not use Kimi for implementation until the provider/OpenCode compatibility issue is fixed.

The recommended path is OpenCode CLI delegation:

```sh
opencode run --model opencode-go/kimi-k2.6 "<handoff prompt>"
```

## Why Use This

Codex token use is highest when one session has to do every step:

- explore the repository
- infer implementation details
- edit files
- debug test failures
- review its own work

This bridge moves the implementation loop to Kimi while keeping Codex responsible for planning, scope control, risk analysis, and final review.

This reduces Codex token use when:

- Codex writes a compact implementation brief.
- Kimi reads the relevant code, edits files, and runs first-pass verification.
- Codex reviews only the final diff, changed files, test output, changelog entry, and Kimi's short summary.

It will not reduce much if:

- Codex still reads the whole codebase first.
- Kimi's full logs are pasted back into Codex.
- Codex reimplements the work after Kimi finishes.
- Handoffs are vague and require repeated clarification.

## Prerequisites

Install and configure OpenCode with access to Kimi K2.6.

Check OpenCode:

```sh
opencode --version
```

List OpenCode Go models:

```sh
opencode models opencode-go
```

You should see:

```text
opencode-go/kimi-k2.6
```

Run a smoke test:

```sh
opencode run --model opencode-go/kimi-k2.6 --format json "Reply with exactly: KIMI_OK"
```

Expected response:

```text
KIMI_OK
```

If the command returns a JSON Schema error, use a different OpenCode Go implementation model temporarily:

```sh
opencode run --model opencode-go/qwen3.6-plus --format json "Reply with exactly: QWEN_OK"
opencode run --model opencode-go/deepseek-v4-flash --format json "Reply with exactly: OK"
```

## Operating Model

### 1. Codex Plans

Use Codex 5.5 with high or xhigh reasoning to produce a small, explicit implementation brief.

The plan should include:

- goal
- current context
- allowed files
- forbidden files and systems
- exact behavior requirements
- verification commands
- changelog expectations
- expected return format

Codex should avoid broad implementation exploration unless needed to make the handoff safe.

### 2. Kimi Implements

Use OpenCode with Kimi K2.6 for bounded implementation.

```sh
opencode run --model opencode-go/kimi-k2.6 "<handoff prompt>"
```

Kimi should:

- inspect only what is needed
- edit only allowed files
- avoid unrelated refactors
- run requested verification
- update the changelog if requested
- return a concise summary

### 3. Codex Reviews

Codex reviews the resulting work.

Codex should inspect:

- `git diff`
- changed files
- test output
- changelog entry
- Kimi's summary and caveats

Codex should then:

- accept the implementation
- make small finishing edits
- ask Kimi for a narrow follow-up
- reject and replan if the implementation is structurally wrong

## Handoff Prompt Template

Use this shape when sending work from Codex to Kimi:

```text
Implement this bounded task.

Goal:
<one paragraph>

Context:
<short project-specific context>

Allowed files:
- <path>
- <path>

Forbidden:
- auth, authorization, session management
- database schema, migrations, recovery paths
- CI, deployment, build pipelines
- unrelated refactors
- files outside the allowed list

Requirements:
- <requirement>
- <requirement>
- <requirement>

Changelog:
- Add or update CHANGELOG.md under an Unreleased section.
- Mention only user-visible or operationally meaningful changes.

Verification:
- <command>
- <command>

Return only:
- files changed
- implementation summary
- changelog entry added
- verification run and result
- unresolved risks or caveats
```

## Example Command

```sh
opencode run --model opencode-go/kimi-k2.6 "Implement this bounded task.

Goal:
Add slug generation for habit names.

Allowed files:
- src/habits/slug.ts
- src/habits/slug.test.ts
- CHANGELOG.md

Forbidden:
- database migrations
- auth/session code
- CI/deployment files
- unrelated refactors

Requirements:
- trim whitespace
- lowercase ASCII letters
- replace non-alphanumeric runs with a single hyphen
- remove leading/trailing hyphens
- return \"untitled\" for empty output

Changelog:
- Add an Unreleased entry describing the slug behavior.

Verification:
- bun test src/habits/slug.test.ts

Return only:
- files changed
- implementation summary
- changelog entry added
- verification run and result
- unresolved risks or caveats"
```

## Changelog Policy

Use a changelog to give both Codex and Kimi a shared memory of what changed.

Recommended format:

```md
# Changelog

All notable changes to this project are documented here.

## Unreleased

### Added

- ...

### Changed

- ...

### Fixed

- ...

### Internal

- ...
```

Rules:

- Keep entries concise.
- Prefer user-visible behavior over implementation trivia.
- Use `Internal` for developer workflow changes.
- Do not paste logs or long test output.
- Update the changelog in the same handoff when the work changes behavior.
- During Codex review, verify that the changelog matches the diff.

## Review Checklist

Codex should use this checklist after Kimi finishes:

- Did Kimi edit only allowed files?
- Does the implementation satisfy the written requirements?
- Are there unrelated refactors?
- Are tests present or updated for the changed behavior?
- Did verification run successfully?
- Is the changelog entry accurate and not inflated?
- Are there security, data-loss, or migration risks?
- Is the code consistent with local project patterns?
- Should the change be accepted, tightened, or sent back for a narrow fix?

## Risk Boundaries

Do not delegate these to Kimi without close Codex supervision:

- authentication
- authorization
- session management
- permission boundaries
- database migrations
- schema authority
- state repair and recovery paths
- billing/payment flows
- secrets handling
- production deployment
- CI gates and release automation
- changes spanning many modules with shared invariants

Good Kimi tasks:

- single-file utilities
- focused UI component edits
- straightforward tests
- mechanical refactors with a clear pattern
- docs updates
- narrow bug fixes with reproduction steps
- first-pass implementation from a strict plan

## Token Budget Rules

To actually reduce Codex token usage:

- Keep Codex plans short and explicit.
- Do not paste Kimi's full JSON event stream back into Codex.
- Return only summaries, file lists, diffs, and test results.
- Let Codex inspect files directly instead of narrating them.
- Prefer one bounded handoff over repeated vague prompts.
- If Kimi gets stuck twice, stop and replan with Codex.

## Suggested `AGENTS.md` Snippet

Add this to projects that should use the workflow:

```md
## Codex + OpenCode/Kimi Workflow

Use Codex 5.5 high or xhigh for planning and final review.

Use OpenCode with `opencode-go/kimi-k2.6` for bounded implementation tasks only.

Operating model:
- Codex writes the implementation brief.
- Kimi implements only the allowed scope.
- Kimi updates CHANGELOG.md when behavior changes.
- Codex reviews the diff, tests, and changelog before accepting.

Kimi handoffs must include:
- allowed files
- forbidden files/systems
- exact requirements
- verification commands
- expected return format

Do not delegate critical paths to Kimi:
- auth/authz/session management
- database migrations/schema authority
- recovery/state repair
- CI/deployment/release automation
- billing/secrets/security-sensitive flows

Use Bun commands where applicable:
- `bun test`
- `bun run <script>`
- `bun install`
```

## Troubleshooting

### OpenCode DB Checkpoint Error

If this appears:

```text
Failed to run the query 'PRAGMA wal_checkpoint(PASSIVE)'
```

Common causes:

- OpenCode desktop/TUI is still running.
- The command needs permission to access OpenCode local state.
- A stale SQLite WAL/checkpoint state exists.

Fixes:

1. Close OpenCode desktop/TUI.
2. Retry the command.
3. If running inside a sandboxed agent, approve OpenCode CLI access to its local state.
4. If needed, run a SQLite integrity check against the OpenCode database.

Expected integrity result:

```text
ok
```

### Kimi Available in OpenCode but Not Codex

If direct Codex-side OSS delegation is not configured, use the OpenCode CLI path instead:

```sh
opencode run --model opencode-go/kimi-k2.6 "<handoff prompt>"
```

### Kimi JSON Schema Error

If Kimi fails with:

```text
JSON Schema not supported: could not understand the instance {'default': 'latest'}
```

Then the model is listed but not currently usable through OpenCode's agent/tool schema path.

Workarounds:

- Try another OpenCode Go model, such as `opencode-go/qwen3.6-plus` or `opencode-go/deepseek-v4-flash`.
- Keep Codex as planner/reviewer and substitute the fallback model for implementation.
- Re-run the Kimi smoke test later after OpenCode or the provider updates.

## Adoption Checklist

For each project:

- [ ] Confirm `opencode-go/kimi-k2.6` is available.
- [ ] Add the `AGENTS.md` snippet.
- [ ] Ensure `CHANGELOG.md` exists.
- [ ] Decide which paths are safe for Kimi.
- [ ] Start with low-risk tasks.
- [ ] Have Codex review every Kimi diff.
- [ ] Tighten the handoff template based on failures.

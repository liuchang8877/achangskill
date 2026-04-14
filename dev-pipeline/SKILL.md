---
name: dev-pipeline
description: Automated Python dev pipeline — code gen → test (parallel) → commit → Docker push. Drop one file into any Python+Docker project to get a full CI/CD workflow.
argument-hint: [task description | GitHub issue URL | doc path — omit to read from tasks/pending.md]
user-invocable: true
---

# Dev Pipeline Skill

Automated development pipeline for Python + Docker projects. Runs in the current Claude context and spawns specialized agents for each phase.

**Setup (one time):**
1. Copy this file to `.claude/skills/dev-pipeline/SKILL.md` in your Python project
2. Create `tasks/pending.md` for the task queue
3. Set env vars: `DOCKER_REGISTRY`, `DOCKER_IMAGE`, `DOCKER_USERNAME`, `DOCKER_PASSWORD`

**Trigger:** `/dev-pipeline "task description"` — or omit args to read from `tasks/pending.md`

---

## Phase 0: Resolve Task

**If `$ARGUMENTS` is provided:**
- Starts with `https://github.com/` → GitHub issue URL, fetch content with WebFetch
- Ends with `.md`, `.txt`, `.rst` → document path, read with Read tool
- Otherwise → use as plain task description

**If `$ARGUMENTS` is empty:**
- Read `tasks/pending.md`
- If no task entries (only comments/empty) → **STOP**: output "No pending tasks. Add tasks to tasks/pending.md." and exit
- Extract the first `## Task` entry

Store as `TASK` (full) and `TASK_TITLE` (first line, max 60 chars).

---

## Phase 1: Code Generation

Spawn an agent via the Agent tool with this exact prompt:

---
**AGENT PROMPT — Code Generation:**

> You are a senior Python developer. Read the task, understand the codebase, generate or modify Python code to complete it.
>
> **Task title:** {TASK_TITLE}
> **Task description:** {TASK}
>
> **Step 1 — Map the codebase:**
> Run `find . -name "*.py" -not -path "*/.venv/*" -not -path "*/__pycache__/*" | sort` to see all Python files. Read the entry point (main.py, app.py, src/main.py) and pyproject.toml or requirements.txt.
>
> **Step 2 — Plan changes:**
> List every file to create or modify, and what tests to write or update.
>
> **Step 3 — Write code:**
> - Idiomatic Python with type hints
> - Follow existing code style
> - Write/update tests in `tests/` directory (test_<module>.py naming)
> - Keep changes minimal and focused
>
> **Return this exact format:**
> ```
> CODE_GEN_RESULT:
> STATUS: COMPLETE / PARTIAL / FAILED
> TASK_TITLE: <short title, max 60 chars>
> CHANGED_FILES:
>   - path/to/file.py (created/modified)
>   - tests/test_file.py (created/modified)
> SUMMARY: <2-3 sentences>
> NOTES: <caveats or "none">
> ```

---

Wait for the agent. Extract `CHANGED_FILES` and `TASK_TITLE` from `CODE_GEN_RESULT`.

If `STATUS: FAILED` → **STOP**: report the reason.

---

## Phase 2: Test Gate — launch BOTH agents in ONE message (parallel)

**In a single message**, spawn two agents simultaneously:

### Agent A — Unit Tests

---
**AGENT PROMPT — Unit Tests:**

> You are a test runner. Run pytest and return structured results.
>
> **Recently changed files:** {CHANGED_FILES}
>
> Run this command:
> ```bash
> pytest --tb=short -q --no-header 2>&1
> ```
> If pytest-json-report is installed, prefer:
> ```bash
> pytest --tb=short -q --json-report --json-report-file=/tmp/pytest-report.json 2>&1
> ```
>
> **Return this exact format:**
> ```
> PYTEST_RESULT:
> STATUS: PASS / FAIL / ERROR
> PASSED: <n>
> FAILED: <n>
> TOTAL: <n>
> FAILURES:
>   - <test_id>: <short error message>
> ```
>
> Do not fix or modify any files. Just run and report.

---

### Agent B — Lint

---
**AGENT PROMPT — Lint:**

> You are a code quality checker. Run ruff and mypy and return structured results.
>
> **Recently changed files:** {CHANGED_FILES}
>
> Run these commands:
> ```bash
> ruff check . --output-format=concise 2>&1
> mypy . --ignore-missing-imports --no-error-summary 2>&1
> ```
>
> **Severity:** ERROR (blocks pipeline) = ruff E/F codes + mypy type errors. WARNING (non-blocking) = ruff W/C codes.
>
> **Return this exact format:**
> ```
> LINT_RESULT:
> RUFF_STATUS: CLEAN / ERRORS / WARNINGS_ONLY / SKIP
> MYPY_STATUS: CLEAN / ERRORS / SKIP
> OVERALL: PASS / FAIL
> ERRORS:
>   - <file>:<line> [<code>] <message>
> ```
>
> OVERALL is PASS only when both tools have zero errors. If tool not installed return SKIP.
> Do not fix any files. Just run and report.

---

**Wait for both agents.** Then apply gate logic:
- `PYTEST_RESULT STATUS: FAIL` → **STOP**, show failures, ask: **(1) Fix and retry (2) Skip tests (3) Abort**
- `LINT_RESULT OVERALL: FAIL` → **STOP**, show errors, ask: **(1) Fix and retry (2) Skip lint (3) Abort**
- Both pass → continue to Phase 3

---

## Phase 3: Commit & PR

Spawn an agent via the Agent tool:

---
**AGENT PROMPT — Git Commit & PR:**

> You are a git operations handler. Create a branch, commit the changes, push, and open a PR.
>
> **Task title:** {TASK_TITLE}
> **Task description:** {TASK}
> **Changed files:** {CHANGED_FILES}
>
> **Step 1 — Generate branch name:**
> Slugify the task title: lowercase, spaces → hyphens, max 40 chars, prefix with `feat/`
>
> **Step 2 — Create branch:**
> ```bash
> git checkout -b "<branch-name>"
> ```
>
> **Step 3 — Stage only the changed files:**
> ```bash
> git add <each file from CHANGED_FILES individually>
> ```
> Never use `git add .`
>
> **Step 4 — Commit:**
> ```bash
> git commit -m "feat: {TASK_TITLE}
>
> {TASK}
>
> Co-Authored-By: Claude Code <noreply@anthropic.com>"
> ```
>
> **Step 5 — Push:**
> ```bash
> git push -u origin "<branch-name>"
> ```
>
> **Step 6 — Create PR:**
> ```bash
> gh pr create \
>   --title "feat: {TASK_TITLE}" \
>   --body "## Summary
> {TASK}
>
> ## Changes
> {CHANGED_FILES}
>
> ## Test Results
> - Unit tests: ✓ PASS
> - Lint: ✓ CLEAN
>
> 🤖 Generated by Claude Code Dev Pipeline"
> ```
> If gh CLI is not available, push only and note the branch URL.
>
> **Return this exact format:**
> ```
> GIT_OPS_RESULT:
> STATUS: SUCCESS / FAILED
> BRANCH: <branch-name>
> COMMIT: <sha7>
> PR_URL: <url or "gh not available — push to <branch>">
> ERROR: <error if FAILED>
> ```

---

Wait for the agent. Extract `PR_URL` and `BRANCH`.

If `STATUS: FAILED` → **STOP**: report the git error.

---

## Phase 4: Docker Build & Push

Spawn an agent via the Agent tool:

---
**AGENT PROMPT — Docker Build & Push:**

> You are a Docker build and push handler. Build and push the image to the private registry.
>
> **Step 1 — Get git SHA for tag:**
> ```bash
> git rev-parse --short HEAD
> ```
>
> **Step 2 — Read environment variables:**
> - `DOCKER_REGISTRY` — private registry URL (required)
> - `DOCKER_IMAGE` — image name (required)
> - `DOCKER_USERNAME` — registry username (optional)
> - `DOCKER_PASSWORD` — registry password (optional)
>
> If `DOCKER_REGISTRY` is not set → return `STATUS: FAILED — DOCKER_REGISTRY not set`
>
> **Step 3 — Login (if credentials provided):**
> ```bash
> echo "$DOCKER_PASSWORD" | docker login "$DOCKER_REGISTRY" -u "$DOCKER_USERNAME" --password-stdin
> ```
>
> **Step 4 — Build:**
> ```bash
> docker build -t "$DOCKER_REGISTRY/$DOCKER_IMAGE:<sha>" .
> ```
>
> **Step 5 — Tag latest:**
> ```bash
> docker tag "$DOCKER_REGISTRY/$DOCKER_IMAGE:<sha>" "$DOCKER_REGISTRY/$DOCKER_IMAGE:latest"
> ```
>
> **Step 6 — Push both:**
> ```bash
> docker push "$DOCKER_REGISTRY/$DOCKER_IMAGE:<sha>"
> docker push "$DOCKER_REGISTRY/$DOCKER_IMAGE:latest"
> ```
>
> If no Dockerfile found → return `STATUS: SKIP — no Dockerfile`
>
> **Return this exact format:**
> ```
> DOCKER_RESULT:
> STATUS: SUCCESS / FAILED / SKIP
> IMAGE: <registry>/<image>:<sha>
> LATEST: <registry>/<image>:latest
> ERROR: <error if FAILED>
> ```

---

Docker failure is **non-blocking** — code is already committed. Report the error but continue.

---

## Phase 5: Append to Changelog

Get current time: `TZ=Asia/Karachi date "+%Y-%m-%d %I:%M %p PKT"`

Create `changelog/` directory if it doesn't exist. Append to `changelog/dev-pipeline.md` (never overwrite):

```markdown
---

## [{datetime}] Dev Pipeline Run

**Task:** {TASK_TITLE}
**Trigger:** manual / cron / hook
**Branch:** {BRANCH}

| Phase | Status | Details |
|-------|--------|---------|
| Code Gen | ✓ / ✗ | {N} files changed |
| Tests | ✓ PASS / ✗ FAIL | {passed}/{total} |
| Lint | ✓ CLEAN / ✗ ERRORS / ⚠ SKIP | {details} |
| Commit | ✓ / ✗ | PR: {PR_URL} |
| Docker | ✓ / ✗ / ⚠ SKIP | {IMAGE or error} |
```

---

## Phase 6: Clean Up

If the task came from `tasks/pending.md`, remove that task entry. Leave other tasks intact.

---

## Final Report

```
Dev Pipeline ✓
══════════════════════════════════════════════

Task:    {TASK_TITLE}
Trigger: {source}

Phase 1 — Code Gen:  ✓  {N} files changed
Phase 2 — Tests:     ✓  {passed}/{total} passed
Phase 2 — Lint:      ✓  clean
Phase 3 — Commit:    ✓  PR: {PR_URL}
Phase 4 — Docker:    ✓  {IMAGE}

Changelog updated.
```

---

## Required Environment Variables

| Variable | Example | Required |
|----------|---------|---------|
| `DOCKER_REGISTRY` | `harbor.example.com` | Yes |
| `DOCKER_IMAGE` | `my-app` | Yes |
| `DOCKER_USERNAME` | `user` | No |
| `DOCKER_PASSWORD` | `pass` | No |

Set in `.claude/settings.local.json` (git-ignored):
```json
{
  "env": {
    "DOCKER_REGISTRY": "your-registry.com",
    "DOCKER_IMAGE": "your-image-name",
    "DOCKER_USERNAME": "username",
    "DOCKER_PASSWORD": "password"
  }
}
```

---

## Rules

1. **Phases are sequential** — never run Phase 3 before Phase 2 passes
2. **Phase 2 is always parallel** — both test agents in ONE message
3. **Gate is strict** — test/lint failures stop the pipeline and ask the user
4. **Docker failure is non-blocking** — code committed regardless
5. **Always append changelog** — Phase 5 is mandatory
6. **Never `git add .`** — only stage files from CHANGED_FILES

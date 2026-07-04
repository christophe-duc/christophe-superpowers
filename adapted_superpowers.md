# Adapting Superpowers: Remove Commits, Serialize Writing Agents, Per-Phase Execution, User-Gated Full-Suite Testing

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Modify the Superpowers skills so that (1) no skill ever creates a git commit, (2) writing (file-modifying) subagents run one at a time — read-only investigation agents may still be dispatched in parallel, (3) subagents execute one plan *phase* at a time (auto-continuing between phases), and (4) skills never run the *full* test suite — focused RED/GREEN test runs, lint, syntax, type, and build checks are allowed; the human runs the full suite and starts the broader testing phase.

**Architecture:** Four orthogonal behavioral changes, each touching distinct skills and scripts. Phase A patches the shared review tooling first (everything else depends on it), then phases B–G update each skill layer in dependency order. Phase H greps for stragglers.

**Tech Stack:** Bash scripts, Markdown skill files (no external dependencies).

## Global Constraints

- No skill, prompt, or script may call `git commit`, `git add` (except inside a throw-away index), `git merge`, `git push`, or `git branch -d/-D`.
- No skill, prompt, or script may run the **full test suite** (`npm test`, bare `pytest`, `cargo test`, `go test ./...`, or any equivalent whole-suite invocation). Running a **focused test** — a single test file or case covering the code under change (e.g. `pytest path/test.py::case`, `npm test path/to/file.test.ts`) — to observe RED/GREEN is allowed and expected for TDD. Lint, type-check, format, and compile/build calls are permitted. The human runs the full suite and starts the broader testing phase.
- Phase notation: a phase is a `## <Letter>. Title` heading; a subtask is `### <Letter><Number>`. One subagent per `## <Letter>` phase; auto-continue between phases.
- Diff review uses working-tree vs. baseline **tree object** diffs (`git diff <tree> <tree>`), never commit-range diffs.
- Only one *file-modifying* subagent dispatch at a time — never two writing agents concurrently. Read-only investigation agents (diagnose and report, no file writes) may still be dispatched in parallel.
- The human commits, merges, and pushes at the end; the agent hands off cleanly.

---

## A. Commit-free snapshot & review tooling (shared scripts)

**Files:**
- Create: `skills/subagent-driven-development/scripts/snapshot-tree`
- Modify: `skills/subagent-driven-development/scripts/review-package`
- Modify: `skills/subagent-driven-development/scripts/task-brief`
- Modify: `skills/subagent-driven-development/scripts/sdd-workspace`

**Interfaces:**
- Produces: `snapshot-tree` prints a tree SHA to stdout; `review-package BASE_TREE [HEAD_TREE] [OUTFILE]` accepts tree SHAs; `task-brief` accepts a phase letter and extracts a full `## <Letter>` section.

### A.1 — Create `snapshot-tree`

Create `skills/subagent-driven-development/scripts/snapshot-tree` with content:

```bash
#!/usr/bin/env bash
# Capture the full working tree (tracked + untracked, honouring .gitignore)
# as a git tree object WITHOUT touching HEAD, the real index, or any branch.
# Prints the tree SHA. Safe to call at any time — no side effects on history.
#
# A fresh (empty) throw-away index + `git add -A` stages every tracked and
# untracked file in the working tree, which is exactly a full snapshot — so
# we do NOT `git read-tree HEAD` first. That also lets this work in a repo
# with zero commits (no HEAD to read), matching review-package's promise
# that diffs work whether or not any commits exist.
#
# Usage: snapshot-tree
set -euo pipefail

idx=$(mktemp)
rm -f "$idx"                      # start from a truly empty index
trap 'rm -f "$idx"' EXIT

GIT_INDEX_FILE="$idx" git add -A
GIT_INDEX_FILE="$idx" git write-tree
```

- [ ] Write the file with the content above.
- [ ] Make it executable: `chmod +x skills/subagent-driven-development/scripts/snapshot-tree`
- [ ] Verify no side effects: run `git status --porcelain > /tmp/before` before and `git status --porcelain > /tmp/after` after calling `snapshot-tree`, and confirm the two are identical (the real index, HEAD, and branch state are untouched).
- [ ] Verify no-commit support: in a scratch repo with an untracked file and **no commits**, `snapshot-tree` must still print a tree SHA (not fail on a missing HEAD).

### A.2 — Update `review-package` to accept tree SHAs

Read the current file: `skills/subagent-driven-development/scripts/review-package`

Replace its content with:

```bash
#!/usr/bin/env bash
# Generate a review package: stat summary and net diff with extended context,
# written to a file the reviewer reads in one call.
#
# Diffs are tree-object to tree-object (never commit ranges), so this works
# whether or not any commits have been made. Pass a baseline tree recorded
# by snapshot-tree before the phase started, and optionally a head tree
# (defaults to the current working tree via snapshot-tree).
#
# Usage: review-package BASE_TREE [HEAD_TREE] [OUTFILE]
# Default OUTFILE: <sdd-workspace>/review-<base7>..<head7>.diff
set -euo pipefail

if [ $# -lt 1 ] || [ $# -gt 3 ]; then
  echo "usage: review-package BASE_TREE [HEAD_TREE] [OUTFILE]" >&2
  exit 2
fi

base=$1

# Resolve HEAD_TREE: use provided or snapshot current working tree
if [ $# -ge 2 ]; then
  head=$2
else
  head=$("$(cd "$(dirname "$0")" && pwd)/snapshot-tree")
fi

git rev-parse --verify --quiet "$base" >/dev/null || { echo "bad BASE_TREE: $base" >&2; exit 2; }
git rev-parse --verify --quiet "$head" >/dev/null || { echo "bad HEAD_TREE: $head" >&2; exit 2; }

if [ $# -eq 3 ]; then
  out=$3
else
  dir=$("$(cd "$(dirname "$0")" && pwd)/sdd-workspace")
  out="$dir/review-$(git rev-parse --short "$base")..$(git rev-parse --short "$head").diff"
fi

{
  echo "# Review package: ${base}..${head}"
  echo "(working-tree diff — no commits required)"
  echo
  echo "## Files changed since phase baseline"
  git diff --stat "$base" "$head"
  echo
  echo "## Diff"
  git diff -U10 "$base" "$head"
} > "$out"

echo "wrote ${out}: $(wc -c < "$out" | tr -d ' ') bytes"
```

- [ ] Overwrite `skills/subagent-driven-development/scripts/review-package` with the content above.
- [ ] Verify the script is still executable (`chmod +x` if needed).
- [ ] Smoke-test: run `scripts/snapshot-tree` twice, capture both SHAs, then call `scripts/review-package SHA1 SHA2` — confirm it writes a `.diff` file and exits 0.

### A.3 — Update `task-brief` to extract a whole phase section

Read the current file: `skills/subagent-driven-development/scripts/task-brief`

The script currently extracts a single task by number. Update it to accept a **phase letter** and extract the entire `## <Letter>.` section (everything from that heading to the next `## ` heading or EOF).

Replace its content with:

```bash
#!/usr/bin/env bash
# Extract a phase section from an implementation plan and write it to a
# uniquely named file so the implementer subagent can read it in one call.
#
# A phase starts at a "## <LETTER>." heading and ends at the next "## "
# heading (or EOF). All subtasks (### <LETTER><N>) are included.
#
# Code-fence tracking prevents false matches on "## " lines inside fenced
# code blocks (e.g. bash comments or YAML keys inside ``` blocks).
#
# Usage: task-brief PLAN_FILE PHASE_LETTER
# Prints the absolute path of the written brief file.
set -euo pipefail

if [ $# -ne 2 ]; then
  echo "usage: task-brief PLAN_FILE PHASE_LETTER" >&2
  exit 2
fi

plan=$1
letter=${2^^}   # uppercase

[ -f "$plan" ] || { echo "plan not found: $plan" >&2; exit 2; }

dir=$("$(cd "$(dirname "$0")" && pwd)/sdd-workspace")
out="$dir/phase-${letter}-brief.md"

# Extract from "## <LETTER>." to next "## " (exclusive) or EOF.
# infence tracks whether we are inside a fenced code block so that
# "## " appearing inside a code block does not terminate extraction.
awk \
  -v letter="$letter" \
  'BEGIN { in_phase=0; infence=0 }
   /^```/ { infence = !infence }
   !infence && /^## / {
     if (in_phase) exit
     if ($0 ~ ("^## " letter "\\.")) { in_phase=1; print; next }
   }
   in_phase { print }
  ' "$plan" > "$out"

[ -s "$out" ] || { echo "phase ${letter} not found in ${plan}" >&2; exit 2; }

echo "$out"
```

- [ ] Overwrite `skills/subagent-driven-development/scripts/task-brief` with the content above.
- [ ] Make it executable if needed.
- [ ] Smoke-test: call it against this plan file (`adapted_superpowers.md`) with letter `A` — confirm it writes a non-empty file containing the `## A.` section and nothing from `## B.`.

### A.4 — Update `sdd-workspace` header comment

Read `skills/subagent-driven-development/scripts/sdd-workspace`.

Update only the comment block at the top to read:

```
# Resolve and ensure the working-tree directory SDD uses for its short-lived
# artifacts: phase briefs, implementer reports, baseline tree snapshots,
# review packages, and the progress ledger. Print the directory's absolute path.
```

(No code change — comment only.)

- [ ] Apply the comment-only edit.

---

## B. Code review without commits

**Files:**
- Modify: `skills/requesting-code-review/SKILL.md`
- Modify: `skills/requesting-code-review/code-reviewer.md`
- Modify: `skills/subagent-driven-development/task-reviewer-prompt.md`

**Interfaces:**
- Consumes: `snapshot-tree` SHA (baseline), working-tree SHA (head), diff file from `review-package`.
- Produces: updated review templates that accept tree SHAs and expect focused-test evidence only (never full-suite run evidence).

### B.1 — Update `requesting-code-review/SKILL.md`

Read the file.

- Replace the "How to Request → 1. Get git SHAs" step block:
  - Old: ```BASE_SHA=$(git rev-parse HEAD~1)  # or origin/main``` / ```HEAD_SHA=$(git rev-parse HEAD)```
  - New:
    ```bash
    # Record baseline BEFORE the phase starts (call once, save the SHA)
    BASE_TREE=$(skills/subagent-driven-development/scripts/snapshot-tree)
    # After phase work is done, generate the review package:
    scripts/review-package "$BASE_TREE"
    # (HEAD_TREE defaults to current working tree snapshot)
    ```
- Replace placeholder definitions `{BASE_SHA}`/`{HEAD_SHA}` → `{BASE_TREE}`/`{HEAD_TREE}` (tree SHAs).
- **Frontmatter `description`:** change `"...or before merging to verify work meets requirements"` → `"...or before handing work off to the human to verify it meets requirements"`.
- Replace **every** "before merge/merging" reference, not just one:
  - "When to Request → Mandatory → Before merge to main" → "Before handing off to the human"
  - "Integration → Ad-Hoc Development → Review before merge" → "Review before hand-off"
- Update the "Example" to use tree SHAs (the example's `BASE_SHA=$(git log … grep "Task 1")` becomes a recorded `BASE_TREE` from `snapshot-tree`).

- [ ] Apply all edits to `skills/requesting-code-review/SKILL.md`.

### B.2 — Update `code-reviewer.md`

Read the file.

- "Git Range to Review" section → rename to "Diff to review (working tree vs. baseline)":
  ```
  **Baseline tree:** {BASE_TREE}
  **Head tree:** {HEAD_TREE}

  Read the review-package diff file. Fallback:
      git diff --stat {BASE_TREE} {HEAD_TREE}
      git diff {BASE_TREE} {HEAD_TREE}
  ```
- In "What to Check → Testing" checklist, replace "All tests passing?" with: "Tests present and meaningful? (Tests are written by the agent but *run by the human* — do not expect passing-test evidence. Check that tests exist and cover real behavior.)"
- Change "Ready to merge?" assessment label → "Ready to hand off?" (also update the same label in the **Example Output** block at the bottom: `"Ready to merge: With fixes"` → `"Ready to hand off: With fixes"`).
- **"Read-Only Review" paragraph:** it suggests `git worktree add /tmp/review-[SHA] [SHA]` to inspect a revision. A *tree* SHA is not a commit and cannot be checked out that way. Change the guidance to: "The review-package diff is your view of the change; if you must inspect a file's full content, read it from the working tree (it already holds the head state) — do not try to check out a tree object."
- Update placeholders at the bottom: `[BASE_SHA]`/`[HEAD_SHA]` → `[BASE_TREE]`/`[HEAD_TREE]`.

- [ ] Apply all edits to `skills/requesting-code-review/code-reviewer.md`.

### B.3 — Update `task-reviewer-prompt.md`

Read the file.

- Replace `[BASE_SHA]` / `[HEAD_SHA]` → `[BASE_TREE]` / `[HEAD_TREE]` throughout.
- "Diff Under Review" block: remove "commit list" wording (the package no longer contains a commit list); diff-file fallback: `git diff [BASE_TREE] [HEAD_TREE]`.
- Rewrite the "## Tests" section:
  ```
  ## Tests

  The implementer ran the FOCUSED tests for exactly this code and reported the
  results with TDD evidence. Do not re-run the full suite — the human runs the
  full suite. Run a single focused test only when reading the code raises a
  specific doubt no existing run answers; never a package-wide suite, race
  detector run, or high-count loop. If heavier validation seems warranted,
  recommend it in your report instead of running it.
  ```
- Keep the sentence about pristine output but scope it to the focused runs: `"Warnings or noise in the implementer's reported focused-test output are findings — that output should be pristine."`
- **Placeholders block at the bottom:** update `[BASE_SHA]`/`[HEAD_SHA]` → `[BASE_TREE]`/`[HEAD_TREE]`; change `[BRIEF_FILE]` note `"scripts/task-brief PLAN N prints the path"` → `"scripts/task-brief PLAN <LETTER> prints the path"`; change `[DIFF_FILE]` note `"scripts/review-package BASE HEAD prints..."` → `"scripts/review-package BASE_TREE prints..."`.

- [ ] Apply all edits to `skills/subagent-driven-development/task-reviewer-prompt.md`.

---

## C. Read-only agents may parallelize; writing agents run one at a time

**Files:**
- Modify: `skills/dispatching-parallel-agents/SKILL.md`
- Modify: `skills/subagent-driven-development/SKILL.md`

### C.1 — Constrain `dispatching-parallel-agents/SKILL.md` to read-only investigation

Read the file.

**Rationale:** Parallel dispatch is safe and valuable when agents only *read and report* (investigation, root-cause diagnosis). It is unsafe when agents *write*, because two writers can touch overlapping files with no coordination. Keep the parallel pattern, but restrict it to read-only investigation and require that any file-modifying work be applied by the controller one change at a time.

Make the following targeted edits:

- **Frontmatter `description`:** Change to `"Use when facing 2+ independent problems that can be investigated read-only in parallel — agents diagnose and report; the controller applies any fixes one at a time."`
- **Overview / Core principle:** Change `"Dispatch one agent per independent problem domain. Let them work concurrently."` → `"Dispatch one read-only investigation agent per independent problem domain. They diagnose in parallel and report back; the controller applies fixes serially. Never dispatch two file-modifying agents concurrently."`
- **Add a bold constraint** near the top of "The Pattern": `"**Read-only only.** Agents dispatched in parallel must not modify files — they investigate and return a diagnosis + recommended fix. The controller (or a serial follow-up agent) applies the actual changes one at a time, so two writers never race on the same files."`
- **Section 2 "Create Focused Agent Tasks":** Change the "Clear goal: Make these tests pass" / "Constraints: Don't change other code" framing to `"Clear goal: Diagnose the root cause and recommend a fix"` and `"Constraints: Read-only — do not modify files; report findings only."`
- **Step 3 "Dispatch in Parallel":** Keep the parallel dispatch, but change the example agent prompts from "Fix X" to "Investigate X and report root cause + recommended fix (read-only)." Keep the note that multiple dispatch calls in one response run in parallel — this is intended for read-only investigation.
- **Step 4 "Run full test suite":** Change → `"Apply the recommended fixes one at a time, then run focused tests for each change; the human runs the full suite."`
- **"Real Example" / "Real-World Impact":** Reframe the outcomes as parallel *investigation* whose fixes were then integrated serially (e.g. `"3 agents investigated in parallel; controller applied each fix serially and ran focused tests"`). Keep the parallelism framing for the investigation itself.
- **Agent Prompt Structure / Common Mistakes:** Add a mistake: `"❌ Letting a parallel agent edit files — parallel writers race. ✅ Parallel agents are read-only; fixes are applied serially."`
- **"## Verification → Run full suite" (step 3 of Verification):** Change → `"Run focused tests for each integrated fix; the human runs the full suite."`

- [ ] Apply all edits to `skills/dispatching-parallel-agents/SKILL.md`.

### C.2 — Reinforce write-serialization in `subagent-driven-development/SKILL.md`

Read the file. The skill already has "Dispatch multiple implementation subagents in parallel (conflicts)" in Red Flags. Add an explicit call-out:

In the **Red Flags** → **Never** list, add as the first bullet:
```
- Dispatch more than one file-modifying subagent at a time — implementer and fixer subagents must run sequentially, never concurrently (read-only reviewers are exempt, but SDD's flow already runs them serially after implementation)
```

Also update the **Advantages** section: change `"Parallel-safe (subagents don't interfere)"` → `"Write-serialized (one file-modifying subagent at a time, so subagents don't interfere)"`.

- [ ] Apply the edit.

---

## D. Per-phase execution model

**Files:**
- Modify: `skills/writing-plans/SKILL.md`
- Modify: `skills/executing-plans/SKILL.md`
- Modify: `skills/subagent-driven-development/SKILL.md`
- Modify: `skills/subagent-driven-development/implementer-prompt.md`

### D.1 — Update `writing-plans/SKILL.md`

Read the file. Apply these edits:

**Overview line:** Remove `"Frequent commits."` from the final sentence. Add: `"No commits — leave all changes in the working tree for the human to commit."`

**File Structure section:** Add after the first paragraph: `"Organize the plan as phases: ## <Letter>. Title headings (e.g. ## A. Setup). Within each phase, use ### <Letter><Number> subtasks. One subagent implements one whole phase per dispatch."`

**Task Right-Sizing / Bite-Sized Task Granularity:** Remove the step:
```
- "Commit"
```
and the example commit block:
```bash
git add tests/path/test.py src/path/file.py
git commit -m "feat: add specific feature"
```
TDD verification steps: **keep** the "Run it to make sure it fails" / "Run the tests and make sure they pass" steps — these run a *focused* test (a single test case/file), which is allowed. Just ensure the commands target one test, not the whole suite (e.g. `pytest tests/path/test.py::test_name`, not bare `pytest`). Add a note that the human runs the full suite at the end.

**Task Structure template (Step 2 / Step 4 run blocks):** Keep the focused `pytest`/`npm test <file>` run commands (they target a single test — allowed). Do NOT change them to lint-only; TDD depends on watching the focused test go RED then GREEN. Just verify each command names a specific test, never the whole suite.

**Step 5 commit block:** Remove the entire `- [ ] **Step 5: Commit**` block including the ```bash git add / git commit``` example.

**Remember section:** Change `"DRY, YAGNI, TDD, frequent commits"` → `"DRY, YAGNI, TDD (focused test runs; human runs the full suite), no commits."`

**Plan Document Header note:** Under `> **For agentic workers:**`, update the sub-skill note to reference per-phase execution: `"Steps are organized as ## <Letter> phases; dispatch one subagent per phase."`

- [ ] Apply all edits to `skills/writing-plans/SKILL.md`.

### D.2 — Update `executing-plans/SKILL.md`

Read the file.

**Step 2 "Execute Tasks":** Rewrite to per-phase execution:
```
### Step 2: Execute Phases

For each `## <Letter>` phase in the plan:
1. Mark phase as in_progress
2. Implement all `### <Letter><N>` subtasks in sequence
3. Run focused tests for the changed code (RED/GREEN) plus lint/syntax/build checks — do **not** run the full suite
4. Mark phase as completed
5. Auto-continue to the next phase without stopping

Do not run the full test suite at any point. Focused test runs (a single
test file/case for the code just changed), lint, type-check, and compile are
fine. Never create git commits.
```

**Step 3 "Complete Development":** Keep the reference to `finishing-a-development-branch` skill (which will be rewritten in Phase F).

**Remember section:** Add `"Never commit."` and `"Never run the full test suite (focused tests are fine; the human runs the full suite)."` Remove any implication of per-task granularity; replace with per-phase.

- [ ] Apply all edits to `skills/executing-plans/SKILL.md`.

### D.3 — Update `subagent-driven-development/SKILL.md` for per-phase model

Read the file. This is the most involved edit. Apply these targeted changes:

**Core principle / description line:** `"Execute plan by dispatching a fresh implementer subagent per task"` → `"Execute plan by dispatching a fresh implementer subagent per **phase** (## <Letter> section)"`

**Process digraph:** Update all `"Task"` → `"Phase"` in node labels. `"task-reviewer-prompt"` label stays; update the implementer dispatch label to `"Dispatch implementer subagent for phase (./implementer-prompt.md)"`. Also change the node `"Implementer subagent implements, tests, commits, self-reviews"` → `"Implementer subagent implements, runs focused tests, self-reviews (no commit)"` (drops the commit step; keeps focused testing).

**Pre-Flight Plan Review:** `"tasks that contradict"` → `"phases or subtasks that contradict"`.

**Handling Implementer Status → DONE:** Replace:
> `"Generate the review package (scripts/review-package BASE HEAD …)`

With:
> `"Generate the review package (scripts/review-package BASE_TREE — where BASE_TREE is the tree snapshot you recorded before dispatching the implementer). The script snapshots the current working tree as HEAD_TREE automatically."`

Remove the `"never HEAD~1"` warning (there are no commits to truncate).

**File Handoffs → Task brief:** Replace `"scripts/task-brief PLAN_FILE N"` → `"scripts/task-brief PLAN_FILE <LETTER>"` (extracts the full `## <Letter>` phase section). Update naming: `…/task-N-brief.md` → `…/phase-<Letter>-brief.md`, `…/task-N-report.md` → `…/phase-<Letter>-report.md`.

**Durable Progress / ledger:** Replace the ledger line format:
> `Task N: complete (commits <base7>..<head7>, review clean)`

With:
> `Phase <Letter>: complete (baseline <tree7> → <tree7>, review clean)`

Replace recovery paragraph: Remove `"the commits it names exist in git even when your context no longer remembers creating them"`. Replace with: `"The baseline tree SHAs it names are objects in git — readable even after context compaction via git cat-file. The working tree holds current state. Never re-dispatch a phase the ledger marks complete."`

Remove: `"git clean -fdx will destroy the ledger (it's git-ignored scratch); if that happens, recover from git log."` Replace: `"git clean -fdx will destroy the ledger and all uncommitted work in the workspace. If that happens, recover from the working tree state and any backup the human has."`

**Between-phase behavior:** Add a note after "Continuous execution": `"Continuous execution applies both within a phase (all subtasks) and between phases — auto-continue to the next phase without stopping for the human after each phase review comes back clean."`

**Example Workflow:** Update all `Task N` → `Phase <Letter>` references; update commit mentions to tree snapshots.

**Constructing Reviewer Prompts → review-package:** Replace `"scripts/review-package BASE HEAD"` → `"scripts/review-package BASE_TREE"` (no HEAD arg; script snapshots automatically). Replace `"never HEAD~1"` warning with `"use the baseline tree recorded before the implementer was dispatched."` Likewise update the final whole-branch review call (`scripts/review-package MERGE_BASE_TREE` — the tree snapshot recorded when the branch/work started).

**Red Flags list:** Update the bullet `"Dispatch a task reviewer without a diff file — generate it first (scripts/review-package BASE HEAD)"` → `scripts/review-package BASE_TREE`.

**Test-run wording (leave mostly intact — focused tests are allowed):** The existing lines "Do not ask a reviewer to re-run tests the implementer already ran" and the fix-dispatch contract ("re-runs the tests covering its change and reports the results … the covering tests, the command run, and the output") describe *focused* test runs on the changed code, which remain allowed under the full-suite-only ban. Add a clarifying parenthetical to each: `(focused tests only — never the full suite)`. Do not delete these; they are the mechanism that keeps TDD evidence flowing.

- [ ] Apply all edits to `skills/subagent-driven-development/SKILL.md`.

### D.4 — Update `implementer-prompt.md`

Read the file.

**"Your Job" numbered list:** Remove `"4. Commit your work"` (renumber remaining items). Change `"3. Verify implementation works"` → `"3. Verify: run the focused test(s) for the code you changed (RED then GREEN) plus lint/type/build. Do **not** run the full suite — the human runs it."`

**While-you-work paragraph:** Replace `"run the full suite once before committing, not after every edit."` with: `"Run the focused test for what you're changing as you iterate. Do not run the full suite — the human runs it at the end."` (Keep the existing "run the focused test for what you're changing" clause.)

**TDD Evidence (in Report Format):** Keep RED/GREEN as *actual* focused-test runs — this is the point of TDD:
- RED: keep `"command run, relevant failing output before implementation, and why the failure was expected"` (focused test).
- GREEN: keep `"command run and relevant passing output after implementation"` (focused test).

**Report Format — return message:**
- Remove `"Commits created (short SHA + subject)"`.
- Change `"One-line test summary (e.g. '14/14 passing, output pristine')"` → `"Focused-test summary + note the full suite is left for the human (e.g. 'focused tests 6/6 pass, full suite not run')"`.

**"After Review Findings" section:** Leave intact — it tells the implementer to re-run the *focused* tests covering amended code, which is allowed. Add `(focused tests, not the full suite)` for clarity.

- [ ] Apply all edits to `skills/subagent-driven-development/implementer-prompt.md`.

---

## E. Testing-phase policy (focused tests OK; the human runs the full suite)

**Files:**
- Modify: `skills/test-driven-development/SKILL.md`
- Modify: `skills/verification-before-completion/SKILL.md`
- Modify: `skills/using-git-worktrees/SKILL.md`

### E.1 — Update `test-driven-development/SKILL.md`

Read the file.

**Design note:** TDD's core — "watch the test fail, then watch it pass" — is preserved. The agent runs the *focused* test for the code under change (a single file or case). The only thing the agent must not do is run the **full suite**; that is the human's job at the end. This keeps the skill coherent with its own philosophy (`AGENTS.md` forbids gutting the Red Flags / rationalization content without eval evidence, so we make the minimum change).

Add a banner directly under the `# Test-Driven Development (TDD)` heading:

```markdown
> **Environment policy:** Run only the **focused** test for the code you're changing (a single test file or case) to observe RED then GREEN. Do **not** run the full test suite — the human runs the full suite and starts the broader testing phase. Lint/type/build checks are always fine.
```

**"Verify RED — Watch It Fail" section:** Keep "MANDATORY. Never skip." Keep the run block but make the command explicitly focused (single test file/case), and add: `"Run the focused test only — never the whole suite."`

**"Verify GREEN — Watch It Pass" section:** Keep the run block as a focused-test run. Change the "Other tests still pass" bullet (which implies the full suite) → `"Focused test passes (the human runs the full suite to catch cross-cutting regressions)."`

**Verification Checklist:**
- Keep `"Watched each test fail before implementing"` and `"Each test failed for expected reason"` (the agent does watch the focused test fail).
- `"All tests pass"` → `"Focused test passes; full suite left for the human"`.
- Keep `"Output pristine (no errors, warnings)"`.

**Red Flags:** Keep `"Test passes immediately"` and `"Can't explain why test failed"` (still valid — the agent runs the focused test). Add one bullet: `"Running the full test suite — run only the focused test; the human runs the suite."`

- [ ] Apply all edits to `skills/test-driven-development/SKILL.md`.

### E.2 — Update `verification-before-completion/SKILL.md`

Read the file.

**Core principle / Iron Law:** Add below the iron law box:
```
Verification = lint, type-check, format, build/compile, and the focused
test(s) for the code you changed. Never claim the FULL suite passes — you
did not run it; the human runs the full suite.
```

**Common Failures table:** Replace the "Tests pass" row with two rows:
| Focused test passes | Focused test command output: 0 failures | Previous run, "should pass" |
| Full suite passes | N/A — agent does not run the full suite; hand off to human | "focused test passed so all pass" |

**Key Patterns → Tests section:** Replace:
```
✅ [Run test command] [See: 34/34 pass] "All tests pass"
❌ "Should pass now" / "Looks correct"
```
With:
```
✅ [Run focused test] [See: pass] "Focused tests pass; full suite left for the human"
❌ "All tests pass" / "All tests passing" — you did not run the full suite
```

**Red Flags:** Add as first bullet: `"Claiming the full suite passes — you only ran focused tests; say 'focused tests pass, full suite not run.'"`

**When To Apply:** Change `"Committing, PR creation, task completion"` → `"Phase/task completion, hand-off."`

- [ ] Apply all edits to `skills/verification-before-completion/SKILL.md`.

### E.3 — Update `using-git-worktrees/SKILL.md`

Read the file.

**Step 3 "Verify Clean Baseline":** Replace the test-suite run (`npm test / cargo test / pytest / go test ./...`) with:
```bash
# Lint/syntax/build baseline only (test suite is run by the human)
# e.g.: npm run lint / cargo check / python -m py_compile **/*.py
```
Update the report text: `"Tests passing (<N> tests, 0 failures)"` → `"Lint/build baseline clean (tests deferred to human)"`

**Safety Verification / `.gitignore` flow (line ~86):** Change:
> `"Add to .gitignore, commit the change, then proceed."`

To:
> `"Add to .gitignore. **Do not commit** — tell the human to commit this change before proceeding."`

Also update the Quick Reference table row: `"Directory not ignored | Add to .gitignore + commit"` → `"Directory not ignored | Add to .gitignore + ask human to commit"`.

Remove `"Prevents accidentally committing worktree contents to repository."` → `"Prevents .gitignore seeing worktree contents; ask the human to commit the .gitignore entry."` (The "Why critical" label can stay.)

**Common Mistakes / Red Flags / Quick Reference (baseline test refs):** The skill still says "Proceed with failing tests without asking", "Skip baseline test verification", "Verify clean test baseline", and "Tests fail during baseline". Reword these to the baseline check now being lint/build (not a suite run): e.g. "Skip baseline lint/build verification", "Proceed with a failing lint/build baseline without asking", "Verify clean lint/build baseline". Leave the intent (don't start on a broken baseline) intact.

- [ ] Apply all edits to `skills/using-git-worktrees/SKILL.md`.

---

## F. End-of-work handoff (finishing — no git write operations)

**Files:**
- Modify: `skills/finishing-a-development-branch/SKILL.md`

### F.1 — Rewrite `finishing-a-development-branch/SKILL.md`

Read the file.

This skill is restructured from a commit/merge/PR workflow to a verify-and-hand-off workflow. Apply these edits:

**Frontmatter `description`:** change `"Use when implementation is complete, all tests pass, and you need to decide how to integrate the work - guides completion ... for merge, PR, or cleanup"` → `"Use when implementation is complete and lint/build passes, and you need to hand the work off - verifies lint/build and presents an uncommitted-changes summary for the human to commit, merge, or PR."`

**Overview core-principle line:** change `"Verify tests → Detect environment → Present options → Execute choice → Clean up."` → `"Verify lint/build → Summarize uncommitted changes → Hand off to the human."` Also update the "Announce at start" line if it references options.

**Step 1 "Verify Tests" → "Verify Lint/Syntax/Build":**
```markdown
### Step 1: Verify Lint/Syntax/Build

Run the project's lint, type-check, and build commands (do **not** run the full test suite — the human starts the testing phase):

```bash
# Examples (use project-appropriate command):
npm run lint && npm run build
cargo check && cargo clippy
python -m py_compile **/*.py && flake8
```

**If lint/build fails:**
```
Build/lint failing. Must fix before handing off:
[Show failures]
```
Stop. Don't proceed to Step 2.

**If lint/build passes:** Continue to Step 2.
```

**Step 2 "Detect Environment":** Keep as-is (needed for the summary).

**Step 3 "Determine Base Branch":** Keep as-is (needed for the diff summary).

**Step 4 "Present Options" → "Present Handoff Summary":** Replace the 4-option / 3-option menus entirely with:

```markdown
### Step 4: Present Handoff Summary

Generate a summary of uncommitted changes for the human. Use `snapshot-tree`
to capture the full working tree including **untracked files** — `git diff
--stat HEAD` shows staged and unstaged changes to tracked files but silently
omits brand-new untracked files, which are often the bulk of new work.

```bash
HEAD_TREE=$(skills/subagent-driven-development/scripts/snapshot-tree)
BASE_COMMIT=$(git rev-parse HEAD)
git status
git diff --stat "$BASE_COMMIT" "$HEAD_TREE"
```

Report:
```
Work is complete and left **uncommitted** in the working tree.

Worktree: <path>
Branch: <branch-name>
Changes since HEAD:
[git diff --stat output]

Review the changes and commit / merge / open a PR yourself.

If you want to **discard all uncommitted changes** instead, type `discard` to confirm.
```

Wait for the human's response. If they type `discard`, proceed to Step 5 (Discard). Otherwise, work is done.
```

**Step 5 "Execute Choice" → "Discard (if requested)":**

Remove Options 1, 2, 3. Keep only the discard path, rewritten:

```markdown
### Step 5: Discard Uncommitted Changes (only if human typed `discard`)

```bash
# Discard all uncommitted changes in the working tree and index
git restore .
git clean -fd
```

**Do NOT** delete branches or remove worktrees — leave that to the human.

Report: "Uncommitted changes discarded. Working tree is clean."
```

**Step 6 "Cleanup Workspace":** Remove entirely (no longer auto-removes worktrees).

**Quick Reference table:** Replace with:
```
| Outcome       | Action                                   |
|---------------|------------------------------------------|
| Lint/build OK | Generate summary, hand off to human      |
| Lint/build fail | Fix failures, then hand off            |
| Human: discard | git restore + git clean (no branch ops) |
```

**Common Mistakes:** Remove merge/push/branch-delete entries. Add:
- "Running the test suite: not the agent's job — the human starts the testing phase."
- "Committing, pushing, or merging: hand off to the human for all git write operations."

**Red Flags — Never:** Replace list with:
```
- Proceed with failing lint/build
- Run the test suite
- Create git commits, push, merge, or delete branches
- Auto-remove worktrees without the human deciding
- Discard changes without typed `discard` confirmation
```

**Red Flags — Always:**
```
- Run lint/build before handing off
- Present a clear diff summary
- Preserve the working tree for human review
- Require typed confirmation for discard
```

- [ ] Apply all edits to `skills/finishing-a-development-branch/SKILL.md`.

---

## G. Brainstorming & remaining commit references

**Files:**
- Modify: `skills/brainstorming/SKILL.md`

### G.1 — Remove commit from brainstorming spec step

Read the file.

**Checklist step 6:** Change:
> `"Write design doc — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` and commit"`

To:
> `"Write design doc — save to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` (do **not** commit — the human commits)"`

**"After the Design → Documentation" bullet:** Remove:
> `"Commit the design document to git"`

**User Review Gate message:** Change:
> `"Spec written and committed to `<path>`. Please review it…"`

To:
> `"Spec written and saved to `<path>`. Please review it…"`

(Leave "check files, docs, recent commits" context-gathering lines intact — reading history is always allowed.)

- [ ] Apply all edits to `skills/brainstorming/SKILL.md`.

---

## H. Consistency verification

**Steps (lint/syntax/grep level; the human runs any real tests):**

### H.1 — Grep for lingering git-write references

- [ ] Search `skills/` for `git commit`, `git push`, `git merge`, `git branch -d`, `git branch -D` — confirm every remaining hit is intentional (e.g., `snapshot-tree`'s throw-away `git add -A` into a temp index is allowed; `finishing-a-development-branch`'s discard path uses `git restore`/`git clean`, not `git commit`).

### H.2 — Grep for full-suite-run instructions

- [ ] Search `skills/` for `npm test`, `pytest`, `cargo test`, `go test ./...`, `run the full suite`, `full test suite` — confirm each remaining hit either (a) targets a **focused** test (allowed), or (b) is reframed as human-run / removed. A bare `pytest` or `npm test` with no target is a full-suite run and must be fixed.
- [ ] Grep the **semantic** phrases literal-command greps miss: `re-run the tests`, `already ran the tests`, `run the tests and make sure`, `all tests pass`, `test evidence`, `test output should be pristine`, `test suite`. For each, confirm it is scoped to focused tests or the human, not a full-suite run.

### H.3 — Grep for parallel-write language

- [ ] Search `skills/` for `in parallel`, `concurrently`, `same response`, `Parallel-safe`, `parallel dispatch` — confirm each remaining hit is either (a) scoped to **read-only investigation** agents (allowed), or (b) forbids parallel **file-modifying** agents. No skill should tell an agent to dispatch two writing subagents at once.

### H.4 — Verify scripts are executable

- [ ] `ls -la skills/subagent-driven-development/scripts/` — confirm all four scripts (`snapshot-tree`, `review-package`, `task-brief`, `sdd-workspace`) have the executable bit.

### H.5 — Smoke-test tree-diff mechanics

- [ ] Run `skills/subagent-driven-development/scripts/snapshot-tree` twice in quick succession; confirm both SHAs match (working tree unchanged). Run `skills/subagent-driven-development/scripts/review-package SHA SHA` and confirm the diff output is empty (identical trees). This is a read-only git operation, not a test run.

### H.6 — Hand off to human

- [ ] Report: "All skill edits applied. Lint/build-level verification complete. Run `tests/` and `evals/` yourself when ready."

---
name: finishing-a-development-branch
description: Use when implementation is complete and lint/build passes, and you need to hand the work off - verifies lint/build and presents an uncommitted-changes summary for the human to commit, merge, or PR.
---

# Finishing a Development Branch

## Overview

Verify lint/build, summarize uncommitted changes, and hand off to the human for all git write operations.

**Core principle:** Verify lint/build → Summarize uncommitted changes → Hand off to the human.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

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

### Step 2: Detect Environment

**Determine workspace state:**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
```

This determines the worktree path and branch name used in the handoff summary.

| State | Notes |
|-------|-------|
| `GIT_DIR == GIT_COMMON` (normal repo) | No linked worktree |
| `GIT_DIR != GIT_COMMON`, named branch | Linked worktree |
| `GIT_DIR != GIT_COMMON`, detached HEAD | Externally managed workspace |

### Step 3: Determine Base Branch

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

Or ask: "This branch split from main - is that correct?"

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

### Step 5: Discard Uncommitted Changes (only if human typed `discard`)

```bash
# Discard all uncommitted changes in the working tree and index
git restore .
git clean -fd
```

**Do NOT** delete branches or remove worktrees — leave that to the human.

Report: "Uncommitted changes discarded. Working tree is clean."

## Quick Reference

| Outcome | Action |
|---------|--------|
| Lint/build OK | Generate summary, hand off to human |
| Lint/build fail | Fix failures, then hand off |
| Human: discard | git restore + git clean (no branch ops) |

## Common Mistakes

**Running the test suite**
- **Problem:** Not the agent's job — the human starts the testing phase
- **Fix:** Run lint/build only; skip test execution

**Committing, pushing, or merging**
- **Problem:** These are git write operations the human owns
- **Fix:** Hand off to the human for all git write operations

**Omitting untracked files from the summary**
- **Problem:** `git diff --stat HEAD` silently omits brand-new untracked files
- **Fix:** Use `snapshot-tree` to capture the full working tree including untracked files

**No confirmation for discard**
- **Problem:** Accidentally discard work
- **Fix:** Require typed "discard" confirmation

## Red Flags

**Never:**
- Proceed with failing lint/build
- Run the test suite
- Create git commits, push, merge, or delete branches
- Auto-remove worktrees without the human deciding
- Discard changes without typed `discard` confirmation

**Always:**
- Run lint/build before handing off
- Present a clear diff summary (including untracked files via snapshot-tree)
- Preserve the working tree for human review
- Require typed confirmation for discard

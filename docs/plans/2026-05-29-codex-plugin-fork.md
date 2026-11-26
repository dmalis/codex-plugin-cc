# Codex Plugin CC Fork — Implementation Plan

> Work through this plan task by task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fork openai/codex-plugin-cc and cherry-pick 6–7 community PRs that fix GPT-5.5 support and rescue agent reliability on macOS.

**Architecture:** Soft fork with a `patched` branch layered on top of upstream `main`. PRs are applied as squashed diffs using `gh pr diff | git apply`. Two conflict groups: broker patches (#300 → #343) and rescue patches (#346 → #325).

**Tech Stack:** JavaScript (Node.js), Claude Code plugin system, GitHub CLI (`gh`)

---

## File Map

No new files are created by us — all changes come from community PRs. Files modified by the patches:

| File | PRs that touch it | Conflict risk |
|---|---|---|
| `plugins/codex/scripts/lib/app-server.mjs` | #255, #346 | HIGH |
| `plugins/codex/scripts/lib/broker-lifecycle.mjs` | #300, #343 | HIGH |
| `plugins/codex/scripts/lib/state.mjs` | #343, #297 | MEDIUM |
| `plugins/codex/agents/codex-rescue.md` | #346, #325, #319 | HIGH |
| `plugins/codex/commands/rescue.md` | #346 | LOW |
| `plugins/codex/scripts/codex-companion.mjs` | #346 | LOW |
| `plugins/codex/scripts/stop-review-gate-hook.mjs` | #297 | LOW |
| `tests/fake-codex-fixture.mjs` | #255, #346 | MEDIUM |
| `tests/runtime.test.mjs` | #255, #346 | MEDIUM |
| `tests/broker-lifecycle.test.mjs` | #300, #343 | MEDIUM |
| New: `plugins/codex/scripts/codex-rescue-completion-hook.mjs` | #346 | NONE |
| New: `plugins/codex/scripts/lib/task-status-token.mjs` | #346 | NONE |
| New: `tests/codex-rescue-hook.test.mjs` | #346 | NONE |
| New: `tests/state-concurrent-worker.mjs` | #343 | NONE |

## Patch application order

1. **#255** — GPT-5.5 clientInfo fix (1 commit, 3 files, clean on main)
2. **#300** — Broker shutdown timeout (2 commits, 2 files, clean on main)
3. **#343** — Stale broker cleanup (1 commit, 5 files, conflicts with #300 in `broker-lifecycle.mjs`)
4. **#297** — Dead-PID detection (1 commit, 2 files, may conflict with #343 in `state.mjs`)
5. **#346** — Rescue reliability pass (18 commits, 16 files, conflicts with #255 in `app-server.mjs` + test files)
6. **#325** — Rescue timeout (1 commit, 3 files, conflicts with #346 in `codex-rescue.md`)

**Dropped patches:**
- **#319** (rescue namespace fix) — Codex review determined this may break `Agent` tool routing. #346 explicitly uses `codex:codex-rescue` as the correct subagent_type. Skip.
- **#350** (rescue empty output) — #346 adds structured error handling via `hasFailureStatus()`, `FAILURE_CONTEXT`, and "include the most actionable stderr lines" instruction. Partially covered. We skip the standalone patch to avoid triple-conflict in `codex-rescue.md`.

---

### Task 1: Fork and Clone

**Files:** None (git/GitHub operations only)

- [ ] **Step 1: Fork the repo on GitHub**

```bash
gh repo fork openai/codex-plugin-cc --clone=false
```

Expected: Fork created at `<your-username>/codex-plugin-cc` on GitHub.

- [ ] **Step 2: Clone the fork locally**

A folder `codex-plugin-cc` already exists in the parent folder, holding only our spec and plan docs. Clone into a temp location and move contents:

```bash
cd <parent folder of codex-plugin-cc>
rm -rf codex-plugin-cc-tmp
git clone git@github.com:$(gh api user --jq .login)/codex-plugin-cc.git codex-plugin-cc-tmp
```

Then move the git repo into place, preserving our docs:

```bash
cp -r codex-plugin-cc/docs codex-plugin-cc-tmp/docs
rm -rf codex-plugin-cc
mv codex-plugin-cc-tmp codex-plugin-cc
cd codex-plugin-cc
```

- [ ] **Step 3: Add upstream remote and create patched branch**

```bash
git remote add upstream https://github.com/openai/codex-plugin-cc.git
git fetch upstream
git checkout -b patched main
```

- [ ] **Step 4: Install dependencies and run baseline tests**

```bash
npm install
npm test
```

Expected: All tests pass (baseline before any patches).

- [ ] **Step 5: Commit**

```bash
git add docs/
git commit -m "docs: add fork design spec and implementation plan"
```

---

### Task 2: Apply Patch #255 — GPT-5.5 Support

**Files:**
- Modify: `plugins/codex/scripts/lib/app-server.mjs:27` (change `DEFAULT_CLIENT_INFO.name`)
- Modify: `tests/fake-codex-fixture.mjs` (add `lastInitialize` tracking)
- Modify: `tests/runtime.test.mjs` (add client name regression test)

- [ ] **Step 1: Download and apply the PR diff**

```bash
gh pr diff 255 --repo openai/codex-plugin-cc > /tmp/pr-255.patch
git apply --3way /tmp/pr-255.patch
```

Expected: Clean apply. If `--3way` is needed, resolve by keeping the PR's changes.

- [ ] **Step 2: Verify the key change**

```bash
grep -n 'codex_claude_code' plugins/codex/scripts/lib/app-server.mjs
```

Expected: Line ~28 shows `name: "codex_claude_code"`.

- [ ] **Step 3: Run tests**

```bash
npm test
```

Expected: All tests pass, including the new `setup initializes the app server with a codex-namespaced client name` test.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "fix: rename clientInfo.name to codex_claude_code for GPT-5.5 compat (PR #255)

Fixes openai/codex-plugin-cc#276, openai/codex-plugin-cc#199"
```

---

### Task 3: Apply Patch #300 — Broker Shutdown Timeout

**Files:**
- Modify: `plugins/codex/scripts/lib/broker-lifecycle.mjs` (add 5s socket timeout to `sendBrokerShutdown`)
- Create: `tests/broker-lifecycle.test.mjs` (or modify if it exists — add unresponsive broker test)

- [ ] **Step 1: Download and apply the PR diff**

```bash
gh pr diff 300 --repo openai/codex-plugin-cc > /tmp/pr-300.patch
git apply --3way /tmp/pr-300.patch
```

Expected: Clean apply — this PR doesn't overlap with #255.

- [ ] **Step 2: Verify the key change**

```bash
grep -n 'timeout\|setTimeout' plugins/codex/scripts/lib/broker-lifecycle.mjs
```

Expected: Shows a timeout (e.g., 5000ms) in the `sendBrokerShutdown` function.

- [ ] **Step 3: Run tests**

```bash
npm test
```

Expected: All tests pass, including the new broker shutdown timeout test.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "fix: add 5s timeout to sendBrokerShutdown (PR #300)

Fixes openai/codex-plugin-cc#288"
```

---

### Task 4: Apply Patch #343 — Stale Broker Cleanup

**Files:**
- Modify: `plugins/codex/scripts/lib/broker-lifecycle.mjs` (stale broker detection + cleanup)
- Modify: `plugins/codex/scripts/lib/state.mjs` (lock state updates)
- Create: `tests/state-concurrent-worker.mjs`
- Modify: `tests/broker-lifecycle.test.mjs`
- Modify: `tests/state.test.mjs`

**Conflict note:** This PR touches `broker-lifecycle.mjs` which was just modified by #300. Conflicts are likely.

- [ ] **Step 1: Download and apply the PR diff**

```bash
gh pr diff 343 --repo openai/codex-plugin-cc > /tmp/pr-343.patch
git apply --3way /tmp/pr-343.patch
```

If `git apply --3way` fails with conflicts:

```bash
git apply --3way --reject /tmp/pr-343.patch 2>&1 || true
```

Then manually resolve `.rej` files in `broker-lifecycle.mjs`. The #300 changes (timeout) and #343 changes (stale cleanup) are in different functions, so the conflict is likely just context lines. Keep both changes.

- [ ] **Step 2: Clean up any .rej files**

```bash
find . -name '*.rej' -type f
```

If any exist, read them to understand the rejected hunks and apply manually. Then delete the `.rej` files:

```bash
find . -name '*.rej' -type f -delete
```

- [ ] **Step 3: Run tests**

```bash
npm test
```

Expected: All tests pass.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "fix: clean up stale brokers and lock state updates (PR #343)

Fixes openai/codex-plugin-cc#286, openai/codex-plugin-cc#108"
```

---

### Task 5: Apply Patch #297 — Dead-PID Detection

**Files:**
- Modify: `plugins/codex/scripts/lib/state.mjs` (add PID liveness check)
- Modify: `plugins/codex/scripts/stop-review-gate-hook.mjs` (treat dead-PID jobs as inactive)

**Conflict note:** This PR touches `state.mjs` which was modified by #343. Minor conflict possible.

- [ ] **Step 1: Download and apply the PR diff**

```bash
gh pr diff 297 --repo openai/codex-plugin-cc > /tmp/pr-297.patch
git apply --3way /tmp/pr-297.patch
```

If conflicts arise in `state.mjs`, the #343 changes (lock state) and #297 changes (PID check) are in different functions. Keep both.

- [ ] **Step 2: Clean up any .rej files**

```bash
find . -name '*.rej' -type f
```

If any exist, resolve manually then delete them.

- [ ] **Step 3: Run tests**

```bash
npm test
```

Expected: All tests pass.

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "fix: treat jobs as inactive when PID is dead (PR #297)

Fixes openai/codex-plugin-cc#164, openai/codex-plugin-cc#222"
```

---

### Task 6: Apply Patch #346 — Rescue Reliability Pass

**Files (16 files, 2515 additions — the biggest patch):**
- Modify: `plugins/codex/agents/codex-rescue.md`
- Modify: `plugins/codex/commands/rescue.md`
- Modify: `plugins/codex/hooks/hooks.json`
- Modify: `plugins/codex/scripts/codex-companion.mjs`
- Create: `plugins/codex/scripts/codex-rescue-completion-hook.mjs`
- Modify: `plugins/codex/scripts/lib/app-server.mjs`
- Modify: `plugins/codex/scripts/lib/codex.mjs`
- Modify: `plugins/codex/scripts/lib/render.mjs`
- Create: `plugins/codex/scripts/lib/task-status-token.mjs`
- Modify: `plugins/codex/skills/codex-cli-runtime/SKILL.md`
- Modify: `plugins/codex/skills/codex-result-handling/SKILL.md`
- Create: `tests/codex-rescue-hook.test.mjs`
- Modify: `tests/commands.test.mjs`
- Modify: `tests/fake-codex-fixture.mjs`
- Modify: `tests/render.test.mjs`
- Modify: `tests/runtime.test.mjs`

**Conflict note:** This PR modifies `app-server.mjs` (also touched by #255) and test files (`fake-codex-fixture.mjs`, `runtime.test.mjs`). Expect conflicts in those files. The #255 change was a one-liner (rename `DEFAULT_CLIENT_INFO.name`); #346's changes are in different parts of the file. Keep both.

- [ ] **Step 1: Download and apply the PR diff**

```bash
gh pr diff 346 --repo openai/codex-plugin-cc > /tmp/pr-346.patch
git apply --3way /tmp/pr-346.patch
```

This is the most conflict-prone step. If `--3way` reports conflicts:

```bash
git status
```

Check for conflicted files. Likely conflicts in:
- `plugins/codex/scripts/lib/app-server.mjs` — #255 changed `name: "codex_claude_code"` on line 28. #346 edits different parts of the file. Keep both changes.
- `tests/fake-codex-fixture.mjs` — #255 added `lastInitialize` field. #346 may add other state fields. Keep both.
- `tests/runtime.test.mjs` — #255 added a client name test. #346 adds other tests. Keep both.

For each conflicted file, resolve by keeping BOTH our prior patches AND the #346 changes (they don't semantically conflict).

- [ ] **Step 2: Clean up any .rej files**

```bash
find . -name '*.rej' -type f
```

If any exist, read each one, apply the rejected hunk manually to the correct file, then delete:

```bash
find . -name '*.rej' -type f -delete
```

- [ ] **Step 3: Verify key changes exist**

```bash
# Our GPT-5.5 fix from #255 survived:
grep -n 'codex_claude_code' plugins/codex/scripts/lib/app-server.mjs

# New completion hook exists:
test -f plugins/codex/scripts/codex-rescue-completion-hook.mjs && echo "OK"

# New status token module exists:
test -f plugins/codex/scripts/lib/task-status-token.mjs && echo "OK"

# PostToolUse hook registered:
grep -c 'PostToolUse' plugins/codex/hooks/hooks.json
```

Expected: All checks pass.

- [ ] **Step 4: Run tests**

```bash
npm test
```

Expected: All tests pass. If test failures occur due to merge issues, fix the specific test file and re-run.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "fix: rescue reliability pass — completion signals, lifecycle hardening (PR #346)

Adds PostToolUse hook for codex-rescue completion context, structured
task status tokens, worker exit detection, and background-first
dispatch default."
```

---

### Task 7: Apply Patch #325 — Rescue Timeout

**Files:**
- Modify: `plugins/codex/agents/codex-rescue.md`
- Modify: `plugins/codex/skills/codex-cli-runtime/SKILL.md`
- Modify: `tests/commands.test.mjs`

**Conflict note:** `codex-rescue.md` was substantially rewritten by #346. The #325 change adds `timeout: 600000` to the Bash call instruction. This may need manual application if the exact context lines have changed.

- [ ] **Step 1: Download and inspect the PR diff**

```bash
gh pr diff 325 --repo openai/codex-plugin-cc > /tmp/pr-325.patch
cat /tmp/pr-325.patch
```

Check what #325 adds. The core change is: the rescue agent's Bash call to `codex-companion.mjs task` must include `timeout: 600000`.

- [ ] **Step 2: Attempt to apply**

```bash
git apply --3way /tmp/pr-325.patch
```

If this fails due to #346 having rewritten `codex-rescue.md`, apply manually:

1. Open `plugins/codex/agents/codex-rescue.md`
2. Find the instruction about the Bash call to `codex-companion.mjs task`
3. Add a line: `- **Always pass `timeout: 600000` on the Bash tool call.** This is Claude Code's Bash max and keeps the call foreground for the full codex run.`
4. Similarly update `plugins/codex/skills/codex-cli-runtime/SKILL.md` if the patch touched it

- [ ] **Step 3: Check if #325's changes are already covered by #346**

#346 changed the rescue agent to default to `--background` (which avoids the timeout issue entirely for the default path). However, `--wait` (foreground) tasks still need the explicit timeout. Check:

```bash
grep -i 'timeout.*600000\|600000.*timeout' plugins/codex/agents/codex-rescue.md
```

If #346 already includes this, skip the manual edit. If not, add it.

- [ ] **Step 4: Run tests**

```bash
npm test
```

Expected: All tests pass.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "fix: pass timeout: 600000 on rescue Bash call (PR #325)

Prevents Claude Code 2-minute auto-background from orphaning foreground
Codex output on --wait tasks.

Fixes openai/codex-plugin-cc#324"
```

---

### Task 8: Run Full Test Suite and Push

- [ ] **Step 1: Run the full test suite one final time**

```bash
npm test
```

Expected: All tests pass.

- [ ] **Step 2: Review the commit log**

```bash
git log --oneline patched ^main
```

Expected: 7 commits (docs + 6 patches) on top of `main`.

- [ ] **Step 3: Push the patched branch**

```bash
git push -u origin patched
```

- [ ] **Step 4: Also push main for completeness**

```bash
git push -u origin main
```

---

### Task 9: Install Fork as Local Plugin

- [ ] **Step 1: Check current plugin installation**

```bash
claude plugin list 2>/dev/null || echo "Check manually"
```

- [ ] **Step 2: Remove the upstream plugin**

```bash
claude plugin remove codex
```

- [ ] **Step 3: Install the fork**

```bash
claude plugin install <path to your codex-plugin-cc clone>
```

If local path install isn't supported, install from GitHub:

```bash
claude plugin install $(gh api user --jq .login)/codex-plugin-cc
```

- [ ] **Step 4: Verify installation**

Start a new Claude Code session and run:

```
/codex:setup
```

Expected output should show:
- Codex CLI installed
- Auth: ChatGPT login active
- GPT-5.5 should NOT produce HTTP 400 errors

- [ ] **Step 5: Smoke test rescue**

In the same session:

```
/codex:rescue explain what the codex-plugin-cc repo does in 3 sentences
```

Expected: Codex processes the request and returns a response (not empty, not a stub, not a hang).

- [ ] **Step 6: Verify no orphaned processes after session end**

Exit the Claude Code session, then:

```bash
ps aux | grep -E 'codex|app-server' | grep -v grep
```

Expected: No orphaned codex or app-server processes.

# Codex Plugin CC — Fork & Patch Design

## Goal

Fork `openai/codex-plugin-cc` (v1.0.4) and cherry-pick community PRs that fix GPT-5.5 compatibility and rescue agent reliability. macOS only. Minimum viable fix set — no original code, no new features.

## Context

- Upstream repo: `openai/codex-plugin-cc` — 122 open issues, 94 open PRs, many with working fixes sitting unmerged
- User has GPT-5.5 subscription; the plugin doesn't work with it due to a `clientInfo.name` bug
- Primary use case: task delegation via `/codex:rescue`
- The rescue agent has multiple reliability issues: hangs, empty output, orphaned processes

## Fork Strategy

**Soft fork** — stay close to upstream structure, no renaming, no restructuring.

- Fork `openai/codex-plugin-cc` to user's personal GitHub (same repo name)
- `main` branch: synced with upstream
- `patched` branch: cherry-picked fixes on top of upstream main
- If upstream ships fixes later, rebase `patched` onto updated `main`

## Patches (Revised Order — Post Codex Review)

> **Codex review findings incorporated:** reordered to apply broad changes before narrow ones,
> flagged #319 as needing verification, grouped broker patches together.

### Patch 1: GPT-5.5 Support
- **Issues:** #276, #199
- **PR:** #255
- **File:** `plugins/codex/scripts/lib/app-server.mjs`
- **Change:** Rename `DEFAULT_CLIENT_INFO.name` from `"Claude Code"` to `"codex_claude_code"`
- **Impact:** Fixes HTTP 400 `invalid_request_error` when using gpt-5.5. THE primary blocker.
- **Tests:** PR includes regression test capturing the initialize payload.

### Patch 2: Broker Shutdown Timeout
- **Issues:** #288
- **PR:** #300
- **File:** `plugins/codex/scripts/lib/broker-lifecycle.mjs`
- **Change:** Add 5-second socket-level timeout to `sendBrokerShutdown` with guarded cleanup
- **Impact:** Prevents `SessionEnd` from hanging indefinitely when broker is unresponsive.
- **Tests:** Regression test simulating unresponsive broker.
- **Conflict group:** Broker (apply before #343).

### Patch 3: Stale Broker Cleanup
- **Issues:** #286, #108
- **PR:** #343
- **Files:** `plugins/codex/scripts/lib/broker-lifecycle.mjs`, `plugins/codex/scripts/lib/state.mjs`
- **Change:** Clean up stale brokers and lock state updates on session start
- **Impact:** Eliminates orphaned broker processes accumulating across sessions.
- **Conflict group:** Broker (apply after #300, resolve conflicts in `broker-lifecycle.mjs`).

### Patch 4: Dead-PID Detection
- **Issues:** #164, #222
- **PR:** #297
- **File:** `plugins/codex/scripts/stop-review-gate-hook.mjs`
- **Change:** Treat job as inactive when its PID is dead (`kill -0` check)
- **Impact:** Jobs whose process died no longer block subsequent task calls forever.

### Patch 5: Rescue Reliability Pass (BROAD — apply before narrow rescue patches)
- **Issues:** Multiple rescue issues
- **PR:** #346
- **Files:** `plugins/codex/agents/codex-rescue.md`, `plugins/codex/scripts/codex-companion.mjs`
- **Change:** Completion signal hardening, launch/cancel edge cases, worker lifecycle
- **Impact:** Comprehensive rescue agent reliability improvement.
- **Conflict group:** Rescue (apply before #325 and #350).

### Patch 6: Rescue Timeout Fix
- **Issues:** #324
- **PR:** #325
- **File:** `plugins/codex/agents/codex-rescue.md`
- **Change:** Add `timeout: 600000` instruction for the inner Bash call to `codex-companion.mjs task`
- **Impact:** Prevents Claude Code's 2-minute auto-background from orphaning Codex output.
- **Conflict group:** Rescue (apply after #346, may need manual merge in `codex-rescue.md`).

### Patch 7: Rescue Empty Output Fix
- **Issues:** #350
- **Related to:** PR #346
- **File:** `plugins/codex/agents/codex-rescue.md`
- **Change:** Forward stderr and exit code on Bash failure
- **Impact:** Errors are surfaced to the user instead of silently returning nothing.
- **Conflict group:** Rescue (apply after #346; skip if #346 already includes this fix).

### Patch 8: Rescue "Unknown Skill" Fix (VERIFY FIRST)
- **Issues:** #266
- **PR:** #319
- **File:** `plugins/codex/commands/rescue.md`
- **Change:** Remove `codex:` namespace prefix from 3 subagent references (`codex:codex-rescue` → `codex-rescue`)
- **Impact:** Fixes 22% failure rate on `/codex:rescue` invocations.
- **Risk:** Codex review flagged that current `main` uses `subagent_type: "codex:codex-rescue"` via `Agent` tool, which is the correct namespaced format. **Verify whether this PR is actually correct before applying.** If the `codex:` prefix is required for `Agent` tool routing, skip this patch.

## Implementation Plan

1. Fork the repo on GitHub
2. Clone to `/Users/damjan/development/projects/codex-plugin-cc`
3. Create `patched` branch from `main`
4. Cherry-pick PRs in revised order: #255, #300, #343, #297, #346, #325, #350
5. Group broker patches (#300 then #343) — resolve conflicts in `broker-lifecycle.mjs`
6. Group rescue patches (#346 then #325 then #350) — resolve conflicts in `codex-rescue.md`
7. Check if PR #346 already includes the stderr/exit-code forwarding from #350. If yes, skip #350.
8. Verify PR #319 (namespace fix) — check if `codex:codex-rescue` is the correct Agent subagent_type format. Apply only if verified as a real bug.
9. Run `npm test` after each patch to catch regressions
10. Install locally: `claude plugin remove codex && claude plugin install .`
11. Smoke test: `/codex:setup` → verify GPT-5.5 auth
12. Smoke test: `/codex:rescue` with a real task

## Testing

- **Automated:** `npm test` — existing test suite (87 tests as of v1.0.4)
- **Smoke test 1:** `/codex:setup` — verify node, npm, codex CLI, GPT-5.5 auth
- **Smoke test 2:** `/codex:rescue "explain what this repo does"` — verify end-to-end rescue flow
- **Smoke test 3:** Exit Claude Code session, verify no orphaned `codex` or `app-server` processes

## Risks

- **Cherry-pick conflicts (broker group):** PRs #300 and #343 both edit `broker-lifecycle.mjs`. Apply #300 first, then #343 with manual conflict resolution.
- **Cherry-pick conflicts (rescue group):** PRs #346, #325, and #350 all edit `codex-rescue.md`. Apply #346 first (broad pass), then layer #325 and #350 on top.
- **PR #319 may be wrong:** Codex review flagged that `codex:codex-rescue` may be the correct `Agent` subagent_type namespace. Verify before applying.
- **PR quality:** Community PRs are reviewed by commenters but not maintainers. Tests mitigate this.
- **Upstream divergence:** If upstream ships v1.0.5, our patches may need rebasing. Maintenance is non-trivial given the overlapping rescue/broker patches.
- **Testing gaps:** `npm test` does not cover Claude Code permission behavior, Agent namespace resolution, or foreground timeout behavior. Smoke tests partially cover these.

## Out of Scope

- Windows compatibility fixes (~20 issues)
- New features (/codex:test, /codex:implement, --context flag, etc.)
- Rate-limit circuit breaker (#306)
- Architecture changes
- Code review flow fixes (we're focused on task delegation)

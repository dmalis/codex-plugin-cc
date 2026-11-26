# codex-plugin-cc

A fork of [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc): a Claude Code plugin that runs the local Codex CLI for reviews and delegated tasks. Branch `main` mirrors upstream. Branch `patched` holds this fork's fixes on top of it and is what users install. There is no deployed service.

## Setup and commands

- Node 18.18 or later (CI uses Node 22). Install: `npm ci`.
- Test: `npm test`. One file: `node --test tests/<name>.test.mjs`. Tests use the fake Codex in `tests/fake-codex-fixture.mjs`, so they need no Codex install or login.
- Type check: `npm run build`. Its `prebuild` step runs `codex app-server generate-ts`, so it needs the Codex CLI on `PATH` (`npm install -g @openai/codex`). The output goes to `plugins/codex/.generated/`, which git ignores.
- Version: `npm run bump-version -- <version>` sets it in `package.json`, `package-lock.json`, `plugins/codex/.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json`. `npm run check-version` verifies they match.

## Layout

- `.claude-plugin/marketplace.json`: the marketplace entry that points at `plugins/codex`.
- `plugins/codex/`: the plugin itself. Slash commands in `commands/`, the `codex-rescue` subagent in `agents/`, skills in `skills/`, hook wiring in `hooks/hooks.json`.
- `plugins/codex/scripts/codex-companion.mjs`: the runtime entry point the slash commands call. The hook scripts (`*-hook.mjs`) sit beside it; shared code is in `scripts/lib/`.

## Rules for this repo

- Change behaviour on a branch from `patched`. Keep `main` equal to upstream, so upstream changes merge in cleanly.
- Change a version only with `npm run bump-version`, never by hand in one manifest.

## Where to read more

- `README.md`: what each slash command does, for users.
- `docs/specs/` and `docs/plans/`: why the fork exists and which upstream pull requests it carries.

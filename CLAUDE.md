# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Deadname Remover is a browser extension (Chrome + Firefox) that replaces deadnames with chosen names on web pages. Built with [WXT](https://wxt.dev) (extension framework), Svelte 5, TypeScript, UnoCSS/Onu UI, and Valibot for validation. Bun is the package manager; all processing is local/on-device by design (privacy-first).

## Commands

```bash
bun install              # install deps (runs `wxt prepare`, which generates .wxt/ types)
bun run dev              # dev server + browser, Chromium (CHROME_PATH=<path> to pick a browser)
bun run dev:firefox      # dev server, Firefox
bun run build            # production build (add :firefox for Firefox)
bun run lint             # ESLint
bun run check            # svelte-check + tsc --noEmit (type checking)
bun run test             # Vitest (watch mode)
bunx vitest run                                      # run tests once
bunx vitest run services/__tests__/siteFiltering.test.ts  # run a single test file
```

Before submitting a PR, run `bun run check`, `bun run lint`, and `bun run test` — all three must pass (CI runs lint).

Type checking depends on generated files in `.wxt/`; if types are missing, run `bun install` or `bunx wxt prepare`. The `@/` import alias resolves to the repo root (configured by WXT).

## Architecture

WXT uses file-based entrypoints in `entrypoints/`; shared logic lives in `services/` and `utils/`.

### Runtime pieces and how they coordinate

- **Content script** ([entrypoints/content/index.ts](entrypoints/content/index.ts)) — runs at `document_start` on all URLs. `configureAndRunProcessor()` is the orchestrator: loads config, checks site filtering, optionally blocks page content to prevent deadname flash (`blockContentBeforeDone`), runs the initial document pass, then attaches a `DOMObserver` for mutations. It re-runs on every config change (via `setupConfigListener`) and diffs previous vs. new state (enabled/names/theme/highlight) to decide what to tear down and reapply. `TextProcessor.revertAllReplacements()` restores original text when disabling or when names change.

- **Background service worker** ([entrypoints/background/index.ts](entrypoints/background/index.ts)) — tracks the active tab (persisted to survive service-worker restarts), updates the extension icon/badge appearance (including stealth mode), and handles install/update events.

- **Parsing-status protocol**: the popup shows whether the current site is being parsed. Content scripts don't write this status directly on load/config change — they send a `CANDIDATE_PARSING_STATUS` message, and the background commits it only if the sender is the active tab (prevents inactive tabs from clobbering status). On tab activation, background sends `RECHECK_PARSING_STATUS` and the content script then writes status directly (`'direct'` mode). Message types are in [utils/types.ts](utils/types.ts).

- **Popup** (`entrypoints/popup/`) — quick toggle + status; **Options** (`entrypoints/options/`) — full settings UI (name mappings, site filtering, themes). Both are Svelte apps.

### Services

- [services/textProcessor.ts](services/textProcessor.ts) — the replacement engine. Builds Unicode-aware regexes with letter-boundary lookarounds; case matching is intentionally approximate (all-upper, all-lower, or first-letter-capitalized — see comment in file). Skips form elements, contenteditable, and other excluded attributes/roles to avoid corrupting user input. Tracks replaced nodes so replacements are revertible.
- [services/domObserver.ts](services/domObserver.ts) — MutationObserver (childList + characterData) that runs a cheap regex pre-check (`hasAnyMatch`) before processing, scheduled via `requestIdleCallback`.
- [services/siteFiltering.ts](services/siteFiltering.ts) — allowlist/blocklist with wildcard support and `defaultAllowMode`; also owns parsing-status storage.
- [services/configService.ts](services/configService.ts) — settings storage and migrations. Settings live in `local:nameConfig` or `sync:nameConfig` depending on `syncSettingsAcrossDevices`; the service handles moving between them.

### Changing the UserSettings shape

Follow [docs/CONFIG_CHANGES.md](docs/CONFIG_CHANGES.md) exactly — this is versioned storage with sequential migrations. Any change requires: bumping `CURRENT_CONFIG_VERSION` in `configService.ts` (with a version-history comment noting the extension release), adding a `UserSettingsStorageVersionN` interface in [utils/types.ts](utils/types.ts) extending the previous version, a migration function, updated `defaultSettings`, and updated Valibot schema in [utils/validations.ts](utils/validations.ts).

## Conventions

- Tests are colocated in `__tests__/` directories next to the code under test.
- ESLint uses `@stylistic` for formatting (no separate formatter) — run `bun run lint` rather than hand-formatting.
- Contributions are expected to be discussed in an issue and approved before a PR is opened (see CONTRIBUTING.md).

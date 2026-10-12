---
name: decky-plugin-dev
description: Hard-won technical knowledge for developing and maintaining Decky Loader plugins for Steam Deck (Python backend + React/TS frontend using @decky/ui and @decky/api). Use this whenever working on ANY Decky Loader plugin, not just one specific project - especially when adding a Python dependency, debugging why a backend RPC method isn't receiving data the frontend clearly sent, implementing a Settings UI control whose spacing or selected-value doesn't behave as expected, detecting Steam app types (game vs tool/demo/server) from the frontend, reading Steam's local data files (localconfig.vdf, shortcuts.vdf, appmanifest .acf), hooking into SteamClient/appStore frontend APIs, or setting up a plugin's GitHub Actions CI/release pipeline. Consult this before guessing at a fix for a "my change had no visible effect" symptom in a Decky plugin - that exact symptom has several known, non-obvious causes documented here.
---

# Decky Loader plugin development

This skill holds lessons from real Decky Loader plugin debugging sessions, organized so a
cold agent can skim this file in under a minute and know which reference file to open for
the deep detail. It applies to any Decky plugin workspace, not one specific repo.

## How to use this

1. Skim the "Quick diagnosis" table below first - it maps symptoms to likely causes and the
   reference file with the fix.
2. Read the relevant `references/*.md` file before implementing a fix, not after guessing.
3. The single biggest time-saver in past sessions: when something "has no visible effect" or
   "isn't receiving data," the cause is almost always one of a handful of known mechanisms
   below, not a mistake in your own logic. Identify *which* mechanism before changing a
   number and re-testing - see "Debugging approach" at the end of this file.

## Quick diagnosis

| Symptom | Likely cause | Read |
|---|---|---|
| Backend method seems to never run / silent failure with no log output | RPC call shape mismatch - method doesn't use the `_extract_params` pattern and was called with a single object instead of positional args | `references/rpc-bridge.md` |
| A field the frontend "definitely sends" always reads as missing/`False` on the backend | Nested wrong - frontend put it one level deeper (e.g. inside a nested data object) than where the backend reads it | `references/rpc-bridge.md` |
| `ModuleNotFoundError` for a third-party package at runtime on the Deck | Decky does **not** install `requirements.txt` - must vendor the package yourself | `references/python-runtime.md` |
| `ModuleNotFoundError` for a *stdlib* module (e.g. `html.parser`) | The frozen Decky Python 3.11 runtime ships a pruned stdlib, not the full one | `references/python-runtime.md` |
| Vendored package still not importable after adding to `sys.path` | It's a stdlib *package* already cached in `sys.modules` by Decky's own bootstrap - needs `__path__` extension, not `sys.path` | `references/python-runtime.md` |
| `[SSL: CERTIFICATE_VERIFY_FAILED]` on outbound HTTPS from the backend | `aiohttp` caches its SSL context at import time; patching `ssl.create_default_context` before import doesn't help | `references/python-runtime.md` |
| Non-Steam game's playtime/last-played lookup in `localconfig.vdf` returns nothing | Wrong appid sign - that file keys non-Steam apps by the *signed* 32-bit appid, not the usual unsigned one | `references/steam-data.md` |
| Need to tell real games apart from tools/servers/demos when scanning the library | Use `appStore.GetAppOverviewByAppID(appid).app_type` bitmask, not a name-based heuristic | `references/steam-data.md` |
| Gamepad focus ring invisible on a button inside a custom modal | `DialogButton` doesn't reliably render focus there - swap to `Focusable` | `references/decky-ui-quirks.md` |
| A `Dropdown` never shows anything selected even though state has a valid value | `selectedOption` must exactly match one option's `data` value, including your "default" | `references/decky-ui-quirks.md` |
| Changed a flex `gap` to tighten/loosen spacing between `TextField`/`Dropdown`/etc. and nothing visibly changed | Those components carry their own intrinsic margin that `gap` doesn't touch | `references/decky-ui-quirks.md` |
| Added `marginBottom`/`marginTop` to add space between two `PanelSectionRow`s and nothing changed | Margin collapse near the row boundary - use `borderTop`/`paddingTop` instead, padding can't collapse | `references/decky-ui-quirks.md` |
| A full-screen modal renders shrunk down to some small wrapper's size instead of covering the screen | An ancestor's CSS `transform` became the modal's containing block - move the modal to be a sibling, not a descendant | `references/decky-ui-quirks.md` |
| Whole plugin fails to load after a SteamUI update, not just one feature | A `findModuleChild`/`findModuleExport` scan at module scope threw uncaught (Steam's webpack shape changed) | `references/decky-ui-quirks.md` |
| CI release workflow fails with `ERR_PNPM_IGNORED_BUILDS` or a 403 creating the release | pnpm 10+ build-script allowlist, or missing `permissions: contents: write` | `references/ci-release.md` |
| Users on `releases/latest/download/...` suddenly get a 404 for the asset name | The fixed-name asset wasn't re-published on the latest release | `references/ci-release.md` |

## Debugging approach that reliably works here

When a change appears to do nothing, don't bump the number and retry - that burns a full
build+deploy+test round trip per guess, and in past sessions repeated guessing without first
identifying the mechanism was consistently slower than reasoning it out. Instead:

- **For a UI value that "didn't apply":** ask whether something else owns that visual space
  (a component's own intrinsic margin, CSS margin collapse) before assuming your own code is
  wrong. See `references/decky-ui-quirks.md`.
- **For a backend "not receiving" a value the frontend clearly sends:** log the exact payload
  on *both* sides of the call - right before the `call()` in JS, and at the very top of the
  Python method - rather than only inspecting one side. The bug is just as likely to be a
  shape/nesting mismatch between the two as a logic error in either one. See
  `references/rpc-bridge.md`.
- **For a fire-and-forget diagnostic `call(...)`:** if you're not `await`ing it and checking
  the result, a backend exception becomes a silent unhandled promise rejection - you'll see
  nothing even though the call genuinely failed. Await and surface the result while debugging.

## Reference files

- `references/rpc-bridge.md` - how `call()` maps to Python method params, the `_extract_params`
  pattern, and payload-nesting bugs
- `references/python-runtime.md` - vendoring deps, the frozen Python 3.11 runtime, stdlib
  pruning/shims, the aiohttp SSL context caching gotcha
- `references/steam-data.md` - reading `localconfig.vdf`/`shortcuts.vdf`, the non-Steam appid
  sign flip, detecting app type, and which `appStore`/`SteamClient` APIs to use for what
- `references/decky-ui-quirks.md` - `@decky/ui` component behaviors that don't match naive
  expectations (focus rings, `Dropdown` selection, spacing/margin collapse, `ButtonItem` style,
  `transform` containing-block, module-scanning fragility)
- `references/ci-release.md` - GitHub Actions/pnpm pitfalls and the stable-download-alias
  pattern for installing a plugin without the Decky Store

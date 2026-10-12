# claude-skill-decky-plugin-dev

Claude Code skill with hard-won fixes for Decky Loader (Steam Deck) plugin development — the
RPC bridge between JS and Python, `@decky/ui` component quirks, the frozen Python 3.11 runtime,
reading Steam's local data files, and CI/release pitfalls.

## What's in here

- `SKILL.md` — quick-diagnosis table mapping common symptoms ("my change had no visible
  effect", "the backend isn't receiving this field") to their known cause and fix.
- `references/` — the deeper detail behind each topic:
  - `rpc-bridge.md` — how `call()` maps to Python method params, and payload-nesting bugs
  - `python-runtime.md` — vendoring deps, the frozen Python 3.11 runtime, stdlib shims, aiohttp SSL
  - `steam-data.md` — `localconfig.vdf`/`shortcuts.vdf`, the non-Steam appid sign flip, app-type detection
  - `decky-ui-quirks.md` — `@decky/ui` component behaviors that don't match naive expectations
  - `ci-release.md` — GitHub Actions/pnpm pitfalls and the stable-download-alias pattern

## Installing

Drop this repo's contents into your Claude Code personal skills directory:

```bash
git clone https://github.com/btd1337/claude-skill-decky-plugin-dev.git ~/.claude/skills/decky-plugin-dev
```

Claude Code picks up skills from `~/.claude/skills/` automatically — no further setup needed.
The skill triggers when you're working on a Decky Loader plugin and hit one of the symptoms it
covers.

## License

MIT - see [LICENSE](LICENSE).

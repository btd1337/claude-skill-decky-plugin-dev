# Reading Steam's own local data and frontend APIs

## Non-Steam game appids: two different representations depending on the file

Non-Steam shortcut appids are stored differently depending on which file you're reading:

- `shortcuts.vdf` and anything user-facing (URLs, the Decky overview API, `app_type` lookups)
  use the **unsigned/positive** 32-bit CRC-based appid - typically a very large number, e.g.
  `> 2,000,000,000`.
- `localconfig.vdf`'s `apps{}` block - which holds the real playtime/last-played data Valve's
  own client writes - keys non-Steam entries by the **signed/negative** 32-bit interpretation of
  that same appid (e.g. `-994305645`).

If you're looking up playtime for a non-Steam shortcut by appid in `localconfig.vdf` and getting
nothing, convert to signed 32-bit first before using it as the key.

`localconfig.vdf` also does **not** have a reliable `LastPlayed` timestamp for non-Steam
shortcuts (only for real Steam games) - don't design a feature assuming it exists for non-Steam
apps.

## Real-time non-Steam playtime tracking

Steam's own `minutes_playtime_forever` field (from `appStore`) is unreliable/stale for
non-Steam shortcuts. For accurate real-time tracking, use
`SteamClient.GameSessions.RegisterForAppLifetimeNotifications` from the frontend to detect
actual session start/stop, and accumulate your own session seconds as a supplement to (or
fallback for) whatever Steam itself reports - don't rely solely on Steam's stored value for
non-Steam apps.

## Detecting whether a Steam app is an actual game vs. a tool/utility

A library scan of "all owned apps" picks up non-game entries - dedicated servers, SDKs,
redistributables, demos, soundtracks, etc. - that pollute any UI meant for real games.

The reliable signal is the `app_type` field Steam's frontend exposes on
`window.appStore.GetAppOverviewByAppID(appid).app_type` - a bitmask. Known bit values observed
in practice:

```
Game = 1
Application = 2
Tool = 4
Demo = 8
Guide = 64
Driver = 128
Config = 256
Hardware = 512
Video = 2048
Plugin = 4096
Music = 8192
```

Two important nuances:

- Treat a **missing/falsy** `app_type` as "unknown, don't filter" rather than "definitely not a
  game" - some entries (certain non-Steam shortcuts) don't populate it, and filtering on absence
  risks hiding a real game due to incomplete overview data.
- If the `Game` bit is set, never treat the entry as a tool even if other bits are also set
  alongside it.

## Useful `appStore`/`SteamClient` frontend APIs

- `window.appStore.GetAppOverviewByAppID(appid)` is the general-purpose source for a lot of
  per-app data: name (`display_name`), playtime (`minutes_playtime_forever`), last-played
  (`rt_last_time_played`), and `app_type`. It works for installed **and**
  uninstalled-but-owned games, unlike anything reading the filesystem, which only sees what's
  actually downloaded.
- For the full owned-games list (not just installed), try sources in this order of reliability,
  falling back if one returns empty:
  1. `SteamClient.Apps.GetAllApps()` - most reliable primary source.
  2. `collectionStore.allAppsCollection.allApps`
  3. `collectionStore.allGamesCollection.allApps`
  4. `appStore.allApps` / `appStore.m_mapApps`

  Early in plugin load, Steam's own caches may not be warm yet and even the primary source can
  return empty - worth a retry-with-backoff loop rather than a single attempt before falling
  back to a weaker source.

## Parsing `shortcuts.vdf` (binary, non-Steam games list)

There is no official parser for this in the stdlib or most Python VDF libraries (which handle
the *text* VDF format, not this binary one). Expect to hand-roll a minimal binary parser keyed
on marker bytes, e.g. `\x01appname\x00` and `\x02appid\x00` preceding each field's value.

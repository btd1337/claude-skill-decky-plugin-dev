# Decky Loader's Python runtime constraints

## Decky does not install `requirements.txt`

Decky Loader does **not** run `pip install -r requirements.txt` for a plugin. If the Python
backend imports a third-party package, you must vendor it into the plugin's own files (e.g. a
`py_modules/vendor/` directory added to `sys.path` at startup) - don't assume a `requirements.txt`
in the repo gets installed anywhere on the Deck.

## The backend runs on a frozen Python 3.11, not the Deck's system Python

Decky Loader bundles its own **PyInstaller-frozen Python 3.11** and runs every plugin's backend
on that, not the Steam Deck OS's system `python3` (which may be a different/newer version, e.g.
3.14). Any wheels you vendor must target this exact runtime. When building/downloading vendor
wheels from a different OS/architecture (e.g. a macOS dev machine), pin the target explicitly
rather than letting pip guess from the host machine:

```bash
pip install --target <vendor_dir> \
  --platform manylinux2014_x86_64 \
  --implementation cp --python-version 3.11 --abi cp311 \
  --only-binary=:all: --no-compile \
  -r requirements.txt
```

If you're unsure which Python the backend is actually running on, a quick diagnostic
(`logger.info(sys.version)` early in the plugin) confirms it directly rather than guessing.

## The frozen runtime's stdlib is pruned

The frozen runtime's stdlib is bundled as a `base_library.zip` containing only the modules
Decky's own bootstrap imports - not the full standard library. Stdlib modules you'd normally
take for granted (e.g. `html.parser`) can be **missing** even though they're technically
"builtin" in a normal Python install.

If you hit `ModuleNotFoundError` for a stdlib name on the Deck but not locally, don't assume
it's a bug in your own vendoring step - it may genuinely not be bundled in this frozen
environment, and you need to ship a copy of that module yourself (e.g. fetched from
`raw.githubusercontent.com/python/cpython/<tag>/Lib/<module>/...`, matching your target Python
version/tag).

## Shimming a missing stdlib *package* requires extending `__path__`, not `sys.path`

If the missing stdlib module is a **package** (not a single file) that's already partially
imported by the time your plugin code runs - Decky's own bootstrap may import `html` itself
before your code executes - fixing it requires extending the *already-cached* package object's
`__path__`, not just prepending your vendor directory to `sys.path`:

```python
import html as _html_pkg
_html_pkg.__path__.append(str(vendor_html_dir))
```

`sys.path.insert()` alone does nothing here: Python has already resolved and cached the
top-level `html` package object in `sys.modules`, so path changes only affect *new* top-level
imports, not submodule lookups (`html.parser`, etc.) inside an already-cached package. You have
to mutate the cached package's own `__path__` list directly.

## aiohttp caches its SSL context at import time

`aiohttp.connector._SSL_CONTEXT_VERIFIED` is built **once**, via `ssl.create_default_context()`,
the moment `aiohttp.connector` is first imported - it is not rebuilt per-request or re-read from
`ssl` module state later.

If outbound HTTPS requests from the backend fail with `[SSL: CERTIFICATE_VERIFY_FAILED]` in this
frozen environment (no system CA bundle wired up the way `aiohttp` expects), monkey-patching
`ssl.create_default_context` *before* importing `aiohttp` does **not** help - the cached context
object from aiohttp's own import is still what gets reused for every connection regardless.

Fix: after `aiohttp` is imported, directly overwrite the cached module-level variable with a
context built from a bundled CA file (e.g. via `certifi.where()`):

```python
import ssl, certifi, aiohttp.connector
aiohttp.connector._SSL_CONTEXT_VERIFIED = ssl.create_default_context(cafile=certifi.where())
```

This matches known upstream aiohttp issues (aio-libs/aiohttp #3180, #7287) describing the same
import-time-caching behavior.

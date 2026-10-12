# CI / release pipeline for Decky plugins (GitHub Actions + pnpm)

## pnpm 10+ blocks dependency build/postinstall scripts by default

Native deps (e.g. `@parcel/watcher`) fail with `ERR_PNPM_IGNORED_BUILDS` unless explicitly
allowed. Fix: add a `pnpm-workspace.yaml` with an explicit `allowBuilds` section:

```yaml
allowBuilds:
  '@parcel/watcher': true
```

Don't put this under `package.json`'s `"pnpm"` field - pnpm no longer reads that location;
`pnpm-workspace.yaml` is the new home (also true for `peerDependencyRules`, etc.).

## `GITHUB_TOKEN` defaults to read-only

Many repos/orgs now default `GITHUB_TOKEN` to read-only. A workflow that creates a GitHub
Release (e.g. via `softprops/action-gh-release`) needs an explicit permissions block or it
fails with `403 Resource not accessible by integration`:

```yaml
permissions:
  contents: write
```

This workflow-level declaration takes precedence over the repo's default "Workflow permissions"
setting - you generally don't need to change the repo-wide setting, just declare the permission
in the specific workflow that needs it. This keeps least-privilege for every other workflow in
the repo.

## Any file under `.github/workflows/` is parsed as a workflow

Even an empty/placeholder `.yml` file there fails every run with
`No event triggers defined in \`on\``. Delete unused stub workflow files rather than leaving
them as placeholders.

## Installing a plugin without the Decky Store: the stable download alias

Decky Loader can install a plugin directly from a zip URL. GitHub's stable alias
`releases/latest/download/<asset-filename>` always resolves to the asset attached to the most
recent non-draft/non-prerelease Release, so a URL like
`https://github.com/<owner>/<repo>/releases/latest/download/<Plugin>.zip` stays valid across
future releases without the user needing to update the link. This is useful for:

- Letting users install a fork/beta build directly.
- Documenting an install method independent of the (sometimes stale or unlisted) official Decky
  Store entry - the Store listing can lag a project's actual GitHub releases significantly,
  since whoever holds the store listing has to push store updates separately.

**Requirement**: this alias needs a **stable, non-versioned filename** attached to *every*
release (in addition to any versioned asset), or the alias 404s whenever the exact filename
changes between versions. Publish both the versioned asset (`myplugin-1.2.3.zip`) and a copy
with the same name every time (`myplugin.zip`) in the same release, and point install docs at
the fixed name, not the versioned one.

## Renaming a GitHub branch that's a PR's head

Renaming a branch via GitHub's dedicated rename feature auto-updates any open PR's head to the
new name - **except when the branch being renamed is the repository's default branch**, where
this can instead silently *close* the PR (GitHub appears to treat the rename as "the branch was
deleted" in that specific case, even though the content is intact under the new name).

If you need to rename the branch a PR is open from, either make sure it is **not** the
repository's current default branch first, or be prepared to recreate/reopen the PR against the
new branch name afterward. Nothing is lost - the commit history survives under the new branch
name either way - but the PR itself needs manual reattachment.

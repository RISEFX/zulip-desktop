# RISEFX fork of `zulip/zulip-desktop`

This is RISEFX's build of the Zulip desktop client. The fork exists for two
reasons:

1. It lets `riselink:` URLs open directly in their RISE application instead of
   going through an interstitial page.
2. It produces installers, because upstream builds installers only for its own
   signed releases.

For upstream's own documentation, see [`README.md`](README.md).

## How this fits with the other RISEFX repos

- [`RISEFX/zulip`](https://github.com/RISEFX/zulip) is the server fork. It
  makes `riselink:` links render in messages, and this fork makes them open.
  Both are required.
- [`RISEFX/docker-zulip`](https://github.com/RISEFX/docker-zulip) runs a daily
  watcher. The watcher opens an issue titled
  `Upstream Zulip Desktop release: v<X.Y.Z>` when upstream publishes a new
  desktop release.
- The `/zulip-release` Claude Code skill in `RISEFX/docker-zulip`
  (`.claude/skills/zulip-release/`) takes such an issue and runs the update
  procedure below.

## Branches

| Branch        | Purpose                                                       |
| ------------- | ------------------------------------------------------------- |
| `main-risefx` | **The fork.** An upstream release tag plus the commits below. |
| `main`        | Mirror of upstream `main`. It is not used for releases.       |

`main-risefx` is rebased onto upstream release tags, not onto upstream `main`.
This keeps every installer built from a known upstream release. To find the
release the branch is based on, run:

```bash
git describe --tags --exact-match "$(git merge-base main-risefx upstream/main)"
```

## Tags

The tag format is `v<version>-risefx`, for example `v5.14.3-risefx`.
`<version>` is the upstream release that `main-risefx` is based on.

Pushing a tag that matches `v*` to any commit on `main-risefx` starts the
installer build, because those commits contain `.github/workflows/build.yml`.
The build produces seven installers and publishes a GitHub Release with them
attached.

The build sets the app version from the tag without its leading `v`, so the
About dialog shows `5.14.3-risefx` and the installer names carry the suffix,
for example `Zulip-5.14.3-risefx-x64.msi`.

Because the scheme has no numeric revision, a re-release at the same upstream
version needs a different tag name. The one past re-release used
`v5.12.4b-risefx`.

## What this fork changes

The fork adds seven commits on top of the upstream release tag. They are listed
here oldest first, by subject, because hashes change on every rebase.

1. **`link_utils: exposing whitelisted protocols to user settings`.** This is a
   cherry-pick of an unmerged upstream pull request (`Fixes #1284`, by Jonas
   Sorgenfrei), not RISEFX-authored code.
   - It adds the `whitelistedProtocols: string[]` key to `configSchemata` in
     [`app/common/config-schemata.ts`](app/common/config-schemata.ts).
   - It adds `getConfigItemWithoutSettingDefault()` to
     [`app/common/config-util.ts`](app/common/config-util.ts). That function
     reads a key without writing the default back into `settings.json`, so a
     later change to the default still reaches installed clients.
   - It changes `openBrowser()` in
     [`app/common/link-util.ts`](app/common/link-util.ts) to check the
     configured list instead of a hardcoded `["http:", "https:", "mailto:"]`.
   - It adds user documentation in
     [`docs/howto/customize-link-protocols.md`](docs/howto/customize-link-protocols.md).

   The default list is `http:`, `https:`, `mailto:`, `tel:` and `sip:`. The app
   opens any other protocol indirectly, through a generated local HTML file in
   the default browser.

2. **`workflows: Add installer build workflow.`** This adds
   [`.github/workflows/build.yml`](.github/workflows/build.yml).
3. **`workflows: Add arm64, deb, and rpm builds; publish releases on tags.`**
   This extends the build matrix and adds the `release` job.
4. **`workflows: Set package version from the build tag.`** Tag builds set the
   `package.json` version from the tag, so a release never needs a version-bump
   commit that would conflict with upstream's version changes on the next
   rebase. Manual `workflow_dispatch` builds keep the committed
   version.
5. **`Implement custom protocol handling in link-util (#1)`.** This is a RISEFX
   pull request by Jonas Sorgenfrei. It changes behavior only on Linux, where
   `openBrowser()` opens a whitelisted protocol other than `http:`, `https:`,
   `mailto:`, `tel:` and `sip:` with a detached `xdg-open` process instead of
   Electron's `shell.openExternal()`. The code treats only those five
   protocols as reliably handled by `shell.openExternal()`. Windows and macOS
   still use `shell.openExternal()` for every whitelisted protocol.
6. **`workflows: Upgrade the actions in the installer build workflow.`** This
   pins the actions in `build.yml` to the versions in upstream's
   `.github/workflows/actions.lock`. An older `pnpm/action-setup` did not
   support upstream's pnpm 12 and broke both Windows MSI builds.
7. **`docs: Add RISEFX fork documentation.`** This is this file. It is kept as
   the last commit, so a README edit is a `git commit --amend` on `HEAD`.

## Making `riselink:` work

**This is the main purpose of the fork, and it does not work without
configuration.** `riselink:` is not in the default list, so a fresh install
still opens `riselink:` URLs through the interstitial page. Each client must
have `whitelistedProtocols` set in its `settings.json`:

```json
{
  "whitelistedProtocols": [
    "http:",
    "https:",
    "mailto:",
    "tel:",
    "sip:",
    "riselink:"
  ]
}
```

- The file is `<userData>/config/settings.json`, as built by `reloadDatabase()`
  in `app/common/config-util.ts`. `userData` is `<appData>/Zulip`, because
  `package.json` sets `productName: "Zulip"`. The paths are
  `%APPDATA%\Zulip\config\settings.json` on Windows,
  `~/.config/Zulip/config/settings.json` on Linux, and
  `~/Library/Application Support/Zulip/config/settings.json` on macOS.
- **Every protocol needs its trailing colon** (`"riselink:"`, not
  `"riselink"`), because the check compares against `URL.protocol`, which
  includes it.
- **The list is read once, when the app starts.** `link-util.ts` evaluates
  `whitelistedProtocols` as a module-level `const`, so a change to
  `settings.json` takes effect only after an app restart.
- Setting the key **replaces** the default list rather than extending it. List
  every protocol that you want to keep.
- The enterprise file `global_config.json` cannot set this key. Its schema
  accepts `whitelistedProtocols`, but the app ignores the value, because
  `link-util.ts` reads only `settings.json` and the renderer copies only a fixed
  list of settings from `global_config.json` into `settings.json` (the
  `settingOptions` list in `app/renderer/js/main.ts`).

## How the build works

[`.github/workflows/build.yml`](.github/workflows/build.yml) runs on `v*` tags
and on `workflow_dispatch`. It has seven matrix jobs:

| Job                     | Runner           | Output      |
| ----------------------- | ---------------- | ----------- |
| `windows-x64-msi`       | `windows-latest` | `.msi`      |
| `windows-arm64-msi`     | `windows-latest` | `.msi`      |
| `linux-x86_64-appimage` | `ubuntu-latest`  | `.AppImage` |
| `linux-x86_64-deb`      | `ubuntu-latest`  | `.deb`      |
| `linux-x86_64-rpm`      | `ubuntu-latest`  | `.rpm`      |
| `macos-x64-dmg`         | `macos-15-intel` | `.dmg`      |
| `macos-arm64-dmg`       | `macos-latest`   | `.dmg`      |

One `.deb` covers Debian and Ubuntu. One `.rpm` covers the RHEL family,
including AlmaLinux 9 and 10. The Windows arm64 app ships inside an x64 MSI
wrapper, because electron-builder's WiX has no arm64 installer support. That
MSI installs correctly on Windows on ARM.

Installer names follow `Zulip-<version>-<arch>.<ext>`, where `<version>` comes
from the tag. For example, `v5.13.2-risefx` produced
`Zulip-5.13.2-risefx-x64.msi` and `Zulip-5.13.2-risefx-x86_64.AppImage`.

Before building, the workflow removes two settings from `package.json`, because
the fork cannot use them:

- `build.win.azureSignOptions` holds upstream's Azure Trusted Signing
  configuration.
- `build.deb.fpm` installs the **official Zulip apt repository**. If it stayed,
  `apt upgrade` on a user's machine would replace this fork's build with
  upstream's.

`CSC_IDENTITY_AUTO_DISCOVERY: "false"` stops electron-builder from searching
the macOS keychain for a signing identity.

**All installers are unsigned.** Windows SmartScreen and macOS Gatekeeper warn
on first run. macOS users need to right-click the app and choose Open, or clear
the quarantine attribute.

On a tag, the `release` job downloads every installer and runs
`gh release create "$GITHUB_REF_NAME" --generate-notes installers/*`.

### Upstream's `Release` workflow also runs

Upstream's `.github/workflows/release.yml` triggers on tags that match
`v[0-9]*`, so it also runs on every `v*-risefx` tag. Its `prepare` job fails at
the step "Check that the tag matches package.json", because the tag is not
`v<package.json version>`, so its build jobs are skipped. This red `Release`
run is expected and harmless. It also happened on `v5.13.2-risefx`. Watch the
`Build installers` run instead.

### GitHub Actions lockfile

Upstream ships `.github/workflows/actions.lock`, which `gh actions-lock`
generates. Our `build.yml` is not listed in it. A workflow that is not listed
still runs normally, unless a "Require lockfile" Actions policy is enabled for
the repository or organization. That policy is opt-in. If someone enables it,
`build.yml` stops running until it is added to the lockfile with
`gh actions-lock .github/workflows/build.yml`. That command needs
`gh extension install github/gh-actions-lock`, and it edits an upstream file,
which can conflict on later rebases.

Keep the action pins in `build.yml` at the versions in upstream's lockfile, so
our build runs the same actions as upstream's release builds.

## Deployment

Linux stations get the app through the `apps::zulip` class in the `it/puppet`
GitLab repository:

- The desktop entry and the autostart entry run
  `/mnt/fhgfs/software/linux/software/zulip/run_zulip_app.bash`.
- According to the class's header comment, the AppImages live on the
  puppetshare, with production and testing symlinks. The class does not manage
  the AppImages or the symlinks, so placing a new AppImage there is a separate
  step outside puppet.
- Puppet installs `/etc/zulip-desktop-config/global_config.json`. It sets
  `presetOrganizations` to `https://chat.risefx.com` and `autoUpdate` to
  `false`.

This repository has no deployment mechanism for Windows or macOS. The GitHub
Release is where those installers are published.

## Updating to a new upstream release

The recommended way is the `/zulip-release` skill. Start Claude Code in a
`RISEFX/docker-zulip` clone and pass it the tracking issue. The skill follows
the steps below and stops before every push.

### One-time setup

A fresh clone has only `origin` (`RISEFX/zulip-desktop`). Add upstream:

```bash
cd ~/workspace/repos/risefx/zulip-desktop
git remote add upstream https://github.com/zulip/zulip-desktop.git
git fetch upstream --tags
```

### 1. Refresh the mirror

```bash
git fetch upstream --tags
git switch main
git merge --ff-only upstream/main
git push origin main
```

### 2. Rebase the fork onto the release tag

```bash
NEW=v5.15.0                      # the new upstream release tag
OLD_BASE=$(git merge-base main-risefx upstream/main)
git switch main-risefx
git rebase --onto "$NEW" "$OLD_BASE"
```

The docs commit is already last, so a plain rebase keeps that order.

### 3. Verify the rebase

```bash
# Expect exactly 7 commits, with the docs commit last.
git log --oneline "$NEW"..main-risefx

# Expect "=" on every row, meaning that every commit applied unchanged.
git range-diff "$OLD_BASE"..origin/main-risefx "$NEW"..main-risefx
```

If `range-diff` shows a `!` row, read that diff before you continue.

**Check that the cherry-picked feature survived.** The first commit is an
unmerged upstream pull request. If upstream merges it, `git rebase` usually
drops the commit as already applied. That outcome is fine, but confirm that the
feature is still there:

```bash
git grep -n whitelistedProtocols -- app/common/
```

If the grep finds nothing, the rebase dropped the feature, and you must restore
it before tagging.

**Compare the action pins with upstream's lockfile.** If upstream bumped an
action, update `build.yml` and amend the
`workflows: Upgrade the actions in the installer build workflow.` commit:

```bash
grep -n 'uses:' .github/workflows/build.yml
git show "$NEW":.github/workflows/actions.lock \
  | grep -A2 -E "'(actions/checkout|actions/setup-node|pnpm/action-setup|actions/upload-artifact)@"
```

Also check `packageManager` in `package.json`. A new pnpm major version needs a
`pnpm/action-setup` release that supports it.

### 4. Tag and push

The branch was rewritten, so it needs a force push:

```bash
git tag "$NEW-risefx"
git push --force-with-lease origin main-risefx
git push origin "$NEW-risefx"
```

### 5. Watch the build

```bash
gh run watch --repo RISEFX/zulip-desktop
```

Pick the `Build installers` run. The `Release` run fails as described above.
On success, seven installers are attached to a new GitHub Release. Before you
roll it out, install one and confirm that `whitelistedProtocols` in
`settings.json` still controls `riselink:`.

## Known issues

These are recorded, not fixed.

- **Installers are unsigned**, as described above.
- **`riselink:` needs per-client configuration.** No default includes it, and
  `global_config.json` cannot set it, so puppet's config does not enable it.
  Until someone edits `settings.json` on a machine, `riselink:` links open
  through the interstitial page there. Adding `riselink:` to the default list
  in `link-util.ts` would make the fork work without configuration.
- **The tag scheme has no numeric revision.** The server fork uses
  `-risefx<P>`. Here, a re-release at the same upstream version needs an ad hoc
  tag name.
- **Upstream's `Release` workflow fails on every fork tag.** This is expected,
  as described above.
- **`xo` reports `no-useless-return` in `link-util.ts`.** The final `return`
  in `openCustomProtocol()` is unnecessary. Upstream's lint workflow runs only
  for `main`, so this does not block our builds.

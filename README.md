# DeepSeek Harness — Linux desktop packaging

Automated `.deb` and `.AppImage` builds of the DeepSeek Harness desktop
application, produced from the **official** [`deepseek-ai/deepseek-harness`](https://github.com/deepseek-ai/deepseek-harness)
source at an official release tag.

This repository holds no source copy. A scheduled workflow resolves the newest
official `dsh-v*` tag, checks that tag out, applies the Linux desktop patches in
[`patches/`](patches/), builds the packages, and publishes them as a GitHub
Release named after the official tag.

## Why patches

Upstream implements the desktop application for macOS and Windows only:

- `apps/desktop/scripts/desktop-build-paths.mjs` and `package-target.ts` accept
  only `mac-arm64`, `mac-x64`, and `win-x64` release targets.
- `apps/desktop/src/main.ts` creates the tray icon only on Windows.
- The packaging configuration ships no Linux tray icon and no Linux installer.

The patches add exactly that Linux support and nothing else:

| Patch | Contents |
| --- | --- |
| `0001-linux-desktop-support.patch` | A `scripts/package-linux-desktop.ts` entry point that builds the deb and AppImage, a Linux tray icon plus its renderer, the Linux tray creation in the main process, and the Linux resource mappings. |

The patches are ordinary `git diff --binary` output against
`dsh-v0.2.0-rc.2`, so they carry the rendered `tray-linux.png` too.

## Version policy

A build is only produced for a published official tag, and the workflow verifies
`dsh-v<package.json version>` equals that tag before building. The result is
therefore the official version, never a locally invented one.

## Running it

The schedule runs daily; a new official tag produces one new release. Manual
runs take an optional tag and a `force` flag from **Actions → Linux desktop
package → Run workflow**.

`force` rebuilds a tag that already has a release, which is useful after a patch
update:

1. Update or add a patch in `patches/` so it applies to the new source.
2. Run the workflow with that tag and `force` enabled.
3. Delete the previous release for that tag, or let the new run fail loudly when
   the release already exists.

## Artifacts

| File | Notes |
| --- | --- |
| `deepseek-harness_<version>_amd64.deb` | Installs to `/opt/DeepSeek Harness`, registers `/usr/bin/deepseek-harness` and a desktop entry. Depends on `libnotify4`, `libxtst6`, `libnss3`. |
| `deepseek-harness-<version>-x86_64.AppImage` | Single file; `chmod +x` and run. Needs FUSE 2, or `--appimage-extract-and-run` where it is unavailable. |
| `SHA256SUMS.txt` | Checksums of both packages. |

Both packages bundle the Electron shell, the dsh runtime it hosts, a pinned
Node.js, pnpm, CPython with the Office libraries, and the LibreOffice engine, so
no system Node.js or Python is required.

## Runtime notes

- The desktop application shares `~/.dsh` with the `dsh` CLI and the `web`
  profile: sessions, workspaces, and plugins are the same data.
- Closing the main window hides it and leaves the Host running. On Linux the top
  bar tray icon is the way back, which needs the GNOME AppIndicator extension
  (`gnome-extensions enable ubuntu-appindicators@ubuntu.com`).
- The tray icon is rendered from `resources/icon-windows.svg` by
  `pnpm --filter @deepseek-ai/dsh-desktop run render:tray-icon`; the committed
  `resources/tray-linux.png` is that output.

## Updating for a new official release

A tag whose source no longer matches a patch fails at the `Apply Linux desktop
patches` step: `git apply` refuses rather than building a silently modified
application. Resolve it by rebasing the patch onto that tag in a local checkout
and committing the refreshed file here.

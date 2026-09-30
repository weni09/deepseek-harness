# DeepSeek Harness — Linux & macOS Intel desktop packaging

Automated `.deb`, `.AppImage` (Linux x86_64) and `.dmg`, `.zip` (macOS Intel x64)
builds of the DeepSeek Harness desktop application, produced from the **official**
[`deepseek-ai/deepseek-harness`](https://github.com/deepseek-ai/deepseek-harness)
source at an official release tag.

This repository holds no source copy. A scheduled workflow resolves the newest
official `dsh-v*` tag, checks that tag out, applies the desktop patches in
[`patches/`](patches/), builds the packages across matrix platforms, and publishes
them as a GitHub Release named after the official tag.

## Why patches

Upstream implements the desktop application with fixed targets:

- `apps/desktop/scripts/desktop-build-paths.mjs` and `package-target.ts` accept
  only `mac-arm64`, `mac-x64`, and `win-x64` release targets, with missing Linux packaging logic.
- Upstream macOS packaging enforces Apple Developer signing keychain and Notary Tool submission, failing without paid Apple credentials.
- `apps/desktop/src/main.ts` creates the tray icon only on Windows.

The patches add the following support:

| Patch | Contents |
| --- | --- |
| `0001-linux-desktop-support.patch` | A `scripts/package-linux-desktop.ts` entry point that builds the deb and AppImage, a Linux tray icon plus its renderer, Linux tray creation in the main process, and Linux resource mappings. |
| `0002-macos-desktop-support.patch` | A `scripts/package-mac-desktop.ts` standalone entry point that builds macOS Intel (`mac-x64`) DMG and ZIP artifacts without requiring Apple Developer signing/notarization credentials. |

## Version policy

A build is only produced for a published official tag, and the workflow verifies
`dsh-v<package.json version>` equals that tag before building. The result is
therefore the official version, never a locally invented one.

**Channels.** Only stable releases and release candidates are packaged. The
`alpha` and `canary` channels are excluded, the same two the release scripts
give their own npm dist-tags; `rc`, `beta`, and any other prerelease channel are
eligible.

The scheduled run picks the newest eligible tag by base version, and prefers the
stable release of a base version over its own prereleases:

| Official tags present | Built |
| --- | --- |
| `0.2.0-rc.2`, `0.3.0-alpha.1`, `0.3.0-canary.2` | `dsh-v0.2.0-rc.2` — a newer-base alpha never wins |
| `0.2.0-rc.2`, `0.2.0` | `dsh-v0.2.0` — stable beats its own release candidate |
| `0.2.0`, `0.3.0-rc.1` | `dsh-v0.3.0-rc.1` — a newer base still wins over an older stable |
| only `0.3.0-alpha.1` | nothing; the run fails with no eligible tag |

A manually supplied alpha or canary tag is rejected before any build starts.

## Running it

The schedule runs daily (03:17 Beijing time / 19:17 UTC); a new official tag produces
one new release. Manual runs take an optional tag and a `force` flag from
**Actions → Desktop package → Run workflow**.

`force` rebuilds a tag that already has a release, which is useful after a patch update.

## Artifacts

| Platform | File | Notes |
| --- | --- | --- |
| Linux | `deepseek-harness_<version>_amd64.deb` | Installs to `/opt/DeepSeek Harness`, registers `/usr/bin/deepseek-harness` and desktop entry. |
| Linux | `deepseek-harness-<version>-x86_64.AppImage` | Single file; `chmod +x` and run. |
| macOS | `deepseek-harness-<version>-mac-x64.dmg` | macOS Intel disk image installer. |
| macOS | `deepseek-harness-<version>-mac-x64.zip` | macOS Intel portable zip bundle. |
| All | `SHA256SUMS.txt` | Checksums of all published packages. |

Packages bundle the Electron shell, the dsh runtime it hosts, a pinned
Node.js, pnpm, CPython with Office libraries, and the LibreOffice engine.

### macOS Note
Because packages are built without an Apple Developer ID signature, macOS Gatekeeper may prevent running it on first launch. If prompted with an unverified developer dialog:
- Right click `DeepSeek Harness.app` -> choose **Open** -> click **Open**.
- Or run `xattr -cr "/Applications/DeepSeek Harness.app"` in Terminal.

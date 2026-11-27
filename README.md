# DeepSeek Harness — linux-arm64 build

Unofficial **linux-arm64** port of [DeepSeek Harness (`dsh`)](https://github.com/deepseek-ai/deepseek-harness), DeepSeek's open-source MIT agent harness. Upstream ships desktop builds for macOS and Windows only; this tree carries the patches needed to build and run the real Electron desktop app (and an AppImage) on **linux arm64**.

> Based on upstream `v0.2.3-alpha.2` (commit `4b399c39`). Everything here is downstream engineering; the upstream code itself already had working linux paths — its release pipeline simply never enabled the target.

## What was patched (20 commits, see `patches/linux-arm64-support/`)

| Area | Change |
|---|---|
| Target whitelists | Add `linux-arm64` to the desktop build paths, packaging targets, auto-update targets, and the publish/release zod schemas |
| Electron executable paths | `prepare-runtime.ts` / `prepare-dsh.ts` resolved a macOS `.app` bundle path unconditionally; added a linux branch |
| **sharp / libvips crash fix** | sharp's prebuilt libvips statically embeds its own glib while Electron leaks the system glib into the process, so the PNG **decode** path dies with `SIGSEGV` in `g_object_unref` (NULL vtable) — the known [electron#46323](https://github.com/electron/electron/issues/46323) clash. Build-time step `runtime:relink-sharp-libvips` now replaces the vendored `libvips-cpp` in the payload with the build host's system libvips, so payload and Electron share a single glib. Reproduction requires the *second* sharp pipeline (encode a buffer, then decode it) — single-step probes pass |
| Packaged smoke | Resolve the linux unpacked layout (`linux-arm64-unpacked/`, executable `deepseek-harness`); skip the Office conversion round trips, which cannot pass on linux (see limitations) |
| AppImage packaging | Clean root-level `executableName: deepseek-harness` (the scoped package name sanitizes to `@deepseek-aidsh-desktop`, which the AppImage tooling rejects) + `syncDesktopName` for window association; do not embed the mandatory-update policy into linux manifests; allow `linux` in the desktop policy identity |
| Wayland app_id | Set `productName` in the packaged manifest (`extraMetadata`): Electron otherwise derives the app_id from the scoped package name (`deepseek-ai-dsh-desktop`), which never matches the desktop entry for dock grouping |

## Build (Fedora 45 aarch64, Node 24.22 + pnpm 11.28)

```sh
# system deps
sudo dnf install -y vips-devel   # or libvips-dev on Debian/Ubuntu — required by the sharp fix

# repo requires node ^24.22.3 / pnpm 11.28.5 (corepack)
pnpm install
# placeholder release settings (unsigned local builds ignore the values but the file must exist):
#   cp apps/desktop/.env.macos.example apps/desktop/.env.macos
#   then set DSH_DESKTOP_APP_ID / AUTO_UPDATE_ENV=test / *_TEST_ORIGIN / DOWNLOAD_TEST_RELEASE_ID
pnpm --filter @deepseek-ai/dsh-desktop run package -- --unsigned
```

Artifact: `apps/desktop/.desktop-build/targets/linux-arm64/unsigned-artifacts/deepseek-harness-<version>-linux-arm64-unsigned.AppImage`

The AppImage itself needs FUSE2; if your system only ships FUSE3, extract instead:

```sh
./deepseek-harness-*.AppImage --appimage-extract
./squashfs-root/AppRun
```

## Known limitations (inherited from upstream)

- **Office document conversion does not work on linux** — upstream ships no LibreOfficeKit engine for linux at all (`@deepseek-ai/libreoffice-kit-linux-arm64-glibc` does not exist on npm); the packaging-time selection falls back to the wasm engine, and the conversion smoke is skipped accordingly.
- No auto-update channel (none exists for linux).

## Model backends

Any OpenAI-compatible endpoint works (`Settings → Models → Custom model API`, or edit `~/.dsh/profiles/desktop/cordis.patch.yml` — hot-reloaded):

```yaml
- id: llm-pi-ai
  config:
    providers:
      my-endpoint:
        api: openai-completions
        baseURL: http://your-host:8080/v1
        models:
          - id: your-model
```

## Provenance & license

Source: [deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) @ `v0.2.3-alpha.2`, MIT (see `LICENSE`, upstream README kept as `README-upstream.md`). The 20 downstream patches are MIT as well. Not affiliated with DeepSeek; the official project remains at the link above.

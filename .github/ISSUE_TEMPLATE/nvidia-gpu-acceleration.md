---
name: 'NVIDIA GPU acceleration on Wayland: upgrade Electron to 43+ (Chromium 150)'
about: 'NVIDIA GPU rendering + VA-API NVDEC hardware video decode on Wayland'
title: 'feat: support NVIDIA GPU acceleration (Electron 43+)'
labels: enhancement
assignees: ''
---

## Background

On **NVIDIA GPU + Wayland** systems (e.g. GTX 1660 Ti / KDE Plasma), the current Electron 28 (Chromium 120) build has two GPU acceleration blockers:

1. **GPU rendering fallback**: `--use-gl=egl` crashes with "Passthrough is not supported" (exit 133). `--use-gl=angle --use-angle=gl` works but the GPU process may fall back to SwiftShader in some configurations.

2. **VA-API video decode blocked**: Chromium's `vaapi_wrapper` has a hard-coded check at `vaapi_wrapper.cc:130`:

   ```
   WARNING:vaapi_wrapper.cc:130] Should skip nVidia device named: nvidia-drm
   ```

   This causes Chromium to skip VA-API initialization on NVIDIA GPUs entirely. The GPU process shows `dec: 0%` in `nvidia-smi dmon`.

3. **System proxy conflict** (Chromium 150+): When a system proxy (e.g. Clash at `127.0.0.1:7897`) is active, the local debug domain `bilipc.bilibili.com` (host-rules → `localhost:3031`) goes through the proxy, which cannot resolve the fake domain and returns `ERR_CONNECTION_CLOSED (-100)` → blank window.

## Root Cause

### For VA-API decode

The feature `VaapiOnNvidiaGPUs` (introduced in Chromium 130) is required to bypass the NVIDIA device skip in `vaapi_wrapper`. This feature **does not exist** in Chromium 120 (Electron 28).

Additionally, Chromium 150 renamed the VA-API decode feature from `VaapiVideoDecoder` (the old name used in Chromium 120) to `AcceleratedVideoDecodeLinuxGL` / `AcceleratedVideoDecodeLinuxZeroCopyGL`.

### For the proxy issue

Chromium 150's NetworkService (out-of-process network stack) honors the system proxy unconditionally. `--proxy-bypass-rules` command-line flag does not override system proxy settings. Must use `session.defaultSession.setProxy()` in the Electron main process with `proxyBypassRules` to bypass.

## Required Changes

### 1. Upgrade Electron from 28.2.1 → 43.1.1 (or ≥ 130)

- `conf/build.json`: `"electronVersion": "43.1.1"`
- `package.json`: `"electron": "^43.1.1"`
- `tools/update-electron.sh`: `electron_version="43.1.1"`

### 2. Add Chromium 150 certificate-error handler

Chromium 150 no longer fully trusts `--ignore-certificate-errors` for navigation. Must add an explicit `certificate-error` event handler:

```ts
app.on("certificate-error", (event, _webContents, url, _error, _certificate, callback) => {
  if (url.startsWith("https://bilipc.bilibili.com") || url.startsWith("https://localhost:3031")) {
    event.preventDefault();
    callback(true);
  } else {
    callback(false);
  }
});
```

### 3. Add proxy bypass for system proxy compatibility

```ts
app.whenReady().then(() => {
  const proxyUrl = (process.env.HTTPS_PROXY || process.env.HTTP_PROXY || "").trim();
  session.defaultSession.setProxy({
    proxyRules: proxyUrl.replace(/^https?:\/\//i, "").replace(/\/+$/, "") || undefined,
    proxyBypassRules: "bilipc.bilibili.com,localhost,127.0.0.1",
  }).catch(() => {});
});
```

### 4. Update `bilibili-flags.conf` recommended options

Replace old `VaapiVideoDecoder` with new GPU/VA-API feature names:

```
--enable-features=AcceleratedVideoDecodeLinuxGL,AcceleratedVideoDecodeLinuxZeroCopyGL,VaapiOnNvidiaGPUs,VaapiIgnoreDriverChecks,PlatformHEVCDecoderSupport
```

`VaapiOnNvidiaGPUs` is the critical new feature that enables VA-API → NVDEC hardware decode on NVIDIA GPUs.

## Compatibility Note

- Electron 43's screen cursor regression (#42519) is **bypassed** on Wayland via the existing `cursor-tool` binary (reads `/dev/input/event*` directly), so upgrading is safe for Wayland users.
- X11 users may still be affected by #42519 and should test before upgrading.

## Reference Fork

A working fork with all changes applied is available at:
https://github.com/wings1848/bilibili-linux/tree/feat/electron43-gpu

The GPU acceleration has been verified:
- ✅ GPU rendering: NVIDIA (gpu-process 379 MiB VRAM, `sm 14-15%`)
- ✅ NVDEC hardware decode: `dec 5-7%` on HEVC video
- ✅ No blank screen / SSL errors
- ✅ System proxy preserved (bypass only bilipc/localhost)

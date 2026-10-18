# Public test: native driver modes (`public-test` branch)

This branch is waydroid-nvidia with two new ways to drive the GPU, next to the
Venus proxy that `main` ships. Pick one with a single key in
`/var/lib/waydroid/waydroid.cfg`:

```ini
[waydroid]
nvidia_driver_mode = nvidia-prolapse   # or glibc-shim, or venus
nvidia_native_gl = off                 # off = ANGLE; on = the mode's own GLES driver
```

| Mode | What runs in the Android container | Notes |
|---|---|---|
| `nvidia-prolapse` (default) | Our Waydroid vendor image, built from source (`vendor-image/`): Mesa NVK/NAK over our Rust RM backend, talking to the host's NVIDIA kernel module directly, plus our gralloc, hwcomposer, ANGLE and Codec2 decoder as vendor modules. No NVIDIA userspace in the guest. `nvidia_native_gl = on` selects our own GLES driver `nvrm` instead of ANGLE | Hardware video decode (Codec2 NVDEC) included. Runs with the official system image |
| `glibc-shim` | The host's own NVIDIA userspace, unmodified, loaded into bionic processes through our glibc-compatibility shims and Vulkan HAL. `nvidia_native_gl = on` selects NVIDIA's own GLES | The guest uses the host's installed driver files, so it needs exactly the host driver version the shim table was generated for |
| `venus` | The existing Venus-over-socket proxy (this repo's `main`) | Slowest, widest availability (open kernel module) |

The mode is announced when the container starts (`NVIDIA driver mode: ...` in
the Waydroid log). Nothing falls back silently: an official vendor image where
ours belongs, a wrong driver version or an unsupported combination stops the
start with a message that names the key and the fix. `nvidia-prolapse` also
records its install shape, `nvidia_prolapse_install = vendor-image` (the setup
writes it; `dev-root` is the developer shape, [dev-setup.md](dev-setup.md)).

The mode was called `rust-driver` during development. That value is now
refused with a message that gives the new name and the command that moves the
root link (`/var/lib/waydroid/nvidia-rust-driver` became
`/var/lib/waydroid/nvidia-prolapse`).

## Install

Release: [v0.2.0-rc1](https://github.com/Shiro836/waydroid-nvidia/releases/tag/v0.2.0-rc1) (prerelease). Until it
is confirmed, the AUR package keeps the previous release; every distro, Arch included, uses the script:

<table>
<tr><th>Step</th><th>Command</th></tr>
<tr><td>1. install</td><td><code>curl -fsSL https://raw.githubusercontent.com/Shiro836/waydroid-nvidia/public-test/packaging/install.sh | sudo bash</code></td></tr>
<tr><td>2. set up</td><td><code>sudo waydroid-nvidia-setup</code></td></tr>
<tr><td>3. start</td><td><code>waydroid session start</code></td></tr>
</table>

What step 2 does in `nvidia-prolapse` mode (the default):

| Stage | Result on failure |
|---|---|
| host checks: render node, `nvidia-drm.modeset=Y`, an initialized Waydroid | stops, names the fix |
| release bundle hash check against `BUILD-RECORD.json`; the vendor image zip against its OTA channel (sha256) and against the bundle (its `ro.vendor.nvidia_prolapse.build_record`); every driver binary in the image byte-identical to the bundle's | stops: reinstall the package |
| install-time gate on your GPU and kernel module, on those bytes: Vulkan compute/graphics/interop, HAL ANB/ASTC (x86_64 and x86), `nvrm` GL smoke (triangle vs llvmpipe reference, dma-buf/fences, Skia shaders, YUV), Codec2: your GPU's decoder list (`nvc2-xmlgen`, checked against the live caps), the service's libraries in the image, device access; CI-built programs, Android runtime read from your `system.img` | stops, prints the failing case and its log (`/var/lib/waydroid-nvidia/gate-*`), installs nothing; suggests `--mode venus` |
| install: your official `vendor.img` kept in `/var/lib/waydroid-nvidia/vendor-image/official/`, ours in its place, `vendor_ota` = this release's channel, `waydroid.cfg` (mode kept if you chose one; GL default off = ANGLE), `waydroid upgrade -o`, container `ExecStartPre` check | previous image and configuration restored |

The image is pinned to the kernel module it was gated on; after an NVIDIA driver update the
container refuses to start and names the fix: run `sudo waydroid-nvidia-setup` again (it re-gates and
regenerates the decoder list). The same after a package update. Waydroid system OTAs stay official;
no Waydroid update replaces our vendor image (`vendor_ota` points at this release). Back to the official
vendor image: `sudo waydroid-nvidia-setup --restore-official-vendor` (package removal runs it).
`glibc-shim` needs shims generated for the exact host driver version and stays a developer mode.
Manual install, file list and uninstall: [install-manual.md](install-manual.md).

Developed on Arch Linux with an RTX 5090 (driver 615, open kernel module) on the official
LineageOS 20 (Android 13) Waydroid image; see the support tables in the README.

## Developers

Building from a checkout, staging with the developer gates and switching modes:
[dev-setup.md](dev-setup.md).

## Known issues

- `nvidia-prolapse` runs the official system image's stock SurfaceFlinger. Its vsync
  snap window is fixed at 3 ms, which can stall above ~333 Hz (500 Hz ran without a
  stall in our test); `--refresh` caps the rate if the display freezes.
- `glibc-shim`, `venus` and the developer `dev-root` shape still bind files into the
  official vendor image; they refuse ours (`--mode venus` puts the official one back).
- Only Android 13 (LineageOS 20, the official Waydroid image) is supported.
- Fixed: SurfaceFlinger crash (SEGV) on the 20260927 vendor image when the Waydroid window was
  resized or activated (hwcomposer patch 0016; upstream waydroid/android_hardware_waydroid#84).
- GTA: San Andreas flicker reported; under investigation (not reproduced yet).
- Minecraft: a short freeze the first time a new item is selected (shader
  compile on the render thread), in both native modes.
- `nvidia_native_gl = on` with `nvrm` is new; if an app misbehaves, compare
  with `off` (ANGLE) before reporting.
- `glibc-shim` stops working when the host NVIDIA driver is updated, until
  the shims are rebuilt for the new version.

## Reporting

Open an issue on https://github.com/Shiro836/waydroid-nvidia/issues with
`[public-test]` in the title and include:

- GPU, `cat /proc/driver/nvidia/version`, open or closed kernel module, distro
  and compositor;
- the mode and `nvidia_native_gl` value, and the output of
  `sudo python3 /usr/lib/waydroid-nvidia/native/packaging/native/vendor_image.py status` and
  `python3 /usr/lib/waydroid-nvidia/native/dev/driver-mode status` and, for a failed setup, the
  gate log directory it printed (`/var/lib/waydroid-nvidia/gate-*`);
- the failing app and what you saw; `sudo waydroid logcat -d` and
  `sudo journalctl -k | grep -i xid` from around the failure;
- whether the same app works in `venus` mode.

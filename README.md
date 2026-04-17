# Fix: HDR washed out / greenish on Raspberry Pi 5 + LibreELEC + Samsung TV

If you're running **LibreELEC on a Raspberry Pi 5** connected to a **Samsung TV** (S90C, S92C, S95C, QN90C, QN95C, or similar recent 4K/8K models), and HDR content plays back looking **washed out with a greenish tint**, this repo explains why and how to fix it.

**TL;DR:** Samsung TVs use a quirk in their EDID (HDMI Forum Extension Override) that the version of `libdisplay-info` shipped with LibreELEC 12.2.1 doesn't understand. Kodi ends up thinking your TV doesn't support HDR, so it sends HDR pixels without the HDR flag. The TV treats them as SDR → washed out picture. The fix is to patch the EDID at boot.

## Symptoms

- HDR movies (HDR10, HDR10+, Dolby Vision) look washed out, low contrast, slight green tint.
- The Samsung TV **does not** show "HDR" in its info panel (press the info button on the remote during playback).
- Kodi log shows `SetHDR: setting connector colorspace to Default` on every HDR playback — never `BT2020_YCC`.
- `winsystem.ishdrdisplay` reports `true` in Kodi, so the surface-level HDR detection looks fine.
- TV settings are all correct (Input Signal Plus ON, Game Mode OFF, HDMI port set to "Cinema" or similar).
- May appear suddenly after a reboot, with no Kodi config changes.

If all of that matches, you're in the right place.

## Affected Setup

- **Hardware:** Raspberry Pi 5
- **OS:** LibreELEC 12.2.1 (Kodi 21.3 Omega)
- **TV:** Samsung S92C confirmed. Very likely affects S90C, S95C, QN90C, QN95C, and other recent Samsung 4K/8K models (any Samsung TV using the HDMI Forum Extension Override in its EDID).
- **Relevant versions:** kernel 6.12.56, `libdisplay-info` 0.1.1, vc4 DRM driver.

## Root Cause

Kodi decides whether to flag its HDMI output as HDR based on what it reads from the TV's EDID. EDID parsing is done by a library called `libdisplay-info`.

Samsung TVs use the **HDMI Forum Extension Override Data Block** (defined in the HDMI 2.1 / CTA-861-H spec) to declare more EDID extension blocks than the base EDID header can describe. On my S92C:

- Base EDID byte 126 says `1` extension block.
- The HDMI Forum Extension Override inside CTA block 1 says `3` extension blocks.
- The actual EDID is 512 bytes (4 blocks total).

This is valid per the HDMI 2.1 spec. However, `libdisplay-info` 0.1.1 doesn't honor the override — it trusts byte 126, finds more data than expected, and rejects the entire EDID:

```
$ di-edid-decode < /sys/class/drm/card1-HDMI-A-1/edid
di_edid_parse failed: Invalid argument
```

The kernel's `edid-decode` parses the same EDID fine; this is specifically a `libdisplay-info` bug.

The failure chain, end to end:

```
Samsung EDID byte 126 = 1 (real count is 3 via HDMI Forum Override)
  → libdisplay-info 0.1.1 rejects the EDID
    → Kodi's CDisplayInfo has no HDR capability flags
      → CWinSystemGbm::SetHDR() silently falls back to SDR
        → DRM connector sends BT.2020 pixels labeled as BT.709
          → TV treats HDR as SDR → washed out / greenish picture
```

The killer is that **nothing logs a warning** along this chain. HDR just silently stops working, and there's no error to search for.

## How to Verify This Is Your Bug

Before applying the fix, confirm by SSHing into the Pi and running:

```sh
di-edid-decode < /sys/class/drm/card1-HDMI-A-1/edid
```

If you see `di_edid_parse failed: Invalid argument`, you've got this exact bug.

You can also check the DRM connector properties during HDR playback:

```sh
modetest -M vc4 -c
```

Look for the `HDMI-A-1` connector. If `HDR_OUTPUT_METADATA` is empty and `Colorspace` is `0` / `Default` while playing an HDR file, HDR signaling is broken.

## The Fix

Patch the EDID so `libdisplay-info` accepts it: change byte 126 from `0x01` to `0x03` (the true extension count), recompute the block 0 checksum, and apply the patched EDID via the DRM debugfs `edid_override` interface before Kodi starts.

### Step 1 — Create the patched EDID (once)

SSH into the Pi as root:

```sh
mount -o remount,rw /flash
mkdir -p /flash/edid_override

python3 -c "
with open('/sys/class/drm/card1-HDMI-A-1/edid', 'rb') as f:
    edid = bytearray(f.read())
edid[126] = 3
edid[127] = (256 - (sum(edid[0:127]) % 256)) % 256
with open('/flash/edid_override/edid-HDMI-A-1.bin', 'wb') as f:
    f.write(edid)
"

mount -o remount,ro /flash
```

### Step 2 — Apply it at every boot

Create `/storage/.config/autostart.sh`:

```sh
#!/bin/sh
# Fix Samsung EDID: patch extension count so libdisplay-info can parse HDR caps.
# Root cause: Samsung EDID says 1 extension in base header but has 3
# (HDMI Forum override). libdisplay-info 0.1.1 doesn't honor the override.

PATCHED_EDID="/flash/edid_override/edid-HDMI-A-1.bin"
EDID_OVERRIDE="/sys/kernel/debug/dri/1/HDMI-A-1/edid_override"

if [ -f "$PATCHED_EDID" ] && [ -f "$EDID_OVERRIDE" ]; then
    sleep 2
    dd if="$PATCHED_EDID" of="$EDID_OVERRIDE" bs=512 2>/dev/null
    echo 'on' > /sys/kernel/debug/dri/1/HDMI-A-1/force 2>/dev/null
    logger -t edid-fix "Patched EDID applied to HDMI-A-1"
fi
```

Make it executable:

```sh
chmod +x /storage/.config/autostart.sh
```

LibreELEC sources `autostart.sh` before launching Kodi, so the EDID override is in place before Kodi reads the display info.

Reboot.

### Step 3 — Verify

After reboot, play an HDR file and run `modetest -M vc4 -c` in another shell. You should see:

```
HDR_OUTPUT_METADATA: 000000000200d084803ec233c4864c1d...   (populated)
Colorspace: 10 (BT2020_YCC)
max bpc: 12
```

And in the Kodi log:

```
SetHDR: setting connector colorspace to BT2020_YCC
```

The Samsung TV should show "HDR" in its info panel, and the picture should look correct.

## Technical Details (for the curious)

### EDID structure on Samsung S92C

- 512 bytes total (4 blocks)
- Block 0: Base EDID — byte 126 = `0x01` (wrong, should be `0x03`)
- Block 1: CTA-861 Extension — contains the HDMI Forum Extension Override Data Block with extension count = 3
- Block 2: CTA-861 Extension (mostly empty)
- Block 3: DisplayID Extension

### Kodi code path

- `CDVDVideoCodecDRMPRIME::SetPictureParams()` — extracts HDR metadata from decoded frames (with fallback to container hints).
- `CVideoLayerBridgeDRMPRIME::Configure()` — calls `SetHDR()` only once, for the first rendered frame. No retry.
- `CWinSystemGbm::SetHDR()` — checks `CDisplayInfo` for `SupportsColorimetry(BT2020_YCC)`, `SupportsHDRStaticMetadataType1()`, `SupportsEOTF(PQ)`. If any fail, it emits `Default` colorspace and empty `HDR_OUTPUT_METADATA`. Returns silently.
- `CDisplayInfo` is populated by `libdisplay-info` parsing the EDID.

### Reproducing the libdisplay-info failure

```python
import ctypes
lib = ctypes.CDLL('libdisplay-info.so.1')
with open('/sys/class/drm/card1-HDMI-A-1/edid', 'rb') as f:
    edid = f.read()
lib.di_info_parse_edid.restype = ctypes.c_void_p
lib.di_info_parse_edid.argtypes = [ctypes.c_void_p, ctypes.c_size_t]
print(lib.di_info_parse_edid(edid, len(edid)))  # NULL on Samsung EDIDs
```

## Where to Report Upstream

- **`libdisplay-info`** (the root bug): https://gitlab.freedesktop.org/emersion/libdisplay-info — v0.1.1 rejects EDIDs using the HDMI Forum Extension Override Data Block.
- **LibreELEC**: https://github.com/LibreELEC/LibreELEC.tv — ships the broken `libdisplay-info`.
- **Kodi**: https://github.com/xbmc/xbmc — `CWinSystemGbm::SetHDR()` silently falls back to SDR with no log message when `CDisplayInfo` is empty, making this near-impossible to diagnose.

## Keywords

For the poor souls Googling this in the future:

Raspberry Pi 5 LibreELEC HDR broken, Samsung S92C HDR washed out, Samsung S90C HDR green, Kodi HDR not working RPi5, BT2020 SDR greenish picture, libdisplay-info parse failed, EDID extension count Samsung, HDR10 not signaled HDMI, HDMI Forum Extension Override, CTA-861-H, Kodi Omega SetHDR Default, RPi5 HDR output metadata empty.

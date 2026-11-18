# Huawei MateBook (`CREF-XX` / i9-12900H) macOS 26 Tahoe — Intel Iris Xe (`0x46A6`) Internal eDP Acceleration Guide

This guide documents how to resolve the `16 MB` unaccelerated framebuffer lockup with `WhateverXe` and achieve full internal eDP Metal acceleration on a **Huawei MateBook (`CREF-XX`, i9-12900H + Intel Iris Xe `8086:46A6`)** with a **`BOE092F` (`2520×1680`)** panel running **macOS 26.7.3 Tahoe**.

## 1. Hardware & System Specifications

| Component | Details | Status | Notes |
| --- | --- | --- | --- |
| **Model / Firmware** | Huawei MateBook (`CREF-XX`, `HUAWEI UEFI V2.80`) | Working | OpenCore UEFI |
| **CPU** | Intel Core `i9-12900H` (14C / 20T: 6P + 8E) | Working | `CpuTopologyRebuild.kext` + `ctrsmt=full` |
| **iGPU** | Intel Iris Xe Graphics 96 EU (`8086:46A6`, rev `0x0C`) | **Accelerated** | 1536 MB dynamic VRAM, Metal 2 |
| **Internal Panel** | `BOE092F` (`2520×1680`, 3:2, eDP) | **Accelerated** | Non-4KB page-multiple resolution |
| **BIOS DVMT** | Fixed 60 MB (`GMS 0xFE`) | Bypassed | `-igfxdvmt` + `framebuffer-stolenmem` |
| **macOS Version** | macOS 26.7.3 Tahoe (`25G342`) | Working | SMBIOS: `MacBookPro16,2` |
| **Wi-Fi / Bluetooth** | Intel Wireless + Bluetooth | Working | Wi-Fi: `itlwm`; Bluetooth requires Tahoe fork (see below) |

## 2. Root Cause Analysis

When booting the internal display directly with upstream [b00t0x/WhateverXe](https://github.com/b00t0x/WhateverXe) (`0.8.2` / `0.8.3`), macOS remains stuck in `16 MB` software framebuffer mode. `ioreg` and diagnostic checks pinpoint three blockers:

1. **`InvalidDPSource` (`linearPlan = 11` / `N97NativePreflight = No`)**: Huawei's UEFI GOP leaves `TRANS_DDI_FUNC_CTL2 bit 3` (`CMTG_SECONDARY_MODE`) set on the eDP transcoder at handoff, causing `dpSource()` to fail and deliberately block 3D acceleration (see [WhateverXe Issue #22](https://github.com/b00t0x/WhateverXe/issues/22)).
2. **Non-4KB Page-Aligned Framebuffer Rejection**: `2520 × 1680 × 4 = 16,934,400` bytes (`4134.375` pages) is not an integer multiple of 4096 bytes, failing the stock `apertureRange()` check.
3. **Combo eDP `mode 0` Handoff & Multi-Output Clock Gate Fault**: The boot port arrives at adoption in `mode 0` (`port+0x550 = 0`), and multi-output VBT records can trigger `adlnSetDualClock` gate failures during panel modeset (see [WhateverXe Issue #89](https://github.com/b00t0x/WhateverXe/issues/89)).

## 3. Solution: `WhateverXe-AlderLake` + `dpSource()` Patch

Start from [NebbaGiveUp/WhateverXe-AlderLake](https://github.com/NebbaGiveUp/WhateverXe-AlderLake) (based on `0.8.2`, which already integrates eDP `mode 0` adoption, single-output `CDCLK` routing, 64-byte pitch `apertureRange` support, and display sleep/wake IRQ fixes), add the **`dpSource()` `CTL2 bit 3` panel-only patch**, and compile in **Release** mode:

### Source Patch (`WhateverXe/kern_alderlake_n.hpp`)

```diff
- !(trans&((1U<<15)|(1U<<9))) && !ctl2 &&
+ !(trans&((1U<<15)|(1U<<9))) && !(ctl2&~(panel?(1U<<3):0U)) &&
```

### Build Instructions (macOS)

Requires Xcode Command Line Tools, Python 3.9+, [MacKernelSDK](https://github.com/acidanthera/MacKernelSDK), and `Lilu.kext` (1.7.2+ **DEBUG** build placed or symlinked in the repository root):

```bash
git clone https://github.com/NebbaGiveUp/WhateverXe-AlderLake.git
cd WhateverXe-AlderLake
# Place/symlink MacKernelSDK/ and Lilu.kext/ in the repo root, then patch kern_alderlake_n.hpp
python3 tools/test.py
python3 tools/build.py --release
```

> **Note**: Build with `--release` so `WhateverXe.kext` matches a Release `Lilu.kext` in your EFI; mixing a Debug `WhateverXe.kext` with a Release `Lilu.kext` can trigger an early prohibitory (Stop) symbol at boot.

## 4. Key OpenCore `config.plist` Settings

### Kexts & Load Order (`Kernel > Add`)

1. `Lilu.kext` (`1.7.3+`)
2. `WhateverXe.kext` (the `--release` build compiled above)
3. `AMFIPass.kext` (**must be `1.4.1 Mod`**; do not enable `AMFIAnchor` or `WhateverGreen.kext` alongside it)

### Userspace Graphics Package

* Install **`WhateverXe-Graphics-0.4.4.pkg`** in macOS (do not mix with the `0.8.3` / `0.8.1` `AppleIntelTGLCompat` package; rebuild AuxKC and reboot).

### Kernel > Quirks

* `DisableLinkeditJettison`: `true`

### DeviceProperties (`PciRoot(0x0)/Pci(0x2,0x0)`)

> Do **not** inject `AAPL,ig-platform-id`, `device-id` spoofing, or hardcoded `framebuffer-con0-*` properties.

| Key | Type | Hex Value | Purpose |
| --- | --- | --- | --- |
| `framebuffer-patch-enable` | `Data` | `01000000` | Enable framebuffer patching |
| `framebuffer-stolenmem` | `Data` | `0000C003` | Handle Huawei BIOS fixed 60 MB DVMT |

### NVRAM (`7C436110-AB2A-4BBB-A880-FE41995C9F82`)

* `csr-active-config` (`Data`): `03080000`
* `boot-args` (`String`):
  ```text
  -allow3d -disablegfxfirmware -igfxdvmt -wexgvadrop=IOGVAScaler,IOGVABGRAEnc,IOGVAHEVCEncodeCapabilities,IOGVACodec -vi2c-force-polling ctrsmt=full -ibtcompatbeta revpatch=sbvmm -no-gpe
  ```
  > `-wexgvadrop=...` unpublishes unstable Tiger Lake VA keys on Alder Lake-P to prevent GIF/fullscreen GPU hangs while retaining hardware HEVC decoding and smooth 60Hz UI rendering.

## 5. Additional Hardware Notes (macOS 26 Tahoe)

* **Brightness Control**: Enable `SSDT-PNLF.aml` (verify `_UID = 19`).
* **Intel Bluetooth**: Stock `2.5.0` `IntelBluetoothFirmware` / `IntelBTPatcher` does not work on Tahoe; use [lshbluesky/IntelBluetoothFirmware v2.5.1](https://github.com/lshbluesky/IntelBluetoothFirmware/releases/tag/v2.5.1) paired with `BlueToolFixup.kext` from [BrcmPatchRAM v2.7.2](https://github.com/acidanthera/BrcmPatchRAM/releases/tag/2.7.2), and add `bluetoothExternalDongleFailed` (`00`) and `bluetoothInternalControllerInfo` (14 zero bytes) in NVRAM.

## 6. Credits

* [b00t0x/WhateverXe](https://github.com/b00t0x/WhateverXe)
* [NebbaGiveUp/WhateverXe-AlderLake](https://github.com/NebbaGiveUp/WhateverXe-AlderLake)
* Contributors to [WhateverXe Issue #22](https://github.com/b00t0x/WhateverXe/issues/22) & [Issue #89](https://github.com/b00t0x/WhateverXe/issues/89)

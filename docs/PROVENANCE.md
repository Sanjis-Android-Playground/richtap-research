# Provenance and licensing

## Research sources

| Source | Use |
|---|---|
| `oplus-kona/android_kernel_oneplus_sm8250` (`aosp-17`) | OPlus AW8697 RichTap kernel driver |
| `oplus-kona/android_vendor_oneplus_sm8250-common` (`aosp-17`) | `libaacvibrator.so`, `aac_richtap.config`, firmware/RTP bins |
| `LineageOS/android_hardware_oplus` (`lineage-23.2`, `aidl/vibrator/richtap`) | Richtap HAL source |
| `PocoF3Releases/Agents.md` | Earlier Alioth haptics history and constraints |
| `PocoF3Releases/hardware_xiaomi` (`cnb`, `vibrator/aw8697`) | Old Alioth custom HAL reference |
| Local Alioth vendor firmware | Stock `aw8697_haptic.bin`, `aw8697_rtp_1.bin` |

## What is not committed

The following are proprietary or large and are ignored by `.gitignore`:

```text
libaacvibrator.so
libaachaptics.so
aac_richtap.config
aw8697_*.bin
*.log
hardware_oplus/
oplus-driver/
agents/
```

Only hashes, paths, and sanitized evidence are committed.

## Licensing

- The OPlus/Richtap userspace blobs are proprietary. Do not redistribute them
  without permission.
- The OPlus kernel driver is GPL-2.0; source references are recorded, but the
  full driver is not copied here.
- The Lineage Richtap HAL source is Apache-2.0.
- This documentation is provided under CC BY 4.0 unless a file says otherwise.

## Safety notes

- Do not ship both the old Alioth AW8697 driver and the OPlus AW8697 driver.
  They claim the same compatible string and I2C address.
- Do not ship both the old Alioth vibrator HAL and the Richtap HAL as the
  default `android.hardware.vibrator.IVibrator/default`.
- Flashing `vendor.img` does not change the kernel, but a vendor image from a
  different build tree can mismatch other vendor blobs. Keep a known-good
  vendor image.
- The OnePlus 8 config is not Alioth-calibrated. Effects may be weak, wrong, or
  absent even when the API returns success.

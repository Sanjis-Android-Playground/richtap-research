> Historical note: this is the pre-implementation feasibility report. It says
> "no changes made" because that was true when written. See
> [../POSTMORTEM.md](../POSTMORTEM.md) for the implemented stack and the
> failures found on device.

# oplus-kona vibrator stack → alioth (AW8697) — feasibility report

Date: 2026-10-30
Target tree: `/home/braindead/luna` (Lunaris / LineageOS 23.2, Android 16, `lineage_alioth`)
Status: research only. No changes made to any device, kernel, vendor or build tree.

Reference context also read:
- https://github.com/PocoF3Releases/Agents.md (`memory/haptics.md`, `CURRENT_STATE.md`, `evidence/haptics/*`)
- https://github.com/PocoF3Releases/hardware_xiaomi (`cnb`, `vibrator/aw8697`)
- https://github.com/oplus-kona/* (kernel, vendor, device trees)
- https://github.com/LineageOS/android_hardware_oplus (`lineage-23.2`, `aidl/vibrator/richtap`)

---

## 1. TL;DR

**Yes, it can be done, but it is a full stack replacement, not an add-on.**

oplus-kona's "vibrator support" is three separate things:

1. An OPLUS/AWINIC kernel driver in `drivers/misc/aw8697_haptic/` that exposes
   `/dev/awinic_haptic` with the AAC RichTap ioctl/mmap ABI (no Linux input-FF).
2. Proprietary OOS userspace: `libaacvibrator.so` + `aac_richtap.config`
   (+ optional RTP waveform bins and RAM firmware).
3. The HAL, which is **not in oplus-kona** — it is
   `LineageOS/android_hardware_oplus/aidl/vibrator/richtap`
   (already merged in `lineage-23.2`, so it matches this ROM's branch).

On alioth the chip is the same (AW8697 at I2C 0x5A), and the OPLUS driver's
compatible string (`awinic,aw8697_haptic`) even matches our DTS node, so the
hardware side is compatible.

The problem is the trade-off:

| | Current alioth stack | oplus-kona RichTap stack |
|---|---|---|
| Kernel interface | input FF + `custom_wave` RTP ring (ABI v2) | `/dev/awinic_haptic` + RichTap ioctls/mmap |
| Effect source | alioth stock RAM firmware + stock PCM | OOS AAC prebaked effects (encrypted config) |
| AIDL composition (`compose`) | yes | **no** |
| Primitive streaming (CLICK/THUD/LIGHT_TICK) | yes (3 stock primitives) | **no** |
| on()/off()/amplitude | yes | yes |
| App-facing RichTap SDK (HE/HE1/vendor extension) | no | **also no** (HAL only implements standard IVibrator) |
| Tuning | alioth stock | OnePlus 8/8T/9R motor tuning |
| HAL maturity | your own, tested | niche Lineage path; 24.0 change still open |

So the oplus route buys you **OOS-tuned standard effects** and a real-time AAC
engine, but it **loses** the composition/primitives work already done here, and
it does **not** close the "full app-facing RichTap" gap noted in the Agents.md
project notes.

Recommended first move is **not** the full port. It is a zero-risk A/B test of
the OOS RTP PCM files against the current HAL (see §7).

---

## 2. What is actually in oplus-kona

### 2.1 Kernel

`oplus-kona/android_kernel_oneplus_sm8250`, branch `aosp-17`, 4.19.325.

```
drivers/misc/aw8697_haptic/
  aw8697.c                 456 KB, ~14,200 lines, `#define AAC_RICHTAP`
  aw8697.h                  17 KB, RICHTAP ioctl numbers, mmap geometry
  aw8697_reg.h              22 KB
  aw8697_config.h           register/algorithm constants
  haptic_hv.c              273 KB generic AWINIC haptic_hv driver (same AAC ABI)
  haptic_hv.h / reg / rtp_key_data.h
  aw8692x.c                 61 KB (other chips)
  Kconfig / Makefile
```

Key properties of the OPLUS `aw8697.c`:

- `AW8697_I2C_NAME "aw8697_haptic"`, `AW8697_HAPTIC_NAME "awinic_haptic"`.
- of_match: `{ .compatible = "awinic,aw8697_haptic" }` — **same as alioth**.
- Registers `misc_register(&aw8697_haptic_misc)` → `/dev/awinic_haptic`.
- `AAC_RICHTAP` is always on (`aw8697.h:339`).
- RichTap UAPI (`aw8697.h:353+`):
  - `RICHTAP_GET_HWINFO` (`_IO(0x52,0x03)`) returns `RICHTAP_AW_8697 = 0x05`
  - `RICHTAP_RTP_MODE`, `RICHTAP_STREAM_MODE`, `RICHTAP_STOP_MODE`
  - `RICHTAP_SETTING_GAIN`, `RICHTAP_GET_F0`, ...
  - 16 × 1000-byte mmap buffers (`RICHTAP_MMAP_BUF_SIZE/SUM`)
- **No `FF_CUSTOM`, no `custom_wave`, no `EVIOCSFF`.** It does not provide the
  alioth HAL's input-FF ABI at all.
- Loads firmware by name with `request_firmware()`; per-`device_id` RAM name
  table (`aw8697_haptic_170.bin` for 815/1815) and a huge RTP name table with
  f0 variants (145…237 Hz).
- `parse_dt()` only reads `reset-gpio`, `irq-gpio`, `qcom,device_id`,
  `qcom,aw8697_boost_voltage`. It ignores the `vib_*` tuning properties in our
  alioth DTS node.

DTS used by OnePlus 8T (`kebab`):
```
aw8697_haptic@5A {
    compatible = "awinic,aw8697_haptic";
    reg = <0x5A>;
    reset-gpio = <&tlmm 72 0>;
    irq-gpio = <&tlmm 11 0>;
    qcom,device_id = <1815>;
    pinctrl-names = "default";
    pinctrl-0 = <&aw_irq_gpio_default>;
};
```
alioth uses the same chip/address but reset GPIO 130 / IRQ 129 and a full
`vib_*` property block plus three `pinctrl-*` states.

### 2.2 Vendor blobs

`oplus-kona/android_vendor_oneplus_sm8250-common`, branch `aosp-17`:

| Path | Size | Notes |
|---|---:|---|
| `proprietary/vendor/lib64/libaacvibrator.so` | 103 KB | ELF aarch64, stripped, `BIND_NOW` |
| `proprietary/odm/etc/aac_richtap.config` | 80 KB | obfuscated/encrypted, version + SHA1 checked by the lib |
| `proprietary/odm/firmware/aw8697_haptic*.bin` | 5–6 KB | RAM firmware per f0 (166/168/170/172/174/235) |
| `proprietary/odm/firmware/aw8697_*_RTP_*.bin` | 1019 files / ~225 MB | **raw signed 8-bit 24 kHz PCM**, no header |

`libaacvibrator.so` dependencies are all standard: `liblog, libcutils, libutils,
libc++, libc, libm, libdl`. Undefined symbol `systemTime` is exported by the
ROM's `libutils.so` (verified in `out/target/product/alioth/*/lib64/libutils.so`),
so there is no ELF/ABI blocker.

Strings confirm: `RICHTAP_DEVICE_PATH`, `ENV_RICHTAP_CONFIG_PATH`,
`vendor.aac_richtap.config.path`, `config file version does not match so version`,
`SHA1 check failed`, `mmap buf list failed`. **The .so and .config must come from
the same OOS build.** Do not mix them.

### 2.3 HAL

`LineageOS/android_hardware_oplus/aidl/vibrator/richtap` (merged on
`lineage-23.2`; a forward-port to `lineage-24.0` is still an open change, 505569):

- `cc_binary "android.hardware.vibrator.service.oplus-richtap"`
- links a stub `libaacvibrator` (real .so copied from vendor blobs)
- rc:
  ```
  on boot
      chown system system /dev/awinic_haptic
      chmod 0600 /dev/awinic_haptic
  service vendor.vibrator /vendor/bin/hw/android.hardware.vibrator.service.oplus-richtap
      class late_start
      user system
      group system input
      setenv RICHTAP_DEVICE_PATH /dev/awinic_haptic
      setenv ENV_RICHTAP_CONFIG_PATH /odm/etc/aac_richtap.config
  ```
- manifest: AIDL `android.hardware.vibrator` version 2, `IVibrator/default`
- `perform()` supports only `CLICK, DOUBLE_CLICK, TICK, THUD, POP, HEAVY_CLICK`
  mapped to AAC strengths 69/89/99
- `compose`, primitives, PWLE, always-on, frequency control: **unsupported**
- `getResonantFrequency` unsupported (so framework still logs "Invalid frequency
  profile")

Device integration in oplus-kona `common.mk:383`:
```
PRODUCT_PACKAGES += android.hardware.vibrator.service.oplus-richtap
$(call soong_config_set_bool,OPLUS_LINEAGE_VIBRATOR_HAL,USES_OPLUS_AWINIC,true)
PRODUCT_COPY_FILES += vendor/qcom/opensource/vibrator/excluded-input-devices.xml:...
```
(`USES_OPLUS_AWINIC` is not consumed by the current HAL's Android.bp; likely a
leftover. Ignore it.)

SELinux in `hardware/oplus/sepolicy/qti/vendor`:
```
type aac_richtap_device, dev_type;
/dev/awinic_haptic  u:object_r:aac_richtap_device:s0
/vendor/bin/hw/android\.hardware\.vibrator\.service\.oplus-richtap u:object_r:hal_vibrator_default_exec:s0
allow hal_vibrator_default aac_richtap_device:chr_file rw_file_perms;
```

Note: the **official LineageOS OnePlus sm8250 device tree does not use this**.
It uses `vendor.qti.hardware.vibrator.service.oplus`, which talks to
`/sys/class/leds/vibrator/*`. alioth's AW8697 driver exposes no LED class, so
that HAL is not an option. oplus-kona's device tree is the Richtap reference.

---

## 3. What alioth has today (in this tree)

| Piece | Location |
|---|---|
| Flag | `device/xiaomi/alioth/device.mk:11` `TARGET_USES_AW8697_VIBRATOR := true` |
| Product selection | `device/xiaomi/sm8250-common/kona.mk:477-490` |
| HAL | `hardware/xiaomi/vibrator/aw8697/` (`android.hardware.vibrator-service.xiaomi_aw8697`) |
| Effect loader | `hardware/xiaomi/vibrator/effect/` (`libqtivibratoreffect.xiaomi`) |
| Kernel driver | `kernel/xiaomi/sm8250/drivers/input/misc/aw8697_haptic/` (4.19.404) |
| Second driver (disabled) | `.../aw8697_haptic_munch/` (`CONFIG_INPUT_AW8697_HAPTIC_MUNCH is not set`) |
| Defconfig | `arch/arm64/configs/vendor/alioth_defconfig:2254` `CONFIG_INPUT_AW8697_HAPTIC=y` |
| DTS | `arch/arm64/boot/dts/vendor/qcom/xiaomi-sm8250-common.dtsi:236` |
| Firmware | `vendor/xiaomi/alioth/proprietary/vendor/firmware/{aw8697_haptic.bin,aw8697_rtp_1.bin}` |
| Primitives | `vendor/etc/vibrator/primitive_effect_{1,2,7}.bin` |
| SELinux | `device/xiaomi/sm8250-common/sepolicy/vendor/{file_contexts,hal_vibrator_default.te}` |

Current HAL capabilities (from source):
- 7 effects: CLICK, DOUBLE_CLICK, TICK, THUD, POP, HEAVY_CLICK, TEXTURE_TICK
- 3 primitives: 1 (CLICK), 2 (THUD), 7 (LIGHT_TICK / Android TICK)
- `compose`, `getSupportedPrimitives`, `getPrimitiveDuration`
- `on()`/`off()`, amplitude, resonant frequency from `f0_value`
- streams signed 8-bit 24 kHz PCM through `custom_wave` (kernel ABI v2),
  effect id 193, ring period 512, ≤4095-byte writes, ≤240000 samples
- direct RAM gain extension (`0x4700|gain`) on top of the legacy FF gain
- built image currently contains only the 3 primitives; `effect_*.bin` are absent,
  so the 7 effects still use the stock RAM bank

This is materially more capable than the Lineage Richtap HAL.

---

## 4. Hardware / software compatibility

| Check | Result |
|---|---|
| Chip | AW8697 both sides (`RICHTAP_AW_8697 = 0x05`) ✅ |
| I2C address | 0x5A both ✅ |
| DT compatible | `awinic,aw8697_haptic` both ✅ |
| Kernel base | 4.19.325 (oplus) vs 4.19.404 (alioth) — same LTS line ✅ |
| Framework | AIDL IVibrator V2 both ✅ |
| Blob ABI | aarch64 only, standard deps, `systemTime` resolves ✅ |
| Config/lib pairing | must be same OOS build; version+SHA1 enforced ⚠️ |
| Motor tuning | config is OnePlus 8/8T/9R; alioth is 170 Hz AW8697 LRA ⚠️ |
| HAL feature coverage | loses compose/primitives ⚠️ |
| Both drivers at once | impossible (same compatible, same chip) ❌ |

Additional porting friction found:

1. `Makefile` force-defines `CONFIG_OPLUS_HAPTIC_OOS`, which includes
   `<soc/oplus/system/boot_mode.h>`. That header does not exist in the alioth
   kernel (`include/oplus/system/` only has project headers). Either drop the
   define or add a stub.
2. `OPLUS_FEATURE_CHG_BASIC` sections include `<soc/oplus/system/oplus_project.h>`
   under a different path; leave that define off.
3. The OPLUS driver does not select alioth's three `pinctrl-*` states. Either add
   a `default` pinctrl state covering reset+IRQ, or patch the driver to select
   them. Otherwise reset may not toggle reliably.
4. RAM firmware name mismatch: OPLUS asks for `aw8697_haptic_170.bin` (device_id
   815/1815), alioth ships `aw8697_haptic.bin`. Rename/copy or patch the table.
5. `device_id` is absent in alioth DTS → defaults to 815 (OnePlus 8 tuning).
   `1815` selects 8T tuning; both use the 170 Hz tables.
6. `hardware/oplus` is not in the Lunaris/Lineage default manifest. It must be
   added (local manifest or `lineage.dependencies`), or the four richtap files
   can be cherry-picked into `hardware/xiaomi`.
7. The ROM has an ODM partition, so `/odm/etc/aac_richtap.config` is possible
   without patching the HAL's rc. `/odm/firmware` is also a standard
   `request_firmware` search path.
8. SELinux additions are small: new `aac_richtap_device` type, `/dev/awinic_haptic`
   label, HAL exec label, allow `hal_vibrator_default` rw, and read access to the
   ODM config.

---

## 5. Options

### Option A — full RichTap port (exact oplus-kona stack)

Replace the AW8697 kernel driver, add blobs, add the Lineage Richtap HAL, remove
the current HAL.

**Gives:** real-time AAC prebaked effects; OOS feel (subject to motor tuning);
f0-adaptive behavior inside the lib; small footprint (≈200 KB + config + one RAM
firmware) if you skip the 225 MB RTP catalog.

**Costs / loses:** compose, primitives, current PCM control, TEXTURE_TICK,
current tested behavior. No app-facing RichTap. Two proprietary blobs with
opaque version coupling. OnePlus-tuned effects on a different motor.

**Effort:** 2–4 days for a working build, plus tuning/debug. Highest risk is the
kernel driver import and the opaque config.

### Option B — graft OOS RTP PCM into the current stack (recommended first test)

The 1019 `aw8697_*_RTP_*.bin` files are plain signed 8-bit 24 kHz PCM, exactly
what the current HAL streams. The HAL already loads
`/vendor/etc/vibrator/effect_<id>.bin` and `primitive_effect_<id>.bin`, and the
kernel already exposes the v2 RTP ABI.

**No kernel change, no HAL code change.** Add a handful of .bin files to the
device/vendor tree, map them to `effect_0/2/3/4/5/6` and `primitive_3/4/5/6/8`,
rebuild, listen. Fully reversible.

**Gives:** OOS waveform texture for the effects you choose, plus more composition
primitives.
**Does not give:** the AAC prebaked effect engine, f0 adaptation, app-facing
RichTap.
**Limits:** files must be ≤240000 samples (10 s); HAL rejects larger; choose short
UI candidates (e.g. `desk_7_RTP_54` 136 ms, `vibrator_remain_12_RTP_56` 217 ms,
`strength_level_bar_edge_RTP_107` 71 ms, `two_error_remaind_RTP_46` 211 ms).
**Effort:** ~1–2 hours + one vendor build.

### Option C — hybrid (best long-term, highest effort)

Port only the RichTap ioctl/mmap ABI into the existing alioth driver (or add the
input-FF/RTP ABI to the OPLUS driver) and call `aac_vibra_*` from the current HAL
for `perform()`, keeping the current compose/primitives paths. This preserves
Android feature coverage and gets AAC effect quality.

**Effort:** high (merge of two driver generations + HAL work). Only worth it if
Option B's sound is insufficient and Option A's feature loss is unacceptable.

### Option D — keep current HAL, use the project's existing AAC renderer

The Agents.md project already has `xiaomi-alioth-richtap-render` using Xiaomi's
own `libaachaptics.so` (AAC 1.0.7, `aac_haptics_*` API, config {512,24000,1,170})
to render RichTap/HE1 events to PCM offline. That renderer is deliberately not
in the HAL (the library has no shutdown API and can block).

A production-safe variant would pre-render a small effect pack at build/boot and
let the current HAL stream it. This is independent of oplus-kona and avoids its
blobs, but it is not "real-time RichTap" either.

---

## 6. Risk register (Option A)

| Risk | Impact | Mitigation |
|---|---|---|
| lib/config version mismatch | HAL init fails, no vibration | ship the matched pair; never mix OOS builds |
| config is tuned for OnePlus motor | wrong feel, buzz, weak effects | test; keep stock RAM bin; allow A/B |
| kernel driver won't compile cleanly | build break | isolate in a branch; fix boot_mode/pinctrl |
| `device_id` mismatch | wrong RAM/RTP tuning | set 815 or 1815 explicitly; test both |
| two IVibrator services | one HAL fails to register | remove the xiaomi HAL package + service |
| SELinux denials | HAL can't open device/config | add the four rules above |
| ODM size / 225 MB RTP | partition overflow | ship ≤20 selected bins or none |
| proprietary redistribution | legal | keep in vendor blobs, not in source |
| Lineage 24.0 change still open | future rebase work | we are on 23.2, code is merged |

---

## 7. Recommended plan

**Phase 0 — baseline (no changes)**
- Rebuild/flash current tree; confirm the new HAL is actually running.
- `adb shell dumpsys vibrator_manager`
- `adb logcat -b all | grep -iE "vibrator|aw8697|qtivibrator"`
- check `custom_wave` fields (`abi_version`, `period_size`, `free_size`,
  `playback_active`, `ram_gain_abi`) and `f0_value`.
- The Oct 27 boot logs predate the new HAL, so ignore their
  `effect_*.bin` / `VibratorInfo` errors until re-checked.

**Phase 1 — Option B A/B test**
- Pick 4–8 short OOS RTP bins (24 kHz, ≤240000 B).
- Install as `effect_N.bin` / `primitive_effect_N.bin` in the vendor image.
- Compare CLICK/DOUBLE_CLICK/TICK/THUD/POP/HEAVY_CLICK/TEXTURE_TICK and compose.
- Decide purely on feel + logcat.

**Phase 2 — only if needed, Option A prototype**
- Branch the kernel; import `drivers/misc/aw8697_haptic` (aw8697 only).
- Add a `default` pinctrl state and `qcom,device_id`.
- Build a vendor image with lib+config+RAM firmware.
- Add `hardware/oplus` and the Richtap HAL; remove the current HAL.
- Validate on device before any public build.

**Do not** ship both HALs or both drivers.

---

## 8. Questions before touching anything

1. Do you want the goal to be "OOS-feeling standard effects" (Options A/B) or
   "app-facing RichTap SDK" (neither A nor B provides it today)?
2. Is the current HAL's compose/primitives coverage worth preserving? If yes,
   Option A is a regression and Option C/D is the real target.
3. Are you OK shipping `libaacvibrator.so` + `aac_richtap.config` (proprietary,
   opaque, version-locked) in the vendor image?
4. Which do you want me to prototype first: the zero-risk RTP-bin A/B test, or
   the kernel driver import in a separate worktree/branch?
5. Do you have the device available for listening tests, or should I prepare a
   build-only change set for you to flash?

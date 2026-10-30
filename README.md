# Alioth RichTap / AW8697 haptics research

Status: **parked, not fixed for UI haptics**. This repository records the
Alioth AW8697 + OPlus AAC RichTap port, the exact failures observed on device,
the evidence, and the remaining fix paths.

The device can produce timed vibration through `on()`. UI haptic effects
(`perform()`) were still not working when this was parked. Two separate bugs
were confirmed; a third playback-layer issue remains open.

## Read this first

| Document | Purpose |
|---|---|
| [docs/POSTMORTEM.md](docs/POSTMORTEM.md) | What was built, what failed, and why |
| [docs/FIX-PLAN.md](docs/FIX-PLAN.md) | Ordered fix options and exact code locations |
| [docs/EVIDENCE.md](docs/EVIDENCE.md) | Logs, disassembly, hashes, and line references |
| [docs/COMMANDS.md](docs/COMMANDS.md) | Working Termux, build, flash, and test commands |
| [docs/ARTIFACTS.md](docs/ARTIFACTS.md) | Local artifact hashes and provenance |
| [docs/PROVENANCE.md](docs/PROVENANCE.md) | Upstream sources and licensing notes |
| [patches/](patches/) | The applied `0x1000` effect-ID patch |

## Confirmed findings

1. **The original effect-file HAL was dead because payload files were missing.**
   The old `libqtivibratoreffect.xiaomi` HAL requested `effect_*.bin` and
   `primitive_effect_*.bin`; the build shipped only
   `primitive_effect_{1,2,7}.bin`. That explained the earlier "vibration dead"
   logs.

2. **The Richtap HAL fixed `on()` but not `perform()`.**
   `AAC init success: 5` means the AAC library opened `/dev/awinic_haptic`, read
   `/odm/etc/aac_richtap.config`, and initialized the actuator.

3. **First confirmed bug: wrong effect-ID namespace.**
   The HAL passed raw AIDL IDs `0..5` to
   `aac_vibra_looper_prebaked_effect()`. The imported `libaacvibrator.so`
   subtracts `0x1000` internally, so CLICK became `-4096` and TICK became
   `-4094`; the config lookup failed, `data_len` became 0, and the HAL returned
   a fake 1 ms success.

   The local fix is:

   ```cpp
   aac_vibra_looper_prebaked_effect(0x1000 + static_cast<uint32_t>(effect),
                                    strength);
   ```

   After that, CLICK resolved to `effect_id:4096`, `data_len=264`,
   `duration_time=12`; TICK resolved to `4098`, `data_len=704`,
   `duration_time=30`.

4. **Second confirmed bug: the imported OnePlus 8 config only exposes three
   prebaked effects.**
   `HEAVY_CLICK` (`0x1000 + 5 = 4101`) still logs:

   ```text
   AacRichTapConfig: effect_id is invalid ,effect_id:5,size:3
   AacRichTapConvert: data_len = 0, duration_time:1
   ```

   The table used reports size 3, so effect indices 0, 1, 2 resolve. THUD (3),
   POP (4), and HEAVY_CLICK (5) are rejected. Long-press uses HEAVY_CLICK, so
   long-press haptic is still dead even after the `0x1000` fix.

5. **Open issue: the RTP playback layer may still be silent.**
   The `boot3` live log proves conversion now returns data, but the user still
   reported no physical haptic. The logs contain no `AacRichTapPerformer` error.
   The kernel's `RICHTAP_RTP_MODE` path ignores the return value of
   `aw8697_i2c_writes()`, so a failed RTP data write is invisible to the HAL.
   This needs one runtime `dmesg` capture to separate "no data" from "data
   written but not played".

## Do not confuse these with the root cause

- `VibratorInfo: Invalid frequency profile ... NaN` is cosmetic. The HAL does
  not implement `getResonantFrequency()`.
- `init` AVC denials on `trigger`, `activate`, `brightness`, `duration`, and
  `state` are init-only sysfs writes. There is no `hal_vibrator_default`
  denial. They are noisy, not the reason `perform()` is silent.
- `aw8697_rtp_loaded: failed to read aw8697_rtp.bin` is a real packaging defect
  because the vendor tree ships `aw8697_rtp_1.bin`, but this is boot preload;
  it is not proven to be the `perform()` failure.
- `vibrate_on=null` is not a problem. The framework defaults it to `1`.
- `cmd vibrator` is the wrong service name. Use `cmd vibrator_manager`.

## Local source state at park time

The `luna` tree contains one local, uncommitted change in
`hardware/xiaomi/vibrator/richtap/Vibrator.cpp`:

```diff
-    int32_t ret = aac_vibra_looper_prebaked_effect(static_cast<uint32_t>(effect), strength);
+    // libaacvibrator expects RichTap effect IDs, not raw AIDL Effect values.
+    int32_t ret = aac_vibra_looper_prebaked_effect(0x1000 + static_cast<uint32_t>(effect),
+                                                   strength);
```

It was built successfully. The resulting HAL and `vendor.img` hashes are in
[docs/ARTIFACTS.md](docs/ARTIFACTS.md). It was **not committed or pushed** in
the device repositories.

## License / blobs

No OPlus/OOS proprietary library or config is committed here. The hashes and
local paths are recorded instead. See [docs/PROVENANCE.md](docs/PROVENANCE.md).

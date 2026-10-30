# Alioth Richtap postmortem

Date: 2026-10-30
Device: Xiaomi Alioth (Poco F3 / Redmi K40)
Chip: AW8697 at I2C `2-005a`
Kernel: 4.19.404, OPlus `drivers/misc/aw8697_haptic`
Userspace: Lineage Richtap HAL + OPlus `libaacvibrator.so` + OnePlus 8
`aac_richtap.config`

## 1. Goal and result

Goal: replace Alioth's stock input-FF AW8697 HAL with the OPlus AAC RichTap
stack so standard Android effects use the OOS AAC engine.

Result: the replacement boots and `on()` works. UI `perform()` effects did not
work reliably when parked. Two bugs were confirmed, one remains open.

## 2. Stack

```text
Android VibratorManagerService
        |
        v
android.hardware.vibrator.IVibrator/default
        |
        v
hardware/xiaomi/vibrator/richtap/Vibrator.cpp
        |
        v
libaacvibrator.so  (OPlus proprietary, aarch64)
        |
        +--> /odm/etc/aac_richtap.config  (OPlus proprietary, encrypted)
        |
        v
/dev/awinic_haptic
        |
        v
kernel/xiaomi/sm8250/drivers/misc/aw8697_haptic/aw8697.c
        |
        v
AW8697 over I2C
```

Relevant source paths:

```text
hardware/xiaomi/vibrator/richtap/Vibrator.cpp
hardware/xiaomi/vibrator/richtap/aac_vibra_function.h
hardware/xiaomi/vibrator/richtap/libaacvibrator.so
kernel/xiaomi/sm8250/drivers/misc/aw8697_haptic/aw8697.c
kernel/xiaomi/sm8250/drivers/misc/aw8697_haptic/aw8697.h
device/xiaomi/sm8250-common/rootdir/etc/ueventd.qcom.rc
device/xiaomi/sm8250-common/sepolicy/vendor/hal_vibrator_default.te
vendor/xiaomi/alioth/proprietary/odm/etc/aac_richtap.config
```

## 3. What the successful init proved

`boot3/logcat (13).txt`:

```text
AacRichTapConfig: config path:/odm/etc/aac_richtap.config
AacRichTapConfig: config version:4
AacRichTapConfig: config file version match so version
AacRichTapConfig: client:Oneplus,device type:oneplus8
AacRichTapConfig: vibrator type:ELA0815
AacRichTapConfig: create prebak_effec effect_num:3
AacRichTapConfig: create prebak_effec effect_num:11
AacRichTapConfig: config file read end 4 g_is_config_read_success:1
AacRichTapVibrator: device_path:/dev/awinic_haptic
android.hardware.vibrator.service.oplus: AAC init success: 5
```

That rules out:

- wrong config path
- config/lib version mismatch
- failure to open `/dev/awinic_haptic`
- SELinux denial of the HAL device
- missing kernel driver

Kernel side in `boot3/dmesg (4).txt`:

```text
aw8697_i2c_probe probe completed successfully!
aw8697_haptic_f0_calibration get f0=1703
aw8697_ram_loaded: check sum pass : 0xe010
aw8697_ram_loaded: fw update complete
```

## 4. Bug 1 — raw AIDL effect IDs vs RichTap IDs

### Symptom

`on()` produced vibration; `perform()` produced nothing and logged no
`AAC perform failed`.

### Before fix

`boot2/30_10-21-05-46_030.log`:

```text
AacRichTapConvert: effect_id:0,strength:89
AacRichTapConfig: effect_id is invalid ,effect_id:-4096,size:3
AacRichTapConvert: data_len = 0,interval_time_us:41.666668,level:1,duration_time:1
```

```text
AacRichTapConvert: effect_id:2,strength:89
AacRichTapConfig: effect_id is invalid ,effect_id:-4094,size:3
AacRichTapConvert: data_len = 0,interval_time_us:41.666668,level:1,duration_time:1
```

The arithmetic is exact:

```text
0 - 4096 = -4096
2 - 4096 = -4094
```

### Cause

The HAL passed the raw AIDL value:

```cpp
// hardware/xiaomi/vibrator/richtap/Vibrator.cpp:96 (before fix)
aac_vibra_looper_prebaked_effect(static_cast<uint32_t>(effect), strength);
```

The imported library expects the RichTap namespace and subtracts `0x1000`.
Disassembly at `libaacvibrator.so` VA `0x14dfc`:

```asm
sub w1, w1, #0x1, lsl #12    ; w1 = w1 - 0x1000
```

Valid input range is checked just before it:

```asm
lsr w9, w1, #12
cmp w9, #0x2
b.ls valid
```

So the accepted input is `0x1000..0x2FFF`; the HAL was feeding `0..5`.

### Fix and proof

```cpp
aac_vibra_looper_prebaked_effect(0x1000 + static_cast<uint32_t>(effect),
                                 strength);
```

The built HAL contains:

```asm
3be8: 32140020  orr w0, w1, #0x1000
```

`boot3/30_10-22-34-13_550.log` after the fix:

```text
AacRichTapConvert: effect_id:4096,strength:89
AacRichTapConvert: data_len = 264,interval_time_us:41.666668,level:1,duration_time:12
```

```text
AacRichTapConvert: effect_id:4098,strength:89
AacRichTapConvert: data_len = 704,interval_time_us:41.666668,level:1,duration_time:30
```

`4096 = 0x1000 + CLICK(0)`, `4098 = 0x1000 + TICK(2)`.

This bug is **fixed at the conversion layer**.

## 5. Bug 2 — the OnePlus config only has three prebaked effects

The same `boot3` live log shows:

```text
AacRichTapConvert: effect_id:4101,strength:89
AacRichTapConfig: effect_id is invalid ,effect_id:5,size:3
AacRichTapConvert: data_len = 0,duration_time:1
```

`4101 = 0x1000 + HEAVY_CLICK(5)`.

The config reports:

```text
create prebak_effec effect_num:3
```

With `size:3`, the valid post-`0x1000` indices are 0, 1, 2. Therefore:

| AIDL effect | ID | Result |
|---|---:|---|
| CLICK | 0 | valid |
| DOUBLE_CLICK | 1 | valid |
| TICK | 2 | valid |
| THUD | 3 | invalid |
| POP | 4 | invalid |
| HEAVY_CLICK | 5 | invalid |
| TEXTURE_TICK | 21 | rejected earlier by HAL |

Android uses HEAVY_CLICK for long-press. The log line occurs immediately after:

```text
NavHandleLongPressHandler: Contextual Search invocation
```

So long-press haptic is still dead even with the `0x1000` fix.

The config also logs a second table:

```text
create prebak_effec effect_num:11
```

It is not yet known whether that table is for another actuator, another
strength set, or another OPlus device. The library selected the 3-entry table
for `vib_id:0`.

## 6. Open issue — conversion succeeds but physical output may still be absent

After the `0x1000` fix, the log proves CLICK/TICK data exists. It does **not**
prove the actuator moved.

The user reported no improvement. The `boot3` live log contains no
`AacRichTapPerformer` error and no `AAC perform failed`. That is expected:
the library only logs failures on some paths, and the HAL returns success when
the library returns a positive duration.

The kernel RTP path is the next suspect:

```c
case RICHTAP_RTP_MODE:
    ...
    aw8697_haptic_play_mode(aw8697, AW8697_HAPTIC_RTP_MODE);
    aw8697_haptic_start(aw8697);
    usleep_range(2000, 2500);
    aw8697_i2c_writes(aw8697, AW8697_REG_RTP_DATA,
                      &aw8697->rtp_ptr[4], tmp);
    break;
```

Two facts matter:

1. `aw8697_i2c_writes()` return value is ignored. If the RTP data write fails,
   userspace still gets ioctl success.
2. The stream path used by `on()` is different: it schedules
   `haptic_rtp_work`, enables the FIFO AEI, and feeds data with
   `aw8697_haptic_set_rtp_aei()` and repeated writes. The `on()` path works.

Hypotheses to test, in order:

- I2C write failure for 264/704-byte RTP payload
- FIFO/start ordering: chip starts before data is written
- missing RTP FIFO interrupt enable in `RICHTAP_RTP_MODE`
- config/actuator mismatch making the data effectively inaudible
- missing `aw8697_rtp.bin` altering the expected boot state

Do not treat any of these as confirmed until a runtime dmesg is captured.

## 7. Non-causes and noise

### `VibratorInfo: Invalid frequency profile ... NaN`

The HAL does not implement `getResonantFrequency()`. This is metadata only.

### `init` AVC denials

`boot3/dmesg.avc (2).txt` shows `init` denied `setattr` on AW8697 sysfs nodes.
These are from init's `chown`/`chmod` actions, not from
`hal_vibrator_default`. The HAL's own access is allowed.

Suggested policy if desired:

```te
allow init sysfs_haptic_aw8697:file { setattr write };
```

or remove the unnecessary init `chown` lines.

### `aw8697_rtp_loaded: failed to read aw8697_rtp.bin`

The kernel table asks for `aw8697_rtp.bin`; the vendor tree ships
`aw8697_rtp_1.bin`. This is a real packaging defect. It is boot preload and does
not by itself explain `on()` working while `perform()` is silent, but it should
be fixed for completeness.

### `TEXTURE_TICK` unsupported

The HAL's `perform()` range check is:

```cpp
if (effect < Effect::CLICK || effect > Effect::HEAVY_CLICK)
```

`TEXTURE_TICK` is AIDL ID 21, so it is rejected before the library. The old
Alioth HAL mapped it to RAM slot 5. This is a fidelity gap; it is not the
current `effect_id:-4096` failure.

### `cmd vibrator` not found

The Android service is `vibrator_manager`, not `vibrator`. The effect
subcommand is `synced`, not `vibrate`.

## 8. Timeline

| Date | Event |
|---|---|
| 2026-10-27 | Old effect-file HAL logs showed missing `effect_*.bin`; vibration dead |
| 2026-10-30 | OPlus driver + Richtap HAL + OnePlus config built |
| 2026-10-30 | New logs: `AAC init success: 5`; no `perform` entries |
| 2026-10-30 | First live trigger log: `effect_id:0 -> -4096`, `data_len=0` |
| 2026-10-30 | Applied `0x1000 + effect`; HAL and `vendor.img` built |
| 2026-10-30 | `boot3`: CLICK/TICK data valid; HEAVY_CLICK still invalid |
| 2026-10-30 | Parked pending decision: fallback mapping vs config replacement vs kernel RTP debug |

## 9. Lessons

- A successful `AAC init` does not mean `perform()` works. Test both paths.
- The OPlus AAC library has a 0x1000 effect namespace. Never pass raw AIDL IDs.
- The OPlus config is opaque and device-specific. A OnePlus 8 config is not an
  Alioth config.
- `getSupportedEffects()` must not advertise more than the config can resolve.
- The kernel RTP path and the stream path are different; a working `on()` does
  not prove the RTP path works.
- The driver can fail silently if its internal write return is ignored. Capture
  runtime dmesg, not only logcat.

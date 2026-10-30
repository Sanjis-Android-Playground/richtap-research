# Evidence index

Local evidence used for the conclusions in [POSTMORTEM.md](POSTMORTEM.md).
Full logs are intentionally not committed: they contain device identifiers and
unrelated app data.

## 1. Old HAL failure (historical)

`bootlogs2/boot/logcat.txt`:

```text
libqtivibratoreffect.xiaomi: Failed to open "/vendor/etc/vibrator/effect_0.bin" for effect 0
libqtivibratoreffect.xiaomi: Failed to open "/vendor/etc/vibrator/effect_1.bin" for effect 1
libqtivibratoreffect.xiaomi: Failed to open "/vendor/etc/vibrator/effect_2.bin" for effect 2
libqtivibratoreffect.xiaomi: Failed to open "/vendor/etc/vibrator/effect_3.bin" for effect 3
libqtivibratoreffect.xiaomi: Failed to open "/vendor/etc/vibrator/effect_4.bin" for effect 4
libqtivibratoreffect.xiaomi: Failed to open "/vendor/etc/vibrator/effect_5.bin" for effect 5
libqtivibratoreffect.xiaomi: Failed to open "/vendor/etc/vibrator/effect_21.bin" for effect 21
libqtivibratoreffect.xiaomi: Failed to open "/vendor/etc/vibrator/primitive_effect_3.bin" for effect 3
libqtivibratoreffect.xiaomi: Failed to open "/vendor/etc/vibrator/primitive_effect_4.bin" for effect 4
libqtivibratoreffect.xiaomi: Failed to open "/vendor/etc/vibrator/primitive_effect_5.bin" for effect 5
libqtivibratoreffect.xiaomi: Failed to open "/vendor/etc/vibrator/primitive_effect_6.bin" for effect 6
libqtivibratoreffect.xiaomi: Failed to open "/vendor/etc/vibrator/primitive_effect_8.bin" for effect 8
```

Only `primitive_effect_{1,2,7}.bin` were shipped. The old HAL was not playing
because its payload set was incomplete.

## 2. Richtap init succeeds

`boot3/logcat (13).txt` lines 8254-8299:

```text
AacRichTapConfig: token:/odm/etc/aac_richtap.config
AacRichTapVibrator: config_count:1
AacRichTapConfig: path_count:1
AacRichTapConfig: config path:/odm/etc/aac_richtap.config
AacRichTapConfig: config_data_len:81920,file_size:81940
AacRichTapConfig: config version:4
AacRichTapConfig: config file version match so version
AacRichTapConfig: client:Oneplus,client item:oneplus,device type:oneplus8
AacRichTapConfig: vibrator type:ELA0815
AacRichTapConfig: interrupt_protect_time:30
AacRichTapConfig: jnd_value:0.140000,jnd_step:10.000000
AacRichTapConfig: config_version:4,so_version:4
AacRichTapConfig: create prebak_effec effect_num:3
AacRichTapConfig: create prebak_effec effect_num:11
AacRichTapConfig: config file read end 4 g_is_config_read_success:1
AacRichTapConfig: get actuator vib_id:0
AacRichTapVibrator: get device raw path:/dev/awinic_haptic
AacRichTapVibrator: device_path:/dev/awinic_haptic
AacRichTapDispEqualiza: hi = 105, freq_coef = 6.379338, jnd_coef = 8.713339
android.hardware.vibrator.service.oplus: AAC init success: 5
```

The `config_data_len` warning is not fatal: `g_is_config_read_success:1`.

## 3. Kernel probe and firmware

`boot3/dmesg (4).txt`:

```text
aw8697 driver version v1.3.6
aw8697_i2c_probe enter
aw8697_read_chipid aw8697 detected
aw8697_haptic_f0_calibration get f0=1703
aw8697_ram_update:crisp haptic bin name  aw8697_haptic.bin
aw8697_ram_loaded: check sum pass : 0xe010
aw8697_ram_loaded: fw update complete
aw8697_rtp_update enter
...
aw8697_rtp_loaded: failed to read aw8697_rtp.bin
```

`aw8697_rtp_1.bin` exists in the vendor tree; the driver asks for
`aw8697_rtp.bin`.

## 4. Bug 1: negative effect IDs

`boot2/30_10-21-05-46_030.log`:

```text
1793378055.766  1000  1838  1838 D AacRichTapConvert: effect_id:4101,strength:89
1793378055.766  1000  1838  1838 E AacRichTapConfig: effect_id is invalid ,effect_id:5,size:3
1793378055.766  1000  1838  1838 D AacRichTapConvert: data_len = 0,interval_time_us:41.666668,level:1,duration_time:1
```

```text
1793378059.363  1000  1838  1838 D AacRichTapConvert: effect_id:2,strength:89
1793378059.363  1000  1838  1838 E AacRichTapConfig: effect_id is invalid ,effect_id:-4094,size:3
1793378059.363  1000  1838  1838 D AacRichTapConvert: data_len = 0,interval_time_us:41.666668,level:1,duration_time:1
```

```text
1793378061.819  1000  1838  1838 D AacRichTapConvert: effect_id:0,strength:89
1793378061.819  1000  1838  1838 E AacRichTapConfig: effect_id is invalid ,effect_id:-4096,size:3
1793378061.819  1000  1838  1838 D AacRichTapConvert: data_len = 0,interval_time_us:41.666668,level:1,duration_time:1
```

## 5. Library disassembly

File: `libaacvibrator.so`, ELF64 aarch64, stripped.

Exported wrapper:

```text
0000000000009604 <_Z32aac_vibra_looper_prebaked_effectji@@Base>
```

It checks the looper state, then forwards `(effect, strength)` to the internal
converter. The converter logs `AacRichTapConvert: effect_id:%d,strength:%d`.

The internal ID conversion is at VA `0x14dfc`:

```asm
0000000000014dd8:
    lsr     w9, w1, #12
    cmp     w9, #0x2
    b.ls    14dec
    mov     w0, #0x1
    ret
14dec:
    ...
    sub     w1, w1, #0x1, lsl #12    ; w1 -= 0x1000
```

So a caller must pass `0x1000 + effect`. Passing `0` underflows to `-4096`;
passing `2` underflows to `-4094`.

The built HAL after the fix contains:

```asm
3be8: 32140020  orr w0, w1, #0x1000
```

## 6. Bug 1 fixed at conversion layer

`boot3/30_10-22-34-13_550.log`:

```text
1793378059.363  1000  1846  1846 D AacRichTapConvert: effect_id:4098,strength:89
1793378059.363  1000  1846  1846 D AacRichTapConvert: data_len = 704,interval_time_us:41.666668,level:1,duration_time:30
```

```text
1793378061.819  1000  1846  1846 D AacRichTapConvert: effect_id:4096,strength:89
1793378061.819  1000  1846  1846 D AacRichTapConvert: data_len = 264,interval_time_us:41.666668,level:1,duration_time:12
```

`4096 = 0x1000 + CLICK`, `4098 = 0x1000 + TICK`.

At 24 kHz, 264 bytes is about 11 ms; 704 bytes is about 29 ms. The reported
durations 12 ms and 30 ms are consistent.

## 7. Bug 2: HEAVY_CLICK still invalid

Same `boot3` live log:

```text
1793378055.766  1000  1846  1846 D AacRichTapConvert: effect_id:4101,strength:89
1793378055.766  1000  1846  1846 E AacRichTapConfig: effect_id is invalid ,effect_id:5,size:3
1793378055.766  1000  1846  1846 D AacRichTapConvert: data_len = 0,interval_time_us:41.666668,level:1,duration_time:1
```

Nearby UI context:

```text
NavHandleLongPressHandler: Contextual Search invocation
```

Android's long-press haptic is `HapticFeedbackConstants.LONG_PRESS`, mapped by
`HapticFeedbackVibrationProvider` to `VibrationEffect.EFFECT_HEAVY_CLICK`.

## 8. AIDL effect IDs

From `hardware/interfaces/vibrator/aidl/android/hardware/vibrator/Effect.aidl`:

```text
CLICK        = 0
DOUBLE_CLICK = 1
TICK         = 2
THUD         = 3
POP          = 4
HEAVY_CLICK  = 5
TEXTURE_TICK = 21
```

The HAL range check:

```cpp
if (effect < Effect::CLICK || effect > Effect::HEAVY_CLICK)
    return EX_UNSUPPORTED_OPERATION;
```

This rejects `TEXTURE_TICK` before the AAC library.

## 9. Why `on()` works

`on()` uses `aac_vibra_looper_on(timeoutMs)`, not the prebaked effect path.
The prebaked path uses `aac_vibra_looper_prebaked_effect()` and the AAC config.

The kernel has two different playback paths:

- `RICHTAP_STREAM_MODE` (`0x520a`) -> worker + mmap + FIFO AEI
- `RICHTAP_RTP_MODE` (`0x5209`) -> direct copy + start + one data write

The stream path is the one exercised by `on()`. The prebaked path uses the RTP
path. A working `on()` therefore does not prove the RTP path works.

## 10. Kernel RTP path evidence

`aw8697.c`, `RICHTAP_RTP_MODE` case:

```c
aw8697_haptic_set_bst_vol(aw8697, 0x11);
aw8697_set_clock(aw8697, AW8697_HAPTIC_CLOCK_CALI_OSC_STANDARD);
aw8697_haptic_play_mode(aw8697, AW8697_HAPTIC_RTP_MODE);
aw8697_haptic_start(aw8697);
usleep_range(2000, 2500);
aw8697_i2c_writes(aw8697, AW8697_REG_RTP_DATA,
                  &aw8697->rtp_ptr[4], tmp);
```

`aw8697_i2c_writes()` returns an `int`, but the RTP case ignores it. If the I2C
transfer fails, userspace still receives ioctl success.

The stream worker, by contrast, enables the FIFO almost-empty interrupt and
feeds data repeatedly:

```c
aw8697_haptic_play_mode(aw8697, AW8697_HAPTIC_RTP_MODE);
aw8697_haptic_set_rtp_aei(aw8697, true);
aw8697_interrupt_clear(aw8697);
aw8697_haptic_start(aw8697);
```

## 11. Settings captured from the device

```text
haptic_feedback_enabled  = 1
haptic_feedback_intensity = 2
keyboard_vibration_enabled = 1
vibrate_on = null
```

`vibrate_on=null` means the setting is unset; the framework default is 1.

## 12. AVC denials

`boot3/dmesg.avc (2).txt`:

```text
avc: denied { setattr } for comm="init" name="trigger" ... tcontext=u:object_r:sysfs_haptic_aw8697:s0
avc: denied { setattr } for comm="init" name="activate" ...
avc: denied { setattr } for comm: "init" name="brightness" ...
avc: denied { setattr } for comm="init" name="duration" ...
avc: denied { setattr } for comm="init" name="state" ...
```

These are `init` writes, not `hal_vibrator_default`. The HAL is allowed by
`hal_vibrator_default.te`.

## 13. Artifact hashes

See [ARTIFACTS.md](ARTIFACTS.md).

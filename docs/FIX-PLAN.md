# Fix plan for Alioth RichTap haptics

Use this after reading [POSTMORTEM.md](POSTMORTEM.md). Do the cheapest test
that separates the remaining failures before changing the kernel.

## 0. First, identify the actual failing action

| User action | Android effect | Current status |
|---|---|---|
| Keyboard tap | CLICK (0) | conversion fixed; physical output unconfirmed |
| Scroll tick | TICK (2) | conversion fixed; physical output unconfirmed |
| Long press | HEAVY_CLICK (5) | **confirmed broken** by config size 3 |
| Back / gesture start | CLICK/TICK | conversion fixed; physical output unconfirmed |
| Text handle / clock tick | TEXTURE_TICK (21) | rejected by HAL |

If the complaint is specifically long-press, go to **Option A**.
If CLICK and TICK are also physically silent, do **Step 1** first.

## Step 1 — separate "no data" from "data not played"

Run one isolated prebaked click and capture kernel output:

```bash
su -c 'dmesg -c > /dev/null; cmd vibrator_manager synced prebaked 0; sleep 1; dmesg | tail -120'
```

Expected on success:

```text
aw8697_file_unlocked_ioctl: cmd=0x5209
```

The driver logs the ioctl command at `dev_info` level. Look for:

```text
i2c master send error
aw8697_i2c_writes: ...
```

If `i2c master send error` appears, the RTP payload write is failing. That is a
kernel/driver issue, not the HAL conversion.

Also capture logcat for the same test:

```bash
su -c 'logcat -b all -c; cmd vibrator_manager synced prebaked 0; sleep 1; logcat -d -v monotonic | grep -iE "AacRichTap|AAC |aw8697|effect_id|data_len"'
```

Interpretation:

- `data_len = 0` -> config/effect mapping still wrong
- `data_len > 0` and no physical output -> playback path
- `i2c master send error` -> RTP write failure
- no `cmd=0x5209` in dmesg -> the library used a different path or failed before ioctl

## Option A — quickest fix for long-press and unsupported effects

The imported config only resolves indices 0, 1, 2. Map unsupported Android
effects to an effect that exists.

Candidate mapping:

```text
HEAVY_CLICK -> TICK      (or CLICK if TICK is too soft)
THUD        -> CLICK
POP         -> TICK
TEXTURE_TICK-> TICK
```

Where:

```text
hardware/xiaomi/vibrator/richtap/Vibrator.cpp
```

Current call:

```cpp
int32_t ret = aac_vibra_looper_prebaked_effect(
        0x1000 + static_cast<uint32_t>(effect), strength);
```

Sketch:

```cpp
uint32_t richtap_effect = static_cast<uint32_t>(effect);

switch (effect) {
    case Effect::THUD:
        richtap_effect = static_cast<uint32_t>(Effect::CLICK);
        break;
    case Effect::POP:
    case Effect::HEAVY_CLICK:
        richtap_effect = static_cast<uint32_t>(Effect::TICK);
        break;
    case Effect::TEXTURE_TICK:
        richtap_effect = static_cast<uint32_t>(Effect::TICK);
        break;
    default:
        break;
}

int32_t ret = aac_vibra_looper_prebaked_effect(0x1000 + richtap_effect, strength);
```

Also update the range check and supported-effects list so they match reality:

```cpp
if (effect < Effect::CLICK || effect > Effect::HEAVY_CLICK) {
    // keep, but add explicit TEXTURE_TICK handling if desired
}
```

```cpp
*_aidl_return = {Effect::CLICK, Effect::DOUBLE_CLICK, Effect::TICK,
                 Effect::THUD, Effect::POP, Effect::HEAVY_CLICK};
```

If mapping is used, advertising THUD/POP/HEAVY_CLICK is honest only because
they now fall back. If no mapping is added, remove them from
`getSupportedEffects()` so the framework does not expect them.

Trade-offs:

- Pros: one small HAL change, no config reverse engineering.
- Cons: not stock feel; long-press will feel like a tick/click.

## Option B — use a config that actually has six prebaked effects

The current config is:

```text
client:Oneplus
device type:oneplus8
vibrator type:ELA0815
vib_id:0
prebaked table size:3
```

It is not an Alioth config. Options:

1. Find an OPlus/Alioth RichTap config with six standard effects and matching
   `libaacvibrator.so`.
2. Determine whether the config's second table (`effect_num:11`) is usable for
   `vib_id` 0 or a different actuator ID.
3. Use the old Alioth stock effect source instead of the OnePlus config.

Hard requirements:

- `libaacvibrator.so` and `aac_richtap.config` must be from the same OOS build.
- The config version must match the library (`config version:4` in this pair).
- The config SHA/version is checked by the library.
- Do not mix a new config with the old library without checking init logs.

Verification after replacing:

```bash
su -c 'logcat -b all -c; cmd vibrator_manager synced prebaked 5; sleep 1; logcat -d | grep -iE "AacRichTap|effect_id|data_len"'
```

A working HEAVY_CLICK must show:

```text
effect_id:4101
data_len > 0
duration_time > 1
```

No `effect_id is invalid`.

## Option C — fix the kernel RTP path if CLICK/TICK are still silent

Only do this after Step 1 shows `data_len > 0` and no physical output.

Relevant code in:

```text
kernel/xiaomi/sm8250/drivers/misc/aw8697_haptic/aw8697.c
```

### C1. Make RTP write failures visible

In the `RICHTAP_RTP_MODE` case, check the `aw8697_i2c_writes()` return:

```c
ret = aw8697_i2c_writes(aw8697, AW8697_REG_RTP_DATA,
                        &aw8697->rtp_ptr[4], tmp);
if (ret < 0)
    dev_err(aw8697->dev, "RTP data write failed: %d\n", ret);
```

This alone may explain why userspace sees success while nothing plays.

### C2. Compare with the working stream path

`on()` uses `RICHTAP_STREAM_MODE`:

```c
haptic_clean_buf(aw8697, MMAP_BUF_DATA_INVALID);
aw8697_haptic_stop(aw8697);
aw8697->done_flag = false;
aw8697->haptic_rtp_mode = true;
aw8697_haptic_set_bst_vol(aw8697, 0x11);
aw8697_set_clock(aw8697, AW8697_HAPTIC_CLOCK_CALI_OSC_STANDARD);
schedule_work(&aw8697->haptic_rtp_work);
```

The worker calls:

```c
aw8697_haptic_play_mode(aw8697, AW8697_HAPTIC_RTP_MODE);
aw8697_haptic_set_rtp_aei(aw8697, true);
aw8697_interrupt_clear(aw8697);
aw8697_haptic_start(aw8697);
```

The direct RTP case does not enable the FIFO AEI and does not use the worker.
Test whether adding `aw8697_haptic_set_rtp_aei(aw8697, true)` before start, or
writing the payload before `aw8697_haptic_start()`, changes output.

Do one change at a time and capture dmesg.

### C3. Fix the missing RTP preload

`aw8697_rtp_update()` asks for:

```text
aw8697_rtp.bin
```

The vendor tree ships:

```text
aw8697_rtp_1.bin
```

Options:

- add `aw8697_rtp.bin` as a copy/symlink of `aw8697_rtp_1.bin`
- change the driver's name table to `aw8697_rtp_1.bin`
- verify the intended file number/index

This is a real defect even if it turns out not to be the `perform()` cause.

### C4. Confirm the ioctl is reached

The driver logs every ioctl at `dev_info`:

```c
dev_info(aw8697->dev, "%s: cmd=0x%x, arg=0x%lx\n", __func__, cmd, arg);
```

Look for:

```text
cmd=0x5209   # RICHTAP_RTP_MODE
```

If it is absent, the library failed before the ioctl or used another path.

## Option D — stop using Richtap for standard effects

The old Alioth custom HAL already had the better Android feature coverage:

```text
hardware/xiaomi/vibrator/aw8697/
```

It supports:

- CLICK, DOUBLE_CLICK, TICK, THUD, POP, HEAVY_CLICK, TEXTURE_TICK
- composition primitives
- stock Alioth RAM effects

The Richtap path loses composition and primitives. If the goal is UI haptics,
not OOS AAC fidelity, reverting to the old HAL is likely the lowest-risk fix.

A hybrid is possible: keep the old HAL for standard effects and use AAC only
for an explicit RichTap path later.

## Build and flash after a HAL change

```bash
cd /home/braindead/luna
source build/envsetup.sh
lunch lineage_alioth-bp4a-user
m android.hardware.vibrator.service.oplus-richtap -j"$(nproc --all)"
m vendorimage -j"$(nproc --all)"
```

Flash the resulting vendor image:

```bash
fastboot flash vendor out/target/product/alioth/vendor.img
fastboot reboot
```

This does not touch the kernel/boot image.

## Verification matrix

Run from Termux root or `adb shell`:

```bash
cmd vibrator_manager list
cmd vibrator_manager synced oneshot 1000
cmd vibrator_manager synced prebaked 0
cmd vibrator_manager synced prebaked 1
cmd vibrator_manager synced prebaked 2
cmd vibrator_manager synced prebaked 3
cmd vibrator_manager synced prebaked 4
cmd vibrator_manager synced prebaked 5
cmd vibrator_manager synced prebaked 21
cmd vibrator_manager feedback 3
```

Expected mapping:

```text
prebaked 0  CLICK
prebaked 1  DOUBLE_CLICK
prebaked 2  TICK
prebaked 3  THUD
prebaked 4  POP
prebaked 5  HEAVY_CLICK
prebaked 21 TEXTURE_TICK
feedback 3  KEYBOARD_TAP
```

For each, capture:

```bash
su -c 'logcat -b all -c; cmd vibrator_manager synced prebaked 0; sleep 1; dmesg | tail -60; logcat -d | grep -iE "AacRichTap|effect_id|data_len"'
```

## Rollback

The vendor image from before the experiment is the safest rollback. The kernel
and boot image are unchanged by the HAL/vendor work. Keep a known-good
`vendor.img` hash before flashing.

Do not ship both AW8697 drivers or both vibrator HALs at the same time.

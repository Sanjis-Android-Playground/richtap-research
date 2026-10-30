# Commands

## Termux / device diagnostics

Get root first:

```bash
su
```

### Correct vibrator service

The Android service is `vibrator_manager`, not `vibrator`.

```bash
cmd vibrator_manager list
cmd vibrator_manager synced oneshot 1000
```

Effect test syntax uses `synced`:

```bash
cmd vibrator_manager synced prebaked 0
cmd vibrator_manager synced prebaked 1
cmd vibrator_manager synced prebaked 2
cmd vibrator_manager synced prebaked 3
cmd vibrator_manager synced prebaked 4
cmd vibrator_manager synced prebaked 5
cmd vibrator_manager synced prebaked 21
```

Haptic feedback constants:

```bash
cmd vibrator_manager feedback 3    # KEYBOARD_TAP
cmd vibrator_manager feedback 1    # VIRTUAL_KEY
cmd vibrator_manager feedback 4    # CLOCK_TICK
```

`cmd vibrator_manager vibrate 1000` returns `Unknown command: vibrate` because
the shell command is `synced`/`combined`, not `vibrate`.

### Settings

```bash
settings get system haptic_feedback_enabled
settings get system haptic_feedback_intensity
settings get system keyboard_vibration_enabled
settings get system vibrate_on
```

Normal captured values:

```text
1
2
1
null
```

`vibrate_on=null` is not an error; the framework defaults it to 1.

### Capture logcat while triggering

```bash
su -c '
  logcat -b all -c
  logcat -v monotonic -b all > /data/local/tmp/haptic.log &
  LP=$!
  sleep 1

  cmd vibrator_manager synced oneshot 1000
  sleep 1
  cmd vibrator_manager synced prebaked 0
  sleep 1
  cmd vibrator_manager synced prebaked 2
  sleep 1
  cmd vibrator_manager synced prebaked 5
  sleep 1

  kill $LP 2>/dev/null
  wait $LP 2>/dev/null
  grep -iE "AacRichTap|AAC |effect_id|data_len|vibrat" /data/local/tmp/haptic.log
'
```

### Capture runtime dmesg for one click

This is the important test for the open playback issue:

```bash
su -c '
  dmesg -c > /dev/null
  cmd vibrator_manager synced prebaked 0
  sleep 1
  dmesg | tail -120
'
```

Look for:

```text
aw8697_file_unlocked_ioctl: cmd=0x5209
i2c master send error
```

`cmd=0x5209` is `RICHTAP_RTP_MODE`.

### Full device dump

```bash
su -c 'dumpsys vibrator_manager > /data/local/tmp/vibrator-dump.txt'
grep -iE "supported|effect|intensity|vibrateOn|keyboard|usage" /data/local/tmp/vibrator-dump.txt
cp /data/local/tmp/vibrator-dump.txt /sdcard/vibrator-dump.txt
```

## Build the HAL and vendor image

From `/home/braindead/luna`:

```bash
source build/envsetup.sh
lunch lineage_alioth-bp4a-user
m android.hardware.vibrator.service.oplus-richtap -j"$(nproc --all)"
m vendorimage -j"$(nproc --all)"
```

Output:

```text
out/target/product/alioth/vendor/bin/hw/android.hardware.vibrator.service.oplus-richtap
out/target/product/alioth/vendor.img
```

## Flash

```bash
fastboot flash vendor out/target/product/alioth/vendor.img
fastboot reboot
```

The vendor image contains the HAL binary. It does not touch the boot/kernel
image.

## Rebuild the full ROM

The user's normal full-build script is:

```bash
bash k1.sh
```

That produces the full zip, but a full ROM build is not needed for a
HAL-only/vendor change.

## Inspect the built HAL

```bash
aarch64-linux-gnu-objdump -d \
  out/target/product/alioth/vendor/bin/hw/android.hardware.vibrator.service.oplus-richtap \
  | grep -n "orr.*#0x1000"
```

Expected:

```asm
3be8: 32140020  orr w0, w1, #0x1000
```

## Inspect the vendor image

```bash
debugfs -R "dump /bin/hw/android.hardware.vibrator.service.oplus-richtap /tmp/hal-from-image" \
  out/target/product/alioth/vendor.img
sha256sum /tmp/hal-from-image \
  out/target/product/alioth/vendor/bin/hw/android.hardware.vibrator.service.oplus-richtap
```

The two hashes must match.

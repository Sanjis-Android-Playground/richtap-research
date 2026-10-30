# Alioth AW8697 haptics

**Stock-backed Android HAL, checkpoint 2026-10-09.** Generated primitives remain
retired. Seven predefined effects and three byte-exact stock primitives are
implemented, with a later LOW_TICK light-tick compatibility fallback. Earlier strength correction received subjective acceptance on
2026-09-30; latest automated tests do not establish perceptual stock parity. [V-HAPTICS](../state/validation.md#v-haptics) distinguishes
the installed baseline from the temporary accepted correction.

## Ownership and interfaces

Current heads: [source map](../state/repositories.yaml). Code:
`hardware/xiaomi:cnb/vibrator/aw8697/`, Alioth's stock driver under
`kernel/xiaomi/sm8250:aosp-17/drivers/input/misc/aw8697_haptic/`, and device/common
packaging, DAC and SELinux. Hardware stays standalone; final playback needs no root.

Module `android.hardware.vibrator-service.xiaomi_aw8697`; init
`vendor.aw8697.vibrator`; Binder `android.hardware.vibrator.IVibrator/default`.
Find input by `EVIOCGNAME=aw8697_haptic`, never a fixed event number. Observed
sysfs base: `/sys/devices/platform/soc/a8c000.i2c/i2c-2/2-005a/`.

- Linux FF uploads/erases effects with `EVIOCSFF`/`EVIOCRMFF`; `EV_FF` starts/stops.
- `custom_wave` reads readiness/status and accepts PCM FIFO writes (ABI 2).
  Optional `ram_gain_abi=1` negotiates tagged RAM strength.
  Service waits for firmware before registration because Android caches capabilities.
- Tagged RAM gain: fourth short `0x4700 | gain` (0..128), eight-byte custom payload;
  legacy ABI is six bytes. Negotiate support; do not send tagged gain to arbitrary RTP IDs.
- `gain` updates timed `on()` amplitude. Predefined effects own their gain separately.
- Read `f0_value` (tenths of Hz) for cached resonance. **Reading `f0` triggers calibration.**

## Effects and stock strength

| Android effect | Zero-based RAM ID | Scheduled duration |
| --- | ---: | --- |
| CLICK | 0 | 20 ms |
| DOUBLE_CLICK | 0 twice | 20 + 80 gap + 20 = 120 ms |
| TICK | 2 | 20 ms |
| THUD | 3 | 20 ms |
| POP | 4 | 28 ms |
| HEAVY_CLICK | 8 | 20 ms; MIUI mesh-heavy |
| TEXTURE_TICK | 5 | 20 ms; MIUI mesh-light |

Hardware RAM slots are one based. Filename numbers are not Android enum IDs.
In particular, stock mesh-heavy uses 8; the light mesh waveform uses 5.

Stock `sys.haptic.infinitelevel=true` activates `VibrateUtils` →
`VibratorExt::configStrengthForEffect` → `InputFFDevice::playEffect`.
Light/medium/strong factors **0.375 / 0.625 / 1.0** yield gains **48 / 80 / 128**.
The old native fallback 30/64/128 was not the active stock UI path. The final
HAL uses the active values; `TEXTURE_TICK` has no extra half-gain multiplier.
At medium it matches `sys.haptic.mesh.light=5,1`.

Android gesture-threshold-deactivate and toggle-off fall back to TEXTURE_TICK
when the requested primitive is absent. The earlier RAM-2 half-gain choice compounded weak
feedback. Long press uses HEAVY_CLICK; gesture activate/toggle-on use TICK.
Trace the actual `HapticFeedbackVibrationProvider` for future Android versions.

Timed amplitude, long-request splitting, cancellation and exactly-once terminal
callbacks are supported; callbacks occur outside the worker lock. Completion
alone does not measure physical strength. Capabilities are `0xA7` when gain and
cached resonance are available: on/perform callbacks, composition, amplitude
control and resonant frequency. **No PWLE, frequency control, FOAM, external
control, always-on or vendor effects.**

Composition supports NOOP plus CLICK (enum 1, 16 ms), THUD (2, 11 ms) and
LIGHT_TICK (7, Android TICK, 13 ms). Their files are
`/vendor/etc/vibrator/primitive_effect_{1,2,7}.bin`: exact stock RAM slices of
381/252/304 samples. LOW_TICK additionally reuses LIGHT_TICK as a compatibility fallback (see below).
Other primitives remain unsupported; do not alias every primitive to CLICK. Discovery uses the exact lookup
from `libqtivibratoreffect.xiaomi`.

## Kernel and firmware

Alioth's [stock driver](https://github.com/MiCode/Xiaomi_Kernel_OpenSource/tree/alioth-r-oss/drivers/input/misc/aw8697_haptic)
and [stock DTS](https://github.com/MiCode/kernel_devicetree/tree/alioth-r-oss)
replace the former dagu/psyche mixture. Live drive/overdrive 73/73 and boost
`0x11` match stock; compensation arithmetic and legacy gain floor are unchanged.
The tagged gain path preserves low timed amplitudes and fades. Do not raise a
global minimum to compensate for a wrong effect mapping or Android intensity setting.

| Shipped `/vendor/firmware/` file | Size | Actual consumer |
| --- | ---: | --- |
| `aw8697_haptic.bin` | 3622 bytes | `aw8697_ram_update`: ten effect slots plus slot 11 constant loop |
| `aw8697_rtp_1.bin` | 120000 bytes | Boot `aw8697_rtp_update` preload and explicit oscillator-calibration sample |

Both match stock; hashes and device evidence are in the
[portable verification record](../evidence/haptics/2026-09-30.md). RTP preload
is not proof oscillator calibration ran. Ordinary Android effects use RAM.
Numbered MIUI/game/ringtone banks and `vibrator_firmware_aidl` duplicates are
not shipped; extended MIUI IDs have no production caller. Stock `libaachaptics.so` is shipped
for the isolated rendering integration, not called by the production HAL.
The large firmware-catalog commit was removed; source README documents actual dependencies.

RAM bank: checksum `0xe010` equals sum(bytes[2:]) modulo 65536; base `0x0800`;
four-byte inclusive descriptors start at offset 5, data at 49. Eleven sample
lengths: 381,443,304,252,681,243,437,169,217,304,142; waves 2 and 9 match.
These digital samples do not provide measured actuator acceleration.

## PCM streaming and AAC boundary

PCM is signed 8-bit, 24 kHz, nonempty and bounded to 240000 samples. Stock RAM
samples replayed through RTP use oscillator trim rather than RAM F0 trim;
byte identity is verified, physical equivalence is not a calibration result.
The HAL uploads custom effect 193, checks FIFO free space and writes at most
4095 bytes. Intermediate writes contain whole periods; only the final write
signals EOF, with a zero pad when needed. Period bounds are 2..2048; observed
period 512, ring capacity 8192. Waiting releases the worker lock for cancellation.
Deadline is nominal duration plus 2000 ms; errors terminate playback/callbacks.
Kernel playback status covers queued work and hardware state. Cancellation may
leave inactive buffered bytes; the next upload resets them.

Stock AAC 1.0.7_20210722 exports `aac_haptics_init`, `aac_haptics_process` and
`aac_haptics_get_frame`; configuration is {512,24000,1,170}. Its detached worker
and blocking frame API have no exported shutdown. Never load it into the HAL
worker as an unbounded call. Android 17 vendor ELF dependencies resolve without
binary patching. The modern upstream `aac_vibra_*` ABI is a different library.

`xiaomi-alioth-richtap-render` is an explicit development target, not a product
package/HAL dependency. It renders native 17-int event records in an isolated
process with a five-second alarm, EOF validation and exclusive output creation.
`--he1` accepts format 3, single actuator, at most 16 events and four curve points
per continuous event; padding, bounds and ordering are checked. It rejects
HE2 and unsupported multi-actuator/16-point layouts. A valid render is capped
at ten seconds and never silently truncated. Usage:

```text
xiaomi-alioth-richtap-render LIBRARY EVENTS.bin OUTPUT.bin [--he1]
```

**Full app-facing RichTap is not implemented.** Android framework/Java routing,
Binder extension, loops and live parameter updates are absent. The production
HAL plays its packaged PCM; AAC rendering still runs separately. Do not claim
full SDK compatibility or expose an unsupported RichTap version. Upstream
reference: [RichTapCoreForAndroidT](https://github.com/richtap-haptics/RichTapCoreForAndroidT)
at the historical candidate.

## Validation and continuation

[October 4 evidence](../evidence/haptics/2026-10-04.md) records the temporary
candidate, restoration, effect/gain matrix, PCM cancellation and renderer tests.
The final two-axis review found no actionable standards defect; the remaining
spec gap is full app-facing RichTap. `dev/alioth-richtap-compat` was fast-forwarded
into `cnb`, then deleted locally/remotely. Latest source heads belong only in
the source map. Do not launch another full build or restore generated effects
from these notes. Identify installed artifacts before any requested regression
check; root is temporary research, normal playback remains enforcing/system UID.
Measured acceleration, exact HyperOS parity and a new human acceptance of the
October 4 candidate remain unestablished.

## October 8 LOW_TICK compatibility

Hardware the historical candidate maps LOW_TICK to stock LIGHT_TICK streaming or RAM selector 2,
preserving scale/delay, duration, callbacks and cancellation; exact files take
priority. It is an approximation, not a verified low-frequency waveform.
Commit-reported target build and temporary enforcing completion tests passed;
reported candidate active until reboot at that time, current device state unknown.
See [latest audit limits](../state/validation.md#v-source-20261009).

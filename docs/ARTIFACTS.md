# Artifacts and hashes

The proprietary OPlus files are **not** committed. These hashes identify the
local files used during the investigation.

## Device source commits at park time

| Repository | Branch | Commit |
|---|---|---|
| `kernel/xiaomi/sm8250` | `main-bpf` | `b8d9584fd570a7c9a638a32aaa92eaee2b8aab76` — input: aw8697: Add the OPLUS AAC RichTap haptic driver |
| `hardware/xiaomi` | `aosp-16` | `c5ea1d7b9f2271184e186ea8f005c1f58f0a8263` — vibrator: Add the OPlus Richtap (AAC) HAL |
| `device/xiaomi/alioth` | `aosp-16` | `b0d8fc3ff0c5815c73a368bfe104994806eb2d46` — alioth: Track the RichTap userspace and configuration |
| `device/xiaomi/sm8250-common` | `aosp-16` | `4489d397df56452a74d04df6078e9b4e17157840` — sm8250-common: Switch AW8697 devices to the RichTap vibrator HAL |
| `vendor/xiaomi/alioth` | `aosp-16` | `7d89f487a0b3296be1baa129ac8028f2403d5bce` — alioth: Add the AAC RichTap library and configuration |

`hardware/xiaomi/vibrator/richtap/Vibrator.cpp` has a local uncommitted
`0x1000` fix on top of `c5ea1d7`.

## Proprietary userspace

| File | Size | SHA-256 |
|---|---:|---|
| `richtap-research/libaacvibrator.so` | 105504 | `41e35389996e71e3a66b49c2831ea18d3bb479752b250f979c03bde0dfea3248` |
| `richtap-research/aac_richtap.config` | 81940 | `5e879534f8799a2d51bca18edced33775e494300e730196dde133a0239d28e08` |
| `richtap-research/libaachaptics.so` | 56344 | `9d70e1d5337f3494ea359eb7532112a0af48fc563c352d5ff816ef7d8f0dd54a` |

Alioth vendor copies:

| File | Size | SHA-256 |
|---|---:|---|
| `vendor/xiaomi/alioth/proprietary/odm/etc/aac_richtap.config` | 81940 | `5e879534f8799a2d51bca18edced33775e494300e730196dde133a0239d28e08` |
| `vendor/xiaomi/alioth/proprietary/vendor/firmware/aw8697_haptic.bin` | 3622 | `7e31b22b591d5f45dcf262529b71fd2b5b3277f98553414b54d3b69d7255d4c4` |
| `vendor/xiaomi/alioth/proprietary/vendor/firmware/aw8697_rtp_1.bin` | 120000 | `9a413b131ecf0ccfb02a837ddb33766d8604cc00da7937b79bd363b53c8d7d86` |

Note: `richtap-research/aw8697_haptic.bin` is 5804 bytes and is a different
file from the Alioth 3622-byte firmware:

```text
9bde39eb8d2ca1688efb046d735f86c2a2c8315a7feee28befb3667d0c9209b7  aw8697_haptic.bin (research copy)
```

Do not mix the two without checking the RAM checksum.

## Built output after the `0x1000` fix

| Artifact | Size | SHA-256 |
|---|---:|---|
| `out/target/product/alioth/vendor/bin/hw/android.hardware.vibrator.service.oplus-richtap` | 27712 | `561daf54dbf1bb574bead5c13b1bc657b5a1dca203e18eddcfbbf14822d99783` |
| `out/target/product/alioth/vendor.img` | 1360031744 | `b3e8af93442322208d06505b066a88923f55df389716cd5ec57199ea542beb81` |

The HAL inside `vendor.img` was verified to match the standalone binary by
dumping it with `debugfs`.

## Logs

The full logs are local only:

```text
boot2/30_10-21-05-46_030.log     # before 0x1000 fix
boot3/30_10-22-34-13_550.log     # after 0x1000 fix
boot3/logcat (13).txt            # boot/init
boot3/dmesg (4).txt              # boot dmesg, ends before the haptic test
boot3/dmesg.avc (2).txt          # init sysfs denials
```

Sanitized excerpts are in [EVIDENCE.md](EVIDENCE.md).

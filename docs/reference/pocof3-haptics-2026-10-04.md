# Alioth haptics verification — 2026-10-04

## Artifact and deployment boundaries

Hardware source the historical candidate was compiled with Android 17 vendor flags in isolated
`out/haptics-validation/`; final HE1 support and validation notes are on `cnb`.
No full ROM build, installclean or permanent deployment was performed. Kernel
reported 4.19.325-cip136-st20-perf-g321ce2cbf8fb. Candidate ran as system UID,
`hal_vibrator_default`, SELinux Enforcing. Executable temporary tmpfs avoided a
nosuid data-bind transition failure; policy was not relaxed.

Original installed HAL restored, SHA256:
`87e7f75fdfaed08fd9e0133840e96ae0bbf69e46114cd35cb0f09314911a603c`.
Stock CLICK primitive file restored, SHA256:
`dcb60279196fcd747381e41c162ad0a745bc0c685e80b7d04d801486f0d1d248`.
Next build must include matching HAL/kernel/device/vendor commits. A source
head is not an installed-artifact manifest.

## Acceptance matrix

| Check | Result and limit |
| --- | --- |
| Seven predefined effects × three strengths | 21/21 gain-register readbacks: 48/80/128 |
| Framework touch intensity 1/2/3 | Checked; prior value 2 restored |
| CLICK/THUD/TICK composition | Scales 0.25/0.5/1 and delays completed |
| Timed amplitude | 25/80/128/255 sequence completed |
| Background long request cancellation | Stopped; final playback_active=0, free_size=8192 |
| Rendered 1200 ms PCM | 28801 samples; temporary exact-file override played/cancelled and restored |
| FIFO model | 60 period/length cases through 240000 samples; bounded writes and final-only EOF |
| Physical maximum-duration/perceptual parity | Not established; timing includes framework/ADB overhead |

Legacy native review client could not locate the current service; it is not
counted as a passing callback/input validation rerun. Framework commands were
used instead. No new human acceptance or broad audio/thermal retest occurred.

## AAC and packet checks

Stock library SHA256:
`9d70e1d5337f3494ea359eb7532112a0af48fc563c352d5ff816ef7d8f0dd54a`.
Dependencies liblog/libc++/libc/libm/libdl resolve through the Android 17 vendor
ELF check. No patch required. Native and supported HE1 60 ms fixtures rendered
identical 1441-byte PCM, SHA256:
`fb1ca160dbbd41cf6905bbd46a1dd9f7b66bb8216b43ab4a37dff46aab592dbe`.
Repeated renders were deterministic. Short transient/continuous fixtures had
no hard-rail clipping. Host sanitizer checks rejected truncated packets,
invalid bounds/order, fifth curve points and nonzero padding.

## Final review

Two-axis Review workflow: standards found no actionable defect; spec retained
one material gap: app-facing RichTap routing, loops and live updates are absent.
Production Android effects/stock composition are separate from the developer
AAC renderer. No full RichTap, PWLE or measured actuator calibration claim.
See [implementation](../../memory/haptics.md) for the authoritative behavior.

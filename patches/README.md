# Patches

## `0001-richtap-use-richtap-effect-id-namespace.patch`

Apply on top of `hardware/xiaomi` commit `c5ea1d7`:

```bash
cd hardware/xiaomi
git apply /path/to/0001-richtap-use-richtap-effect-id-namespace.patch
```

Effect:

```diff
-aac_vibra_looper_prebaked_effect(static_cast<uint32_t>(effect), strength);
+aac_vibra_looper_prebaked_effect(0x1000 + static_cast<uint32_t>(effect), strength);
```

Status: built and verified at the conversion layer.

- CLICK (`0x1000`) now resolves to `data_len=264`, `duration_time=12`.
- TICK (`0x1002`) now resolves to `data_len=704`, `duration_time=30`.
- HEAVY_CLICK (`0x1005`) still fails because the imported OnePlus config only
  has a 3-entry prebaked table.

This patch does **not** fix the remaining long-press/config or RTP-playback
issues. See `docs/FIX-PLAN.md`.

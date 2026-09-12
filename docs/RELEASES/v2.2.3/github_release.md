# Prospector Scanner v2.2.3

**🛡️ Scanner stability release — freeze root causes fixed + self-recovery**

---

## Why update

If your scanner has ever been found frozen after hours of use, this release
is for you. A deep review found and fixed several independent root causes,
and the scanner now recovers itself (with a logged cause) if anything ever
hangs or crashes again.

## What's Fixed (scanner)

- **Stack corruption at 10Hz** — a struct defined in two files drifted apart
  in v2.2.0; the display thread's stack was overwritten ~8 bytes per update
- **LVGL renderer stall** — the layer-change pulse used a scale transform
  needing a 4–7KB contiguous allocation per frame; pool fragmentation made
  it fail and the renderer spun forever. Now an in-place opacity pulse, and
  the LVGL pool grew 32KB → 48KB
- **Permanent "Scanning..."** — after all keyboards timed out, a returning
  keyboard in a different slot was never re-selected
- **Cross-thread races** — display-side readers now lock; pointer access to
  the keyboard table replaced with copy-under-lock snapshots
- **Auto-brightness** now starts at boot when the saved setting is "auto"

## New: Crash recovery + watchdog

Fatal errors and hung threads now **reboot the scanner with a recorded
cause** (`Recovered from crash: ...` in the boot log) instead of freezing on
the last frame. Software-only watchdog (30s) — UF2 firmware updates are
unaffected. Opt out: `CONFIG_PROSPECTOR_SCANNER_TASK_WATCHDOG=n`.

## New (keyboard-side, optional): AUX central

Split keyboards whose central is neither half (e.g. trackball unit):

```conf
CONFIG_ZMK_STATUS_ADV_CENTRAL_SIDE="AUX"
CONFIG_PROSPECTOR_EXPECTED_PERIPHERAL_COUNT=2
```

Halves map to the L/R battery slots; the central's battery shows as Aux1.

## Scope

- **Scanner firmware**: updated — **reflash recommended for everyone**
- **Keyboard module**: optional (no fixes needed over v2.2.2; update for the
  AUX option)
- **BLE protocol**: unchanged — all v2.2.x keyboards ↔ scanners interop

## 📦 Downloads

- `prospector_scanner-xiao_ble_nrf52840_zmk-zmk.uf2` — Non-touch mode
- `prospector_scanner_touch-xiao_ble_nrf52840_zmk-zmk.uf2` — Touch mode
- `settings_reset-xiao_ble_nrf52840_zmk-zmk.uf2` — Settings reset

Full details: [v2.2.3 release notes](https://github.com/t-ogura/zmk-config-prospector/blob/main/docs/RELEASES/v2.2.3/release_notes.md)

---

**🏷️ Version**: v2.2.3
**📅 Released**: September 2026

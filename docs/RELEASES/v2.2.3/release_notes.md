# Prospector Scanner v2.2.3 Release Notes

**Release Date**: September 2026
**Type**: Scanner stability release (+ optional keyboard-side feature)

## Summary

A deep review of the scanner firmware traced the long-standing "display is
frozen after hours of use" reports to several independent root causes — a
stack corruption shipped since v2.2.0, an LVGL rendering stall, a stale
keyboard-selection state, and a set of cross-thread races. All are fixed in
this release, and the scanner now recovers itself (with a logged cause)
should anything ever hang or crash again.

**Scanner firmware update strongly recommended for all users.**
Keyboard-side update is optional (one new feature, no fixes required over
v2.2.2).

## Scanner Fixes

- **Display-thread stack corruption (shipped since v2.2.0)**
  `struct pending_display_data` was defined independently in two source
  files; v2.2.0 added fields to only one of them. The bulk copy handed to
  the display thread was sized by the larger definition and overwrote ~8
  bytes of stack at 10Hz. The struct now lives in a single shared header
  (`include/zmk/scanner_core.h`).

- **LVGL renderer stall on pool fragmentation**
  The layer-change pulse animated a scale *transform*, which forces LVGL
  (with `LV_USE_MATRIX` off) to render the label through a sub-layer whose
  4–7KB contiguous buffer is allocated from the LVGL pool on every frame.
  Once the pool fragmented, that allocation failed and the renderer spun
  forever in `lv_draw_dispatch_wait_for_request()` (`LV_USE_OS=0`) — last
  frame stays on the LCD, BLE keeps running. The pulse is now a text-opacity
  dip drawn in place (no layer allocation possible), and the LVGL pool grew
  from 32KB to 48KB as defence in depth.

- **Permanent "Scanning..." after all keyboards time out**
  When every keyboard timed out (default 8 min) and one returned into a
  different table slot, nothing re-armed the selection if it had pointed at
  another slot — the display stayed on "Scanning..." at the 1% timeout
  brightness while advertisements flowed. An arriving keyboard is now
  adopted whenever the selected slot is inactive.

- **Cross-thread race pass**
  The display runs on a dedicated LVGL thread while keyboard data is
  processed on the system workqueue. Display-side getters now take the data
  mutex (previously unlocked bulk copies could observe torn records), and
  the pointer-returning keyboard accessor was replaced by copy-under-lock
  snapshots (`zmk_status_scanner_copy_keyboard()`), removing a window where
  a name buffer could be read mid-write without a terminator.

- **Misc**: removed a debug heartbeat that called a display-driver API from
  ISR context every 15s; auto-brightness sensor polling now starts at boot
  when the persisted setting is "auto" (previously required toggling the
  switch once per power-up); settings-slider drag state can no longer stick
  and lock out swipe navigation; Operator layout no longer rebuilds its
  battery widgets on every advertisement when a peripheral's battery byte
  flaps; Field layout animation no longer stops after days of continuous
  typing; advertisement ring buffer 16 → 32 entries.

## New: Crash Recovery + Software Watchdog

Previously, any fatal error halted the CPU with the last frame on screen —
indistinguishable from a freeze, with no record of the cause. Now:

- A custom fatal-error handler records the reason, faulting thread, PC/LR
  and uptime in noinit RAM and **reboots**; the next boot logs
  `Recovered from crash: ...` (visible via `zmk-usb-logging`).
- A pure-software task watchdog (no hardware WDT — the nRF52 WDT would keep
  running inside the UF2 bootloader during firmware updates) reboots the
  scanner if the display thread or the processing pipeline stops for 30s,
  recording which one hung.
- A crash-loop guard halts after 5 consecutive early faults so a broken
  build cannot reboot endlessly.
- Opt out with `CONFIG_PROSPECTOR_SCANNER_TASK_WATCHDOG=n`.

## New (keyboard-side, optional): AUX central battery mapping

For split keyboards whose central is neither half — e.g. a trackball or
other auxiliary unit acting as split central:

```conf
CONFIG_ZMK_STATUS_ADV_CENTRAL_SIDE="AUX"
# Both halves are peripherals in this topology:
CONFIG_PROSPECTOR_EXPECTED_PERIPHERAL_COUNT=2
# If connection order doesn't match physical layout:
# CONFIG_ZMK_STATUS_ADV_LEFT_PERIPHERAL=0
# CONFIG_ZMK_STATUS_ADV_RIGHT_PERIPHERAL=1
```

The halves map to the left/right battery slots and the central's own battery
is shown in the Aux1 slot. Purely a keyboard-side mapping — older scanners
display it correctly.

## Internal

The keyboard-tracking core (keyboard table, timeouts, rate calculation,
display handoff) moved from the color-LCD shield into the shared module as
`src/scanner_core.c` + `include/zmk/scanner_core.h`, so future shields (the
upcoming Scanner Pocket, Sharp Memory LCD) reuse it instead of forking it.

## Compatibility

- **Wire protocol unchanged** — every v2.2.x keyboard works with every
  v2.2.x scanner, in both directions.
- **Scanner**: reflash with the v2.2.3 firmware (non-touch or touch).
- **Keyboard**: optional. To pick up the AUX option or keep versions
  aligned:
  ```yaml
  - name: prospector-zmk-module
    remote: prospector
    revision: v2.2.3
    path: modules/prospector-zmk-module
  ```
  then `west update`, rebuild, flash.

## Verified

- Scanner non-touch and touch builds (ZMK main / Zephyr 4.1,
  `xiao_ble/nrf52840`), firmware version stamp v2.2.3
- Keyboard-side on split central (Zephyr 3.5) and uni-body (nice_nano_v2 +
  reviung41) builds
- Multi-day continuous operation on the pre-release build with no freeze

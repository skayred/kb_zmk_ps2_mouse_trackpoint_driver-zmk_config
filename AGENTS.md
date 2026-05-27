# AGENTS.md

Guidance for working in this repository.

## What this is

A ZMK (`zmk-config`) firmware configuration for several split keyboards. Firmware is
built by GitHub Actions from `build.yaml` (there is no local ZMK checkout). Each
keyboard is a ZMK *shield* running on a `nice_nano_v2` controller.

Shields currently defined under `boards/shields/`:
- `charybdis` — split with a PMW3360 **trackball** (on the right half).
- `tp-sofle` — split with a PS/2 **trackpoint**.
- `corne_tp` — Corne with a trackpoint.

## Directory layout — and the rule that matters

### `boards/shields/<name>/` — the hardware definition (defaults)
The "stock" definition of a keyboard and its built-in defaults: matrix transform and
physical layouts, `kscan` GPIOs, per-half `.overlay` files, sensor wiring (trackball /
trackpoint and its `input-listener` processing pipeline), `Kconfig.*` (including which
half is `ZMK_SPLIT_ROLE_CENTRAL`), and a default `.keymap` / `.conf`.

Edit these when you are **correcting or extending the hardware definition itself** —
wiring, matrix, column order, sensor setup, or split role.

### `config/` — your overrides (these win at build time)
This is the ZMK user-config directory (`config/west.yml` has `self.path: config`). Files
here take precedence over the shield defaults:

- **Keymap — full replacement.** ZMK prepends `config/` to the keymap search path, and
  the shield *directory name* is a search prefix, so `config/<name>.keymap` is used
  **instead of** `boards/shields/<name>/<name>.keymap`. The in-tree shield keymap is
  ignored whenever a `config/` one exists.
  ⇒ **For any keymap or layer/behavior change, edit `config/<name>.keymap`**
  (e.g. `config/charybdis.keymap`, *not* `boards/shields/charybdis/charybdis.keymap`).
- **`.conf` — layered override.** The shield's `<shield>.conf` is applied first as
  defaults; then `config/<name>.conf` and `config/<shield>_<half>.conf` are merged on
  top (last value wins). Put half-specific symbols in
  `config/<shield>_left.conf` / `config/<shield>_right.conf` so they are not applied to
  the other half (e.g. a trackball symbol on the half without the sensor would warn).
- **`.overlay` — layered override.** The shield overlay applies, then the first matching
  `config/*.overlay` is layered on top.

### Rule of thumb for where an edit goes
- Wiring / matrix / column order / sensor presence / split role / fixing a shield's own
  definition → `boards/shields/<name>/`.
- Keymap, layers, and personal tuning (pointer sensitivity, rotation, layer timeouts)
  → `config/`.

## Build / flash notes
- `build.yaml` is the GitHub Actions build matrix (board + shield). Comment/uncomment
  `include:` entries to choose what gets built.
- Split roles: exactly one half is central (`ZMK_SPLIT_ROLE_CENTRAL` in the shield's
  `Kconfig.defconfig`). After changing which half is central — or any time the halves
  won't pair — flash the `settings_reset` firmware to **both** halves to clear stale BLE
  bonds, then flash the real left/right firmware.

## Pointer / input-processor notes
- Pointer movement comes from the sensor hardware (trackball / trackpoint), never from
  keymap keys; there are no `&mmv` bindings.
- "Auto-mouse" and "scroll" layers are implemented with input processors on the device's
  `input-listener` (in the shield files): `&zip_temp_layer <layer> <ms>` activates a
  layer on motion, and a `scroller { layers = <N>; input-processors = <&zip_xy_to_scroll_mapper>; }`
  child node turns motion into scrolling while layer `N` is active. These require
  `#include <input/processors.dtsi>` (not auto-included).

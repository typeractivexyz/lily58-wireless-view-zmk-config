<div align="center">

# Lily58 Wireless — ZMK Config

**Typeractive Lily58 · nice!nano v2 · nice!view · Kailh Choc Sunset · custom 3D-printed case**

[![Build ZMK firmware](https://github.com/KrishPatel1404/lily58-zmk-config/actions/workflows/build.yml/badge.svg)](https://github.com/KrishPatel1404/lily58-zmk-config/actions/workflows/build.yml)
![ZMK](https://img.shields.io/badge/ZMK-v0.3-blue)
![Board](https://img.shields.io/badge/board-nice!nano_v2-9cf)
![Wireless](https://img.shields.io/badge/split-BLE-success)

</div>

---

## ⌨️ The Build

| | |
|---|---|
| **Keyboard** | [Typeractive Lily58 wireless](https://typeractive.xyz/products/lily58-partially-assembled-pcb) — 58-key column-staggered split (6×4+4 per half), designed by [kata0510](https://github.com/kata0510/Lily58) |
| **Controllers** | 2× [nice!nano v2](https://nicekeyboards.com/docs/nice-nano/) (nRF52840, BLE 5, UF2 bootloader) |
| **Displays** | 2× [nice!view](https://nicekeyboards.com/docs/nice-view/) — Sharp memory-in-pixel LCD, 160×68, 1-bit, ~1000× less power than OLED. Custom widgets via [zmk-nice-oled](https://github.com/mctechnology17/zmk-nice-oled) |
| **Switches** | 60× Kailh **Choc Sunset** tactile — 40 gf actuation, 55 gf bump, 3.0 mm travel, factory-lubed |
| **Keycaps** | Blank Choc v1, all white — 8× convex 1u (thumbs), 2× homing 1u, 2× 1.5u, rest 1u |
| **Batteries** | 2× 1800 mAh LIP1359 (PS3-controller replacement cells) — months per charge |
| **Case** | Custom CAD-modeled faceplate + bottom shell, 3.25° typing angle, printed in white |
| **Encoder** | 1× **EC12 rotary encoder, no push button** on the **left** half — hand-soldered to the left nice!nano v2's `P0.31` (A) + `P0.29` (B) pads, common to `GND` |

## 🗺️ Keymap

<div align="center">

![Keymap diagram](keymap-drawer/lily58.svg "Lily58 keymap")

</div>

3 layers (Base / Lower / Raise), based on the stock upstream Typeractive layout.

**Homerow mods** on row 3 — tap for the letter, hold for the modifier, using urob's ["timeless"](https://github.com/urob/zmk-config#timeless-homerow-mods) recipe (balanced flavor, 280 ms tapping term, 225 ms quick-tap, 150 ms require-prior-idle, opposite-hand positional triggers):

| A | S | D | F | | J | K | L | ; |
|---|---|---|---|---|---|---|---|---|
| ⌃ | ⇧ | ⌥ | ⌘ | | ⌘ | ⌥ | ⇧ | ⌃ |

How the recipe works and why these numbers: [`docs/timeless-homerow-mods.md`](docs/timeless-homerow-mods.md).

Left encoder (the violet ⟳ key): **page up/down** on Base, **brightness** on Lower, **volume** on Raise.

Edit with [Keymap Editor](https://nickcoutsos.github.io/keymap-editor/) or [`config/lily58.keymap`](config/lily58.keymap).

## 🖥️ Displays

Both nice!views run [zmk-nice-oled](https://github.com/mctechnology17/zmk-nice-oled)'s `nice_epaper` shield instead of ZMK's stock widget, pinned by commit sha in [`config/west.yml`](config/west.yml).

**Left (central):** battery · USB/BT output + profile dots · **live modifier indicators** (⌘ ⌥ ⇧ ⌃ as real macOS glyphs in a 2×2 box, live as you hold them) · current layer name.

**Right (peripheral):** battery + a looping cat animation. Swap it for spaceman, pokemon, head, gem, or a still image in [`config/lily58_right.conf`](config/lily58_right.conf) — the priority order there is load-bearing, read the comments.

> ⚠️ The module draws on a 68 px wide portrait canvas and rotates it onto the panel, but several of its own nice!view X defaults sit past that edge (modifiers 62, CapsLock 100, Luna 100) — see upstream [issue #26](https://github.com/mctechnology17/zmk-nice-oled/issues/26). Every position here is set explicitly. Don't leave one to the module default without checking it against 68.

Deliberately off, each for a verified reason: Luna and Bongo Cat ([issue #30](https://github.com/mctechnology17/zmk-nice-oled/issues/30) — they freeze while typing whenever the modifier row is on), WPM (its gauge collides with the modifier box, and [PR #33](https://github.com/mctechnology17/zmk-nice-oled/pull/33) shows it forces a full-canvas redraw every second while typing), CapsLock (renders an empty string unless an animation is attached), the responsive bongo cat (doesn't link), smart battery, and Raw HID.

Every flag, every trap, and how custom artwork would work: [`docs/zmk-nice-oled.md`](docs/zmk-nice-oled.md).

## 🔨 Building & Flashing

Firmware-relevant pushes build in CI ([ZMK v0.3](https://github.com/zmkfirmware/zmk/releases)); successful main builds auto-publish to [**Releases**](../../releases).

1. Grab `lily58_left.uf2` / `lily58_right.uf2` from the [latest release](../../releases/latest)
2. Plug a half in via USB-C, **double-tap the reset button** → it mounts as a `NICENANO` drive
3. Drag the matching `.uf2` on (left file → left half, right → right)

> Keymap-only changes usually need just the **left** (central) half reflashed. Anything touching split behavior: flash both.

**Halves not pairing?** Flash `settings_reset.uf2` (Actions `firmware` artifact) to both halves, then re-flash normal firmware.

## 🔋 Battery Notes

- Both halves report their own battery on their own screen; the left half also proxies the right half's level to the host, so macOS shows both under Bluetooth.
- **Deep sleep is off** and *both* BLE links are pinned to a 7.5 ms interval with no skipped events (lowest latency ZMK can ask for) — the host link via `CONFIG_BT_PERIPHERAL_PREF_LATENCY` in `lily58.conf`, and the split link between the halves via `CONFIG_ZMK_SPLIT_BLE_PREF_LATENCY` in `lily58_left.conf`. So real runtime is well under the [power profiler](https://zmk.dev/power-profiler)'s ~4 months central / ~1 year peripheral estimate. Raise either back toward 30 to trade latency for runtime.
- Charges via USB-C at 100 mA — full charge takes overnight.

**⚠️ Battery safety:** replacement-pack polarity isn't standardized — multimeter-verify red/+ lands on **B+** before connecting (reversed = dead board, fire risk). nice!nano has no low-voltage cutoff; use packs with a protection circuit.

## 📁 Repo Layout

```
build.yaml                    # build matrix: left/right + nice!view, settings_reset
config/
  lily58.keymap               # homerow mods + layers + encoder sensor-bindings
  lily58.conf                 # Kconfig: BT power + low-latency conn, debounce, display, battery, encoder
  lily58_left.conf            # central-half only: 3 BLE host profiles, split-link latency, central screen widgets
  lily58_right.conf           # peripheral-half only: which animation the right screen runs
  west.yml                    # ZMK pinned to v0.3 + the zmk-nice-oled module pinned by sha
keymap-drawer/
  config.yaml                 # keymap diagram styling (hand-written)
  lily58.svg / lily58.yaml    # auto-generated diagram (CI output — don't hand-edit)
docs/
  zmk-nice-oled.md            # nice!view display module dossier (flags, traps, custom art)
  timeless-homerow-mods.md    # how the homerow-mod recipe works, why these numbers
.github/workflows/
  build.yml                   # CI → zmkfirmware reusable build workflow
  draw-keymaps.yml            # CI → keymap-drawer diagram render
```

## 🔗 Links

[Typeractive build guide](https://docs.typeractive.xyz/build-guides/lily58-wireless) · [firmware guide](https://docs.typeractive.xyz/build-guides/lily58-wireless/firmware) · [ZMK docs](https://zmk.dev/) · [ZMK GitHub](https://github.com/zmkfirmware) · [Lily58 upstream](https://github.com/kata0510/Lily58) · [keymap-drawer](https://github.com/caksoylar/keymap-drawer)

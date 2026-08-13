# zmk-nice-oled research dossier

Everything worth knowing about [`mctechnology17/zmk-nice-oled`](https://github.com/mctechnology17/zmk-nice-oled)
before putting it back on Krish's Lily58: what it is, whether it fits this hardware, how to
customise it, how to draw your own images and animations, and every bug and gotcha found.

> **Status: research only.** Nothing in this repo is configured to use the module right now.
> Researched 2026-08-07 against module HEAD `46f824ab` (2026-01-26) and ZMK `v0.3.0`.
> Facts below were verified against the module's and ZMK's actual source unless marked otherwise.

This is the single reference for the displays; the older, shorter `displays.md` was deleted 2026-08-13 as redundant.

---

## 1. TL;DR

- **It works on this exact keyboard.** We already shipped a green CI build of it: release
  `v2026.07.16-f930103`, shields `lily58_left/right nice_view_adapter nice_epaper`. It was never
  flash-tested, but it compiled and linked for both halves and `settings_reset`.
- **It replaces the stock nice!view widget entirely** with a vertical, information-dense status
  screen: battery, output/profile, layer, WPM (number / speedometer / graph), live modifier
  indicators, CapsLock, plus Luna the dog or Bongo Cat, and an animation or static image on the
  peripheral half.
- **Repositioning anything is pure Kconfig.** Every widget has an X/Y pair. No fork needed.
- **Drawing your own images or animations requires forking the module.** Assets are compile-time
  C arrays wired into `animation.c` + `CMakeLists.txt` + `Kconfig.defconfig`. There is no runtime
  image loading and no "drop a PNG in your config" path. See [section 6](#6-your-own-images-and-animations).
- **The module is ~6.5 months stale and ZMK-v0.3-only.** Last commit 2026-01-26. It will not build
  on ZMK v0.4 (Zephyr 4.1 / LVGL 9.3), the maintainer has not started that port, and five PRs
  have sat unreviewed since July 2026. Fine while we stay pinned to v0.3; it becomes the blocker
  the day we want v0.4.

---

## 2. The hardware it has to drive

Krish's screens are nice!views. Nice Keyboards markets them as "a SSD1306 OLED replacement" and
that is the right way to read it: it is a *replacement for* OLEDs, not an OLED itself.

| Property | Value | Why it matters here |
|---|---|---|
| Panel | Sharp **LS011B7DH03** memory-in-pixel LCD | Not emissive, not backlit. Holds its image with almost no power |
| Resolution | **160×68**, 1.08" diagonal | The module rotates this in software and treats it as **portrait 68×160** |
| Colour | **1 bit** | Pure black/white. No grey, no anti-aliasing, no fades. Dither or nothing |
| Refresh | 30 Hz | Enough for the module's animations; nothing is going to look smooth-smooth |
| Power | **<10 µA typical** | The panel is not your battery problem. The **CPU redrawing it** is |
| Interface | 3-wire SPI, 3.3 V | Comes off `nice_view_spi`, defined by the `nice_view_adapter` shield |

The practical consequence for custom art: design for **1-bit, 68 px wide, pre-dithered, pre-rotated
90°**. The module deliberately never rotates images at runtime because geometric transforms are
expensive on an nRF52840 (stated outright in its README's Limitations section).

---

## 3. Does it fit this keyboard

Yes, and the shield dependency chain checks out end to end. Verified against `zmkfirmware/zmk@v0.3.0`:

```
lily58.zmk.yml            exposes:  [i2c_oled]
  └─ nice_view_adapter    requires: [i2c_oled]      exposes: [nice_view_header]
     │                    defines `nice_view_spi` in boards/nice_nano_v2.overlay
     ├─ nice_view         requires: [nice_view_header]   ← what we use today
     └─ nice_epaper       requires: [nice_view_header]   ← the module's shield
```

`nice_epaper` is a **drop-in swap for `nice_view`** at the same position in the shield list. Its
overlay re-declares the panel node on the adapter's SPI bus:

```dts
&nice_view_spi {
    status = "okay";
    nice_view: ls0xx@0 {
        compatible = "sharp,ls0xx";
        spi-max-frequency = <1000000>;
        reg = <0>;
        width = <160>;
        height = <68>;
    };
};
```

Other compatibility points:

- **The rotary encoder is unaffected.** It sits on `pro_micro 21/20` = P0.31 / P0.29; the display
  is on the SPI header. Different pins, no contention. `lily58_left.overlay` still sets
  `&left_encoder { status = "okay"; }` regardless of which display shield is stacked.
- **Our split battery proxy still applies.** `ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING` and
  `_PROXY` are core ZMK, untouched by the module. The module *additionally* offers
  `NICE_OLED_WIDGET_CENTRAL_SHOW_BATTERY_PERIPHERAL_*` to render the right half's level on the
  left screen, which the stock widget cannot do.
- **Our 3 BLE profiles still apply.** The module's profile widget draws dots, it does not hardcode
  a count.
- **Flash is not the constraint.** Measured from our own release assets (UF2 is exactly 2× the
  binary, 512-byte blocks carrying 256 bytes each):

  | Build | left.uf2 | ≈ binary | right.uf2 | ≈ binary |
  |---|---|---|---|---|
  | With the module (`v2026.07.16-f930103`) | 661,504 B | ~323 KB | 514,560 B | ~251 KB |
  | Current stock (`v2026.07.31-afda909`) | 750,080 B | ~366 KB | 537,600 B | ~262 KB |

  Both are comfortable on the nRF52840's 1 MB. **RAM is the real limit**, not flash.

---

## 4. Installing it (exact changes for this repo)

Four edits. All are firmware-affecting, so all are already covered by `build.yml`'s `paths:`
allowlist (`config/**` and `build.yaml`) — no allowlist change needed.

**1. `config/west.yml`** — add the module remote and project:

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
    - name: mctechnology17
      url-base: https://github.com/mctechnology17
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: v0.3
      import: app/west.yml
    - name: zmk-nice-oled
      remote: mctechnology17
      revision: 46f824abb2bd41f1287c5c68abd14122af6042a3
  self:
    path: config
```

Pin the sha, never `main`. The module has tags (`v0.0.1`, `v0.0.2`) but `v0.0.2` is from Dec 2025
and predates the v0.0.3 fixes that are on `main` unreleased — the sha above **is** current `main`
HEAD, so pinning it costs nothing and protects against a surprise force-push.

**2. `build.yaml`** — swap `nice_view` → `nice_epaper` on both halves:

```yaml
  - board: nice_nano_v2
    shield: lily58_left nice_view_adapter nice_epaper
  - board: nice_nano_v2
    shield: lily58_right nice_view_adapter nice_epaper
```

Order still matters: `nice_view_adapter` must come before `nice_epaper`.

**3. `config/lily58.conf`** — the module needs the custom status screen turned on and told which
panel family it is driving:

```conf
CONFIG_ZMK_DISPLAY=y
CONFIG_ZMK_DISPLAY_STATUS_SCREEN_CUSTOM=y
CONFIG_NICE_EPAPER_ON=y
CONFIG_NICE_OLED_ON=n
```

`NICE_EPAPER_ON` is what selects every ePaper-flavoured default (canvas 68×160, `LV_Z_VDB_SIZE=100`,
`LV_DPI_DEF=161`, all the ePaper widget coordinates). Getting this wrong is the single most common
way people end up with widgets rendered off-screen.

**4. Remember the post-push rule.** This changes firmware for both halves, so both halves must be
reflashed, and the uf2s must land in `firmware/` as `left.uf2` / `right.uf2` / `reset.uf2`.

---

## 5. What you can turn on, and what it costs

### Central screen (left half)

| Feature | Flag | Default |
|---|---|---|
| WPM widget (master switch) | `NICE_OLED_WIDGET_WPM` | `y` |
| WPM as a number | `..._WPM_NUMBER` | `n` |
| WPM speedometer gauge | `..._WPM_SPEEDOMETER` | `n` on ePaper |
| WPM graph | `..._WPM_GRAPH` | `n` on ePaper |
| Luna the dog, driven by WPM | `..._WPM_LUNA` | `y` |
| Bongo Cat, driven by WPM | `..._WPM_BONGO_CAT` | `n` |
| Live modifier indicators (⌃⇧⌥⌘) | `..._MODIFIERS_INDICATORS` | `y` |
| Modifier style: real icons | `..._MODIFIERS_INDICATORS_FIXED_SYMBOL` | `y` |
| Modifier icons: macOS glyphs | `..._MODIFIERS_INDICATORS_FIXED_SYMBOL_MACOS` | `y` |
| CapsLock / HID indicators | `..._HID_INDICATORS` | `y` |
| Layer name | `..._WIDGET_LAYER` | `y` |
| Big profile icons | `..._PROFILE_BIG` | `n` |
| Peripheral battery on central screen | `..._CENTRAL_SHOW_BATTERY_PERIPHERAL_ALL` / `_ONLY` / `_AND_CENTRAL` | `n` (experimental) |
| Invert the whole display | `..._WIDGET_INVERTED` | `n` |

The macOS modifier glyphs pair well with the homerow mods — a live ⌃⇧⌥⌘ readout is arguably the
most genuinely *useful* thing this module adds over stock, given the [HRM setup](../CLAUDE.md).
But see the Luna freeze bug in [section 7](#7-known-bugs-and-gotchas) before combining it with Luna.

### Peripheral screen (right half)

| Feature | Flag | Default |
|---|---|---|
| Animate the peripheral | `..._ANIMATION_PERIPHERAL` | `y` |
| Cat (8 frames) | `..._ANIMATION_PERIPHERAL_CAT` | **`y`** |
| Head (16 frames) | `..._ANIMATION_PERIPHERAL_HEAD` | `n` |
| Spaceman (20 frames) | `..._ANIMATION_PERIPHERAL_SPACEMAN` | `n` |
| Pokemon (48 frames) | `..._ANIMATION_PERIPHERAL_POKEMON` | `n` |
| Gem / crystal (16 frames) | `..._ANIMATION_PERIPHERAL_GEM` | `n` — **and this flag is a no-op, see below** |
| Static image instead (battery saver) | `..._STATIC_IMAGE_PERIPHERAL` | `y` when animation is off |
| Static: vim logo | `..._STATIC_IMAGE_PERIPHERAL_VIM` | `n` |
| Animation frame-cycle duration | `..._ANIMATION_PERIPHERAL_MS` | 960 (cat/gem), 4800 (head/spaceman), 10000 (pokemon) |
| Sleep art on idle / on sleep | `NICE_OLED_SHOW_SLEEP_ART_ON_IDLE` / `_ON_SLEEP` | `n` |

### Raw HID (host-side companion app)

Optional and heavy. Enabling `NICE_OLED_WIDGET_RAW_HID=y` lets a Rust host app
([`zzeneg/qmk-hid-host`](https://github.com/zzeneg/qmk-hid-host), with the module author's macOS
additions) push system time, volume, and keyboard layout to the screen on all three OSes, plus
**weather and Spotify track info on macOS only**. Costs roughly +1 KB RAM / +4 KB flash, more with
its font, and wants `DEDICATED_THREAD_STACK_SIZE=3072`. It also requires running a login-item
daemon. Probably not worth it here unless Krish specifically wants the Spotify readout.

### Memory budget

The module's `OPTIMIZE.md` presets, which are the numbers to reach for if anything misbehaves:

| Preset | `LV_Z_MEM_POOL_SIZE` | thread stack | notes |
|---|---|---|---|
| Minimal | 4096 | 2048 | static peripheral image, no WPM, no HID indicators |
| **Balanced (recommended)** | 8192 | 2048–2560 | WPM + Luna at 300 ms, peripheral animation at 960 ms |
| Full | 12288 | 4096 | everything including Raw HID |

Module defaults already do the sensible split-aware thing: pool `8192` on the central, `4096` on the
peripheral; stack `2560`, or `3072` on a central with Raw HID; thread priority `5`.

Symptoms of running out of RAM, per the README: display artefacts, freezing, BLE instability. That
last one is the scary one — **the display can destabilise Bluetooth**, which is exactly the class of
problem we already fought once with the [connection-latency bug](../CLAUDE.md#learnings-log).
Do not stack WPM graph + Bongo Cat + Raw HID.

---

## 6. Your own images and animations

This is the part with no shortcut, so here is the honest shape of it.

### How assets actually work

Every image in this module is a **compile-time C array**. `assets/vim.c` opens with:

```c
/* Generated using lvgl_img_converter.py (bundle_images_to_c - Default BlackBG/WhiteFG) */
static const uint8_t vim_32x100_map[] = {
#if CONFIG_NICE_OLED_WIDGET_INVERTED
  0x00,0x00,0x00,0xff, 0xff,0xff,0xff,0xff,   /* palette: idx0 black, idx1 white */
#else
  0xff,0xff,0xff,0xff, 0x00,0x00,0x00,0xff,   /* palette: idx0 white, idx1 black */
#endif
  /* 1 bit per pixel packed bitmap ... */
```

Format: LVGL **`LV_IMG_CF_INDEXED_1BIT`**, a 2-entry palette (with the inversion flag baked in as a
preprocessor branch), then packed 1-bpp pixel data. Animations are just N of these plus an
`lv_img_dsc_t *` array handed to `lv_animimg_set_src()`.

Adding one means touching **three** files inside the module:

1. `assets/<name>.c` / `.h` — the generated frames
2. `widgets/animation.c` — `LV_IMG_DECLARE()` each frame, build the `<name>_imgs[]` array, add an
   `#elif` branch calling `lv_animimg_set_src(art, (const void **)<name>_imgs, <count>)`
3. `CMakeLists.txt` — `target_sources_ifdef(CONFIG_..._<NAME> app PRIVATE assets/<name>.c)`
4. `Kconfig.defconfig` — declare the `CONFIG_..._<NAME>` bool and its `_MS` default

**There is no way to do this from our own `config/` directory.** You must fork
`mctechnology17/zmk-nice-oled`, commit the asset there, and point `west.yml` at your fork.

### The easy path: the GIF converter script

Unmerged **PR #28** by @Furglitch adds exactly the tool for this. It is not in `main`, so grab it
from the author's fork:

```bash
# script lives at Furglitch/zmk-niceview @ 58054f7, docs/CONVERT.md + gif-convert.py
pip install Pillow          # plus ImageMagick for preprocessing
python3 gif-convert.py my-animation.gif -n myanim -W 64 -H 64 --animation-ms 4000
```

What it does, in order:

1. ImageMagick preprocess: `-coalesce` → `-rotate 90` → `-colorspace Gray` → optional `-crop`
2. **Atkinson dithering** down to pure 1-bit (this is the step that makes photos survive a
   black-and-white panel at all; threshold is tunable with `-t`)
3. Emits `<name>.c` / `<name>.h` with the LVGL descriptors and the inversion-aware palette
4. **Auto-registers** itself in `animation.c`, `CMakeLists.txt` and `Kconfig.defconfig`, using the
   existing pokemon/spaceman/cat entries as insertion anchors

Useful flags: `-W`/`-H` canvas size (default 64×64), `--crop 48x48+16+16`, `--center`,
`--no-rotate`, `--rotate <deg>`, `--no-register` to generate only.

The script is derived from [`danielsodium/nice-view-gifs`](https://github.com/danielsodium/nice-view-gifs),
which is a standalone ZMK module for the same job if you would rather not fork this one.

### For a single static image

Overkill to run the GIF pipeline. Use the [LVGL image converter](https://docs.lvgl.io/latest/en/html/overview/image.html)
with **CF_INDEXED_1BIT**, at 68 px wide, pre-rotated 90°, then drop it in as
`assets/<name>.c` and wire it into the `STATIC_IMAGE_PERIPHERAL` `#elif` chain the same way `vim`
is. Static images are the cheapest thing you can put on the peripheral: roughly **−4 KB flash and
near-zero CPU** versus an animation, and they let the memory-in-pixel panel do what it is good at,
which is holding a picture for free.

### Repositioning without forking

Everything visible has an X/Y Kconfig pair, so layout changes need **no fork at all**. ePaper
defaults, for reference:

| Widget | X | Y | | Widget | X | Y |
|---|---|---|---|---|---|---|
| Layer | 0 | 146 | | Battery | 26 | 19 |
| Profile | 18 | 129 | | Profile text | 25 | 32 |
| Output USB | 45 | 2 | | Output BT | 49 | 0 |
| WPM gauge | 16 | 43 | | WPM label | 0 | 103 |
| Luna | 100 | 15 | | Bongo Cat | 100 | 8 |
| HID indicators | 100 | 15 (Luna) / 8 (Bongo) | | Modifiers | 2 | 110 |
| Peripheral animation | 36 | 0 | | Sleep status | 0 | 0 |

Canvas is 68 wide × 160 tall. Set them via `CONFIG_NICE_OLED_WIDGET_<THING>_CUSTOM_X` / `_Y`.

⚠️ **The table above is the module README's, and it is wrong in places** — it lists modifiers at
X=2 where `Kconfig.defconfig` actually says 62. Read the Kconfig, not the README, and check every
X against the 68 px limit (see [section 7](#-the-68-pixel-rule-half-the-modules-own-x-defaults-are-off-screen)).

There is also a third shield, **`nice_custom`**, which is the same widget set with the panel
geometry itself moved into Kconfig (`CUSTOM_CANVAS_WIDTH` / `_HEIGHT`) and its overlay left for you
to write. Only worth it for non-nice!view panels; for us `nice_epaper` already has the right
defaults.

### The no-fork escape hatch (unmerged)

**PR #35** adds `CONFIG_NICE_OLED_EXTERNAL_WIDGET`, an 18-line hook that lets a *separate* Zephyr
module supply the peripheral widget:

```c
extern int nice_oled_external_widget_init(lv_obj_t *parent);
```

That would let custom art live in our own repo instead of a fork. Its first consumer,
[`delneet/zmk-peripheral-animations`](https://github.com/delneet/zmk-peripheral-animations),
computes ~10 effects procedurally on the MCU (plasma, rule 30, starfield, matrix rain, …) at
1–2 KB each instead of storing frames — a genuinely clever fit for a 1-bit panel. Unmerged since
2026-07-03 with no maintainer response.

---

## 7. Known bugs and gotchas

Verified against module source at `46f824ab` unless noted.

### 🔴 The 68-pixel rule: half the module's own X defaults are off-screen

The single most important thing on this page. The module draws onto a **portrait canvas
`CONFIG_NICE_OLED_CUSTOM_CANVAS_WIDTH = 68`** px wide, then software-rotates it 90° onto the
160×68 panel (`widgets/util.c: rotate_canvas`, and `screen.c` sizes the object 160×68). **Any
pre-rotation X of 68 or more never reaches the glass.**

Several `nice_epaper` defaults in the module's own `Kconfig.defconfig` break that rule:

| Widget | Module's ePaper default X | Width it draws | Right edge | Visible? |
|---|---|---|---|---|
| Modifiers (2×2 box) | **62** | 30 (2×14 + 2 gap) | 92 | right column gone, left clipped to 6 px |
| Modifiers (horizontal) | **62** | 62 (4×14 + gaps) | 124 | mostly gone |
| HID indicators | **100** | — | — | entirely gone |
| Luna | **100** | — | — | entirely gone |
| `PROFILE_BIG` | 18 | 70 (5×14) | 88 | last two dots clipped |

This is upstream **issue #26**, where a user on a Typeractive Corne with nice!view reports
*"my modifiers are also sent all the way off the screen and I can only see a sliver of it on
the right side"*. Exactly this.

Two further traps in the same area:

- **The README's position tables disagree with `Kconfig.defconfig`.** The README says modifiers
  default to X=2, Y=110 on ePaper; the Kconfig says **62, 62**. The Kconfig wins. Treat the
  README tables as unreliable and read `Kconfig.defconfig` directly.
- **`FIXED_VER` is the only layout with ePaper-aware X.** `screen.c` hardcodes
  `base_x = (68 - img_size) / 2` for the centred vertical layout; `BOX` and `HOR` both use
  `CONFIG_NICE_OLED_WIDGET_MODIFIERS_CUSTOM_X` raw. But `VER` needs 62 px of *height* for its
  four stacked glyphs, which then fights the profile row at Y=129.

**Rule: set every X explicitly and check it against 68.** Y has the full 160 to play with and is
far more forgiving. The maths for a centred 2×2 modifier box is `(68 − 30) / 2 = 19`.

### 🔴 `HID_INDICATORS=y` renders nothing unless an animation is attached

`widgets/hid_indicators.c` defines `HID_HAS_ANIMATION 1` **only** under
`HID_INDICATORS_BONGO_CAT` or `HID_INDICATORS_LUNA`. With both off, the `#else` branch compiles
to an unconditional:

```c
// HID_HAS_ANIMATION = 0: No animation enabled
// Widget remains functional but displays nothing visually
lv_label_set_text(label, "");
```

So `CONFIG_NICE_OLED_WIDGET_HID_INDICATORS=y` on its own buys **zero visible output** while still
forcing `CONFIG_ZMK_HID_INDICATORS=y` and its BLE/HID plumbing and RAM. And attaching an
animation to fix it walks straight into the issue #30 freeze below. On a nice!view with the
modifier row enabled, this widget is not salvageable — leave it `n`.

### 🟠 The WPM label position knob is a no-op on ePaper

`widgets/wpm.c` has two branches. The ePaper one hardcodes `#define DRAW_LABEL_WMP_Y 103` and
draws at a literal `x=0`; only the non-ePaper branch reads
`CONFIG_NICE_OLED_WIDGET_WPM_LABEL_CUSTOM_X/_Y`. So the documented way to move the WPM readout
does nothing on this hardware.

### 🟠 The profile widget does not adapt to the profile count

Unlike ZMK's stock nice!view widget, `widgets/profile.c` draws a **fixed five** slots on both
code paths — a flat `for (int i = 0; i < 5; i++)` under `PROFILE_BIG`, and a single fixed sprite
without it. With `BT_MAX_PAIRED=4` (3 host profiles) the screen still shows 5 markers, 2 of which
can never be selected. Cosmetic, but don't expect the stock behaviour.

### 🔴 Peripheral animation selection is an `#elif` chain, and cat wins

`widgets/animation.c` resolves the animation like this:

```
HEAD > CAT > SPACEMAN > POKEMON > crystal (the "gem")
```

Two consequences that bite everyone:

- **`CAT` defaults to `y`.** Setting `SPACEMAN=y` or `POKEMON=y` without also setting `CAT=n` gives
  you the cat, silently. You must explicitly disable the higher-priority ones.
- **`CONFIG_NICE_OLED_WIDGET_ANIMATION_PERIPHERAL_GEM` does nothing.** There is no `#elif` branch
  for it. The gem is the `#else` fallback you land on when nothing else is enabled. The flag exists
  in Kconfig and is documented in the README, but no code reads it.

### 🔴 `RESPONSIVE_BONGO_CAT=y` will not link (unreported)

Found while reading the build wiring, not present in any issue. `CMakeLists.txt` has the asset file
commented out:

```cmake
# target_sources_ifdef(CONFIG_NICE_OLED_WIDGET_RESPONSIVE_BONGO_CAT app PRIVATE assets/responsive_bongo_cat_images.c)
target_sources_ifdef(CONFIG_NICE_OLED_WIDGET_RESPONSIVE_BONGO_CAT app PRIVATE widgets/responsive_bongo_cat.c)
```

…and the widget's live code path references `bongo_cat_both1_open_90`, `bongo_cat_both1_90`,
`bongo_cat_right2_90`, `bongo_cat_left2_90`, which are **declared in that widget and defined
nowhere in the repository** (the `idle_img*` / `fast_img*` names that *do* exist in
`responsive_bongo_cat_images.c` sit inside a `/* … */` comment block). Enabling this flag is an
undefined-reference link failure. Leave `CONFIG_NICE_OLED_WIDGET_RESPONSIVE_BONGO_CAT=n`.

### 🟠 Smart battery compiles out every animation (issue #31, open)

`CONFIG_NICE_OLED_WIDGET_ANIMATION_PERIPHERAL_SMART_BATTERY=y` wraps the whole of `animation.c` in
`#if !IS_ENABLED(...SMART_BATTERY)`, and `battery.c` only ever compiles the gem. So smart battery
plus any animation choice = gem, always. The maintainer separately warned back in issue #4 that
this feature "consumes a significant amount of memory and CPU… the screen goes blackout and the
CPU stops responding after a while" and advised against using it. **Don't enable it.**

### 🟠 Luna freezes while typing when the fixed modifier row is on (issue #30, open, no reply)

Reported on exactly our setup — Corne with nice!view, `nice_view_adapter nice_epaper`, ZMK v0.3.0,
module `46f824a`. With `MODIFIERS_INDICATORS_FIXED=y` (horizontal symbols) plus `WPM_LUNA=y`, Luna
stops animating while typing and only recovers when you stop. Turning the modifier row off fixes
it; turning the WPM graph off does not. The reporter's diagnosis: the fixed-modifier listener fires
on `keycode_state_changed` and calls `draw_canvas()` → `rotate_canvas()`, starving the `animimg` on
a 1-bpp panel.

This matters because the modifier row is the feature most worth having here. **Pick one: live
modifier indicators, or Luna. Not both.**

### 🟠 Full-canvas redraw on every WPM tick (PR #33, open)

With the shipped defaults (`WPM_NUMBER`, `WPM_SPEEDOMETER`, `WPM_GRAPH` all `n`, only Luna or Bongo
Cat on) `draw_wpm_status()` draws nothing, but `screen.c` still subscribes the whole status screen
to `zmk_wpm_state_changed`. Result: about once a second while typing, the entire canvas is redrawn
and software-rotated for no visual change, competing with key scanning and BLE on the central.
Same root cause family as issue #30. PR #33 fixes it; unmerged.

### 🟡 Animation-speed Kconfigs are ignored (PR #34, open)

`WPM_LUNA_ANIMATION_MS` and `WPM_BONGO_CAT_ANIMATION_MS` exist, are documented as *the* CPU tuning
knob in `OPTIMIZE.md`, and **neither widget reads them** — `luna.c` and `bongo_cat.c` use hardcoded
200 ms. So the documented way to calm these animations down currently does nothing. Unmerged.

### 🟡 "Luna stuck barking" (issues #20, #26, both closed)

Long-running complaint, mostly resolved by the December 2025 rework plus explicitly disabling what
you don't want. The residue worth knowing: after v0.0.3, HID indicators and modifier indicators show
**no animation by default** — you must opt in with `..._LUNA=y` or `..._BONGO_CAT=y`. Anyone
following an older guide gets confusing results. Also note issue #20's other lesson: pin the
**workflow** `@ref` to v0.3 too, not just `west.yml` — which we already do in both places.

### 🟡 Shield-level config surprises

`nice_epaper.conf` is merged automatically and forces two things worth being aware of:

- `CONFIG_ZMK_EXT_POWER=y`, with the comment *"de alguna manera rara esto no se activa bien"*
  ("somehow this doesn't activate properly"), which is not confidence-inspiring but is harmless on
  a nice!nano v2.
- `CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE` defaults to **`y`** for ePaper (it's `n` for OLED). Screens
  blank on idle. Since [deep sleep is off](../CLAUDE.md) on this board, this is the main display
  power saver. Override to `n` if Krish wants the screens always on.

### 🟡 Repo health

- **Last commit 2026-01-26**, ~6.5 months ago as of 2026-08-07.
- **Five open PRs**, four of them substantive fixes from July 2026, none reviewed.
- Issue #22 (Zephyr 4.1 / LVGL 9.3 breaks the module) has been open since 2025-12-12. Maintainer on
  2026-01-06: "I haven't started updating to 4.1 yet." A user pinged again 2026-07-06 with no reply.
- The `v0.3/dev` branch is **3 commits behind `main`** — already merged, ignore it.
- 485 stars, 157 forks, MIT licensed. Historically the maintainer was very responsive; the current
  quiet stretch is the longest in the project's life.

**Bottom line on health:** perfectly safe pinned to a sha on ZMK v0.3. It becomes a hard blocker the
day ZMK v0.4 ships and we want it, and there's no evidence the port is underway.

---

## 8. Alternatives, if this one doesn't fit

| Project | What it gives you | Trade-off |
|---|---|---|
| [`M165437/nice-view-gem`](https://github.com/M165437/nice-view-gem) | Polished fixed design, animated gem | No customisation; its `main` needs Zephyr 4.1, so use the v0.3.0 release |
| [`GPeye/nice-view-mod`](https://github.com/GPeye/nice-view-mod) | The stock ZMK nice!view shield repackaged as a module, for easy forking | You draw everything yourself |
| [`danielsodium/nice-view-gifs`](https://github.com/danielsodium/nice-view-gifs) | Purpose-built GIF → nice!view module with a converter script | GIF only, no status widgets |
| [`qwerty22121998/nice-view-anim`](https://github.com/qwerty22121998/nice-view-anim) | Another animation-focused shield | Less widely used |
| Stock `nice_view` (what we run now) | Battery, output/profile, layer, WPM. Zero risk | No modifier indicators, no custom art |

If the goal is *only* "my own image or animation on the right half", `nice-view-gifs` or
`nice-view-mod` is a smaller blast radius than forking zmk-nice-oled. If the goal is the **live
modifier readout** plus a richer central screen, zmk-nice-oled is the one that has it.

---

## 9. Sources

- [mctechnology17/zmk-nice-oled](https://github.com/mctechnology17/zmk-nice-oled) — README, `Kconfig.defconfig`, `CMakeLists.txt`, `widgets/animation.c`, `assets/*.c`, `docs/OPTIMIZE.md`, `docs/NICE_CUSTOM_EXAMPLE.md`, `CHANGELOG.txt`, issues #1/#4/#5/#9/#11/#20/#22/#26/#30/#31, PRs #28/#32/#33/#34/#35
- [zmkfirmware/zmk @ v0.3.0](https://github.com/zmkfirmware/zmk) — `lily58.zmk.yml`, `nice_view_adapter/*`, `nice_view/*`, `lily58_left.overlay`
- [nice!view documentation](https://nicekeyboards.com/docs/nice-view/) — panel specs
- [ZMK: Zephyr 4.1 Update](https://zmk.dev/blog/2025/12/09/zephyr-4-1) — LVGL 9.3 breaking changes, v0.4 plans
- [Furglitch/zmk-niceview @ 58054f7](https://github.com/Furglitch/zmk-niceview) — `docs/CONVERT.md`, `gif-convert.py` (PR #28 source)
- [danielsodium/nice-view-gifs](https://github.com/danielsodium/nice-view-gifs), [zzeneg/qmk-hid-host](https://github.com/zzeneg/qmk-hid-host), [delneet/zmk-peripheral-animations](https://github.com/delneet/zmk-peripheral-animations)
- This repo's own history: release `v2026.07.16-f930103` and commit `f930103` (`config/west.yml`, `build.yaml`, `config/lily58.conf` as they were when the module was live)

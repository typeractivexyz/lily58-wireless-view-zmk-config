# Timeless homerow mods

Research notes on urob's "timeless" homerow-mod (HRM) recipe, what each setting actually
does, how a second config (linkarzu's Toucan) implemented it and tuned it in practice, and
what it would take to run it on this Lily58.

Researched 2026-08-13. Nothing here is implemented - this is a findings doc.

**Primary sources**

| Source | What it is | Checked |
|---|---|---|
| [urob/zmk-config#timeless-homerow-mods](https://github.com/urob/zmk-config#timeless-homerow-mods) | The original write-up | readme.md @ `main`, 2026-08-11 |
| [urob/zmk-config `config/base.keymap`](https://github.com/urob/zmk-config/blob/main/config/base.keymap) | His live config (readme can lag) | lines 44-53 |
| [linkarzu/zmk-keyboard-toucan](https://github.com/linkarzu/zmk-keyboard-toucan) | A 42-key config using the recipe | `config/toucan.keymap` @ `main`, 2026-07-07 |
| [ZMK hold-tap docs](https://zmk.dev/docs/keymaps/behaviors/hold-tap) | Official property semantics | 2026-08-13 |
| ZMK `v0.3.0` source | What our pinned firmware actually does | `app/src/behaviors/behavior_hold_tap.c` |
| [sunaku, bilateral combinations](https://sunaku.github.io/home-row-mods.html) | The positional-HRM idea, pre-ZMK | background |

---

## 1. The problem being solved

A homerow mod is a hold-tap: tap `A` and you get `a`, hold `A` and you get Ctrl. The naive
version is decided purely by a stopwatch:

- held **longer** than `tapping-term-ms` → mod
- held **shorter** → letter

That needs your typing rhythm to be consistent, and it never is. Two failure modes:

- **False positive** - you lingered on `f` mid-word and got Cmd instead of `f`.
- **False negative** - you meant Cmd+S, released a fraction early, got `fs`.

And even when it works there is a **delay**: the firmware cannot emit the letter until it
knows the key was a tap, so every homerow letter appears at release, not at press.

urob's insight is that you should not be solving this with a timer at all. Set the tapping
term so long that it effectively never fires, then add **short-circuits** that decide the
key correctly and instantly in each situation you actually encounter.

## 2. The four ingredients

All four exist in ZMK `v0.3.0` (verified in `app/dts/bindings/behaviors/zmk,behavior-hold-tap.yaml`),
so nothing here needs a version bump.

| Property | Default | What it does | Role in the recipe |
|---|---|---|---|
| `flavor = "balanced"` | `hold-preferred` | Resolves as **hold** if another key is pressed *and released* while the hold-tap is down | The main short-circuit. Cmd+S resolves the instant you release `s` - no waiting |
| `tapping-term-ms` | 200 | The stopwatch | Deliberately set **long** (280) so it almost never decides anything |
| `require-prior-idle-ms` | disabled | If the hold-tap is pressed within X ms of another key, force **tap** immediately | Kills the typing delay. Mid-word, homerow keys are plain letters with zero latency |
| `hold-trigger-key-positions` | `[]` | Only the listed positions may trigger the hold. Anything else → **tap** | "Bilateral combinations": a mod only counts if the next key is on the *other* hand |
| `hold-trigger-on-release` | off | Delay the positional check from the next key's *press* to its *release* | Lets you stack two mods on the same hand |
| `quick-tap-ms` | disabled | Re-pressing the **same** hold-tap key within X ms of tapping it forces a tap | Lets you type `ff`, `ll` without the second press becoming a mod |

### How they chain together

Pressing an HRM key, in order:

1. **Was another key pressed less than `require-prior-idle-ms` ago?** → it's a letter. Done,
   emitted immediately. This is the mid-typing case, i.e. most keystrokes.
2. **Is this the same key I just tapped, within `quick-tap-ms`?** → letter. Handles doubles.
3. Otherwise ZMK waits. Then:
   - Another key is pressed *and released* while you hold → `balanced` says **hold**...
   - ...but only if that key is in `hold-trigger-key-positions`. If it's on the same hand,
     it's a roll, not a chord → **tap**.
   - Nothing happens for 280 ms → **hold** (this is how you press a bare Cmd).

The result: cross-hand chords fire instantly, same-hand rolls never misfire, and normal
typing has no delay. The tapping term only matters in two edge cases (bare modifier press,
and same-hand shortcuts).

### The two rough edges urob calls out

**Nested rolls.** Typing fast you sometimes get `key1↓ key2↓ key2↑ key1↑`. That is exactly
the `balanced` hold pattern, so it would falsely fire a mod. `hold-trigger-key-positions`
fixes it: if key2 is on the same hand, force a tap.

**Same-hand mod stacking.** By default the positional check runs when the next key is
*pressed*, which blocks holding two mods on one hand. `hold-trigger-on-release` defers the
check to *release*, so held keys can combine while tapped keys still resolve as taps.

## 3. urob's exact configuration

Verified against his live `config/base.keymap` (not just the readme) - the values match:

```c
// config/base.keymap lines 46-53, with helper macros expanded
flavor = "balanced";
tapping-term-ms = <280>;
quick-tap-ms = <175>;
require-prior-idle-ms = <150>;
bindings = <&kp>, <&kp>;
hold-trigger-key-positions = <KEYS_R THUMBS>;   // left-hand HRMs: opposite hand + thumbs
hold-trigger-on-release;
```

Two behaviors, `hml` and `hmr`, identical except that each lists the *opposite* hand plus
all thumbs as its trigger positions.

**Where 150 comes from.** urob gives a rule of thumb: `require-prior-idle-ms` ≥ `10500 / WPM`.
The derivation is `(60 × 1000) / (5.7 × WPM)`, using an average English word length of 4.7
characters plus one space. At 70 WPM that is 150 ms. Slower typist → larger value.

**His caveat, and it is the important one:** this setup works best with a **dedicated shift
key** (he uses sticky-shift on a thumb). Capitalising a letter is the one case where you
press a mod *immediately after* another key, which is precisely what `require-prior-idle-ms`
forces to a tap. So homerow shift produces false negatives for fast typists, by design.

## 4. linkarzu's Toucan, and what he learned tuning it

His `config/toucan.keymap` copies the recipe (with a comment pointing straight at urob's
section) but drifts on two numbers:

| | urob | Toucan macOS | Toucan Windows |
|---|---|---|---|
| flavor | balanced | balanced | balanced |
| `tapping-term-ms` | 280 | **300** | **250** |
| `quick-tap-ms` | 175 | **300** | **250** |
| `require-prior-idle-ms` | 150 | 150 | 150 |
| `hold-trigger-on-release` | yes | yes | yes |
| `retro-tap` | no | no | **yes** |

He also keeps two behaviors per hand per OS (`hrm_l`/`hrm_r`, `hrm_w_l`/`hrm_w_r`) because
the positional lists must stay hand-specific - the same reason we would need two.

The `quick-tap-ms = 300` is a real deviation, not a typo: it equals his tapping term, so any
re-press of the same key within 300 ms of tapping it is forced to a letter. Very forgiving
for doubled letters, slightly worse if you tap a mod and immediately want to hold it.

**His mod order** is `GUI SHIFT ALT CTRL` on `A S D F` (pinky→index), mirrored. urob uses
`GUI ALT SHIFT CTRL`. Both put GUI on the pinky and Ctrl on the index.

### The tuning history is the useful part

From `git log` on his keymap:

| Date | Change | Reading |
|---|---|---|
| 2026-06-01 | `require-prior-idle-ms` 125 → **75** ("for shift in homerowmods") | Hit exactly the shift problem urob warns about, tried to fix it by shortening the idle window |
| 2026-06-05 | 75 → 125, **and added a physical `&kp LSHFT` thumb key** ("reddit was right, I'll think I'll use a thumb key for shift") | The tuning fix didn't work; the structural fix did |
| 2026-06-25 | Swapped Cmd and Ctrl across both base layers | Mod placement is the thing people churn on, not the timings |
| 2026-07-03 | 125 → **150** | Landed back on urob's stock value |

So an independent user walked the tuning ladder in both directions and ended up at urob's
numbers plus a dedicated thumb shift. That is a strong signal: **don't tune first, add a
thumb shift first.**

## 5. What ZMK v0.3.0 actually does (source-verified)

Read from `app/src/behaviors/behavior_hold_tap.c` at the tag we're pinned to, because these
details are not in the docs and they change how you tune.

- **`require-prior-idle-ms` is global, not per-key.** There is one static `last_tapped`
  struct (line 130) shared by every hold-tap on the board. Any key resets the window for all
  HRMs.
- **Modifiers do not reset the window.** Line 792: `if (ev->state && !is_mod(...))`. So
  holding Shift and then pressing an HRM does *not* count as "prior activity" - the HRM can
  still resolve as a hold. Layer keys don't count either (the code has a `// we want to catch
  layer-up events too... how?` comment at line 789).
- **`quick-tap-ms` only applies to the same physical key.** Line 148 requires
  `last_tapped.position == hold_tap->position`, and that position is only recorded when a
  hold-tap resolves as a tap (`store_last_hold_tapped`). A plain `&kp` press sets it to
  `INT32_MIN`, so it can never match. Quick-tap is strictly a "tap the same HRM twice"
  feature.
- **Prior-idle is checked before quick-tap**, in the same function `is_quick_tap()` (lines
  144-151). Both funnel into the same "force a tap" decision.
- `hold-while-undecided` and `hold-while-undecided-linger` **exist in v0.3.0**. Undocumented
  in most HRM write-ups but potentially useful: `hold-while-undecided` applies the modifier
  the instant you press the key and drops it if the key resolves as a tap. The ZMK docs
  recommend it for **using modifiers with a mouse** (Cmd-click, drag-select), which the
  standard recipe handles badly. Caveat from the docs: if the key position is in any combo,
  activation waits for all combo timeouts.

## 6. What this looks like on the Lily58

> **Implemented 2026-08-13.** Live values: balanced, tapping-term 280, **quick-tap 225**
> (Krish's pick, between urob's 175 and Toucan's 300), require-prior-idle 150,
> `hold-trigger-on-release`. Mod order is **⌃⇧⌥⌘ on `A S D F`, mirrored** — the reverse of
> both reference configs, putting ⌘ on the index finger, which suits macOS where ⌘ is the
> most-used modifier. The `&mt` thumbs were left as they were.


### Key positions

Enumerated from `config/lily58.keymap` (58 bindings, index order):

```
KEYS_L  0  1  2  3  4  5    12 13 14 15 16 17    24 25 26 27 28 29    36 37 38 39 40 41
KEYS_R  6  7  8  9 10 11    18 19 20 21 22 23    30 31 32 33 34 35    43 44 45 46 47 48 49
THUMBS 50 51 52 53 54 55 56 57
```

Notes:
- **Position 42 is deliberately excluded.** That is the left inner-bottom dead slot where the
  rotary encoder physically sits - no switch is fitted, so it can never be pressed. It holds
  a marker keycode only so keymap-drawer draws the knob.
- Position 43 (`'`) is on the right half, so it belongs to `KEYS_R`.
- Homerow keys would be `A`=25 `S`=26 `D`=27 `F`=28 and `J`=31 `K`=32 `L`=33 `;`=34.

### The behaviors block

```c
#define KEYS_L  0  1  2  3  4  5 12 13 14 15 16 17 24 25 26 27 28 29 36 37 38 39 40 41
#define KEYS_R  6  7  8  9 10 11 18 19 20 21 22 23 30 31 32 33 34 35 43 44 45 46 47 48 49
#define THUMBS 50 51 52 53 54 55 56 57

/ {
    behaviors {
        hml: homerow_mod_left {
            compatible = "zmk,behavior-hold-tap";
            #binding-cells = <2>;
            bindings = <&kp>, <&kp>;
            flavor = "balanced";
            tapping-term-ms = <280>;
            quick-tap-ms = <175>;
            require-prior-idle-ms = <150>;
            hold-trigger-key-positions = <KEYS_R THUMBS>;
            hold-trigger-on-release;
        };

        hmr: homerow_mod_right {
            compatible = "zmk,behavior-hold-tap";
            #binding-cells = <2>;
            bindings = <&kp>, <&kp>;
            flavor = "balanced";
            tapping-term-ms = <280>;
            quick-tap-ms = <175>;
            require-prior-idle-ms = <150>;
            hold-trigger-key-positions = <KEYS_L THUMBS>;
            hold-trigger-on-release;
        };
    };
};
```

Base layer row 3 then becomes:

```
&kp ESCAPE  &hml LCTRL A  &hml LSHFT S  &hml LALT D  &hml LGUI F  &kp G   ...
        ...  &kp H  &hmr RGUI J  &hmr RALT K  &hmr RSHFT L  &hmr RCTRL SEMI  &kp BACKSPACE
```

That is `CTRL SHIFT ALT GUI` pinky→index, mirrored, which is **the reverse of both reference
configs** - urob (`GUI ALT SHIFT CTRL`) and Toucan (`GUI SHIFT ALT CTRL`) both fix GUI on the
pinky and CTRL on the index, and linkarzu got there deliberately via a 2026-06-25 commit that
swapped Cmd and Ctrl. The case for reversing it on macOS: ⌘ is by far the most-used modifier
there, and the index finger is the strongest, so ⌘ belongs on `F`/`J` rather than the pinky.

### Conflicts specific to this keyboard

1. **The thumbs already hold mod-taps.** Positions 50/51 are `&mt LALT UP` and `&mt LGUI DOWN`.
   `&mt` defaults to `flavor = "hold-preferred"`, `tapping-term-ms = 200` (verified in
   `app/dts/behaviors/mod_tap.dtsi` @ v0.3.0) - i.e. the exact naive behaviour this whole
   recipe exists to avoid, and it fires a hold on *any* key press. Alt and Gui would also
   then exist in two places. Decide: keep the thumbs and drop those two mods from the
   homerow, or convert the thumbs to the same balanced treatment.
2. **Both thumbs are in `THUMBS`**, so they are valid hold-triggers. A thumb mod-tap
   resolving as a hold while an HRM is undecided is the messiest interaction here and is
   worth testing first.
3. **Shift already has a dedicated key**: `&kp RSHFT` at position 49 and `&kp LCTRL` at 36.
   That satisfies urob's "keep a dedicated shift" requirement, though it's a row-4 pinky
   rather than a thumb. Both reference configs moved shift to a thumb eventually.
4. **No combos in this keymap**, so the combo/HRM interaction problems (ZMK
   [#544](https://github.com/zmkfirmware/zmk/issues/544), the `hold-while-undecided` combo
   caveat) do not apply here.
5. **BLE latency config is unaffected.** HRM delay is resolved in firmware before anything is
   sent over the air; it cannot reintroduce the chunked-typing problem, which lived in the
   connection parameters. Confirmed by reasoning about the layers, and by the fact that the
   previous HRM build on this board was hardware-tested without the stutter returning.
6. **The nice!view modifier widget** already repaints on every keystroke regardless, so HRMs
   add no new display load.

### Why the previous attempt on this board may have felt annoying

The HRM setup removed on 2026-08-13 was *not* stock urob. It used six behaviors with
per-key `hold-trigger-key-positions` that permitted same-hand stacking only for ⌘⇧ and ⌥⇧.
Every other same-hand mod pair resolved as letters by design. Neither reference config does
this - both list the whole opposite hand plus thumbs, uniformly, and let
`hold-trigger-on-release` handle stacking generically. If HRMs are retried, the stock
uniform version is the thing to try first.

## 7. Tuning ladder

Straight from urob's troubleshooting section, in the order to try things:

| Symptom | Fix |
|---|---|
| Noticeable delay when tapping homerow keys | **Increase** `require-prior-idle-ms` (≥ `10500 / your WPM`) |
| False negatives, same hand (mod ignored) | **Reduce** `tapping-term-ms`, or drop `hold-trigger-key-positions` |
| False negatives, cross hand | **Reduce** `require-prior-idle-ms`, or switch flavor to `hold-preferred` (needs [a patch](https://github.com/celejewski/zmk/commit/d7a8482712d87963e59b74238667346221199293) to keep `hold-trigger-on-release`) |
| False positives, same hand (mod fires while typing) | **Increase** `tapping-term-ms` |
| False positives, cross hand | **Increase** `require-prior-idle-ms`, or switch flavor to `tap-preferred` |
| Shift specifically is unreliable | Don't tune. Use a dedicated shift key (Toucan's lesson) |

## 8. Appendix: the "faster GitHub Actions workflow" bonus

urob's readme §"Bonus: A (moderately) faster Github Actions Workflow" is a **Nix-based
drop-in replacement** for ZMK's standard build workflow, described by him as "mainly a
proof-of-concept" that "runs moderately faster, especially with a cold cache". It depends on
his whole `flake.nix` local build environment.

Not a small edit for this repo: our `.github/workflows/build.yml` delegates to the upstream
reusable workflow `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3`, and
adopting his approach means replacing that delegation with a self-hosted Nix build. Our
builds are ~3 minutes and cached; the payoff is small. Noted, not recommended.

## 9. Open questions before implementing

1. Which mod order - `CTRL SHIFT ALT GUI` (both reference configs, Cmd on index) or the
   `A=Ctrl … F=Cmd` order used previously here?
2. What happens to the `&mt LALT UP` / `&mt LGUI DOWN` thumbs?
3. All eight homerow keys, or six (skip the pinkies)? sunaku rates index fingers best and
   pinkies worst for HRMs.
4. Is `hold-while-undecided` worth enabling for Cmd-click and drag-select on macOS?

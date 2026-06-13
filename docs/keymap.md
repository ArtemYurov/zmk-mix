[← Hardware](hardware.md) · [Back to README](../README.md) · [Configuration →](configuration.md)

# Keymap

How `config/mix.keymap` is structured: layers, behaviors, combos, encoders, and the ZMK Studio path. Use this page when you want to edit your bindings.

## File Layout

```dts
#include <behaviors.dtsi>
#include <dt-bindings/zmk/bt.h>
#include <dt-bindings/zmk/keys.h>
#include <dt-bindings/zmk/pointing.h>

#define ZMK_POINTING_DEFAULT_SCRL_VAL 40

/ {
    combos { … };
    behaviors { … };
    keymap { … };
};
```

- Include `behaviors.dtsi` (gives you `&kp`, `&mt`, `&lt`, `&mo`, `&mkp`, `&trans`, `&none`, etc.).
- Include `dt-bindings/zmk/*` for keycodes, BT slot codes, and pointing constants.
- Pointing macros (`ZMK_POINTING_DEFAULT_SCRL_VAL`, etc.) are project-local `#define`s — set them before the root node.

## Layers

Four layers, all 5 rows × 12 columns. Layout in row-major order — the **first 6 cells** of each row are the left half, the **last 6** are the right half.

| Layer | Display name | Purpose                                              |
|-------|--------------|------------------------------------------------------|
| 0     | `Default`    | QWERTY, navigation row, modifier row                 |
| 1     | `Layer 1`    | F-keys, arrow clusters, brackets                     |
| 2     | `Layer 2`    | Shifted symbols (`!@#$%^&*()`, `_+`, `{}`, `<>?:"|`) |
| 3     | `Bluet:th`   | BT profile select / clear                            |

Activation (from the default layer's bottom row):
- `&mo 1` — momentary Layer 1 (held)
- `&mo 2` — momentary Layer 2
- `&mo 3` — momentary Layer 3 (Bluetooth)

To rename a layer in ZMK Studio, change `display-name = "…"` on that layer.

## Bindings Cheat Sheet

| Binding                  | Meaning                                                              |
|--------------------------|----------------------------------------------------------------------|
| `&kp <KEY>`              | Tap key (e.g. `&kp A`, `&kp LCTRL`, `&kp F12`)                       |
| `&mo <N>`                | Momentary layer N                                                    |
| `&mt <MOD> <KEY>`        | Mod-tap (hold = MOD, tap = KEY) — not currently used in Layer 0      |
| `&lt <N> <KEY>`          | Layer-tap (hold = layer N, tap = KEY)                                |
| `&mkp <BTN>`             | Mouse button — `MB1`/`MB2` used on default layer bottom row          |
| `&bt <CMD>`              | BLE — `BT_SEL n`, `BT_CLR`, `BT_CLR_ALL` (used on Layer 3)           |
| `&trans`                 | Transparent — falls through to lower layer                           |
| `&none`                  | Inert — eats the keypress                                            |
| `&bootloader`            | Reset into UF2 bootloader (used by the `boot` combo)                 |

Keycode reference: <https://zmk.dev/docs/keymaps/list-of-keycodes>.

## Combos

```dts
combos {
    compatible = "zmk,combos";

    boot { bindings = <&bootloader>; key-positions = <0 11 54 53>; };
    combo_ru_x  { bindings = <&kp LEFT_BRACKET>;  key-positions = <22 23>; };
    combo_ru_tb { bindings = <&kp RIGHT_BRACKET>; key-positions = <34 35>; };
};
```

- `boot` — press all four corner keys (positions `0`, `11`, `54`, `53`) to jump into UF2 bootloader without touching the reset button.
- `combo_ru_x` / `combo_ru_tb` — left and right bracket via two-key chords (helpful on Russian keymaps where `[` / `]` are awkward).

Key positions are zero-indexed in the **default_transform** (row-major across the full 60-key matrix; see `boards/shields/mix/mix.dtsi`).

## Custom Behaviors

```dts
behaviors {
    encoder_sc: encoder_sc {
        compatible = "zmk,behavior-sensor-rotate-var";
        label = "ENCODER_SC";
        #sensor-binding-cells = <2>;
        bindings = <&msc>, <&msc>;
        tap-ms = <40>;
    };
};
```

`encoder_sc` rotates `&msc` (mouse-scroll) — both CW and CCW emit `&msc` with different scroll values. Used on Layer 0's first sensor binding (`<&encoder_sc SCRL_UP SCRL_DOWN>`).

## Encoder Bindings (`sensor-bindings`)

Each layer can override what the two encoders do:

| Layer | Left encoder                          | Right encoder                                |
|-------|---------------------------------------|----------------------------------------------|
| 0     | `&encoder_sc SCRL_UP SCRL_DOWN`       | `&inc_dec_kp C_VOLUME_UP C_VOLUME_DOWN`      |
| 1     | `&inc_dec_kp PG_UP PG_DN`             | `&inc_dec_kp C_VOL_UP C_VOL_DN`              |
| 2     | `&inc_dec_kp UP_ARROW DOWN`           | `&inc_dec_kp C_VOL_UP C_VOL_DN`              |
| 3     | `&inc_dec_kp UP_ARROW DOWN_ARROW`     | `&inc_dec_kp C_VOL_UP C_VOL_DN`              |

`&inc_dec_kp <CW> <CCW>` is a built-in ZMK helper for tap-on-rotate.

Order of `sensor-bindings` mirrors the `sensors` node order in `mix.dtsi`: `<&left_encoder &right_encoder>`.

## Trackpad

The trackpad is hardware (right half) — its bindings live in the matrix bottom row, not in `sensor-bindings`:

- `&mkp MB1` / `&mkp MB2` — primary / secondary click on the bottom row.
- Scroll behavior is activated when one of the scroll-mode layers is held — see `MIX_SCROLL_LAYERS` in `config/mix_trackpad_config.dtsi`.

## ZMK Studio

ZMK Studio is enabled on the right (central) half via the `studio-rpc-usb-uart` snippet in `build.yaml` and `CONFIG_ZMK_STUDIO=y` in `config/mix_right.conf`. The `mix.zmk.yml` shield metadata + `mixlayout.dtsi` physical layout let Studio render the keyboard.

Locking is disabled for convenience:

```kconfig
CONFIG_ZMK_STUDIO_LOCKING=n
CONFIG_ZMK_STUDIO_LOCK_ON_DISCONNECT=n
```

Connect the right half via USB and open <https://zmk.studio/> to edit the keymap live.

## Editing Workflow

1. Edit `config/mix.keymap`.
2. **If you add/remove keys**, mirror the change in `config/mix.json` (the keymap-editor / drawer layout) — otherwise the rendered SVG will desync.
3. Commit + push. The **Draw ZMK keymaps** workflow renders `doc/mix.svg` and auto-commits it (see `.github/workflows/draw-keymaps.yml`).
4. The **Build ZMK firmware** workflow produces new `.uf2` files in parallel — flash as in [getting-started.md](getting-started.md#5-flash-each-half).

## See Also

- [Hardware](hardware.md) — matrix layout, encoders, trackpad pins
- [Configuration](configuration.md) — `mix_trackpad_config.dtsi`, Studio Kconfig, build matrix

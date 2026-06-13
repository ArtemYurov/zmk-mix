[← Getting Started](getting-started.md) · [Back to README](../README.md) · [Keymap →](keymap.md)

# Hardware

What the shield defines, pin assignments, and per-half responsibilities. All values below come from `boards/shields/mix/`.

## Controllers

Both halves use the **`nice!nano v2`** (nRF52840), an Arduino Pro-Micro-pinout BLE board.

## Split Roles

| Half  | Role        | Source                                                          | Responsibilities                                                                          |
|-------|-------------|-----------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| Left  | Peripheral  | Default (no `ZMK_SPLIT_ROLE_CENTRAL` override)                  | Keys, left encoder, `nice!view` display fed by relayed central state                       |
| Right | Central     | `Kconfig.defconfig`: `ZMK_SPLIT_ROLE_CENTRAL=y` if `SHIELD_MIX_RIGHT` | Keys, right encoder, Cirque trackpad, BLE to host, USB host, ZMK Studio                  |

State (active layer, BLE, battery) flows **right → left** through `zmk-central-states-relay`; the left half's display renders it via `nice-view-central-relay`.

## Matrix

Defined in `boards/shields/mix/mix.dtsi`.

- Driver: `zmk,kscan-gpio-matrix`
- Diode direction: `col2row`
- Dimensions: 5 rows × 12 columns total (each half contributes 6 columns)
- Rows (shared, from `pro_micro` pins, pull-down): `16, 10, 1.7, 1.2, 1.1`
- Columns (per half, see overlays):
  - Left half (`mix_left.overlay`): `pro_micro 21, 20, 19, 18, 15, 14`
  - Right half (`mix_right.overlay`): `pro_micro 14, 15, 18, 19, 20, 21` — note the right half overlay applies `col-offset = <6>` to `default_transform`, placing right-side keys in columns 6–11 of the combined transform.

## Encoders

Defined in `boards/shields/mix/mix.dtsi` as `alps,ec11`. Each half **enables only its own encoder** via its overlay:

```dts
&left_encoder  { status = "okay"; };   // in mix_left.overlay
&right_encoder { status = "okay"; };   // in mix_right.overlay
```

- A/B GPIOs: `pro_micro 0` / `pro_micro 9` (same on both halves — each board only wires the encoder on its own side)
- Steps: 80
- Triggers per rotation: 48 (`sensors` node)
- Required Kconfig (in shared `mix.conf`): `CONFIG_EC11=y`, `CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y`

Sensor order in the keymap: `<&left_encoder &right_encoder>`. The first `sensor-bindings` entry maps to the left encoder, the second to the right.

## Display (left half only)

The left half mounts a `nice!view` panel. Build variant adds two upstream shields:
`mix_left nice_view_adapter nice_view_gem`. Enabled via `config/mix_left.conf`:

```kconfig
CONFIG_ZMK_DISPLAY=y
CONFIG_ZMK_DISPLAY_STATUS_SCREEN_CUSTOM=y
CONFIG_NICE_VIEW_GEM_PERIPHERAL_CENTRAL_RELAY=y
```

The display reads relayed central data (layer, BLE state, battery) rather than its own peripheral state.

## Cirque Pinnacle Trackpad (right half only)

Defined in `boards/shields/mix/mix_right.overlay` on `pro_micro_spi`:

| Property               | Value                                                                |
|------------------------|----------------------------------------------------------------------|
| Driver                 | `cirque,pinnacle` (geeksville/cirque-input-module on `zmk-v0.3`; built-in on ZMK `main`) |
| Bus                    | SPI on `pro_micro_spi`                                               |
| Chip select            | `gpio0 24` active-low                                                |
| SPI max frequency      | 1 MHz                                                                |
| Data-ready interrupt   | `gpio1 4` active-high                                                |
| Sensitivity            | `MIX_CIRQUE_SENSITIVITY` from `config/mix_trackpad_config.dtsi` (default `"2x"`) |
| Sleep mode             | Disabled — trackpad stays always-on                                  |
| Tap-to-click           | Enabled (default)                                                    |

Pointer / scroll speed and which layers activate scroll-mode are tuned in `config/mix_trackpad_config.dtsi` — see [Configuration → Trackpad](configuration.md#trackpad-tuning).

Required Kconfig (in `config/mix_right.conf`):

```kconfig
CONFIG_ZMK_POINTING=y
CONFIG_ZMK_MOUSE=y
CONFIG_INPUT_THREAD_STACK_SIZE=2048
```

## SPI Wiring (right half)

Defined in `mix_right.overlay`:

```dts
spi1_default {
    psels = <NRF_PSEL(SPIM_SCK,  1, 00)>,
            <NRF_PSEL(SPIM_MOSI, 0, 11)>,
            <NRF_PSEL(SPIM_MISO, 0, 22)>;
};
```

`spi3` on the left half is also pre-pinned (`mix_left.overlay`) for `nice!view` MOSI on `0,6`, ready for the upstream `nice_view_*` shields.

## Bluetooth

- Profile management lives on the right (central) half.
- TX power boost: `CONFIG_BT_CTLR_TX_PWR_PLUS_8=y` (shared `mix.conf`).
- Experimental connection: `CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y`.
- Bonding wipe: flash `settings_reset` firmware (see [getting-started.md](getting-started.md#5-flash-each-half)).

## Sleep

Shared sleep policy in `config/mix.conf`:

```kconfig
CONFIG_ZMK_IDLE_TIMEOUT=450000          # 7m30s before idle (USB activity wakes)
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=1800000   # 30m before deep sleep
```

## See Also

- [Configuration](configuration.md) — Kconfig overrides, build matrix, trackpad tuning
- [Keymap](keymap.md) — how keys map onto the matrix and encoders

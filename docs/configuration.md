[← Keymap](keymap.md) · [Back to README](../README.md)

# Configuration

Reference for every knob you can turn without touching shield/hardware files: build matrix, module pinning, Kconfig fragments, and trackpad tuning.

## `build.yaml` — Build Matrix

Top-level file that CI iterates over. Each entry is `{board, shield[, snippet]}`.

```yaml
include:
  - board: nice_nano_v2
    shield: mix_left nice_view_adapter nice_view_gem
  - board: nice_nano_v2
    shield: mix_right
    snippet: studio-rpc-usb-uart
  - board: nice_nano_v2
    shield: settings_reset
```

| Field    | Notes                                                                                                |
|----------|------------------------------------------------------------------------------------------------------|
| `board`  | Always `nice_nano_v2` — this repo only targets the nice!nano v2.                                     |
| `shield` | Space-separated. First token is the Mix half; additional tokens are stacked upstream shields (e.g. `nice_view_adapter nice_view_gem`). |
| `snippet`| Optional. `studio-rpc-usb-uart` enables ZMK Studio over USB on the right half.                       |

Add a new variant by appending a row. **Do not** add a third "main" entry — split keyboards must have both `mix_left` and `mix_right` present in the matrix or bonding breaks.

## `config/west.yml` — Module Pinning

West manifest that pulls in ZMK + third-party modules at build time. Branch-scoped: `main` and `zmk-v0.3` use deliberately different revisions.

```yaml
manifest:
  remotes:
  - name: zmkfirmware
    url-base: https://github.com/zmkfirmware

  projects:
  - name: zmk
    remote: zmkfirmware
    revision: v0.3                  # ← pinned per branch
    import: app/west.yml
  - name: cirque-input-module
    url: https://github.com/geeksville/cirque-input-module
    revision: main
    path: modules/cirque-input-module
  - name: zmk-central-states-relay
    url: https://github.com/ArtemYurov/zmk-central-states-relay
    revision: main
  - name: nice-view-central-relay
    url: https://github.com/ArtemYurov/nice-view-central-relay
    revision: zmk-v0.3
  self:
    path: config
```

| Module                          | Purpose                                                                  | Pinning rule                                          |
|---------------------------------|--------------------------------------------------------------------------|-------------------------------------------------------|
| `zmk`                           | Core ZMK firmware                                                        | `v0.3` on `zmk-v0.3`, `main` on `main`                |
| `cirque-input-module`           | Cirque Pinnacle trackpad driver (out-of-tree)                            | Used on `zmk-v0.3` only — ZMK `main` has the binding built-in |
| `zmk-central-states-relay`      | Relays layer/BLE/battery state from central to peripheral                | Same revision on both branches                        |
| `nice-view-central-relay`       | Drives `nice!view` on the peripheral half from the relayed central state | `main` on ZMK `main`, `zmk-v0.3` on the v0.3 branch   |

**Rules:**
- Every entry must carry an explicit `revision:`. Don't rely on the default branch.
- Never copy `west.yml` across branches without re-verifying each `revision:`.
- Adding a module: append under `projects:` with `name`, `url` (or `remote`), `revision`, optional `path`.

## Kconfig Fragments (`config/*.conf`)

Three files, applied in this order at build time:

1. `config/mix.conf` — applies to both halves.
2. `config/mix_left.conf` — applies only when shield `mix_left` is built.
3. `config/mix_right.conf` — applies only when shield `mix_right` is built.

### `mix.conf` (shared)

```kconfig
# Encoder
CONFIG_EC11=y
CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y

# Sleep
CONFIG_ZMK_IDLE_TIMEOUT=450000        # ms before idle (7.5 min)
CONFIG_ZMK_SLEEP=y
CONFIG_ZMK_IDLE_SLEEP_TIMEOUT=1800000  # ms before deep sleep (30 min)

# Bluetooth
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y         # +8 dBm TX power
CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y
```

### `mix_left.conf` (left half = peripheral, with `nice!view`)

```kconfig
CONFIG_ZMK_DISPLAY=y
CONFIG_ZMK_DISPLAY_STATUS_SCREEN_CUSTOM=y
CONFIG_NICE_VIEW_GEM_PERIPHERAL_CENTRAL_RELAY=y
```

### `mix_right.conf` (right half = central, with trackpad + Studio)

```kconfig
# Pointing / mouse
CONFIG_ZMK_POINTING=y
CONFIG_ZMK_MOUSE=y
CONFIG_INPUT_THREAD_STACK_SIZE=2048

# ZMK Studio
CONFIG_ZMK_STUDIO=y
CONFIG_ZMK_STUDIO_LOCKING=n
CONFIG_ZMK_STUDIO_LOCK_ON_DISCONNECT=n
```

### Shield Defaults (read-only reference)

Hardware defaults live in `boards/shields/mix/Kconfig.defconfig`. The right-shield-only block sets the central role:

```kconfig
if SHIELD_MIX_RIGHT
config ZMK_KEYBOARD_NAME      default "mix"
config ZMK_SPLIT_ROLE_CENTRAL default y
config ZMK_POINTING           default y
endif

if SHIELD_MIX_LEFT || SHIELD_MIX_RIGHT
config SPI                    default y
config ZMK_SPLIT              default y
endif
```

These are defaults — anything in `config/*.conf` overrides them.

## Trackpad Tuning (`config/mix_trackpad_config.dtsi`)

`#define` macros consumed by `mix_right.overlay` and the trackpad input processor. Edit values to taste, then rebuild.

```dts
/* Scroll layers: layer numbers that activate scroll mode */
#define MIX_SCROLL_LAYERS 1 2

/* Cirque hardware sensitivity: "1x" (default), "2x", "4x" */
#define MIX_CIRQUE_SENSITIVITY "2x"

/* Mouse pointer speed: numerator / denominator (higher = faster) */
#define MIX_MOUSE_SPEED_NUM 5
#define MIX_MOUSE_SPEED_DEN 2

/* Scroll speed: lower ratio = slower scroll */
#define MIX_SCROLL_SPEED_NUM 1
#define MIX_SCROLL_SPEED_DEN 30
```

Guidance:

| Symptom                                  | Try                                                              |
|------------------------------------------|------------------------------------------------------------------|
| Pointer too slow                         | Increase `MIX_MOUSE_SPEED_NUM` (e.g. `3/1`) or raise sensitivity |
| Pointer overshoots, hard to land cursor  | Drop `MIX_CIRQUE_SENSITIVITY` to `"1x"` or lower the speed ratio |
| Scroll too fast                          | Increase `MIX_SCROLL_SPEED_DEN` (e.g. from `30` to `60`)         |
| Scroll mode never activates              | Adjust `MIX_SCROLL_LAYERS` to include the layer you actually hold |

## `config/mix.json`

JSON layout consumed by `keymap-editor` and `keymap-drawer`. Keep it **in sync with `config/mix.keymap`** whenever you add or remove keys — otherwise the auto-rendered SVG (`doc/mix.svg`) drifts from the firmware truth.

## CI Workflows

| Workflow                      | Trigger                                              | Purpose                                                         |
|-------------------------------|------------------------------------------------------|-----------------------------------------------------------------|
| `.github/workflows/build.yml` | `push`, `pull_request`, manual                       | Compiles every `build.yaml` entry → `firmware.zip` artifact     |
| `.github/workflows/draw-keymaps.yml` | Manual or `push` touching `config/mix.keymap`, `mixlayout.dtsi`, or `keymap_drawer.config.yaml` | Renders `doc/mix.svg` + `doc/mix.yaml` via `caksoylar/keymap-drawer` and commits them |

The build workflow delegates to `zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3` (`@main` on the `main` branch).

## See Also

- [Getting Started](getting-started.md) — fork, build, flash
- [Hardware](hardware.md) — pin assignments, encoders, trackpad
- [Keymap](keymap.md) — layers, behaviors, combos, Studio

[Back to README](../README.md) · [Hardware →](hardware.md)

# Getting Started

How to fork this repo, build firmware in GitHub Actions, and flash it onto the Mix keyboard.

## Prerequisites

- A GitHub account (forking + Actions).
- A Mix split keyboard with two `nice!nano v2` controllers.
- A USB-C cable and a thin tool (paperclip / tweezers) to press the reset button.
- That's it — **no local toolchain required**. Everything compiles in CI.

## 1. Fork or Clone

```bash
# Fork via the GitHub UI, then:
git clone git@github.com:<your-user>/zmk-mix.git
cd zmk-mix
```

## 2. Choose a Branch

The repo maintains two parallel ZMK lines. Modules in `config/west.yml` are pinned per branch — pick one and stay on it.

| Branch     | ZMK         | Notes                                                                           |
|------------|-------------|---------------------------------------------------------------------------------|
| `main`     | ZMK `main`  | Latest ZMK on Zephyr 4.1+. Trackpad binding is upstream.                        |
| `zmk-v0.3` | ZMK `v0.3`  | Stable. Uses `geeksville/cirque-input-module` for the trackpad. **Recommended** until the drag-select regression on `main` is fixed. |

```bash
git checkout zmk-v0.3   # or: git checkout main
```

## 3. Trigger a Build

Push a commit (or just the branch you forked) — GitHub Actions runs `Build ZMK firmware` automatically. You can also trigger it manually:

- GitHub UI → **Actions** → **Build ZMK firmware** → **Run workflow** → pick branch.

The workflow uses `zmkfirmware/zmk/.github/workflows/build-user-config.yml` and reads `build.yaml` for the variant matrix:

| Variant         | Shield                                            | Purpose                                  |
|-----------------|---------------------------------------------------|------------------------------------------|
| `mix_left`      | `mix_left nice_view_adapter nice_view_gem`        | Left half: keys + `nice!view` display    |
| `mix_right`     | `mix_right` (snippet `studio-rpc-usb-uart`)       | Right half: keys + trackpad + ZMK Studio |
| `settings_reset`| `settings_reset`                                  | Bonding/Bluetooth wipe firmware          |

## 4. Download Firmware

When the workflow finishes (green check):

1. Open the workflow run.
2. Scroll to **Artifacts** → download `firmware.zip`.
3. Unzip — you get three `.uf2` files:
   - `mix_left-nice_nano_v2-zmk.uf2`
   - `mix_right-nice_nano_v2-zmk.uf2`
   - `settings_reset-nice_nano_v2-zmk.uf2`

## 5. Flash Each Half

For **each half** (left and right):

1. Connect the half via USB-C.
2. Double-press the on-board reset button on the `nice!nano v2`. The board reboots into UF2 bootloader mode — a `NICENANO` disk appears on your computer.
3. Drag the matching `.uf2` onto `NICENANO`. The disk auto-unmounts; the board reboots into the new firmware.
4. Disconnect and repeat for the other half — **use the half's matching `.uf2`** (`mix_left*.uf2` on the left half, `mix_right*.uf2` on the right half).

> The `settings_reset` firmware is for when Bluetooth bonding gets stuck. Flash it onto a half, wait ~10 seconds for it to wipe settings, then re-flash the normal `mix_left*` / `mix_right*` firmware.

## 6. Verify

- Both halves should power on. With Bluetooth: pair the **right (central)** half to your host — the left half connects to the right over BLE, not to the host directly.
- Type something — keys from both sides should land.
- Touch the trackpad on the right half — pointer moves.
- Look at the `nice!view` on the left half — it shows layer/BLE/battery info relayed from the central.

## Next Steps

- See [hardware.md](hardware.md) for the physical layout (matrix, encoders, displays, trackpad pins).
- See [keymap.md](keymap.md) to customize layers, behaviors, and combos.
- See [configuration.md](configuration.md) to tune Kconfig, pin modules, or add build variants.

## See Also

- [Hardware](hardware.md) — what the shield actually defines
- [Configuration](configuration.md) — `west.yml`, `build.yaml`, `*.conf`

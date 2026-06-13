# zmk-mix

## Overview

ZMK firmware user-config for the **Mix** split mechanical keyboard. This repository contains shield definitions, layout/keymap, build manifest, and CI to compile firmware for the `nice!nano v2` (nRF52840) controller. There is no traditional source code — all artifacts are Devicetree overlays, Kconfig fragments, and YAML/JSON configuration consumed by the Zephyr/ZMK build system.

Firmware (`.uf2`) is produced by GitHub Actions and downloaded from the workflow artifacts.

## Core Features

- Mix split keyboard shield (`mix_left` / `mix_right`) on nice!nano v2
- 5×12 split matrix, `col2row` direction
- One rotary encoder per half (`left_encoder`, `right_encoder`, both `alps,ec11`)
- `nice!view` display on the **left (peripheral)** half, driven by relayed central state via `nice-view-central-relay`
- Cirque Pinnacle trackpad on the **right (central)** half
- Split roles: **right = central** (`CONFIG_ZMK_SPLIT_ROLE_CENTRAL=y` in `Kconfig.defconfig` under `if SHIELD_MIX_RIGHT`), left = peripheral (default)
- ZMK Studio enabled on the right (central) half via `studio-rpc-usb-uart` snippet
- `settings_reset` build for bonding/Bluetooth reset
- Keymap rendering CI workflow (keymap-drawer)

## Tech Stack

- **Build system:** West (Zephyr meta-tool) + ZMK + Zephyr RTOS
- **Controller:** nice!nano v2 (nRF52840)
- **Configuration languages:** Devicetree (`.dtsi`, `.overlay`, `.keymap`), Kconfig (`.conf`, `Kconfig.shield`, `Kconfig.defconfig`), YAML (`build.yaml`, `west.yml`), JSON (`mix.json` keymap-editor layout)
- **CI:** GitHub Actions (`.github/workflows/build.yml`, `draw-keymaps.yml`)
- **External modules** (pinned in `config/west.yml`):
  - `zmkfirmware/zmk` — core firmware
  - `geeksville/cirque-input-module` — trackpad driver (zmk-v0.3 branch only; main has it built-in)
  - `ArtemYurov/zmk-central-states-relay` — relays layer/BLE/battery state from central to peripheral
  - `ArtemYurov/nice-view-central-relay` — `nice!view` display driven by relayed central data

## Branch Strategy

The repo has parallel branches for two ZMK lines. Modules in `west.yml` are pinned per branch — pick a branch and stay on it.

| Branch     | ZMK     | Description                                                                                                |
|------------|---------|------------------------------------------------------------------------------------------------------------|
| `main`     | `main`  | Latest ZMK on Zephyr 4.1+                                                                                  |
| `zmk-v0.3` | `v0.3`  | Stable ZMK v0.3, uses external `geeksville/cirque-input-module` for trackpad                              |

Known issue on ZMK `main`: holding `&mkp MB1` while touching the trackpad releases the held button mid-drag (drag-select broken). Caused by the post-PR-#2477 mouse-subsystem refactor. Prefer `zmk-v0.3` for daily use until upstream fixes it.

## Architecture

See `.ai-factory/ARCHITECTURE.md` for the full architecture guidelines.
**Pattern:** Layered ZMK Shield Configuration (Hardware Definition → Runtime Configuration → Build Orchestration).

## Architecture Notes

- Shield-only project — no board files (`nice!nano v2` is provided by ZMK upstream).
- Files under `boards/shields/mix/` define hardware (matrix, OLED, encoders, trackpad pins, display).
- Files under `config/` are user-facing: `mix.keymap` (keys/layers/behaviors), `mix.conf` (global Kconfig), `mix_left.conf` / `mix_right.conf` (per-side Kconfig), `mix_trackpad_config.dtsi` (trackpad tuning), `west.yml` (module pins), `mix.json` (keymap-editor layout).
- Builds are declared in `build.yaml` — each entry is a `{board, shield[, snippet]}` tuple consumed by the GitHub Actions matrix.
- Keymap-drawer renders SVGs from `mix.keymap` + `mix.json` on every keymap change.

## Non-Functional Requirements

- **Reproducibility:** firmware builds are deterministic — pinned module revisions in `west.yml` guarantee identical CI output across runs.
- **Branch isolation:** never cross-pollinate keymap/shield changes between `main` and `zmk-v0.3` without re-validating the matching ZMK revision.
- **No secrets:** repository contains only configuration — no credentials, tokens, or PII.

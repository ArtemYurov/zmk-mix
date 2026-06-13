# AGENTS.md

> Structural map for AI agents working in this repo. Keep factual; update when the project layout changes significantly. The Documentation section is maintained by `/aif-docs`.

## Project Overview

ZMK firmware user-config for the **Mix** split mechanical keyboard (`nice!nano v2`). See `.ai-factory/DESCRIPTION.md` for the full description.

## Tech Stack

- **Build system:** West + ZMK + Zephyr RTOS
- **Controller:** nice!nano v2 (nRF52840)
- **Configuration languages:** Devicetree, Kconfig, YAML, JSON
- **CI:** GitHub Actions (firmware build, keymap render)
- **No source code:** repository contains only declarative configuration

## Project Structure

```
zmk-mix/
├── boards/shields/mix/        # Hardware shield definition (matrix, OLED, encoders, trackpad, display)
│   ├── mix.dtsi               # Common shield Devicetree
│   ├── mix.zmk.yml            # Shield metadata
│   ├── mix_left.overlay       # Left half overlay
│   ├── mix_right.overlay      # Right half overlay
│   ├── mix_trackpad.dtsi      # Cirque trackpad node
│   ├── mixlayout.dtsi         # Physical layout (for studio/render)
│   ├── Kconfig.shield         # Shield Kconfig declarations
│   └── Kconfig.defconfig      # Default Kconfig values
├── config/                    # User-facing configuration
│   ├── mix.keymap             # Layers, behaviors, key bindings
│   ├── mix.conf               # Global Kconfig overrides
│   ├── mix_left.conf          # Left half (peripheral): nice!view display + central-state relay
│   ├── mix_right.conf         # Right half (central): trackpad, ZMK Studio, pointing
│   ├── mix_trackpad_config.dtsi  # Trackpad tuning (CPI, scroll, taps)
│   ├── mix.json               # keymap-editor / keymap-drawer layout
│   └── west.yml               # Pinned module manifest (per-branch)
├── .github/workflows/         # CI
│   ├── build.yml              # Compile firmware (matrix from build.yaml)
│   └── draw-keymaps.yml       # Render keymap SVGs via keymap-drawer
├── build.yaml                 # CI build matrix entries
├── doc/                       # Repo documentation assets
├── firmware/                  # Reserved (empty) — local firmware output
├── zephyr/                    # West-managed Zephyr placeholder
└── .ai-factory/               # AI Factory context (DESCRIPTION, ARCHITECTURE, rules, config)
```

## Key Entry Points

| File                                  | Purpose                                                                          |
|---------------------------------------|----------------------------------------------------------------------------------|
| `build.yaml`                          | CI build matrix — add/remove firmware variants here                              |
| `config/west.yml`                     | Pinned external modules (ZMK, trackpad, relays) — branch-scoped revisions        |
| `config/mix.keymap`                   | Keymap (layers, behaviors, encoder bindings) — primary user-facing file          |
| `config/mix.conf`                     | Global Kconfig overrides (BLE, sleep, USB)                                       |
| `config/mix_left.conf` / `mix_right.conf` | Per-side Kconfig overrides (display on left peripheral; trackpad + Studio on right central) |
| `boards/shields/mix/mix.dtsi`         | Common shield hardware definition                                                |
| `boards/shields/mix/mix_left.overlay` / `mix_right.overlay` | Per-side hardware specifics                                |
| `.github/workflows/build.yml`         | Firmware build pipeline                                                          |
| `.github/workflows/draw-keymaps.yml`  | Keymap SVG render pipeline                                                       |

## Documentation

| Document        | Path                          | Description                                                           |
|-----------------|-------------------------------|-----------------------------------------------------------------------|
| README          | `README.md`                   | Project landing page, branches, module pinning, doc links             |
| Getting Started | `docs/getting-started.md`     | Fork, build firmware in CI, flash `.uf2` onto each half               |
| Hardware        | `docs/hardware.md`            | Shield definition: matrix, encoders, display, trackpad, split roles   |
| Keymap          | `docs/keymap.md`              | Layers, behaviors, combos, encoder bindings, ZMK Studio               |
| Configuration   | `docs/configuration.md`       | `build.yaml`, `west.yml`, Kconfig fragments, trackpad tuning          |

## AI Context Files

| File                                | Purpose                                                                       |
|-------------------------------------|-------------------------------------------------------------------------------|
| `AGENTS.md`                         | This file — structural map and agent rules                                    |
| `.ai-factory/DESCRIPTION.md`        | Detailed project description, tech stack, branch strategy                     |
| `.ai-factory/ARCHITECTURE.md`       | Architecture pattern, folder rules, conventions (generated by `/aif-architecture`) |
| `.ai-factory/rules/base.md`         | Detected naming/structure/style conventions                                   |
| `.ai-factory/config.yaml`           | AI Factory run-scoped settings (language, paths, git, rules)                  |

## Agent Rules

- **Branch awareness:** This repo runs two parallel ZMK lines (`main`, `zmk-v0.3`). Module revisions in `config/west.yml` differ per branch. Never copy keymap/config changes between branches without verifying the matching ZMK revision and behaviour.
- **No source code:** Do not propose adding `.c`/`.cpp`/`.py` etc. Changes must stay within Devicetree, Kconfig, YAML, or JSON.
- **Shell commands:** decompose composite commands into single steps.
  - Incorrect: `git checkout main && git pull`
  - Correct: first `git checkout main`, then `git pull origin main`.
- **Builds run in CI:** there is no local toolchain. Verification = open the workflow run, check the matrix, download the `.uf2`, and flash to test.
- **Keymap + layout in sync:** if you modify `config/mix.keymap` (keys added/removed), keep `config/mix.json` aligned so the keymap-drawer workflow renders correctly.

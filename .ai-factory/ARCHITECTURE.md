# Architecture: Layered ZMK Shield Configuration

## Overview

This repository is a **ZMK user-config** for the Mix split mechanical keyboard. It contains no traditional source code — only declarative configuration consumed by the Zephyr/ZMK build system. The natural architecture for such a project is a **three-layer separation** between *Hardware Definition* (what the PCB looks like), *Runtime Configuration* (how the user wants it to behave), and *Build Orchestration* (which firmware variants CI must produce).

This layering matches the conventions established by upstream ZMK and Zephyr: shield files live under `boards/shields/`, user overrides live under `config/`, and a top-level `build.yaml` declares the build matrix. Keeping the three layers strictly separated makes the project diff-friendly, branch-portable, and CI-reproducible.

## Decision Rationale

- **Project type:** Firmware configuration (no application code, no runtime executed on a host machine).
- **Tech stack:** Devicetree (`.dtsi`, `.overlay`, `.keymap`), Kconfig (`.conf`, `Kconfig.shield`, `Kconfig.defconfig`), YAML (`build.yaml`, `west.yml`), JSON (`mix.json`), built by West + Zephyr + ZMK in GitHub Actions.
- **Key factor:** Architecture must mirror upstream ZMK conventions exactly — divergence breaks the build, the keymap-drawer render, and ZMK Studio.

## Folder Structure

```
zmk-mix/
├── boards/shields/mix/        # Layer 1: Hardware Definition
│   ├── mix.dtsi               #   Shared Devicetree (matrix, encoders, OLED, common nodes)
│   ├── mix.zmk.yml            #   Shield metadata (id, name, features) for ZMK Studio
│   ├── mix_left.overlay       #   Left half: GPIO matrix, central role, display
│   ├── mix_right.overlay      #   Right half: peripheral role, trackpad pins
│   ├── mix_trackpad.dtsi      #   Cirque trackpad node template
│   ├── mixlayout.dtsi         #   Physical key layout for studio/keymap-drawer
│   ├── Kconfig.shield         #   Shield Kconfig symbol declarations
│   └── Kconfig.defconfig      #   Default Kconfig values for the shield
├── config/                    # Layer 2: Runtime Configuration
│   ├── mix.keymap             #   Layers, behaviors, encoder bindings (user-editable)
│   ├── mix.conf               #   Global Kconfig overrides (BLE, sleep, USB)
│   ├── mix_left.conf          #   Left-half overrides (central, display)
│   ├── mix_right.conf         #   Right-half overrides (peripheral, ZMK Studio)
│   ├── mix_trackpad_config.dtsi  # Trackpad tuning (CPI, taps, scroll)
│   ├── mix.json               #   keymap-editor / keymap-drawer layout map
│   └── west.yml               #   Pinned external modules (branch-scoped revisions)
├── build.yaml                 # Layer 3: Build Orchestration
├── .github/workflows/         #
│   ├── build.yml              #   Compiles every entry in build.yaml
│   └── draw-keymaps.yml       #   Renders SVGs from mix.keymap + mix.json
└── .ai-factory/               # AI-agent context (out of build scope)
```

## Dependency Rules

Information flows **upward only**: Build Orchestration consumes Configuration, which references Hardware Definition. Hardware never depends on user configuration, and configuration never depends on the build matrix.

- ✅ `build.yaml` references shield names (`mix_left`, `mix_right`, `settings_reset`) defined in `boards/shields/mix/Kconfig.shield`.
- ✅ `config/mix_left.conf` may override Kconfig symbols declared in `boards/shields/mix/Kconfig.shield`.
- ✅ `config/mix.keymap` includes `<behaviors.dtsi>`, `<dt-bindings/zmk/...>`, and references node labels from `boards/shields/mix/mix.dtsi`.
- ✅ `config/mix_trackpad_config.dtsi` extends nodes declared in `boards/shields/mix/mix_trackpad.dtsi`.
- ✅ `config/west.yml` pins external modules used by both shield files and user config.
- ❌ Shield files (`boards/shields/mix/*`) must NOT reference anything under `config/` — shields are reusable, configs are project-specific.
- ❌ Shield Kconfig must NOT hardcode user preferences (key bindings, BLE PIN, display rotation).
- ❌ `build.yaml` must NOT contain Kconfig overrides — those belong in `config/*.conf`.
- ❌ Never `#include` files across branches (`main` ↔ `zmk-v0.3`) — module revisions differ, behavior diverges.

## Layer / Module Communication

- **Hardware → Configuration:** via Devicetree node labels (`&kscan0`, `&trackpad`, `&encoder_left`) and Kconfig symbols (`CONFIG_ZMK_DISPLAY`, `CONFIG_ZMK_STUDIO`). Hardware exposes the surface; configuration overrides it.
- **Configuration → Build:** via shield name + snippet tuples in `build.yaml`. The build matrix selects which Kconfig fragment merges with which overlay.
- **External modules:** `config/west.yml` is the single integration point — no module is referenced anywhere else by URL. All shield/keymap references go through ZMK-published Devicetree bindings.
- **Side-to-side runtime communication:** at firmware runtime the two halves communicate via the `zmk-central-states-relay` and `nice-view-central-relay` modules. These are wired only via `west.yml` + Kconfig — never via shield-level Devicetree hacks.

## Key Principles

1. **Mirror upstream ZMK conventions.** If a pattern exists in `zmkfirmware/zmk` reference shields (e.g., `corne`, `lily58`), follow it byte-for-byte rather than inventing local style.
2. **Branch-pinned modules are immutable per-branch.** `west.yml` on `main` and `zmk-v0.3` are independent files. Never cherry-pick `west.yml` changes across branches without a manual revision-by-revision review.
3. **Configuration is declarative.** A configuration file describes the desired state; it never describes how the build should reach that state.
4. **Both halves are first-class.** Every change that touches one side's overlay/`.conf` must be evaluated for the other side. A split keyboard with mismatched halves is non-functional.
5. **CI is the source of truth.** Builds run in GitHub Actions. Local "it compiles" is meaningless — only `build.yml` artifacts count.
6. **Keymap and layout stay in sync.** `config/mix.keymap` is the firmware truth; `config/mix.json` is the visual truth for keymap-drawer / keymap-editor. Drift between them breaks the rendered SVG.

## Code Examples

### Example 1 — Hardware layer exposes; configuration layer overrides

`boards/shields/mix/Kconfig.shield` — declaration only:

```kconfig
config SHIELD_MIX_LEFT
	def_bool $(shields_list_contains,mix_left)

config SHIELD_MIX_RIGHT
	def_bool $(shields_list_contains,mix_right)
```

`boards/shields/mix/Kconfig.defconfig` — sane defaults, no opinions:

```kconfig
if SHIELD_MIX_RIGHT

config ZMK_KEYBOARD_NAME
    default "mix"

config ZMK_SPLIT_ROLE_CENTRAL
    default y

config ZMK_POINTING
    default y

endif

if SHIELD_MIX_LEFT || SHIELD_MIX_RIGHT

config SPI
    default y

config ZMK_SPLIT
    default y

endif
```

`config/mix_left.conf` — user override at the configuration layer:

```kconfig
# User preference: enable nice!view display + central-state relay on the left (peripheral) half
CONFIG_ZMK_DISPLAY=y
CONFIG_ZMK_DISPLAY_STATUS_SCREEN_CUSTOM=y
CONFIG_NICE_VIEW_GEM_PERIPHERAL_CENTRAL_RELAY=y
```

The shield never knows whether the user wants the display on. The configuration layer decides.

### Example 2 — Devicetree dependency direction

`boards/shields/mix/mix.dtsi` — defines the label:

```dts
/ {
	kscan0: kscan {
		compatible = "zmk,kscan-gpio-matrix";
		diode-direction = "col2row";
		row-gpios = <&pro_micro 21 (GPIO_ACTIVE_HIGH | GPIO_PULL_DOWN)>;
		/* … */
	};
};
```

`config/mix.keymap` — references the label, never redefines it:

```dts
#include <behaviors.dtsi>
#include <dt-bindings/zmk/keys.h>

/ {
	keymap {
		compatible = "zmk,keymap";
		default_layer {
			bindings = <
				&kp Q  &kp W  &kp E  /* … */
			>;
		};
	};
};
```

Configuration consumes hardware. Never the other way around.

### Example 3 — Build matrix is a thin consumer

`build.yaml` — orchestration only, no logic:

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

Each entry is `{board, shield[, snippet]}`. No Kconfig overrides, no module pins, no keymap mentions — those belong to the lower layers.

## Anti-Patterns

- ❌ Editing `boards/shields/mix/Kconfig.defconfig` to set `CONFIG_ZMK_BLE=y` because "I want BLE on." That's a user preference — it belongs in `config/mix.conf`.
- ❌ `#include`-ing a file from `config/` inside `boards/shields/mix/*.dtsi`. Shields must stay self-contained and reusable.
- ❌ Copy-pasting `config/west.yml` from `main` to `zmk-v0.3` (or vice versa). The module revisions are deliberately different per ZMK line.
- ❌ Changing `mix.keymap` without updating `mix.json` (or vice versa). The keymap-drawer SVGs will desync.
- ❌ Adding a new firmware variant via `boards/` instead of appending an entry to `build.yaml`.
- ❌ Encoding "left" or "right" assumptions in the shared `mix.dtsi`. Side-specific configuration belongs in `mix_left.overlay` / `mix_right.overlay`.
- ❌ Running ad-hoc local builds and trusting the result. Only GitHub Actions output is authoritative.
- ❌ Adding traditional source files (`.c`, `.py`, `.ts`) — this repository is configuration-only.

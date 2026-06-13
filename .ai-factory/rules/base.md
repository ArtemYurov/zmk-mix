# Project Base Rules — zmk-mix

> Auto-detected conventions for a ZMK user-config repository. There is no traditional source code — only Devicetree, Kconfig, YAML, and keymap-editor JSON. Adjust as needed.

## Repository Layout

- `boards/shields/mix/` — hardware definition (matrix, OLED, encoders, trackpad pins, display, Kconfig). Touch only when wiring/HW changes.
- `config/` — user-facing configuration (`mix.keymap`, `mix.conf`, per-side `.conf`, `mix_trackpad_config.dtsi`, `west.yml`, `mix.json`).
- `build.yaml` — top-level CI build matrix. Add a row to produce a new firmware artifact.
- `.github/workflows/` — `build.yml` (firmware) and `draw-keymaps.yml` (keymap SVG render).
- `firmware/` — empty dir, reserved for local artifacts (ignored by CI).
- `doc/` — repository documentation assets.

## Naming Conventions

- Shield files use the `mix_*` prefix: `mix_left.overlay`, `mix_right.overlay`, `mix.dtsi`, `mix.keymap`, `mix.conf`.
- Devicetree node labels use `snake_case` (`mix_layout`, `kscan0`, `trackpad`).
- Devicetree node names use `kebab-case` (`gpio-keys`, `zmk,kscan-gpio-matrix`).
- Kconfig symbols are upper-`SNAKE_CASE` (`CONFIG_ZMK_SLEEP`, `CONFIG_ZMK_KEYBOARD_NAME`).

## Devicetree

- Indent with **tabs**, mirror the style used in upstream ZMK shields.
- Group related properties; keep one property per line.
- Comments use `// …` (single-line) or `/* … */` (block); both are accepted by the DTS preprocessor.
- Always close a node with `};` on its own line.
- Reference labels with `&label` and define them with `label: node-name { … }`.

## Kconfig (`.conf`)

- One `CONFIG_*=value` per line.
- Use `=y` / `=n` for booleans, integers without quotes, strings with `"…"`.
- Per-side overrides live in `mix_left.conf` / `mix_right.conf`; cross-side defaults in `mix.conf`.
- Comment justifying non-obvious values is welcome (`# needed because …`).

## Keymap (`config/mix.keymap`)

- Layer names are uppercase short identifiers (`DEFAULT`, `NUM`, `NAV`, `SYM`).
- Behaviors use ZMK shorthand (`&kp`, `&mt`, `&lt`, `&mo`, `&mkp`, `&trans`, `&none`).
- Keep one row of keys per source line where reasonable for diff-readability.
- Keep `mix.json` (keymap-editor layout) in sync with `mix.keymap` when adding/removing keys; otherwise `draw-keymaps.yml` will render incorrectly.

## `west.yml` / Module Pinning

- Every module entry must carry an explicit `revision:` — never rely on the default branch.
- Module revisions are branch-scoped: `main` and `zmk-v0.3` pin different upstreams. Do not copy `west.yml` between branches without re-checking each `revision:`.
- Adding a module: append under `projects:` with `name`, `url` (or `remote`), `revision`, optional `path`.

## `build.yaml`

- Each entry is `{board, shield[, snippet]}`. The split-keyboard pair must have **both** `mix_left` and `mix_right` entries — omitting one breaks bonding.
- The `settings_reset` build is intentional; keep it.
- `studio-rpc-usb-uart` snippet belongs to the **right** half only (peripheral, ZMK Studio).

## Git / Branching

- Default working branch is whichever ZMK line you target (`main` or `zmk-v0.3`). PRs target the matching branch — do not merge across lines.
- Keep keymap-render commits (`keymap-drawer render`) separate from logic commits when possible — the drawer workflow auto-commits SVGs.

## CI

- All firmware builds run in GitHub Actions; no local toolchain is required.
- Artifacts (`firmware/*.uf2`) are downloaded from the workflow run.
- A failing build is almost always a Devicetree or Kconfig syntax error — read the workflow log from the bottom up to find the first `error:` line.

## Out of Scope

- No application logic, no business rules, no tests in the traditional sense — verification is "does it compile + boot + behave as expected on the keyboard."
- No linters/formatters are enforced for `.dtsi/.overlay/.keymap` (the Zephyr DTS preprocessor is permissive). Maintain consistent style by mimicking surrounding files.

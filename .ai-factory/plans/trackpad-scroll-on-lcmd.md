# Trackpad Scroll Activated by LCMD (Layer-Mod)

Branch: zmk-v0.3 (no new branch — user requested in-place)
Created: 2026-06-13
ZMK target: v0.3 (revision pinned in `config/west.yml`)

## Summary

Today the Cirque trackpad enters scroll mode only when keymap layer 1 or 2 is
active (raised via `&mo 1` / `&mo 2`). The user wants scroll mode to also be
available while holding the left Command (LGUI) key, so the same gesture used
for mouse pointer becomes a scroll wheel without releasing the modifier — and
`Cmd+scroll` shortcuts (browser zoom, Figma zoom, etc.) start working.

The ZMK 0.3 input-listener has no native "activate override when modifier held"
hook — the `scroller` child node's `layers = <...>` property is the only
trigger. So the implementation introduces a virtual MOUSE layer and uses the
official ZMK `layer-mod` (`&lm`) macro to press LGUI **and** activate the
MOUSE layer in one keystroke. When the user releases LCMD, both are released.

Layer numbering shifts: the existing Bluet:th layer moves from slot 3 to slot 4
to make room for the new MOUSE layer at slot 3, and `&mo 3` becomes `&mo 4` in
LAYER0.

## Settings

- Testing: no (no automated test harness for keymap configs)
- Logging: not applicable (devicetree/keymap files have no runtime logging)
- Docs: yes — refresh `.ai-factory/DESCRIPTION.md` / `ARCHITECTURE.md` if layer
  semantics need to be reflected (covered by `/aif-docs` after implement)

## Background

Findings from `/aif-explore` (see chat history; not persisted to RESEARCH.md):

- `app/src/pointing/input_listener.c` (line ~216) gates `scroller` activation
  on `zmk_keymap_layer_active(layer)` only — no modifier-state hook.
- `zmk,behavior-macro-two-param` provides the official `lm` example in
  `docs/docs/keymaps/behaviors/macros.md`; it is built into ZMK 0.3 with no
  external module required.
- Modifier-stuck issue (#2847) only fires for **nested** macros with
  `macro_pause_for_release`. Our MOUSE layer is `&trans`-only (except MB1/MB2),
  so no nested macro triggers — bug does not apply.
- Layers attribute on `scroller` (#2967) is closed-completed by reporter — no
  outstanding bug.

Alternative path (patch ZMK `input-listener.c` to accept a `modifiers`
property) was considered and rejected for simplicity: the `&lm` route requires
zero firmware patches and zero fork maintenance.

## Affected Files

- `config/mix_trackpad_config.dtsi` — `MIX_SCROLL_LAYERS` macro
- `config/mix.keymap` — behaviors block, LAYER0, LAYER1, LAYER2, LAYER3,
  LAYER4 (new)
- (No changes to `boards/shields/mix/*` or `config/*.conf` files)

## Tasks

### Phase 1 — Trackpad scroll config

- [x] **Task 1.** Update `MIX_SCROLL_LAYERS` in
  `config/mix_trackpad_config.dtsi` from `1 2` to `1 2 3`.
  - The macro is consumed at `boards/shields/mix/mix_trackpad.dtsi:19`
    (`scroller { layers = <MIX_SCROLL_LAYERS>; }`).
  - This single edit makes layer 3 a scroll-active layer.

### Phase 2 — Keymap restructure

- [x] **Task 2.** Add `lm` behavior in `config/mix.keymap` under
  `/ { behaviors { ... } }` alongside `encoder_sc`:

  ```dts
  lm: lm {
      compatible = "zmk,behavior-macro-two-param";
      wait-ms = <0>;
      tap-ms = <0>;
      #binding-cells = <2>;
      bindings
          = <&macro_param_1to1>
          , <&macro_press &mo MACRO_PLACEHOLDER>
          , <&macro_param_2to1>
          , <&macro_press &kp MACRO_PLACEHOLDER>
          , <&macro_pause_for_release>
          , <&macro_param_2to1>
          , <&macro_release &kp MACRO_PLACEHOLDER>
          , <&macro_param_1to1>
          , <&macro_release &mo MACRO_PLACEHOLDER>
          ;
  };
  ```

  Source: `zmkfirmware/zmk/docs/docs/keymaps/behaviors/macros.md` (verbatim).

- [x] **Task 3.** Replace LAYER3 contents in `config/mix.keymap` with the new
  MOUSE layer.
  - `display-name = "Mouse"`
  - 5 rows × 12 cols of bindings, all `&trans` EXCEPT:
    - Position 54 (row 4, col 6) → `&mkp MB1`
    - Position 55 (row 4, col 7) → `&mkp MB2`
    - (Same positions as LAYER0; explicit duplication for Studio/visual clarity.)
  - `sensor-bindings = <&encoder_sc SCRL_UP SCRL_DOWN>, <&inc_dec_kp C_VOL_UP C_VOL_DN>;`

- [x] **Task 4.** Add a new LAYER4 immediately after the new LAYER3 in the
  keymap node. Its contents are the **old** LAYER3 (Bluet:th) verbatim:
  - `display-name = "Bluet:th"`
  - bindings: BT_CLR, BT_SEL 0..4, BT_CLR_ALL, rest `&trans` (copy from current
    LAYER3).
  - `sensor-bindings = <&inc_dec_kp UP_ARROW DOWN_ARROW>, <&inc_dec_kp C_VOL_UP C_VOL_DN>;`

- [x] **Task 5.** Update LAYER0 bindings (depends on Tasks 2, 3, 4):
  - Position 50 (row 4, col 2): `&kp LEFT_COMMAND` → `&lm 3 LEFT_COMMAND`
  - Position 58 (row 4, col 10): `&mo 3` → `&mo 4`
  - Other LAYER0 positions unchanged (MB1/MB2 stay at 54/55).

- [x] **Task 6.** Update LAYER1 `sensor-bindings`:
  - Left encoder: replace `&inc_dec_kp PG_UP PG_DN` with `&encoder_sc SCRL_UP SCRL_DOWN`.
  - Right encoder slot: keep `&inc_dec_kp C_VOL_UP C_VOL_DN` (no physical right encoder).
  - LAYER1 key bindings unchanged.

- [x] **Task 7.** Update LAYER2 `sensor-bindings`:
  - Left encoder: replace `&inc_dec_kp UP_ARROW DOWN` with `&encoder_sc SCRL_UP SCRL_DOWN`.
  - Right encoder slot: keep `&inc_dec_kp C_VOL_UP C_VOL_DN`.
  - LAYER2 key bindings unchanged.

### Phase 3 — Verification

- [x] **Task 8.** Push branch and verify CI build (depends on Tasks 1–7):
  - GitHub Actions builds both `mix_left` and `mix_right` shields.
  - Confirm devicetree compiles (`MACRO_PLACEHOLDER` resolves, `&lm` accepted,
    `MIX_SCROLL_LAYERS = 1 2 3` accepted as `<phandle>`-style int array).
  - Confirm both halves produce `.uf2` artifacts.
  - Manual smoke after flashing (out of plan scope, but worth noting):
    - Press LCMD: trackpad becomes scroll, encoder scrolls, LGUI reaches host
      (verify with Cmd+Tab + Cmd+scroll zoom in browser).
    - `&mo 1`/`&mo 2`: trackpad still scrolls (legacy behavior preserved).
    - `&mo 4`: Bluet:th controls work, trackpad reverts to pointer mode.

## Dependencies

- Task 5 blocked by Tasks 2, 3, 4 (`&lm` behavior and LAYER3/LAYER4 must exist
  before referencing them in LAYER0).
- Task 8 blocked by Tasks 1–7.
- Tasks 1, 2, 3, 4, 6, 7 are independent of each other within the same file
  and can be applied in any order (or batched).

## Commit Plan

Single atomic commit at the end of Phase 2 (all keymap+config changes must
land together to keep the firmware buildable):

- **Commit 1** (after Tasks 1–7):
  ```
  feat(keymap): add trackpad scroll on LCMD via layer-mod

  - Introduce Mouse layer (LAYER3) activated by holding LCMD
  - Use ZMK built-in &lm (behavior-macro-two-param) to press LGUI and
    activate Mouse layer in one keystroke
  - Move Bluet:th layer from slot 3 to slot 4
  - Add layer 3 to MIX_SCROLL_LAYERS so scroller engages on Mouse layer
  - Switch left encoder to mouse-scroll on layers 1, 2, 3 (Mouse)
  ```

Push triggers CI build (Task 8).

## Notes

- LCMD remains a one-key affordance: short tap still sends `LGUI` press+release
  to the host. During the tap window (~30 ms), MOUSE layer flickers active,
  but since the layer is `&trans`-only (except MB1/MB2 which match LAYER0),
  no observable side effect.
- `&kp LEFT_COMMAND` and `&kp LGUI` are full synonyms (`dt-bindings/zmk/keys.h`
  lines 785–790). `LEFT_COMMAND` is used to match existing keymap style.
- Single LCMD key only — RCMD/RGUI is not currently bound on this keymap, so
  no other modifier path needs scroll activation.

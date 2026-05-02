# zmk-mix

Cube studio
qq交流群：1037094476

ZMK configuration for the Mix split keyboard (nice!nano v2).

## Branches

> [!IMPORTANT]
> The repo has parallel branches for two ZMK lines. Pick a branch and stay on it — modules in `west.yml` are pinned per branch.
>
> - ZMK `v0.3` (stable) → use the `zmk-v0.3` branch
> - ZMK `main` (Zephyr 4.1+, latest) → use the `main` branch

| Branch | ZMK | Description |
|---|---|---|
| [`main`](../../tree/main) | `main` | base config on latest ZMK (Zephyr 4.1+) |
| [`zmk-v0.3`](../../tree/zmk-v0.3) | `v0.3` | base config on stable ZMK v0.3, uses [geeksville/cirque-input-module](https://github.com/geeksville/cirque-input-module) for the trackpad |

> [!WARNING]
> **Known issue on ZMK `main`** — holding `&mkp MB1` while touching the Cirque trackpad releases the held button mid-drag (drag-select stops working). Caused by the post-PR-#2477 mouse subsystem refactor: trackpad input events overwrite the held button state in the global HID mouse report. Not reproducible on ZMK `v0.3`.
>
> Until fixed upstream, prefer the `zmk-v0.3` branch for daily use.

## External modules

Wired in via `config/west.yml`:

| Module | `main` | `zmk-v0.3` | Purpose |
|---|---|---|---|
| [zmk](https://github.com/zmkfirmware/zmk) | `@main` | `@v0.3` | core ZMK firmware |
| [cirque-input-module](https://github.com/geeksville/cirque-input-module) | — *(Zephyr 4.1 has the binding built-in)* | `geeksville@main` | Cirque Pinnacle trackpad driver |
| [zmk-central-states-relay](https://github.com/ArtemYurov/zmk-central-states-relay) | `ArtemYurov@main` | `ArtemYurov@main` | relays state (layer, BLE, battery) from central to peripheral |
| [nice-view-central-relay](https://github.com/ArtemYurov/nice-view-central-relay) | `ArtemYurov@main` | `ArtemYurov@zmk-v0.3` | nice!view display driven by central data via the relay |

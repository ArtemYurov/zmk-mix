# zmk-mix

Cube studio
qq交流群：1037094476

ZMK configuration for the Mix split keyboard (nice!nano v2).

## External modules

Wired in via `config/west.yml`:

| Module | Source | Purpose |
|---|---|---|
| [zmk](https://github.com/zmkfirmware/zmk) | `zmkfirmware/zmk@main` | core ZMK firmware |
| [zmk-central-states-relay](https://github.com/ArtemYurov/zmk-central-states-relay) | `ArtemYurov@main` | relays state (layer, BLE, battery) from central to peripheral |
| [nice-view-central-relay](https://github.com/ArtemYurov/nice-view-central-relay) | `ArtemYurov@main` | nice!view display driven by central data via the relay |

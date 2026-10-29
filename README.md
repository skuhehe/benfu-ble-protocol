# Benfu BLE Charger Protocol

Independent interoperability documentation for Benfu battery chargers. This repository describes observed protocol behavior and is not a vendor specification.

## Protocol reference

See [benfu-ble-charger.md](benfu-ble-charger.md) for the observed GATT profile, telemetry fields, settings frames, CRC rules, third-stage import layout, evidence limits, and safety guidance.

The reference distinguishes readback-verified settings from transport-only writes. In particular, successful writes to the third-stage import characteristic do not confirm that the charger accepted or applied the data.

## Safety

Charging parameters can affect batteries and charger protections. Prefer read-only access; use manufacturer limits; preserve unchanged fields in whole-frame writes; verify settings by readback when available. Do not bypass charger or BMS protections.

No vendor source code or application package is reproduced. This repository does not assign a license to vendor material.

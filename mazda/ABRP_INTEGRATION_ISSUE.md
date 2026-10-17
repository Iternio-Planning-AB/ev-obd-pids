# ABRP "OBD App" — per-PID TX-header request

Filed against the ABRP "OBD App" Bluetooth integration that consumes
`obdble_cars.json` + per-car PID profiles from this repository.

## Summary

The Mazda 6e profile in this directory polls three distinct ECUs each
tick (broadcast `7DF` for Mode 01 PIDs, `7A1/7A9` for the BMS, `7E2/7EA`
for the HPCM). When loaded into ABRP and connected via a VLink BLE
ELM327 v2.3, SoC and odometer (both on `7EA`) load correctly, but pack
voltage / current (`22F228` / `22F229` on `7A9`) consistently report
NO_DATA — even though identical UDS requests reliably succeed when sent
from a Python+BLE probe outside ABRP on the same hardware and adapter.

## Reproduction

Profile: `mazda/mazda6e.json` on this branch (commit `df3ef76`).
Adapter: VLink iOS-Vlink BLE ELM327 v2.3.
Vehicle: Mazda 6e EU (LVRHDA… VIN prefix), ignition ON, parked.

1. Pair the adapter, connect ABRP, select the Mazda 6e profile.
2. Tick: ABRP shows `soc = 81 %`, `odometer = 7715 km` correctly.
3. Same tick: `voltage` and `current` show as NO_DATA / blank.

Standalone control (same adapter, same parked car, same minute):

```
ATSP6
ATH1
ATSH7A1
ATCRA7A9
22F228     -> 7A9 05 62 F228 0F 95     => 398.9 V  ✓
22F229     -> 7A9 05 62 F229 17 7A     => +2.7 A   ✓
```

So the DIDs and the adapter are fine. The integration is what fails to
reach the BMS.

## Root cause (hypothesis)

ABRP appears to send `Data_Commands` once at session start, then per-PID
only emits the `command` field plus `ATCRA<ecu>` derived from the PID
definition. It does **not** issue a fresh `ATSH<tx>` per PID. Whatever
`ATSH` was set last during init / data-commands stays in force, so when
ABRP polls `22F228`, the request goes out with the last-active TX
header — `7E2` (HPCM) or `7DF` (broadcast) in our case. The BMS at
`7A1` does not listen on either, so it does not answer, and ELM returns
NO_DATA.

This matches the behaviour we see: every PID whose `ecu` lines up with
the conventional `RX = TX + 8` chain `7E0/7E8 → 7E1/7E9 → 7E2/7EA` works,
but DIDs that require a non-trivial TX header switch (`7A1 ↔ 7A9`) do
not.

## What we tried (and what doesn't work)

- **Ordering `Data_Commands`** so BMS is polled before HPCM and the
  required `ATSH7A1 ATCRA7A9` precedes `22F228 22F229` — no effect, as
  `Data_Commands` is treated as a one-shot init.
- **Chained command (`ATSH7A1\r22F228`)** in the `command` field —
  rejected by the ABRP UI; the `\r` is treated as a literal character,
  not an ELM line separator.
- **Setting `ecu` to the TX header (`7A1`)** instead of RX (`7A9`) —
  worth a try, but breaks the response parser because ABRP filters on
  the RX side using `ecu`.

## Proposed enhancement

Add an optional **`header`** field per PID definition (parallel to the
existing `ecu` field) that, when present, causes the integration to
emit `ATSH<header>` immediately before the PID's `command`. With that
field, the Mazda 6e profile becomes:

```json
"voltage": {
  "command":  "22F228",
  "header":   "7A1",
  "ecu":      "7A9",
  "equation": "((A<<8)+B)/10",
  ...
},
"current": {
  "command":  "22F229",
  "header":   "7A1",
  "ecu":      "7A9",
  ...
}
```

Backwards-compatible: when `header` is absent, behaviour is unchanged.
This is the same pattern Car Scanner Pro and the openpilot/comma.ai
DBC stack use; it is also how a hand-rolled Python ELM driver naturally
addresses the problem.

## Why this matters beyond Mazda

Any car whose battery management ECU sits outside the canonical
`7E0…7E7` UDS chain hits the same wall — including several Changan,
BYD, GAC, and SAIC platforms. A single `header` field unlocks the whole
class without per-vehicle workarounds in the integration code.

## What we ship in the meantime

The current `mazda/mazda6e.json` keeps voltage/current entries with
`"ecu": "7A9"` because that is the documented schema. We do not silently
swap `ecu` to `7A1`, because that breaks the integration's response
parser and would only help users on builds that happen to use `ecu` for
TX. ABRP can already derive `power` from the SoC + an assumed pack
voltage when `voltage`/`current` are absent; that degradation is
acceptable for now and reverts cleanly once the `header` field is
supported.

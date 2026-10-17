# Mazda

## Mazda EZ-6 / 6e (EU BEV, EPA1 platform)

Rebadged Changan Deepal SL03. EU model is pure BEV (the China REEV variant is not sold in Europe). Usable battery capacity is 68.8 kWh (LFP, 120 cells, 24 module thermistors).

### Vehicle details used to derive the profile

Reverse-engineered from a 2026 Mazda 6e (VIN prefix `LVRHDA...`) over a VLink BLE ELM327 adapter using Car Scanner Pro v2.1.25 captures and a follow-up live verification drive (190 sample ticks, mixed urban + brief highway burst). Every field was cross-checked against a value visible on the dashboard at capture time and validated against expected magnitudes under load.

| ABRP field | DID / PID | Decoded | Verification observation |
|---|---|---|---|
| `soc` | `22F1AF` on HPCM (RX `7EA`) | `(A<<8\|B)/10` % | drops monotonically with energy use; 1 % over ~3 km matches 68.8 kWh × consumption rate |
| `voltage` | `22F228` on BMS (RX `7A9`) | `(A<<8\|B)/10` V | range 370.6 V (sag under peak discharge) — 404.2 V (rest) |
| `current` | `22F229` on BMS | `((A<<8\|B)-6000)/10` A | range −231.6 A (regen) / +515.6 A (acceleration); positive = discharge / negative = regen — matches ABRP convention |
| `odometer` | `22F1AE` on HPCM | `(A<<24\|B<<16\|C<<8\|D)/10` km | monotonic increase, tracks dashboard within rounding |

Speed comes from the phone GPS (the standard ABRP behaviour). Ext temp is
omitted — see "Not on OBD App" below.

### Derived sanity check (not posted to ABRP — included here for verification)

`power = voltage × current / 1000` peaked at +191.55 kW under heavy acceleration, matching the Mazda 6e EU spec motor peak of 190 kW. Regen peak was −93 kW under hard braking. Both values lie cleanly inside the `current` minValue/maxValue envelope.

### What was reachable but intentionally omitted

The same vehicle exposes much more over OBD:

- 12 V auxiliary battery (`0142`, `(A<<8|B)/1000` V → ~14.25 V),
- BMS cell aggregates: min/max cell V (`22F250`/`F251`), min/max cell V position (`22F252`/`F253`),
- BMS cell min/max temperature aggregates (`22F254`/`F255`, 1-byte raw — scale not yet locked; vendor lookup table suspected),
- 120 individual cell voltages (`22F29F`-`F2EF`, `22F3A9`-`F3B0`, `22F3E1`-`F3FF`, each `(A<<8|B)` mV),
- 24 module temperatures (`22F1B0`-`F1B7` + `22F2F0`-`F2FF`, each `raw-40` = °C),
- HPCM pack-V mirror (`22F1BD`, ~10 V below `F228` — post-relay DC-link sense).

These are kept out to keep the poll cycle short. A separate diagnostic profile can expose them.

### Not on OBD for this car

- `is_charging` and `plug_detected` — exhaustively searched and **not exposed
  via OBD-II**. Search trail:
  1. All 192 named PIDs in the OEM/aftermarket "Mazda EZ-6 / 6e" profile
     used by Car Scanner Pro were enumerated by decoding the PID-definition
     records in the iOS `.brc` recording binary. No name matches
     `Charging`, `Plug`, `Cable`, `EVSE`, `Port`, `Pre-cond`, `HVAC`, or
     `Cabin`.
  2. The full SAE J1979-2 / WWH-OBD standardised DID set (`22F40C`,
     `22F40D`, `22F411`, `22F412`, `22F45B`, `22F47x`, `22F80C`-`22F81F`,
     etc. — 68 DIDs total) was probed against the BMS at `7A1/7A9` in
     default session. **Every single one returned NO_DATA — not even a
     UDS negative response.** The BMS UDS handler is whitelisted to its
     proprietary `F22x` / `F25x` / `F2xx` / `F3xx` range only.
  3. ECU header discovery against 70 candidate 11-bit IDs surfaced one
     extra live ECU at `0x754 / 0x75C` — the VCU. A 1789-DID wide sweep
     across F2/F3/F4/FC/FD/FE/FF on this ECU returned 22 hits, of which
     six were boolean / counter candidates (`F1F7`, `FECA`, `FEF6`,
     `FEF9`, `FEFA`, `FD0C`). A live 4-state delta capture (parked
     unplugged → plugged not charging → AC 4.2 kW charging → cable
     physically unplugged) showed every candidate bit-identical across
     all four states. They are firmware build-time constants, not live
     state.
  4. Earlier `0x700` DID `0xFD03` candidate (`0x00`=charging, `0x40`=idle)
     produced clean transitions but during a 190-tick live verification
     drive flipped 13 times on door-open events — it is a body-network
     wake-source bit, not a charging flag.

  ABRP / any telemetry consumer must therefore **derive `is_charging`
  from current sign**: `is_charging = (pack_I_A < -1) sustained 5 s`.
  The negative-equals-into-pack convention is confirmed by direct
  measurement: `F229` raw 5904-5921 during a 4.2 kW AC charge
  (`(raw - 6000) / 10 = -8.4 to -9.6 A`) versus 6024-6033 during the
  immediately following idle drain (`+2.4 to +3.3 A`). Matches the
  car's spec at 247 V × 17 A × ~93 % charger efficiency = ~9.7 A into
  pack at 404 V.
- `batt_temp` aggregate — `22F254`/`F255` return single bytes whose raw values vary widely (8-112) while the displayed temperature stays stable (18-22 °C), implying a vendor lookup table rather than a linear formula. A v2 of this profile should expose `batt_temp = mean(22F1B0..F1B7, 22F2F0..F2FF, raw − 40)` — the per-module formula is proven across all 24 active probes.
- `soh` — no DID identified on either BMS (`7A1`) or VCU (`0x754`).
  Standardised candidate `22F412` returns NO_DATA on every ECU probed
  so far. Dashboard shows SOCE 100 % at present; revisit when the pack
  has measurable degradation.
- `est_battery_range` — no OBD-reachable DID matches the dashboard value.
  Let ABRP derive server-side from `soe + avg kWh/km`.
- HVAC state / setpoint, TPMS, cabin temp — live on body-CAN behind the
  SVDC gateway; not reachable via the OBD port without a mid-bus tap.

### Not on OBD App (Mode 01 PIDs unsupported by the parser)

Mazda 6e exposes vehicle speed via standard SAE J1979 Mode 01 PID `0x0D`
and ambient temperature via PID `0x46`. We have raw-confirmed both on
the wire (`7EA 03 41 0D 00` for speed = 0 km/h parked; `7EA 03 41 46 41`
for ambient = `0x41 − 40 = 25 °C`). However, when these PIDs are added
to a profile and consumed by the ABRP OBD App, both consistently report
`failed to parse response / got response null`, even though the raw
bytes are visibly received by the integration. Cross-checking against
all 12 other EV profiles in this repository: **none of them uses any
Mode 01 PIDs** — every working profile reads its values exclusively via
Mode 22 UDS DIDs. The empirical conclusion is that the obdble parser in
the ABRP OBD App layer only decodes `62 …` (Mode 22 positive) responses
correctly and does not handle `41 …` (Mode 01 positive) responses.

For the Mazda 6e profile we therefore ship four Mode 22 fields only.
Speed gets picked up from the phone GPS (the standard ABRP fallback)
and `power` / `is_charging` / `is_dcfc` / `is_parked` / `soe` are
derived server-side from `voltage × current` + GPS speed.

If a Mode 01-capable variant of the parser becomes available, the two
omitted PIDs can be re-added trivially:

```json
"speed":    { "command": "010D", "ecu": "7EA", "equation": "A",      "type": "Number", "minValue": 0,   "maxValue": 250 },
"ext_temp": { "command": "0146", "ecu": "7EA", "equation": "A-40",   "type": "Number", "minValue": -40, "maxValue": 80  }
```

with `ATSH7DF` + `ATCRA7EA` + `010D` + `0146` prepended to
`data_commands`. Both have been verified to read cleanly from a
standalone Python+BLE probe on the same adapter and car.

### Multi-ECU header switching — known limitation in the OBD App layer

This profile polls three distinct ECUs each tick (broadcast `7DF` for
Mode 01 PIDs, `7A1/7A9` for BMS proprietary, `7E2/7EA` for HPCM
proprietary). On some ELM327 + integration combinations we have observed
`22F228` / `22F229` returning NO_DATA in the ABRP UI despite the same
queries working consistently from a standalone Python probe. The pattern
is:

- SoC (`22F1AF` on `7EA`) loads correctly.
- Odometer (`22F1AE` on `7EA`) loads correctly.
- Pack voltage / current (`22F228`/`22F229` on `7A9`) report NO_DATA.

Root cause appears to be that the OBD App plugin sends `Data_Commands`
once at session start, then per-tick only emits the `command` field and
an `ATCRA<ecu>` from each PID definition — it does **not** automatically
issue a fresh `ATSH<tx>` per PID. The last `ATSH` from the init / data
sequence wins, so requests for BMS DIDs go out with the HPCM (or
broadcast) header active and the BMS does not answer.

Mitigations, in increasing order of effort:

1. Verify that the ABRP UI accepts `Data_Commands` as a per-tick
   sequence (not a one-shot init). If it does, the order shown in the
   JSON here is sufficient.
2. If your build of the OBD App exposes a separate "TX header" field
   per PID, set it to `7A1` for `22F228` / `22F229` (and `7E2` for
   `22F1AE` / `22F1AF`).
3. As a last resort, override the `ecu` field with the TX header rather
   than the RX header (some integrations interpret `ecu` as TX). E.g.
   change `"ecu": "7A9"` to `"ecu": "7A1"` for the BMS DIDs.

Verified working on a Python+BLE control run (see the upstream
research repo) — the limitation is in the consuming integration, not
in the DIDs themselves.

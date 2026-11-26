# smart EQ fortwo / forfour (453, 2017–2024)

**Status: draft, not yet tested with ABRP.** The requests and formulas were verified with manual
requests on a smart EQ forfour (77,390 km) against the values of the
[OVMS](https://github.com/openvehicles/Open-Vehicle-Monitoring-System-3) smart EQ vehicle module
(`vehicle_smarteq`), which reads the same ECUs. The car uses 11 bit CAN IDs (protocol 6).

## ECUs

| ECU | Request ID | Response ID | Requests |
|---|---|---|---|
| BMS (LBC) | 79B | 7BB | `2104`, `2107`, `2161` (KWP style "read data by local identifier") |
| EVC | 7E4 | 7EC | `222002`, `22320C` |
| 743 (temperature sensors) | 743 | 763 | `22200C` |

## Values

Letters refer to the data bytes after the service and PID echo (`61 07`, `62 20 02`, ...).

| Value | Request | Formula | Check on the car | Comparison |
|---|---|---|---|---|
| `soc` | 7E4 `222002` | `0.02*(A*256+B)` | `13 23` → 98.0 % | see "SOC" below |
| `voltage` | 79B `2107` | `(I*256+J)/64` | `62 74` → 393.8 V | OVMS 391 V (frame 0x42E, read later) |
| `current` | 79B `2107` | `-1*(Signed(K)*256+L)/32` | `fe f1` → +8.5 A | OVMS +12.5 A, same sign (discharge positive) |
| `batt_temp` | 79B `2104` | `(Signed(A)*256+B+Signed(C)*256+D)/128` | `01 c0 01 40` → 6.0 °C | OVMS 6 °C (27 sensors 5…7 °C, average 5.7 °C) |
| `soh` | 79B `2161` | `I/2` | `c8` → 100 % | OVMS 100 % |
| `odometer` | 79B `2161` | `J*16777216+K*65536+L*256+M` | `00 01 2e 4e` → 77390 km | OVMS 77390.9 km (BMS mileage, whole km) |
| `capacity` | 7E4 `22320C` | `(A*256+B)*0.005` | `0c 80` → 16.0 kWh | OVMS 16.05 kWh, see open points |
| `ext_temp` | 743 `22200C` | `((C*256+D)-400)/10` | `01 85` → −1.1 °C | OVMS −0.8 °C (read later) |

Other details:

- `2107` byte M is the contactor state (`02` = closed), which confirms the byte offsets.
- `2104`: A,B is the maximum, C,D the minimum module temperature (unit 1/64 °C), followed by
  27 sensor values. The third value (E,F) is not the average, so the formula uses
  (max + min) / 2.
- `2161` G,H is the full capacity in units of 10 As (`47 6e` → 50.8 Ah).

## SOC

The dashboard SOC is only available in the broadcast frame 0x654 (byte 3, whole percent), which
cannot be read with a request. Measurements during a drive:

| Dashboard (0x654) | EVC `222002` | BMS `2108` kernel SOC (G,H / 16) | BMS `2108` real SOC min / max |
|---|---|---|---|
| 100 % | 98.00 % | 98.0 % | 93.6 / 95.4 % |
| 100 % | 97.44 % | 97.0 % | 93.5 / 95.25 % |
| 99 % | 96.96 % | 97.0 % | 93.06 / 95.0 % |

The dashboard changed together with the EVC value while the kernel SOC stayed the same, so the
dashboard follows the EVC value with a scale. The EVC value is used for `soc` (resolution 0.02 %).
`2125` (B,C / 16, 97.1 %) is another BMS SOC and is not used.

## Other apps

The ABRP live data page for the smart EQ forfour
(`https://abetterrouteplanner.com/resources/live-data/smart-eq?car=smart:eq:20:17:forfour`)
recommends Car Scanner. Requests sent by Car Scanner on the same car (counted at the adapter):

| ECU | Requests | Result |
|---|---|---|
| 79B (BMS) | `2104`, `2107`, `2108`, `2141`, `2142`, `220209` | answered |
| 7E4 (EVC) | `223024`, `223045`, `223307`, `223376`, `2233AE`, `223436`, `223438` | answered |
| 75A | `222004`, `227001`, `22700E`, `22700F`, `227010`, `227033` | answered |
| 740 (ABS) | `224B04` ("[ABS] Vehicle speed" in Car Scanner) | **no answer** on response ID 748 (28 of 28) |
| 7E4 | `2104` | NRC 12 |

No generic OBD2 requests (`01xx` to 7DF) were sent; the car does not seem to support them.

## Open points

- **SOC scale:** both "EVC / 0.97" and "EVC + 3" fit the three points near 100 %. Points at a
  low SOC are necessary to select one and add it to the formula.
- **`capacity`:** 16.0 kWh at 98 % SOC can be the usable capacity or the remaining energy.
  A request at a low SOC will show it.
- **`current` while charging:** must be negative; not yet tested.
- **`batt_temp` below 0 °C:** not yet tested.
- **Speed:** no working diagnostic DID known (OVMS reads the speed from a broadcast frame).
  Car Scanner uses ABS 740 `224B04`, which did not answer on 748; response ID 760 is not yet
  tested with the ignition on.
- **`is_charging`:** not available. `22339D` (EVC) only shows that the charging plug is connected.
- **`obdble_cars.json`:** the entry `smart:eq:*` uses the car ID of the ABRP live data page
  (`smart:eq:20:17:forfour`) and the `tester` group until the list is tested with ABRP.

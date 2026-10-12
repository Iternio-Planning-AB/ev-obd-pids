# Chery Q (QQ3 EV) OBD Data

Community-tested PIDs for the 2026 Chery Q EV (sold as QQ3 EV in China; Gotion LFP pack). Found in 2026 with a Bluetooth ELM327 adapter and a custom scanning tool that only sends standard read requests (service 0x22, plus 0x10 03 to open the extended diagnostic session), on protocol 6 (ISO 15765-4 CAN 11-bit 500k). Most fields come from the BMS (request 7E5, response 7ED); is_charging comes from the on-board charger / DC-DC unit (request 7E6, response 7EE); is_parked, ext_temp and hvac_power come from the VCU (request 7E0, response 7E8); tyre pressures come from the body module BDM (request 760, response 770).

Starting point was the Omoda E5 BMS DID list (github.com/sl3per/OmodaE5Mod); several of those DIDs carry over, the SoC block (22441E) does not.

| Key | DID | Status |
| --- | --- | --- |
| soc | 221038 | matched dash (parked, driving, AC, DC) |
| soh | 221048 | reads 100 on a new car |
| voltage | 224404 | cross-checked with 220001 and charger display |
| current | 224406 | + discharge / − charge; matches 220001 and AC/DC charge power |
| power | 220001 | V (bytes A,B ×0.02) × I (bytes AA,AB, offset 30000) |
| batt_temp | 22440E | average module temperature |
| odometer | 224402 | matched dash |
| speed | 224401 | matched dash |
| is_dcfc | 220001 | DC inlet voltage (bytes E,F ×0.02) > 100 V; matched charger display |
| is_charging | 7E6 224314 | on-board charger flag: 1 while AC charging, 0 unplugged and plugged-not-charging (Car Scanner, default session). Not yet checked on DC; is_dcfc covers DC |
| tire_pressure_fl/fr/rl/rr | 760 223404/3405/3406/3407 | A × 1.38 kPa; all four matched the dash TPMS screen per wheel (2.55/2.51/2.49/2.51 bar), default session |
| is_parked | 7E0 22520D | VCU gear byte: 1 P, 2 R, 3 N, 4 D; followed every shift in a P/R/N/D/P test. Only answers in the extended session, so `1003` is sent to 7E0 just before it |
| ext_temp | 7E0 226802 | A/2-55 °C; matched the dash outside temperature exactly at 14.5, 17 and 16.5 °C on three days, did not move with the A/C on, and followed the outside temperature as it rose. (226859 reads close but rises with the A/C, so it is not the ambient sensor) |
| hvac_power | 7E0 22684E | Byte M × 0.05 kW = A/C compressor load (0 with the A/C off; tracked the HV current step when the A/C switched on at several set temperatures). Covers the compressor only, not the cabin heater. Extended session (same `1003` as is_parked) |

## Not found yet (help wanted)
- **soe, est_battery_range, capacity, cabin_temp, hvac_setpoint**: not found.
- **kwh_charged**: not mapped yet. 22FD0A bytes A-D (UInt32 mAh) are a lifetime charge counter, not a per-session one (it kept climbing across separate charges), and bytes E-H count lifetime discharge. Another UInt32 at data bytes 78-81 rises with charging but also counts from car-on, so it is not used.

## Notes for contributors
- Modules answering 11-bit physical requests: 7E0 VCU, 7E4 MCU, 7E5 BMS, 7E6 charger/DC-DC (OBC). 7E7 answers but returns NRC 0x11 to everything tried.
- Body modules (IPB 720, EPS 724, e-shifter 72B, FCM 72E, BDM 760, liftgate 76C, head unit 788, climate panel 789, ...) reply at request + 0x10 (760 -> 770), so they need `ATCRA` and only answer with the car on. They are slow (often `7F 22 78` response-pending first, about 0.6 s per read), hence `ATSTFF` around the BDM reads; `ATAR` / `ATST32` restore the defaults for the next loop. Some cheap ELM327 clones never hear these modules at all.
- BDM 760 also has: 223400-223403 TPMS sensor IDs, 223408-22340B tyre temperatures (likely A-50 °C, same wheel order), 22720F real-time clock (YY MM DD hh mm ss).
- 220001 and 220002 are long (110/130-byte) BMS blocks. 22442F is a cell-voltage list (259-byte reply, UInt16 mV per cell, cell 1 = bytes A,B) but only fills cells 1-125; 220013 (853 bytes) lists all 126 cells the same way. 220014 is a summary: max/min cell mV and cell numbers, max/min temperature (float32) and sensor numbers, average cell mV, then the 12 module temperatures as float32 °C. 220009-220012 are per-balancing-chip blocks (raw cell channels, NTC raw values, module voltage). Cheap BLE ELM327 clones return only the first frame of replies over 255 bytes; use STmin 20 (ATFCSD300014) and an adapter that handles long ISO-TP replies.
- VCU (7E0) live data sits in 225200-22524A and 226800-226806, extended session only. Besides the gear byte (520D): 5212 = HV voltage ((A*256+B)*0.05 V), 5213 = HV current ((A*256+B-32000)*0.1 A, + discharge); both looked right in one test, not yet cross-checked.
- 22FD0A (223 bytes, also answers in the default session) bytes 162-163 (UInt16) count down the minutes to full while charging (matched the dash, about 6 h on a ~2.9 kW AC charge).
- On the charger module, 224304 = AC input current (A), 224303 = DC output current ((Int16(A,B)-30000)/100, A; reads 0x0200 when idle), 22431C = HV voltage (/10). 224314 is the charging flag used here.
- Pack: Gotion LFP, 126 cells in series (220020 lists cells per balancing chip: 13, 13, 12, 13, 12, 12, 13, 12, 13, 13), 105 Ah cells, 41.278 kWh rated (126 × 105 Ah × 3.12 V, matching the owner's spec sheet). The pack code in 22104D follows GB/T 34014 ('03H' = Gotion, 'P' = pack, 'B' = LFP). No usable-energy figure found.

## Licenses
This data is licensed by Apache 2.0

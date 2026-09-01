# Mercedes Sprinter (2024, W907/VS30) — CAN Bus Codes

Reverse-engineered CAN bus signals for the **Mercedes-Benz Sprinter, model year 2024 (W907/VS30 platform)**,
captured passively from the **OBD2 diagnostic port, pins 6 (CAN-H) and 14 (CAN-L)**.

This data was gathered while building [CANBUSSY](https://github.com/Electricmats), a CAN bus interface for a
camper van conversion. It's shared here so other people working on Sprinter camper conversions, telematics,
or home automation integrations don't have to re-discover the same frame IDs from scratch.

## Bus details

| Property | Value |
|---|---|
| Vehicle | Mercedes-Benz Sprinter W907/VS30, model year 2024 |
| Access point | OBD2 port, pin 6 (CAN-H) / pin 14 (CAN-L) |
| Bus | Powertrain CAN (gateway-filtered) |
| Baud rate | 500 kbps |
| Mode used for capture | **Listen-only** (no frames written to the bus) |

⚠️ **This is the gateway-filtered powertrain bus, not the full vehicle bus.** A lot of body/comfort signals
(door-open-per-door, indicator left/right, ignition/voltage) are **not** broadcast here — they live on a
separate body CAN (CAN-B), which is a different physical pair with a different (likely 125 kbps) baud rate.
See [CAN_SIGNALS.md](./CAN_SIGNALS.md) for details per signal, including what was tried and ruled out.

## What's confirmed vs. what's a guess

Every signal below has a confidence note. Some are hardware-confirmed over multiple test sessions, some are
still a hypothesis. **Do not treat "unconfirmed" entries as reliable** — they're documented so nobody else has
to re-run the same dead-end capture.

| Signal | Frame ID | Status |
|---|---|---|
| Engine running | `0x077` | ✅ Confirmed on hardware (activity-based, not value-based) |
| Door lock / unlock | `0x307` | ✅ Confirmed on hardware |
| High beam | `0x33D` | ✅ Confirmed on hardware |
| Daytime running light (DRL) | `0x33D` | ✅ Confirmed on hardware |
| Accelerator pedal position | `0x03D` | ✅ Confirmed, linear range |
| Fuel tank level | `0x3C0` | ✅ Stable/confirmed, scale not fully calibrated |
| Door open (count only, no per-door) | `0x15A` | ⚠️ Partially reliable, see notes |
| Indicators (active/inactive only, no L/R) | multiple (`0x096`, `0x351`, `0x209`, `0x15A`) | ⚠️ "blinking" only, no direction |
| Outside temperature | `0x3EF` | ⚠️ Unconfirmed, conflicting data |
| Battery / system voltage | — | ❌ Not found on this bus (not broadcast) |
| Coolant temperature | — | ❌ Not found |
| Engine RPM | — | ❌ Not found |
| Per-door open status (L/R) | — | ❌ Not on this bus, needs body CAN (CAN-B) tap |

Full write-up, including bit-level detail, rejected hypotheses, and the reasoning behind each conclusion, is in
[CAN_SIGNALS.md](./CAN_SIGNALS.md). A machine-readable [`sprinter_w907_2024.dbc`](./sprinter_w907_2024.dbc) file
covers the hardware-confirmed signals.

## Safety note

All of this was captured in **listen-only mode**. If you're doing your own sniffing on a vehicle you drive:
never write/query frames on the powertrain CAN while driving, and treat anything you send to the bus as
potentially affecting vehicle systems. This repo does not encourage or provide guidance for transmitting on
the bus.

## Contributing

If you've confirmed, refuted, or extended any of these signals on your own 2024 Sprinter (or a closely related
platform like the Crafter/VS30), please open an issue or PR. Include:
- how you captured it (tool, listen-only or not),
- how many independent test sessions confirmed it,
- raw example frames if possible.

## License

Data and documentation in this repository are released under [CC0 1.0](./LICENSE) (public domain) — use it
however you like, attribution appreciated but not required.

# CAN Signals — Mercedes Sprinter 2024 (W907/VS30), OBD2 pin 6/14

All captures were done in **listen-only mode** on the powertrain CAN accessible via OBD2 pins 6 (CAN-H) /
14 (CAN-L), 500 kbps. Frame IDs are 11-bit standard IDs shown in hex (`0x...`).

Confidence markers used throughout:
- ✅ Confirmed on hardware over multiple independent test sessions
- ⚠️ Uncertain / partially confirmed / conflicting evidence
- ❌ Not found on this bus / ruled out

---

## Engine running — `0x077` ✅

Detected by **activity**, not by value.

- `0x077` changes only while the engine is running (1492 changes observed during a drive, zero changes
  outside of engine-running periods across ~44,000 log lines).
- The byte *values* are not usable as a threshold: bytes 2+3 range `0x0900`–`0x0CFF`, and the "off" sentinel
  `0x0BB8` (3000) also occurs 67× while the engine is running — any fixed threshold flickers.
- **Detection logic:** engine considered running if `0x077` sent a changed (non-rolling) byte within the last
  1500 ms; otherwise considered off.
- Confirmed reliable on hardware, including under throttle (no false "off" transitions).

Rejected approaches (kept for reference, do not reuse):
- `0x087` timeout — frame stays present even with engine off.
- `0x0B1` byte 4 — oscillates, not a stable engine-state signal.
- `0x0B1` byte 6 — looked stable at idle, but flips `0x08`→`0x07` under throttle while engine keeps running
  (false "off"). Reflects idle/load mode, not on/off.
- `0x077` with threshold `> 0x0400` — wrong direction; the "off" sentinel (3000) is *above* idle (~2450).
- `0x03D` — turned out to be the accelerator pedal (see below), not engine state.

Engine RPM itself was **not found** — `0x077` bytes 2+3 barely move with throttle (2400–2740 across the full
pedal range), so it's unusable as RPM.

---

## Accelerator pedal position — `0x03D` ✅

- Bytes 2 and 4: pedal position (always equal to each other); byte 3 leads slightly.
- Range: `0x00` (released) → ~`0xFA` (full throttle), linear.
- Present on ignition, not just engine-running — **not** usable as an engine-state signal.
- Frame is absent entirely when the pedal isn't touched.

---

## Door lock / unlock — `0x307` ✅

- Byte 2: `0x11` = locked, `0x22` = unlocked, `0x00` = idle (ignore).
- Byte 0: `0x07` when active, `0x1F` when idle.
- Frame repeats ~10× per action.
- Confirmed across multiple lock/unlock cycles.

---

## High beam — `0x33D` ✅

- Byte 4: `0x03` = on, `0x00` = off.
- Byte 7: `0x18` = on, `0x10` = off (bit 3 toggles).
- Bytes 0 and 1 are rolling counters — ignore.
- Confirmed across 4 on/off cycles in one log.

### High-beam gesture switch (project-specific, not a vehicle signal)

Not a Sprinter signal — a feature built on top of the high-beam frame: flash high beam twice quickly, then
hold once, to toggle an auxiliary output without a physical switch. Documented here only because it reuses the
`0x33D` byte-4 edge detection above. See this repo's companion CANBUSSY firmware if you want the implementation.

---

## Daytime running light (DRL) — `0x33D` (same frame as high beam) ✅

Byte 2 is a bitmask of light state:

| Value | Meaning |
|---|---|
| `0x00` | off |
| `0x1F` | DRL via the light switch |
| `0x7F` | DRL "welcome light" mode (`0x1F` + extra bits `0x60`) |

Detection: `(byte2 & 0x1F) == 0x1F` → DRL is on, for **either** switch-activated or welcome-light DRL (both
have the low 5 bits set). Testing for exact equality with `0x1F` misses the welcome-light case.

Confirmed on hardware: an LED wired to this logic follows the physical light switch 1:1.

### DRL "welcome" mode (unlock-triggered)

After unlocking with the key, DRL turns on as a welcome/courtesy light and stays on for ~30 s (delayed off),
even with ignition off.

- Welcome mode: `byte2 = 0x7F`, `byte3 = 0x0C`, `byte7 = 0x12` (bit 1 set).
- Switch-activated DRL: `byte2 = 0x1F`, `byte3 = 0x00`, `byte7 = 0x10`.

There is no separate "welcome mode" output in this dataset — it's folded into the general DRL-on detection
above via the low bits of byte 2. If you want to detect "someone just unlocked the vehicle" specifically as
its own event, key off `byte2 == 0x7F` (or `byte2 & 0x40`) instead.

Rejected candidates:
- Byte 6 `0x40` — initially suspected as a DRL flag; ruled out with a controlled test (DRL toggled on/off with
  engine off, byte 6 stayed `0x00` throughout). It correlates with engine running, not DRL.
- Byte 5 — inconsistent between stationary and driving captures (contradicted itself), not used.

---

## Fuel tank level — `0x3C0` ✅ (stable, scale not fully calibrated)

- Byte 1 **and** byte 3: percentage, decimal (e.g. `0x0D` = 13%). Both bytes agreed and were stable across an
  entire session.
- Byte 2 varies (`0xC8`–`0xCE`) — a different signal, not fuel level.
- Byte 4 is a fast rolling counter — ignore.
- Cross-checked against OBD2 DID `0x5050` via response ID `0x7E8`: value `0x0B` (11) ≈ 11.7 L, consistent with
  a percentage-of-tank reading.
- Not yet confirmed across a refuel event (expected to rise to roughly `0x16`–`0x1A` after +10 L, not yet
  tested).

---

## Door open/closed — `0x15A` ⚠️ partially reliable, count only, no per-door

- Only trust an event when byte 4 = `0x55` (a real event). At baseline (`byte4 = 0xFF`), byte 6 contains noise
  (toggles between `0x41`/`0x51`/`0x11`) — ignore those.
- When `byte4 == 0x55`, byte 6 was originally read as an absolute open-door count:
  - `0x00` = 0 doors open
  - `0x40` = 1 door open
  - `0x50` = 2 doors open simultaneously
- **However**, a later, cleaner capture contradicted this: every door action (driver/passenger/slider, open or
  close) produced the *same* event `byte4=0x55, byte6=0x20`, with no distinction by count or by which door —
  not even open vs. closed. So `0x15A` is at best an "something happened to a door" trigger, not a reliable
  open/closed state, on this bus.
- Practical usage in the source project: track only "at least one door not fully closed" as a sticky binary
  flag (remember the last non-baseline reading), since individual events can be missed.
- A previously suspected root cause for missed events: serial-port backpressure overflowing the CAN RX queue
  on the capturing microcontroller — not a Sprinter/CAN issue, a capture-hardware issue. Worth ruling out
  before trusting any "still missing events" conclusion on your own setup.

### Per-door status (which door) — not available on this bus

Three controlled tests (driver-only, passenger-only, mixed, ignition on) all produced identical `0x15A`
behavior regardless of which door was opened — no frame distinguishes driver vs. passenger. `0x3C2` (initially
suspected as a driver-door-specific frame) did not appear in any of these tests.

Per-door open status appears to live on the **body/comfort CAN (CAN-B)**, which the OBD2 gateway does not
forward to pins 6/14. Corroborated by:
- Mid City Engineering (a commercial CAN alarm interface for the W907) documents tapping door triggers and
  lock/unlock via a separate brown / brown-red twisted pair at the SAM module / fuse box, passenger side —
  not via OBD2.
- General Mercedes documentation pattern: body signals (ED5/PSM module) are read via body CAN, not the
  diagnostic bus.

If you want real per-door status, you likely need a second tap on that brown/brown-red pair (CAN-B), with a
baud rate scan (try 125/250/500 kbps — body CAN is often 125 kbps, not 500). This was **not** attempted in
this project. Stay listen-only.

---

## Indicators / turn signals — ⚠️ "active" only, no left/right

During left/right/hazard activation, several frames pulse in sync with the blink rate:
`0x096` byte 3 (`0x06`↔`0x00`), `0x351` byte 4 (`0x0F`↔`0x5F`) / byte 5, `0x209` byte 3/6, `0x15A` byte 2.

All of these are **identical** for left, right, and hazard — they indicate "something is blinking," not which
side. No byte reliably splits left vs. right; every left/right candidate tested turned out to be noise or a
blink-phase artifact.

Left/right distinction likely lives on the body CAN, same as per-door status (see above) — not on pin 6/14.

A community source (opendbc) documents `BH_Blinker_li`/`BH_Blinker_re` signals, but those are for the VW MQB
platform (Golf/Passat) comfort CAN — a different platform and bus, **not** applicable to the Sprinter
907/VS30 (or the Crafter, which isn't MQB either).

If you want real L/R indicator state: tap CAN-B at the SAM module (see door section above) and scan baud
rates. On pin 6/14, "indicator active" (no direction) is the practical ceiling.

---

## Outside temperature — `0x3EF` ⚠️ unconfirmed, do not rely on this

- Byte 4: hypothesized as direct Celsius, no offset (e.g. `0x10` = 16 °C, `0x11` = 17 °C). Stable within a
  session and consistent across two sessions (16 °C and 17 °C).
- **Contradicted by a later capture**: at a known outside temperature of 16.5 °C, byte 4 read `0x14` (= 20),
  not 16/17. The stable byte pattern observed (`EA 07 06 0D 14`) doesn't encode 16.5 in any obvious way.
- Also, a genuinely stable temperature barely changes, which makes it hard to isolate in a changes-only
  capture — the original identification may simply have been coincidence.
- Byte 5 also varies between sessions (possibly dew point or a second sensor) — unclear.
- **Do not use this without re-testing** across clearly different outside temperatures, or confirming via an
  OBD2 query instead.

---

## Battery / system voltage — ❌ not found (not broadcast on this bus)

- No byte or 2-byte value moved reliably from ~12.6 V → 14.2 V at engine start across any capture.
- A variance-based search (values stable within each of engine-off/engine-on, but different between them)
  only produced noise candidates (`0x094` byte 7, `0x341` byte 4, `0x351` byte 6) — none plausible as voltage.
- **Conclusion**: system voltage is not broadcast on this gateway-filtered powertrain bus at all.
- Alternatives, not implemented here:
  - OBD-II PID `01 42` (Control Module Voltage) — requires an active *query* (writing to the bus), which
    conflicts with a listen-only safety policy. Do not do this while driving.
  - A separate ADC voltage divider on the battery, outside of CAN entirely (recommended — also removes any
    need for the `0x077` engine-activity signal as a stand-in for charge state).
- Corroborated by a third-party finding: the Mid City 907 interface also does not read ignition status from
  CAN, but from a direct wire to the fuse box — confirming voltage/ignition isn't broadcast on this bus.

## Coolant temperature — ❌ not found

`0x341` byte 3 dropped quickly during a short engine run — too fast for coolant temp, likely something else
(RPM-derived, oil pressure?). Not investigated further.

## Movement / vehicle speed — ❌ not investigated

Not captured/analyzed in this project.

---

## Summary table

| Signal | Frame ID | Byte(s) | Notes | Status |
|---|---|---|---|---|
| Engine running | `0x077` | activity-based | see detection logic above | ✅ |
| Accelerator pedal | `0x03D` | 2, 4 (3 leads) | `0x00`–`0xFA` linear | ✅ |
| Lock/unlock | `0x307` | 2 (`0x11`/`0x22`), 0 | repeats ~10× per action | ✅ |
| High beam | `0x33D` | 4 (`0x03`/`0x00`), 7 (bit 3) | | ✅ |
| DRL | `0x33D` | 2, bitmask `& 0x1F` | `0x7F` = welcome mode | ✅ |
| Fuel level | `0x3C0` | 1 and 3 | % direct, decimal | ✅ (scale to verify) |
| Door open (count) | `0x15A` | 4 (`0x55` = event), 6 | no per-door, event-based | ⚠️ |
| Indicator active | `0x096`/`0x351`/`0x209`/`0x15A` | various | no L/R distinction | ⚠️ |
| Outside temp | `0x3EF` | 4 (hypothesized) | conflicting evidence | ⚠️ |
| Voltage | — | — | not broadcast | ❌ |
| Coolant temp | — | — | not identified | ❌ |
| RPM | — | — | not identified | ❌ |
| Per-door status, L/R indicator | — | — | needs body CAN (CAN-B) tap | ❌ |

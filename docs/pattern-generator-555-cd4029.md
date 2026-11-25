# 0 / 15 V pattern generator: NE555 + CD4029B

A "fake 4004" for the bench: four lines counting 0 to 15 at real 0 / 15 V levels, with rail-to-rail CMOS edges and no overshoot. Built entirely from the shelf. Replaces the DDS generator for Part D step D4 of `stage0-bench-plan.md`, and can be built before the Mouser parts arrive.

Sources: TI CD4029B datasheet SCHS034C (`datasheets/CD4029B-TI-SCHS034C.pdf`) [DS]; NE555 standard pinout and astable formula. [CALC] is my arithmetic.

## Why it works

- CD4029B at VDD 15 V: outputs 0.05 V max low and 14.95 V min high, specified at 15 V [DS p.3-85]. Sink and source 3.4 mA min (6.8 typ) at 15 V. Edges tens of ns, no overshoot.
- Its output resistance at 15 V is a few hundred ohms, about the same as a 4004 data line (ROH 150 to 250 Ω).
- Supply range 3 to 18 V, absolute maximum 20 V [DS]. 15 V is well inside. Make sure the part is a **CD4029B** (B series); old A-series 4000 parts are only good to about 15 V.
- NE555 runs to 16 V (18 V absolute). Its output high is about VCC − 1.7 V, so about 13.3 V at 15 V. The CD4029B clock input needs 11 V min for a high at VDD 15 V [DS], so the 555 drives it directly.

## Schematic

```
                +15 V ──┬──────────────┬─────────────────────────┐
                        │              │                         │
                       R1 1k          ─┴─ 0.1 µF + 10 µF          │
                        │              │  (at 555 pin 8)          │
          ┌─────────────┼── pin 7 DIS  │                         │
          │            R2 6.8k        GND                  CD4029B pin 16 VDD
          │             │                                  0.1 µF at pin 16
          │   ┌─────────┼── pin 6 THR                            │
          │   │         └── pin 2 TRIG                           │
          │   │         │                                        │
          │   │        C 1 nF (≈99 kHz)  or 220 pF (≈450 kHz)    │
          │   │         │                                        │
          │   │        GND                                       │
NE555: pin 1 GND, pin 4 RESET → +15 V, pin 5 CTRL → 10 nF → GND
       pin 3 OUT ───────────────────────────────→ CD4029B pin 15 CLOCK

CD4029B (count up, binary, free running):
  pin 1  PRESET ENABLE → GND   (low = count, no preset)
  pin 3, 4, 12, 13  JAM 4, 1, 2, 3 → GND
  pin 5  CARRY IN     → GND   (datasheet: must go to VSS when unused; low enables counting)
  pin 9  BINARY/DECADE → +15 V (high = binary, 0 to 15)
  pin 10 UP/DOWN      → +15 V (high = up; ground it to count down)
  pin 8  VSS → GND

Outputs, each through 1k in series to its "pin node":
  pin 6  Q1  (LSB, ÷2 of the clock)   → "D0"
  pin 11 Q2  (÷4)                     → "D1"
  pin 14 Q3  (÷8)                     → "D2"
  pin 2  Q4  (MSB, ÷16)               → "D3"
  pin 7  CARRY OUT  → scope trigger (one pulse per 16 counts)
```

All pin numbers are from the CD4029B terminal diagram [DS p.3-84].

## Preferred clock: the 4004 clock board instead of the 555

The TTL clock board in `clock-generator.md` already makes the 4004's real clock. Use its /φ2 (74LS00 pin 6) as the CD4029B clock:

```
clock board /φ2, 5 V TTL ──→ one DM7407 gate (open collector, 30 V rated)
                              4.7k pull-up from its output to +15 V ──→ CD4029B pin 15 CLOCK
```

- Rate: one step per 4004 clock cycle, 511 kHz (1.956 µs). Q1 toggles at 256 kHz, Q4 at 32 kHz.
- The CD4029B counts on the rising edge of its clock. /φ2 rises at the φ2 trailing edge, which is the edge on which a real 4004 clocks its outputs. The pattern changes when a 4004's outputs would.
- The 7407's rising edge through 4.7k is about 100 ns; the CD4029B accepts clock edges up to 15 µs at 15 V [DS]. Clock input high needs 11 V min; the pull-up gives 15 V.
- Once the CD4504B is on the bench, its φ2 output (0 / 15 V, already level-shifted) can clock the CD4029B directly and the 7407 goes away.

The 555 below remains the stand-alone option when the clock board is not built yet.

## Values

555 astable: f = 1.44 / ((R1 + 2·R2)·C) [CALC].

| C | R1 | R2 | Clock | Q1 | Q4 | Use |
|---|---|---|---|---|---|---|
| 1 nF | 1k | 6.8k | ≈ 99 kHz | 49 kHz | 6.2 kHz | easy to watch, start here |
| 220 pF | 1k | 6.8k | ≈ 450 kHz | 225 kHz | 28 kHz | near the 4004's 511 kHz subcycle rate |

Duty is (R1 + R2)/(R1 + 2·R2) ≈ 53 %; it does not matter, the counter only uses the rising edge. A 10k pot in series with R2 makes the rate adjustable. The CD4029B is good to 5.5 MHz at 15 V [DS], so either rate is easy for it; the bipolar 555 is near its own limit at 450 kHz, fine for this.

Decoupling: 0.1 µF ceramic plus 10 µF at the 555 (it pulls current spikes on every edge), 0.1 µF at CD4029B pin 16. A 10 µF tantalum is not for the 15 V rail; use an electrolytic rated 25 V or more there, or two of the 1 µF X7R ceramics.

## Power and connection rules

- **Its +15 V must be the same node as the interface's 15 V** (the 4004's VSS node). Never a separate supply. A second rail could put a "data" line above VSS, and with the board's 15 V off, the CD4029 outputs would back-feed the TP0606 body diodes through the 1k resistors.
- Power it on and off together with the board's 15 V.
- On the bench with no board, any 15 V bench supply is fine.

## Bring-up, before it touches the breadboard

1. Power from the bench supply at 15 V, current limit 30 mA. Expect about 5 to 10 mA, almost all of it the 555.
2. Scope 555 pin 3: square at the expected rate, high about 13.3 V.
3. Scope Q1 to Q4 directly (no load): 0 and 15 V plateaus, Q1 at half the clock, each next output at half the previous. Measure Max and Min with the ground spring: want ≤ 15.3 V and ≥ −0.3 V.
4. Trigger on CARRY OUT (pin 7) and display Q1 to Q4 together: the binary count 0 to 15 repeating.
5. Into a 1k series plus 12k to ground (the pin-node load): highs about 13.8 V, edges a few tens of ns, no overshoot.

## Patterns

- Count up: as wired.
- Count down: UP/DOWN (pin 10) to GND.
- BCD 0 to 9: BINARY/DECADE (pin 9) to GND.
- Hold a value: CARRY IN (pin 5) to +15 V stops the count; JAM inputs plus a high pulse on PRESET ENABLE load any nibble.
- Walking one on 16 lines: add an MC14515BCP (4-to-16 decoder, on the shelf) after Q1 to Q4.

## Use in the bench plan

Part D, step D4: the four outputs, each through its 1k, go to the four pin nodes in place of the generator. Bus released (BUS_OE = 0). Every receiver should follow its line; the host side reads the count. D5 (deliberate contention) works the same way with one output against the driver.

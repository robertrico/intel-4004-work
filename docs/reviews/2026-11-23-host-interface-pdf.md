# Review of "Intel 4004 host interface" (4004Schematic.pdf), parts on hand only

Constraint: build only from what is already here. Mouser order of 2026-11-13 (BS170 x10, TP0606 x6, CD4504B x2, SN74AHCT541N x2, LM2901N x3, LM319N x1, TMR 1-0513, zeners, caps, sockets) plus the shelf (74LS/74S/7400 logic, 74HC14, CD4049AE, 7406/7407, LM339, 9602, cans, resistors, 10 uF tantalums).

## Verdict

Sound design; keep its architecture. Four changes: three swap parts we do not have for parts we do, one fixes a timing error. No new order needed.

## 1. Replace 74AHCT04 / 74AHCT08 / 74AHCT32 (not on hand)

Use the two ordered SN74AHCT541N as level restorers and 74LS00 for the logic.

- AHCT541 #1, host side: D_OUT0-3, BUS_OE, RUN, TEST in (10k pull-downs on its inputs, unchanged), 1 spare. 3.3 V in, full 5 V out.
- Per data line, one 74LS00 (four packages, all gates used):
  - /D = NAND(D, D)
  - n1 = NAND(BUS_OE, D);  low = NAND(n1, n1) = BUS_OE AND D
  - hs = NAND(BUS_OE, /D) -> CD4504B input (TTL mode, LS level is fine)
- AHCT541 #2: the four "low" signals in, full 5 V out to the BS170 gates (an LS high of 2.7 V min does not clear a 3.0 V worst-case BS170 threshold). Plus the RESET restore below. 3 spare.
- Truth table unchanged: BUS_OE=0 released; BUS_OE=1, D=1 -> BS170 on, pin 0 V; BUS_OE=1, D=0 -> P-FET on, pin 15 V. Both on is impossible.
- Rejected shortcut: hs = NAND(BUS_OE, n1) saves a gate per line but glitches low for one gate delay when BUS_OE rises with D=1. The four-gate form has no race.
- Use LS, not standard 7400, for these: BUS_OE fans out to eight inputs; 8 x 0.4 mA LS = 3.2 mA is inside the AHCT541's 8 mA, 8 x 1.6 mA standard TTL is not.

## 2. RESET and TEST without 74AHCT04

- TEST: host -> AHCT541 #1 -> BS170 gate directly. Non-inverting already gives the right sense (host 1 -> pin 0 V -> TEST logic 1); default low via the pull-down keeps TEST inactive.
- RESET: RUN -> AHCT541 #1 -> spare fourth gate of the clock's 74LS00 (NAND as inverter) -> AHCT541 #2 -> 1k -> BS170 gate; push button from gate to +5 V unchanged. RUN default low holds reset.

## 3. Replace 74LVC2G34 (surface-mount only, not on hand)

/φ1, /φ2 from the clock 74LS00 -> 1k series -> 74HC14 powered from the host's 3.3 V -> host PH1, PH2.

- 74HC14 VT+ at 3.3 V is about 2.4 V max; LS VOH min 2.7 V clears it. The 1k limits clamp current if an LS high exceeds 3.8 V.
- The HC14 inverts: the host sees φ1 and φ2 active-high. Invert in firmware or rename the signals.
- Not a resistor divider: worst-case LS high divided lands under the host's 2.0 V threshold.

## 4. Bus release timing (firmware, no parts)

PH1/PH2 are taken from the TTL side, ahead of the CD4504B. The real pin edges are 140 to 280 ns later. The P-FET release also goes through the CD4504B, so the two delays cancel and the high side releases at about the pin's φ2 trailing edge; the 12k pull-down then crosses 13.5 V in about 40 ns. Total hold about 40 ns plus host latency, against the 150 ns bus hold the 4004 needs after M2 and X2.

Fix: release BUS_OE about 200 ns after PH2 rises, not on the edge. Hold becomes about 240 ns; the P-FET is still off about 200 ns before X1's φ1 (tφD2 449 ns). Low-driven lines are unaffected, the 12k holds them.

## 5. Drawing and text corrections

- Data line figure: "-> LM2901 IN-" is drawn at the TP0606 drain, above its 680. It must connect at the 4004 pin node, below both 680s, or the receiver reads the driver instead of the pin and the loopback proves nothing.
- Sampling margin: worst case is about 250 ns (1956 - 280 - 930 - 500), not 0.5 us. Still positive.
- 47k pull-downs on SYNC and CM lines: fine for the 11.5 V threshold (highs about 14.4 V); falling edges depend more on the 4004's own sink. Check on the scope.

## Parts, all on hand

| Part | Qty | Source | Job |
|---|---|---|---|
| SN74S163 | 1 | shelf | clock ÷4 |
| DM74LS163A | 1 | shelf | clock mod 3 |
| DM74LS74A | 1 | shelf | Johnson counter |
| SN74LS00 | 1 | shelf | φ decode, mod-3 clear, RESET inverter |
| SN74LS00 | 4 | shelf (confirm count) | data line gating |
| 74HC14 | 1 | shelf | PH1/PH2 to 3.3 V |
| SN74AHCT541N | 2 | ordered | host buffers, BS170 gate restore |
| CD4504B | 1 (+1 spare) | ordered | φ1, φ2, four P-FET gates |
| BS170 | 6 | ordered (10) | D0-D3 low side, RESET, TEST |
| TP0606N3 | 4 | ordered (6) | D0-D3 high side |
| LM2901N | 3 | ordered | 10 receivers |
| Resistors | | shelf | as the PDF, plus 1k x2 for the HC14 inputs |

No 74AHCT04, 74AHCT08, 74AHCT32 or 74LVC2G34 required.

## Response to the reviewer's reply (same day)

- **Item 4, release delay: reviewer is right, withdrawn.** My hold estimate treated the P-FET as off when the CD4504B output switches. It turns off only when its gate climbs past about VSS − 2.4 V (TP0606 VGS(th) max), i.e. the last ~15 % of the swing, and the CD4504B is charging 80 to 150 pF of Ciss plus Miller charge with a few mA near the rail (IOH −3.4 mA min, −6.8 typ at 15 V). 1 to 2 nC at those currents is 150 to 300 ns. That sits on top of the 40 ns pull-down crossing, so hold is probably met with no firmware delay, and adding 200 ns moves release toward X1's φ1, the side with no data. The "delays cancel" argument also assumed channel tracking the CD4504B does not specify. Adopted: release on the PH2 edge; release delay is a firmware parameter, 0 to 300 ns; set it from one scope capture of a high-driven line across M2 to X1, probing the pin, the TP0606 gate and φ2 at the pin together. First test is safe either way (overlap limited by the 680s).
- **Item 5a, comparator tap: accepted.** The tap is on the pin node; the drawing misleads. Redraw so the tap leaves the node clear of the upper 680.
- **100 µF ECA-1CM101 (16 V):** in the plan these sit on the 5 V input side of the module, not on 15 V, so 16 V is fine there. Agreed they must never go on the 15 V rail.
- **VSS decoupling, new point from this check:** the PDF puts 0.1 µF + 10 µF at 4004 pin 5. If the 10 µF is a tantalum, a 15 V low-impedance rail wants a 35 V part (usual 50 % derating), 25 V minimum. Simpler: use the ordered 1 µF 50 V X7R ceramics (two in parallel) plus 0.1 µF at the pin, and keep tantalums off the 15 V rail.
- **LS00 count:** 5 needed (1 clock + 4 data). to confirm on the shelf.

## Second reply (same day)

- **Turn-off tail and time range: accepted.** |VGS(th)| spans 1.0 to 2.4 V, so a low-threshold TP0606 is not off until its gate is within 1.0 V of the rail, where the CD4504B sources least. 2 nC at 3.4 mA is about 600 ns. Expect 150 to 600 ns on the scope; firmware release delay stays 0 to 300 ns, default 0.
- **Bulk at the module output: accepted.** The TMR 1 datasheet gives only a maximum (680 µF on a 15 V single output), no recommendation. Plan: 47 to 100 µF electrolytic, 25 V or more, at the module output. Not in the order; from the shelf if one is there. 0.1 µF parts are shelf, as noted.
- **Power budget: accepted, with two additions.** 4004 output current into its own pull-downs is outside IDD: SYNC, CM-ROM and four CM-RAM high into 47k is about 6 × 0.32 = 1.9 mA. Clock pin charging at 50 pF per pin (pin, socket, probe) is 2 × 50 pF × 15 V × 511 kHz ≈ 0.8 mA, not 0.3. Revised worst case about 59 mA, 89 % of the TMR 1-0513's 67 mA. Within rating (current limit 80 mA min); Stage 0 test A3 already loads the module to 60 mA and checks regulation, so the confirmation is in the plan.
- **Splitting the 15 V loads across two modules: disagree.** Every 15 V reference must be the same node as the 4004's VSS. RESET and TEST pull-ups on another module could sit above VSS (the ROE-0515S is unregulated and drifts high at light load) and exceed the +0.3 V input limit. A reference divider on the ROE would move the comparator threshold with the ROE's drift: 11.54 × 17 / 15 ≈ 13.1 V, too close to the 14.4 V CM-line high. The ROE stays a measurement reference only. If margin is wanted, reduce load on the one node: RESET and TEST pull-ups 10k → 22k (static lines; saves about 1.6 mA).
- **New consequence of the budget:** first chip power-up was planned on the bench supply with a 60 mA limit. With the whole interface on that rail, the empty board alone draws roughly 15 to 20 mA. Set the limit to the measured empty-socket draw plus 45 mA, not a flat 60.
- **Zeners: purpose confirmed.** The 1N4745A is an absolute-maximum clamp (4004 limit 20 V across the supply), not an operating-window clamp; the TMR's ±1 % does the regulating. One caveat: a 16 V −5 % part (15.2 V) has an unspecified knee between its 12.2 V leakage spec point and 15.2 V and may draw current at 15.0 V. Stage 0 test A6 already checks the rail is unchanged within 10 mV with the zener fitted; if it is not, use a 1N4746A (18 V).

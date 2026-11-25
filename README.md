# intel-4004-work

Work toward running a real Intel 4004 safely, and eventually building a standalone 4004 calculator with no FPGA in it.

The chips are on loan: two 1970s boards, an HP MCS-4 Exerciser (COS-2) and a Pro-Log PWM 100674XA, each carrying a white ceramic C4004 with its 4001/4002/4008/4009/1702A companions. They are scarce and effectively irreplaceable, so chip safety outranks convenience everywhere in this repo. Nothing with a chip in it gets power until the empty-socket bench plan passes.

## Plan

1. **Bench harness.** A 3.3 V host (ECP5 FPGA first, Pico 2 as the alternative) drives and reads a 4004 through discrete level shifting. Rails VSS +15 V, VDD ground. Drive side: CD4504B for the two clocks and for the gates of TP0606 P-FETs that make a tri-state push-pull on the four bus lines, BS170 open drain for TEST and RESET. Receive side: LM2901 comparators from +15 V against an 11.5 V reference, open collector pulled up to 3.3 V. The FPGA is bench gear only.
2. **Reverse-engineer the loaned boards**, statically, with a meter and power off, before any of them sees power.
3. **Standalone calculator.** Intel's +5 / −10 V rails, real MCS-40 parts for clock, memory and I/O. The TTL clock timing section carries over from the harness; its CD4504B level stage does not.

## Layout

| Path | What |
|---|---|
| `docs/tutorial.md` | Step-by-step walkthrough from what arrives to a program blinking an LED. Start here. |
| `docs/stage0-bench-plan.md` | Empty-socket test plan with pass criteria: rails, level shifters, signal-generator learning sequence. |
| `docs/bench-measurements.md` | Running list of things the simulation or the design review said to measure on the bench, with results as they come in. |
| `docs/clock-generator.md` | Two-phase 4004 clock in TTL from shelf parts: 24.545 MHz can, 74S163, 74LS163A, 74LS74, 74LS00, CD4504B. Pin tables and scope checklist. |
| `docs/pattern-generator-555-cd4029.md` | NE555 + CD4029B at 15 V, a four-line 0 / 15 V counting pattern that stands in for the 4004 on the bench. |
| `docs/pmos-level-shifting-notes.md` | The analysis: 4004 datasheet limits, the owner's circuit checked against them, every part's datasheet numbers. Section 1 is earlier 8008 reasoning that led here. |
| `docs/spice-lessons.md` | Learning the KiCad simulator (ngspice) lesson by lesson, with the power sheet simulated and the clamp-zener tolerance experiment. |
| `docs/baker-4004-sbc.md` | Scott Baker's 4004/4040 SBC read as prior art for the calculator: Intel rails, 4201 clock, 4289 and 4265 as the level shifters. |
| `docs/hp-board-reverse-engineering.md` | Five-step static plan for the HP board. |
| `docs/reviews/` | Design review rounds of the host interface schematic. |
| `hardware/host-interface/` | KiCad 10 project of the bench harness: power, clock, host buffers, line drivers, receivers, 4004 socket. `4004-host-interface.pdf` is the export. ERC clean. |
| `hardware/spice-tests/` | KiCad scratch project for the simulator lessons; sub-sheet `power` is the simulated power sheet. |
| `bom/` | Parts list with each part's role and the datasheet number behind it, plus the matching Mouser cart upload. |
| `datasheets/` | BS170, LM2901, TP0606, LM319, Traco TMR 1, CD4029B, and from [bitsavers](http://www.bitsavers.org/components/intel/MCS4/) the MCS-40 User's Manual (Nov 1974) and Advance Specifications (Sep 1974). |
| `reference/` | The [RetroShield k4004s](https://github.com/8bitforce/retroshield-hw) PCB imported from gEDA. Prior art only, not this design. |

## Conventions

- Intel datasheets and measurement are the authority. Prior-art boards and the owner's circuit are evidence.
- Guaranteed margins over typical values.
- KiCad files are edited in KiCad or through tooling that preserves them, never by hand.
- Through-hole only.

## Prior art

Studied, not copied:

- 8bitforce RetroShield: [hardware](https://github.com/8bitforce/retroshield-hw), [Teensy firmware](https://github.com/8bitforce/retroshield-teensy), [8bitforce.com](https://8bitforce.com)
- Scott Baker's 4004/4040 single board computer: [repository](https://github.com/sbelectronics/4004), [write-up](https://www.smbaker.com/?p=3953)
- Intel MCS-4 and MCS-40 documentation on [bitsavers](http://www.bitsavers.org/components/intel/MCS4/), including the 4004 datasheet, the MCS-4 User's Manual (1973), the MCS-40 User's Manual (1974) and the MCS-40 Advance Specifications (1974)
- Traco [TMR 1 series](https://www.tracopower.com/tmr1-datasheet) DC/DC converter datasheet

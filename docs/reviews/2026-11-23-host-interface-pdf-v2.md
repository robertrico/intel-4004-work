# Review of "Intel 4004 host interface" v2 (4004Schematicv2.pdf)

Checked every sheet against v1 and both review rounds. All agreed changes are in: 22k RESET/TEST pull-ups, AHCT541 + four-gate LS00 data gating, 74HC14 at 3.3 V with 1k series for PH1/PH2 (active high, table edges renamed to match), comparator tap drawn at the pin node, release delay as a 0 to 300 ns parameter defaulting to 0, sampling margin 250 ns, single 15 V node stated, no tantalums on 15 V, 100 µF 16 V on the 5 V side only, electrolytic bulk at the TMR output, budget, scope list. Resistor counts check out (10k ×18 = 7 host pull-downs + 10 receiver series + 1 divider; 1k ×3 = 2 HC14 + 1 RESET gate).

Ready to build after four small items.

1. Unused CMOS inputs are not tied off. Floating CMOS inputs draw current and can oscillate.
   - AHCT541 #1: 1 spare input; #2: 3 spare inputs. Tie to GND.
   - 74HC14: 4 spare inputs. Tie to GND (its VCC is the host 3.3 V).
   - LM2901 U3: 2 spare channels. Tie IN+ to GND and IN- to the 11.54 V reference; leave the outputs open.

2. LM2901 current in the budget is typical, not maximum. TI gives 1 mA typ, 2.5 mA max per package at high VCC; three packages are up to 7.5 mA. Worst case becomes about 58 mA, 87 % of 67 mA. Conclusion unchanged; the number should read 58.

3. Bench current limit. v2 says empty-board draw + 50 mA; the tutorial says + 45. 50 is the better figure: 4004 IDD 40 + its outputs into the 47k pull-downs about 1.9 + data lines driven high by the 4004 into 12k up to 5 + its own clock-pin charging about 0.3 = about 47 mA. Adopt 50 in both places.

4. Decoupling wording. "0.1 µF per chip" should say per supply pin: the CD4504B needs one at pin 1 (VCC, 5 V) and one at pin 16 (VDD, 15 V).

Optional: state the S163 tie-high resistor count (one 1k shared by /CLR, /LOAD, ENP, ENT is fine).

Nothing else found.

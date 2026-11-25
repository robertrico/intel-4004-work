# Bench measurements

Running list. Each entry comes from the simulation, a design review, or a datasheet gap: something only the real part can answer. Fill in the result column when measured. Keep the number even when it passes.

| # | What | Why | Where in the plan | Pass | Result |
|---|---|---|---|---|---|
| M1 | D1 (1N4745A) current at 15.0 V, no chip | SPICE: a part at the low end of its 15.2..16.8 V band draws 3 mA at 15.0 V and 20 mA at 14.9 V; the model's knee is sharper than a real one, so only the meter settles it | Stage 0 A11 | under 100 uA; milliamps means 1N4746A or no clamp | |
| M2 | TMR 1-0513 output at 0, 15, 30, 45, 60 mA | Load regulation and the 80..87 mA foldback are not in the model | Stage 0 A4 | 14.85..15.15 V, no foldback below 67 mA | |
| M3 | TMR 1-0513 ripple at 60 mA, scope AC-coupled | Datasheet 110 mVp-p, not modelled | Stage 0 A5 | under 150 mVp-p | |
| M4 | 5 V input current at 60 mA out | Polyfuse MF-R025 holds 250 mA; sim at 80 % efficiency says about 220 mA | Stage 0 A4 | under 250 mA with margin, or fit MF-R050 | |
| M5 | Rail step when a 40 mA load switches in | Sim: 50 mV dip, corners rounded by 1.25 ohm x 47 uF | Stage 0 A6 | under 100 mV, no ring | |
| M6 | CD4504B output edges into the 4004 clock pin load | Datasheet 140..280 ns typ/max; the 489 ns clock budget subtracts one edge at each reference point | Stage 0 B3, clock sheet | both channels under 280 ns, skew between channels recorded | |
| M7 | TP0606 turn-off time with the 680 ohm gate resistor | Sets the bus release delay parameter (default 0) | Stage 0 B6 | recorded | |
| M8 | LM2901 propagation delay at 11.5 V reference, 15 V swing | Sampling margin estimate of 250 ns assumes the datasheet typical | Stage 0 B8 | recorded | |
| M9 | TP0606 pin order by diode test | The KiCad symbol assumes S=1, G=2, D=3 | before building sheet 4 | matches, or symbol corrected | |
| M10 | Tantalums on the loaned boards for shorts | Age | HP board static plan step 2 | none shorted | |
| M11 | HP board ground symbol vs the -10 V entry | The silk says GND near the 5 V entry; whether that is 0 V or -10 V decides which rail is which | HP board static plan step 3 | continuity recorded | |

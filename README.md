# HARD3

Video-rate triple hard keyer with exposed positive and negative key outs as well as some logic combinations

## Files

- `Hard3 Rev D Schematic.pdf`: main board Rev D, control board Rev D, power board Rev B
- `Hard3 BOM Rev D.csv`: current BOM
- `Hard3 Rev B Schematic.pdf`, `Hard3 BOM Rev B.csv`: previous revision, kept for reference
- `HARD3Main/Ctrl/Pwr iBOM.html`: interactive placement guides, still Rev B values (see below)

## Rev D: OFFSET fix

### Symptom (Rev B and Rev C)

With anything patched into a CV input, a soft band of mid-level chatter (about 25 lines tall on a vertical ramp) follows the key edge whenever OFFSET is anywhere except fully CCW. Fully CCW is clean because it grounds the pot wiper.

### Cause

The OFFSET pot wiper feeds the + input of the second op amp in each channel (U4B, U8B, U3B pin 5) over the board-to-board header. The 10K pot fed through a 4.99K resistor leaves that node at up to about 3.7K, so it picks up the comparator edges and turns them into positive feedback.

Just adding a cap to that node fixes the chatter, but it breaks the zero. OFFSET at noon nulls the CV by cancellation, and a 100 pF cap on a 3K node phase-shifts video-rate CV (corner around 480 kHz). A DWO3 sine then leaks through at noon, about 30% on the bench, and no knob position removes it. The fix is a lower-impedance wiper plus the cap.

### Changes

| Ref | Board | Rev C | Rev D |
| --- | --- | --- | --- |
| RV4, RV5, RV6 (OFFSET 1-3) | Control | 10K linear | 1K linear |
| R22, R42, R21 (channels 1-3) | Main | 4.99K | 499R |
| C35, C36, C37 | Main | none | 100 pF C0G, OFF12/OFF22/OFF32 to GND |

The series resistor must be half the pot value for the CV null to sit at noon. The channel gain is 3 × wiper, so the noon wiper has to be 1/3 of the buffered CV.

The Rev D PCB layout has not been updated yet. The values have changed, but C35 to C37 still need placing and routing, ideally right at U4B/U8B/U3B pin 5.

### Retrofitting a Rev B or Rev C build

This is the rework done on the prototype (2026-10-27):

1. Replace OFFSET pots RV4, RV5 and RV6 on the control board with B1K 9 mm pots (Alpha RD901F-40-15K-B1K). Clip the old pot's legs and tabs from the front, then remove each stub separately so the plated holes survive.
2. Measure each new pot end to end. On the main board, solder a resistor across the existing 4.99K (R22 for channel 1, R42 for channel 2, R21 for channel 3) so the parallel value is half that pot's measured value. Alternatively, replace the 4.99K with 499R.
3. Solder a 100 pF cap from each wiper net to ground on the solder side of the main board, across header XN3 pins 7-8, 9-10 and 11-12.
   - On Rev C, the +2V5 reference trace runs right beside XN3 pins 7-12 on the board-edge side.
   - Bend the cap leads inboard. A bridge to that trace kills every THRESHOLD and turns all channels white.

On the prototype, the measured pots were 1090, 1137 and 1104 Ω, with 660, 665 and 663 Ω in parallel.

- The caps alone gave a clean edge at both noon and fully CCW.
- With the pot rework added, OFFSET behaved much better on the bench.
- Calculated, not measured: the CV null should land about 1 to 3.5° clockwise of noon, because those parallel values are slightly above half the pot values.

### Check after the rework

1. Patch a V ramp into KEY. The edge should move with THRESHOLD.
2. Patch a DWO3 sine into CV with OFFSET at noon. The edge should be flat, with no ripple.
3. Compare OFFSET at noon with fully CCW. Both should be clean, with no chatter band.

## Known issues and Rev E candidates

Nothing in this section has been built or tested. These are bench findings from the Rev D prototype (2026-10-27) and the changes they point to.

### 1. − output has a mid-level shelf on its edge (fix proposed)

- **Symptom:** with a V ramp in KEY and nothing in CV, the − output edge rises to about 60% and holds there for about 5 lines before reaching full. The + output switches cleanly, and so does the same edge sent to the − jack through the XOR path.
- **What the tests prove:** the problem is at the LM361's second output (pin 9, nets OUT1B/OUT2B/OUT3B). The switch, the ribbon and the U6A/U10A/U10B buffers are fine, because they carry the XOR signal cleanly.
- **Likely cause:** pin 9 drives a long, capacitive run (off the board over the ribbon to SW1 and back, and on channel 1 alongside THRESH12 for 15 mm). When the comparator is in its in-between region on a slow key, that output probably oscillates.
- **Proposed fix:** a 100 Ω series resistor right at pin 9 of U2, U5 and U1. If the shelf survives, try 220 Ω. The output level drops about 3% (or 6% with 220 Ω) against the 3.3K load downstream.
- **Retrofit if the LM361s are socketed:** bend pin 9 out of the socket and solder the resistor between the pin and the socket hole.

### 2. Vertical edges are softer than Ribbons (no hardware fix planned)

- **Measured:** a vertical edge on the + output with a plain H ramp key takes about 13 capture pixels to switch. Ribbons through the same ESG3 and capture chain takes about 7 px, so the extra softness comes from HARD3.
- **+ output:** the LM361 only gets a small overdrive from a gentle ramp, so its output slews slowly. A steeper, clean key should sharpen it (not yet tested).
- **Logic (XOR/XNOR) outputs:** these pass through the CD4070 at 5 V. 4000-series gates have edges on the order of 100 ns, which caps how sharp the logic outputs can be at about the same 13 px. A faster logic family, for example 74HC86 (pin-compatible with the CD4070 for XOR, check the XNOR wiring), would remove that limit. Not verified on this board.

### 3. Line-to-line edge jitter (mostly upstream)

- **Measured:** every line switches exactly once, so this is not chatter, and hysteresis would not help.
- **Jitter, plain H ramp:**
  - HARD3: about ±1.0 px line to line, plus ±0.6 px slow wander.
  - Ribbons, same ramp: about ±0.8 px and ±0.3 px.
- **Where it comes from:** most of it is in the ramp source. HARD3 adds a little, mainly the slow wander. The suspect is low-frequency noise on the THRESHOLD lines, which come from the TL074 on the power board over the ribbon. Not investigated.
- **Avoid noisy gain stages:** adding gain through SumVertDub DUB roughly doubled the jitter, because a noisy gain stage adds more noise than slope.

### 4. Layout notes for any respin

- On the main board, outputs run close to sensitive lines. Gaps are measured edge to edge, Rev C layout:

| Output trace | Sensitive line | Gap | Run alongside |
| --- | --- | --- | --- |
| OUT1B | THRESH12 | 0.9 mm | 15 mm |
| SWITCH1B | OFF32 | 0.32 mm | 4 mm |
| SWITCH1B | OFF12 | 0.67 mm | 2 mm |
| OUT1A_5V | CV2 | 0.20 mm | 7 mm |
| SWITCH3B | CV2 | 0.36 mm | 3 mm |

- C35 to C37 (the Rev D wiper caps) still need placing on the PCB, right at U4B/U8B/U3B pin 5.
- The +2V5 trace runs beside XN3 pins 7 to 12 on Rev C, which makes rework there risky.

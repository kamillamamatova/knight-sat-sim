# RF bench path: IC-9700 -> attenuators -> IC-R8600

Research background; current product scope and implementation requirements are in the [documentation index](../specs/project-plan.md#build-specifications).

Written for a non-RF audience. Every number below is either quoted from an Icom manual / Icom spec page / a manufacturer datasheet (cited), or is arithmetic on those numbers, or is flagged as **engineering practice** (a conservative convention, not a manufacturer figure).

## Project bench constraints

For later RF work, design the attenuator chain to protect the IC-R8600 even if the IC-9700 is keyed at full power. Ask ARC@UCF to lend a 100 W first pad before buying. Require an RF-competent sign-off before cabling or transmitting. This bench is outside the first software demo.

> ## DO NOT CONNECT ANYTHING UNTIL AN EXPERIENCED OPERATOR HAS VERIFIED THIS
>
> This document is a plan, not a permission slip. A 100 W transmitter connected to the wrong thing destroys equipment in well under a second, and the IC-R8600 has no published damage limit. Nothing in this chain gets plugged in, and the IC-9700 is not keyed, until a licensed, RF-experienced operator has (1) checked every attenuator's dB value, power rating, direction and connector against this note, (2) set and confirmed the IC-9700 power settings, and (3) done the first key-up themselves.

**Scenario.** Icom IC-9700 transmitting at its lowest power setting on 2 m (144-148 MHz) or 70 cm (430-450 MHz), AX.25 packet at AFSK 1200 baud (FM), through a short coax and a chain of attenuators directly into an Icom IC-R8600 receiver. No antenna. Part 97 questions are covered separately in `part-97-rules.md`.

---

## 0. Headline numbers

| Quantity | Value | Where it comes from |
| --- | --- | --- |
| IC-9700 minimum output, 144 MHz and 430 MHz (SSB/CW/FM/RTTY/DV) | **0.5 W = +27 dBm** | IC-9700 Basic Manual §11 p. 11-2; Icom America spec page |
| IC-9700 minimum output, 1200 MHz | 0.1 W = +20 dBm | same |
| IC-9700 maximum output (the accident case) | 100 W (2 m) = +50 dBm; 75 W (70 cm) = +48.8 dBm; 10 W (23 cm) = +40 dBm | same |
| IC-R8600 published absolute-maximum RF input | **Not published by Icom** (checked Instruction Manual §14 and Icom America/Europe spec pages) | - |
| IC-R8600 never-exceed level used in this plan | **+10 dBm (10 mW)** absolute ceiling; design so the input can never see more than **-10 dBm** even in a fault | **Engineering practice**, see §2.3 |
| IC-R8600 comfortable operating window for a clean AFSK/FM signal | **-50 to -30 dBm** (S9+23 to S9+43 on its meter) | Derived from manual S-meter calibration, sensitivity, and Icom's IP3 figures, see §2.4 |
| Required attenuation, min power to comfortable window | **57 to 77 dB; design target 60-70 dB** (+27 dBm - (-30 to -50 dBm)) | arithmetic, §3 |
| Recommended chain | **30 dB (rated >= 100 W) + 20 dB (2 W) + 20 dB (2 W) = 70 dB** -> -43 dBm at the receiver | §3, §4 |
| First attenuator power rating | **>= 100 W** (survives a full-power accident on any band). Absolute minimum 50 W *and* IC-9700 TX PWR LIMIT set to <= 25 % | §3.3, §6 |
| Connectors | IC-9700: 144 MHz = SO-239; 430 MHz and 1200 MHz = Type-N. IC-R8600: ANT1 = Type-N (the only one usable above 30 MHz) | IC-9700 Basic Manual p. 11-1, p. 13-3; IC-R8600 Instruction Manual p. 2-2, p. 14-1 |

---

## 1. IC-9700 (transmitter)

### 1.1 Output power range

From the IC-9700 Basic Manual, section 11 "Specifications", page 11-2, "Transmit output power":

- 144 MHz band: SSB/CW/FM/RTTY/DV **0.5 to 100 W**; AM 0.125 to 25 W
- 430 MHz band: SSB/CW/FM/RTTY/DV **0.5 to 75 W**; AM 0.125 to 18.75 W
- 1200 MHz band: SSB/CW/FM/RTTY/DV/DD **0.1 to 10 W**; AM 0.025 to 2.5 W

Icom America's product page states the same figures (https://www.icomamerica.com/lineup/products/IC-9700/, "Output power"). Packet/AFSK is sent in FM, so the planning figure is **0.5 W on 2 m and 70 cm**. (AM goes lower, but AM is not the mode we use and its "power" is carrier power that rises with modulation; do not plan around it.)

### 1.2 How to set the lowest power (two independent settings; use both)

1. **RF POWER** (Basic Manual p. 3-10, "Adjusting the transmit output power"): push MULTI to open the Multi-function menu, touch "RF POWER", rotate MULTI. The range is shown as **0 to 100 %**. The radio displays percent, not watts; the lowest setting corresponds to the specification minimum of about 0.5 W. Set it to the minimum. The Po meter also shows percent, not watts (p. 3-10).
2. **TX PWR LIMIT** (Basic Manual p. 3-10, "Transmit Power Limit function"): "limits the output power to the preset level for each band." On the FUNCTION screen touch [TX PWR LIMIT] to turn it ON, touch it for 1 second, rotate MULTI to set the maximum, push MULTI to close. The manual's own caption: "Even when set RF POWER exceeds 'LIMIT,' the actual output power is limited to this value." It is per band, so set it on 144, 430 and 1200 MHz separately. This is the setting that protects the bench if someone bumps RF POWER.
3. **Power ON Check** (Basic Manual p. 8-20, SET > Display): "Selects whether or not to display the RF Power level at power ON" (default ON). Leave it ON so the power level is shown on the screen every time the radio is switched on.

Neither setting is a hardware interlock. Both can be changed by touching the screen. That is why the first attenuator must be able to absorb full power (§3.3).

### 1.3 Antenna connectors and one hidden hazard

Basic Manual p. 11-1: "Antenna impedance: 50 Ω unbalanced. Antenna connectors: SO-239 × 1 (for the 144 MHz band), Type-N × 2 (for the 430/1200 MHz band)." The rear-panel diagram on p. 13-3 labels them [144MHz ANT] (SO-239), [430MHz ANT] (Type N), [1200MHz ANT] (Type N), each "Input/Output impedance: 50 Ω (unbalanced)".

Hazard: p. 13-3 carries a boxed WARNING: "A DC voltage can be applied to the antenna coax connector to power an external preamplifier." This is controlled by MENU > SET > Connectors > External P.AMP (per band). Fixed attenuators are resistor networks: they do not block DC, they load it (drawing current from the radio's feed through their shunt resistors) and pass a reduced DC voltage onward to the receiver. Neither is wanted. **Confirm External P.AMP is OFF on every band before connecting the chain.**

### 1.4 What the IC-9700 does if the load is bad

- Advanced Manual p. 7-2, "Protection function": a 2-step protection for the final amplifier "in case the antenna SWR becomes high" or the PA temperature becomes too high: first "Power down transmission" (output reduced, "LMT" shown), then "TX inhibit" (transmitter disabled).
- Basic Manual troubleshooting (section 10, p. 10-6): "No power output or the output power is too low" - "The antenna SWR is more than 3:1. Adjust the antenna for an SWR of less than 3:1." The same table lists "The output power is limited because of power amplifier protection. Stop transmitting, and then wait without turning off the transceiver until the temperature of the power amplifier FET drops sufficiently."
- Advanced Manual p. 7-2, "Measuring SWR": the built-in SWR meter; "If the SWR meter indicates 1.5 or less, the antenna is matched."

So a badly mismatched or open load will not normally destroy a modern IC-9700; the radio folds back. But protection is a backstop, not a plan, and it does nothing to protect the attenuators or the receiver.

---

## 2. IC-R8600 (receiver)

### 2.1 Connectors

IC-R8600 Instruction Manual p. 14-1, "Antenna connectors":

| | ANT1 | ANT2 | ANT3 |
| --- | --- | --- | --- |
| Frequency range | 10 kHz - 3000 MHz | 10 kHz - 30 MHz | 10 kHz - 30 MHz |
| Impedance | 50 Ω unbalanced | 50 Ω unbalanced | 500 Ω unbalanced |
| Connector | **N type** | SO-239 | RCA |

Page 2-2 repeats it in the connection diagram ("ANT 1 connector (N type) 0.01 MHz ~ 3000 MHz (50 Ω)"). For 144/430/1200 MHz the only usable input is **ANT1, Type-N**. Do not put the SO-239 on ANT2 into the plan; it is HF-only.

### 2.2 Sensitivity and meter calibration (manual figures)

- Sensitivity, 30-1099.999 MHz, preamp ON (p. 14-2): SSB/CW/FSK (10 dB S/N) -10 dBμ; **FM (12 dB SINAD) -6 dBμ**; DIGITAL (1 % BER) -2 dBμ. Icom's dBμ is dBμV across a 50 Ω terminated load (p. 3-4), so dBm = dBμ - 107: FM sensitivity **-6 dBμ = -113 dBm**, SSB -117 dBm.
- S-meter calibration (p. 3-4): "At S9, the input signal level is 50 μV (34 dBμ). At S9 +20 dB, the input signal level is 54 dBμ." In dBm: **S9 = -73 dBm, S9+20 = -53 dBm**, S9+40 = -33 dBm. The meter can be switched to read dBm directly ("dBm meter: Absolute power. 0 dBm is the level corresponding to 1 mW that is produced at a 50 Ω terminated load", p. 3-4). Icom Europe states the dBm meter is accurate to ±3 dB between 0.5 and 1100 MHz (https://www.icomeurope.com/en/product/ic-r8600/). This is what the operator will use to confirm the level actually arriving.
- Front-end controls (p. 5-1): preamplifier gain "approximately 20 dB on the HF bands, 14 dB on the VHF and UHF bands"; "When you use the preamplifier while receiving a strong signal, the receiving signal may be distorted. In such case, turn OFF the preamplifier." Attenuator: 10 / 20 / 30 dB steps. Overflow: "If a strong signal is received and OVF (Overflow) appears, reduce the RF gain or turn ON the attenuator until it disappears." The troubleshooting table (p. 12-5) gives the same remedy for "OVF is displayed: An excessively strong signal is received."
- Icom Europe's product page gives third-order intercept: "IP3 performance is +10 dBm at 144 MHz and 0 dBm at 440 MHz."

### 2.3 Maximum safe input: what Icom says and what we assume

**Icom does not publish an absolute maximum RF input level for the IC-R8600.** The Instruction Manual's Precautions (p. viii) and Specifications (§14) contain no such figure, and neither do the Icom America and Icom Europe spec pages. The manual only describes OVF, which is an overload *indicator*, not a damage rating.

Because there is no manufacturer number, this plan uses a conservative convention (**engineering practice, not an Icom specification**):

- Treat **+10 dBm (10 mW)** as an absolute never-exceed at ANT1. Wideband receivers with a semiconductor preamp and relay-switched attenuator on the input, as this one has, are typically specified by their makers somewhere in the +10 to +20 dBm region; we have no evidence the IC-R8600 tolerates more than the low end of that.
- Design the chain so that **even a full-power (100 W) accident delivers no more than about -10 dBm** to the receiver. That gives 20 dB of margin below the assumed ceiling, and it is below the level at which the receiver's digitizer starts overloading (an independent test report measured OVF onset at -8 dBm with preamp off and -27 dBm with preamp on, at 14.1 MHz: Adam Farson AB4OJ, "IC-R8600 User Evaluation & Test Report", Table 2, https://www.qsl.net/ab4oj/icom/r8600/r8600notes.pdf; HF direct-sampling data, but the right order of magnitude).

Ask the experienced operator, or Icom America support, before relying on any figure higher than 0 dBm.

### 2.4 Comfortable operating level: -50 to -30 dBm, and why

"Comfortable" means: far above the noise, below any overload, and not dependent on the preamp. From the manual figures in §2.2:

- FM 12 dB SINAD sensitivity is -113 dBm. A signal at -40 dBm is 73 dB above that: full quieting, error-free 1200 baud AFSK with any decoder.
- -50 to -30 dBm reads S9+23 to S9+43 on the S-meter (S9 = -73 dBm). This is the "very strong local station" region that the radio is designed to handle every day with the preamp off.
- Icom's own IP3 figures are +10 dBm at 144 MHz and 0 dBm at 440 MHz. Keeping the signal at least 30 dB below IP3 (i.e. <= -30 dBm at 440 MHz) keeps the receiver's own distortion products negligible. That fixes the upper edge of the window at -30 dBm.
- Below about -50 dBm nothing breaks; the window is simply chosen so the receiver's dBm meter reads well inside its ±3 dB accurate range and the signal dominates any leakage or nearby real stations.

Receiver settings for the test: preamp **OFF**, internal ATT OFF (keep it available as a 10/20/30 dB trim, but do not count it in the safety budget; it is a menu setting, not a guarantee), RF gain 100 %, FM mode, 15 kHz filter.

---

## 3. The arithmetic

### 3.1 Watts to dBm

dBm = 10 × log10(P in milliwatts).

| Power | dBm |
| --- | --- |
| 0.5 W = 500 mW (IC-9700 minimum, 2 m / 70 cm) | 10 × log10(500) = **+27.0 dBm** |
| 0.1 W (IC-9700 minimum, 23 cm) | +20.0 dBm |
| 10 W (23 cm max) | +40.0 dBm |
| 75 W (70 cm max) | +48.8 dBm |
| 100 W (2 m max) | **+50.0 dBm** |

### 3.2 Required attenuation

Attenuation needed = source dBm - target dBm.

- To the top of the window (-30 dBm): 27 - (-30) = **57 dB**
- To the middle (-40 dBm): 27 - (-40) = **67 dB**
- To the bottom (-50 dBm): 27 - (-50) = **77 dB**

So anything from about 60 to 75 dB works. Coax loss in two 3 ft jumpers is small enough to ignore (LMR-240 is roughly 3 dB per 100 ft at 150 MHz and 5 dB per 100 ft at 450 MHz per Times Microwave, i.e. under 0.2 dB per jumper).

Chosen design: **30 + 20 + 20 = 70 dB** (nominal). Result at ANT1: 27 - 70 = **-43 dBm** (about S9+30). If only a 30 + 30 = 60 dB chain is available: -33 dBm (S9+40), still inside the window. With the 23 cm minimum of 0.1 W the same 70 dB chain gives -50 dBm, the bottom of the window; use 60 dB there.

### 3.3 Power at every point in the chain, normal case and accident case

Attenuators turn almost all of the input into heat: a 30 dB pad dissipates 99.9 % of what goes in and passes 0.1 %. That is why only the first stage needs a serious power rating.

| Point | Normal (0.5 W, +27 dBm) | Accident: 100 W keyed (+50 dBm) | What must survive it |
| --- | --- | --- | --- |
| Into stage 1 (30 dB) | 0.5 W | **100 W** | Stage 1 rating >= 100 W |
| Out of stage 1 / into stage 2 | 0.5 mW (-3 dBm) | 0.1 W (+20 dBm) | Stage 2 rating >= 0.1 W; a 2 W pad has 13 dB of margin |
| Out of stage 2 (20 dB) / into stage 3 | 5 μW (-23 dBm) | 1 mW (0 dBm) | trivial |
| Out of stage 3 (20 dB) / into IC-R8600 ANT1 | **-43 dBm** | **-20 dBm** | Below the -10 dBm design ceiling in §2.3, so the receiver survives the accident |

Key consequence: **if, and only if, the first attenuator is rated for the IC-9700's full output, the whole chain is fail-safe against a full-power key-up.** Every later element, including the receiver, sees at most 0.1 W. If the first attenuator is rated 50 W instead, a 100 W key-up (2 m) or 75 W key-up (70 cm) exceeds its rating; it may survive a brief burst (Mini-Circuits rates its 50 W unit for 500 W peak at 5 μs pulses, which says nothing about a 1 second key-up) but must not be relied on, so TX PWR LIMIT must be set so the radio cannot exceed 25 W (25 % on 2 m) with a 50 W unit.

Also check the direction. Mini-Circuits' BW-N30W50+ datasheet: "50 W ... input N-Male. **5 W max. at N-Female.**" It is unidirectional; installed backwards it is a 5 W device. Bird's 50-A and 100-A series are bidirectional. Whatever is lent, the operator reads the label and the datasheet, and marks the transmitter end with tape.

---

## 4. The chain

```
 IC-9700 (transmit)                                                     IC-R8600 (receive)
 RF POWER = min (0.5 W)                                                 ANT1, Type-N
 TX PWR LIMIT = ON, minimum                                             Preamp OFF, ATT OFF
 External P.AMP = OFF
 +---------------+                                                      +---------------+
 | [144MHz ANT]  | SO-239                                               |               |
 | [430MHz ANT]  | Type-N                                               |    [ANT1]  N  |
 +-------+-------+                                                      +-------+-------+
         |                                                                      ^
         | coax jumper, 3 ft (1 m)                                              | coax jumper, 3 ft (1 m)
         | 2 m: PL-259 jumper + N-male-to-SO-239 adapter at stage 1 input       | N male <-> N male
         | 70 cm: N male <-> N male                                             |
         v                                                                      |
 +------------------+     +----------------+     +----------------+             |
 |  STAGE 1         |     |  STAGE 2       |     |  STAGE 3       |             |
 |  30 dB           |---->|  20 dB         |---->|  20 dB         |-------------+
 |  >= 100 W rated  |     |  2 W rated     |     |  2 W rated     |
 |  N connectors    |     |  SMA (adapters)|     |  SMA (adapters)|
 |  GETS WARM: 0.5 W|     |                |     |                |
 |  IN ACCIDENT:    |     | sees <= 0.1 W  |     | sees <= 1 mW   |
 |  100 W -> heat   |     | in an accident |     | in an accident |
 +------------------+     +----------------+     +----------------+
    +27 dBm in                -3 dBm in             -23 dBm in            -43 dBm at receiver
   (+50 dBm accident)        (+20 dBm accident)    (0 dBm accident)      (-20 dBm accident)

 Total 70 dB.  Normal: 0.5 W -> -43 dBm (S9+30).  Full-power accident: 100 W -> -20 dBm (safe).
 TX end of every attenuator marked with tape. No antenna anywhere. Nothing else on the bench connected to either radio's antenna port.
```

The three-pad version is preferred over one big 60 or 70 dB pad because 30 dB high-power units are common loan items (clubs, university RF labs) and the small SMA pads are cheap; one 70 dB / 100 W attenuator is a specialist part.

---

## 5. Alternative: dummy load plus a "sniffer" or sampling tap

**Arrangement B1: dummy load + leakage.** IC-9700 into a proper 50 Ω dummy load (e.g. MFJ-260C: 300 W for 30 s, 25 W continuous, SO-239, 0-650 MHz, VSWR <= 1.3:1, https://www.gigaparts.com/mfj-260c.html). The IC-R8600 gets a short whip or a few centimetres of wire on ANT1 and is placed across the room; it hears whatever leaks from the load, connectors and radio chassis.

- Safer for the *receiver* than anything else: there is no cable path by which transmitter power can reach it. Also the cheapest ($70-85 for the load).
- Worse for everything else: the received level is unknown and not repeatable (it depends on distance, orientation, which way the coax lies), it may be too weak with a well-built load or too strong if someone "helps" by moving the receiver closer, the receiver also hears real on-air traffic and lab noise, and it deliberately relies on radiation, which is the thing the bench test is meant to avoid (see the Part 97 note). It teaches nothing about levels.

**Arrangement B2: dummy load + directional coupler (-20/-30 dB tap).** IC-9700 -> coupler -> dummy load, with the coupler's sampled port -> pads -> IC-R8600. This is what an RF lab would do to monitor a transmitter, and it gives a known, repeatable level. The catch for a novice team is the coupler's through-line power rating. The common low-cost broadband unit, Mini-Circuits ZFDC-20-5+ (0.1-2000 MHz, 19.5 dB coupling, ~$127-132), is rated for only **0.5 to 2 W through the main line** (datasheet: https://www.minicircuits.com/pdfs/ZFDC-20-5+.pdf, "Power input (W) ... 0.5 ... 2.0"); a full-power key-up destroys it and then the sampled level is undefined. A coupler that survives 100 W (Bird line section, Werlatone, etc.) costs as much as or more than the 100 W attenuator, and the dummy load is still needed.

**Verdict for this team.** The **direct attenuated path with a >= 100 W first stage (§4) is the safer arrangement for novices**, because its safety comes from physics that cannot be mis-set: once the first pad is rated for full power, nothing downstream, receiver included, can ever see more than 0.1 W, and the level at the receiver is known to within a few dB and reproducible from session to session. Keep a dummy load on the bench anyway: it is the thing the IC-9700 should be connected to whenever the chain is not, and it is what the operator uses for the very first key-up (§6). B1 is acceptable only as a five-minute "does Direwolf decode at all" sanity check under supervision; B2 is only worth it with a high-power coupler, which nobody is going to lend a first-year team.

---

## 6. Risks and mitigations

### 6.1 Risk to the IC-9700 from the attenuator load

A correctly rated attenuator is a very good 50 Ω load, better than most antennas: Mini-Circuits specifies VSWR <= 1.45:1 for the BW-N30W50+ (typ. 1.30), Bird specifies <= 1.10:1 to 1 GHz for the 50-A/100-A series. The IC-9700 is happier into this than into an antenna. The realistic ways the transmitter sees a bad load are:

- The attenuator is overdriven (see §3.3) and its resistors burn open or short. The IC-9700's protection (Advanced Manual p. 7-2) then folds back power or inhibits TX; the radio is normally fine, the attenuator is scrap, and if the failure is intermittent the downstream gear may see uncontrolled power. Mitigation: first stage rated >= radio maximum; TX PWR LIMIT set; watch the SWR meter on the first key-up.
- Keying with nothing connected, or with a loose connector. Same protection applies; same mitigation: never key unless the operator has physically traced the path end to end.
- A unidirectional attenuator installed backwards (5 W end toward the radio). Mitigation: tape-mark the TX end; the operator checks the label against the datasheet.
- Thermal: at 0.5 W nothing gets warm. At a 100 W accident a convection-cooled 100 W attenuator gets hot within seconds; it is rated for it, but keep hands off and keep transmissions short.
- DC on the coax from External P.AMP (§1.3). Mitigation: confirm OFF per band.

### 6.2 Risk of accidentally keying at full power

The failure mode that actually destroys things: RF POWER left at 100 % from someone else's use, or bumped on the touch screen, and PTT pressed. 100 W into a 2 W SMA pad, a low-power coupler, or directly into the receiver is instant, silent damage.

Mitigations, in order of how much they are worth:

1. **First attenuator rated >= 100 W** (§3.3). This is the only mitigation that is not a setting somebody can change. With it in place, every other mistake on this list costs nothing.
2. **TX PWR LIMIT ON and set to the minimum on every band** (Basic Manual p. 3-10). Caps the output even if RF POWER is turned up.
3. **RF POWER at minimum**, and **Power ON Check ON** so the level is displayed at every power-up (p. 8-20).
4. **Pre-session check, every session, before the receiver is connected:** IC-9700 into the 100 W attenuator (or the dummy load) *with the receiver not yet attached*; operator keys briefly in FM; confirms Po meter at minimum, SWR meter at or near 1.0 (Advanced Manual p. 7-2), no "LMT" indicator. Only then connect stages 2, 3 and the receiver.
5. **Level check at the receiver:** with the chain connected, key briefly; the IC-R8600 dBm meter should read about -43 dBm (±3 dB meter accuracy, ±1.5 dB or so of attenuator tolerance). If it reads much higher, stop: a pad is missing, wrong, or backwards.
6. Housekeeping: VOX OFF; microphone unplugged unless needed (PTT then requires the front-panel TRANSMIT key); the IC-9700 sits on the dummy load, not the chain, when nobody is testing; a laminated card with the settings taped to the radio; one named person (the control operator) is the only one who keys.

---

## 7. What to ask ARC@UCF to lend, and what it costs if that fails

Representative products only, for the loan request and cost estimate, not endorsements. Prices are list/online prices seen 2026-09-17 and move around.

| Item | Spec to ask for | Representative product | Approx. cost if bought |
| --- | --- | --- | --- |
| **Stage 1 attenuator** | 30 dB, 50 Ω, **>= 100 W continuous**, DC to >= 500 MHz (>= 1.3 GHz if 23 cm is ever tested), N connectors, bidirectional preferred | Bird 100-A-FFN-30 (100 W, DC-2.4/3 GHz, N female both ends, bidirectional) https://birdrf.com/rf-equipment/attenuators/100w-series ; JFW 50FH-030-100-3 (100 W, DC-3 GHz, N) https://www.jfwindustries.com/product/50fh-xxx-100-3-fixed-attenuator/ | $475-745 (Bird, various dealers); JFW by quote |
| Stage 1, minimum acceptable | 30 dB, **>= 50 W**, N; requires TX PWR LIMIT <= 25 % | Mini-Circuits BW-N30W50+ (50 W at 25 °C, unidirectional, N-male in / N-female out, 5 W max in reverse) https://www.minicircuits.com/pdfs/BW-N30W50+.pdf ; Bird 50-A-FFN-30 (50 W, bidirectional) | $424 (Mini-Circuits list); ~$410 (Bird) |
| **Stage 2 and 3 pads** (x2) | 20 dB each, 50 Ω, >= 2 W, DC-18 GHz | Mini-Circuits BW-S20W2+ (2 W, SMA female to SMA male) https://www.minicircuits.com/pdfs/BW-S20W2+.pdf | $46 each |
| Alternative stage 2 (one pad instead of two, N connectors, fewer adapters) | 20 dB, 20 W, N | Mini-Circuits BW-N20W20+ (N female to N male) https://www.minicircuits.com/pdfs/BW-N20W20+.pdf | ~$100-150 (check current list) |
| **Dummy load** (always on the bench) | 50 Ω, >= 100 W short-term, covers 144 and 430 MHz, SO-239 or N | MFJ-260C (300 W / 30 s, 25 W continuous, 0-650 MHz, SO-239) https://www.gigaparts.com/mfj-260c.html | $70-85 |
| Coax jumper, 2 m path | 3 ft, 50 Ω, PL-259 (UHF male) both ends, the ordinary ham jumper; it plugs into the IC-9700's SO-239 directly and reaches the attenuator through the adapter below | any LMR-240 / RG-8X / RG-58 assembly from a reputable maker | $15-40 |
| Coax jumper, 70 cm path and receiver side (x2) | 3 ft, 50 Ω, N male to N male | LMR-240 N-N 3 ft, e.g. https://us.infinitecables.com/products/lmr-240-ultra-flex-n-type-male-to-n-type-male-cable | $22-45 each |
| Adapter, 2 m path into stage 1 | N male to UHF female (SO-239): the N male goes into the attenuator's N female input, the PL-259 jumper plugs into the SO-239 side. (If the only jumper is N-N, the alternative is a UHF male to N female adapter on the radio's SO-239 instead.) | Amphenol 242155 (N male to UHF female) https://www.rfparts.com/242155.html | $10-30 |
| Adapters for the SMA pads (x2-4) | N male to SMA female, N female to SMA male, as needed to mate SMA pads between N gear | any Amphenol / Pasternack / Fairview between-series adapter | $10-25 each |
| Optional: through-line wattmeter | 5 W and 100 W scales, 144/430 MHz, for the operator's first key-up | Bird 43 with 5C / 100C / 5D / 100D slugs (clubs usually own one) | loan only; not worth buying |

**Loan request, in one sentence:** "One 30 dB attenuator rated 100 W (or at least 50 W) with N connectors, two 20 dB / 2 W SMA pads, a 50 Ω dummy load good to 450 MHz, three 3 ft coax jumpers (N-N, N-N, N-PL259), N-to-SMA and N-to-SO-239 adapters, and a Bird wattmeter with 2 m / 70 cm slugs; plus an hour of an experienced member's time for the first key-up."

**Cost if the loan fails entirely:** about **$700-1,100** for the recommended set (100 W Bird attenuator being $475-745 of it), or about **$650-750** with the Mini-Circuits 50 W unit. Used 30 dB / 50-150 W attenuators (Bird, Narda, Weinschel) turn up second-hand for $60-150; that is a reasonable route only if the experienced operator can measure the unit's attenuation and check it on a wattmeter before it goes near the receiver.

---

## 8. Operator checklist (for the person who does the first key-up)

1. Datasheet of every attenuator in hand; dB value, power rating, direction, frequency range confirmed against §3 and §4; TX end taped.
2. IC-9700: RF POWER minimum; TX PWR LIMIT ON and minimum on 144, 430, 1200; Power ON Check ON; External P.AMP OFF on all bands; VOX OFF; mode FM.
3. IC-R8600: preamp OFF; ATT OFF; RF gain 100 %; meter set to dBm; **not yet connected**.
4. IC-9700 -> stage 1 only (or dummy load). Key briefly. Po at minimum, SWR ~1.0, no LMT. Optional: wattmeter confirms ~0.5 W.
5. Add stages 2 and 3, then the receiver. Key briefly. IC-R8600 reads about -43 dBm (60 dB chain: about -33 dBm). If it reads above -30 dBm, stop and find out why.
6. Log frequency, power setting, chain, and the dBm reading. Leave the IC-9700 on the dummy load when done.

---

## Sources

- Icom IC-9700 Basic Manual (local copy: `Documentation/Radios/Icom IC-9700 - Basic Manual.pdf`): p. 3-10 (RF POWER, Transmit Power Limit), p. 8-20 (Power ON Check), p. 11-1 (antenna impedance/connectors), p. 11-2 (transmit output power), p. 10-6 troubleshooting (SWR > 3:1, PA protection), p. 13-3 (rear-panel connectors; External P.AMP DC warning).
- Icom IC-9700 Advanced Manual (local copy): p. 7-2 (Measuring SWR; Protection function).
- Icom IC-R8600 Instruction Manual (local copy: `Documentation/Radios/Icom IC-R8600 - Instruction Manual.pdf`): p. viii (Precautions), p. 2-2 (antenna connection), p. 3-4 (S-meter / dBμ / dBm calibration), p. 5-1 (preamp gain, attenuator, OVF), p. 12-5 (troubleshooting, OVF), p. 14-1 and 14-2 (connectors, sensitivity).
- Icom America IC-9700 specifications: https://www.icomamerica.com/lineup/products/IC-9700/
- Icom America IC-R8600 specifications: https://www.icomamerica.com/lineup/products/IC-R8600/
- Icom Europe IC-R8600 page (IP3 at 144/440 MHz; dBm meter accuracy): https://www.icomeurope.com/en/product/ic-r8600/
- Adam Farson AB4OJ, IC-R8600 User Evaluation & Test Report (independent measurements; OVF onset levels): https://www.qsl.net/ab4oj/icom/r8600/r8600notes.pdf
- Mini-Circuits datasheets: BW-N30W50+ https://www.minicircuits.com/pdfs/BW-N30W50+.pdf ; BW-N20W20+ https://www.minicircuits.com/pdfs/BW-N20W20+.pdf ; BW-S20W2+ https://www.minicircuits.com/pdfs/BW-S20W2+.pdf ; ZFDC-20-5+ https://www.minicircuits.com/pdfs/ZFDC-20-5+.pdf
- Bird Technologies 100-A and 50-A series attenuators: https://birdrf.com/rf-equipment/attenuators/100w-series ; https://birdrf.com/rf-equipment/attenuators/50w-series
- JFW 50FH-XXX-100-3: https://www.jfwindustries.com/product/50fh-xxx-100-3-fixed-attenuator/
- MFJ-260C dummy load: https://www.gigaparts.com/mfj-260c.html

> ## DO NOT CONNECT ANYTHING UNTIL AN EXPERIENCED OPERATOR HAS VERIFIED THIS
>
> The numbers above are a plan derived from manuals and datasheets by people who are not RF engineers. The IC-R8600's damage threshold is not published; the IC-9700 can put out 200 times its minimum power with two touches of a screen. The first key-up, and every check in §8, is done by a licensed, experienced operator, or it is not done.

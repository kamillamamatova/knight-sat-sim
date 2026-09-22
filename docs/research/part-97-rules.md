# FCC Part 97 and the RF bench test

Research background; current product scope and implementation requirements are in the [documentation index](../specs/project-plan.md#build-specifications).

Written for a non-radio audience. This describes what the rules say and where they are unclear; it is not legal advice.

## Project operating constraints

For later RF work, treat the cabled bench as on-air: Diab must be present whenever it transmits, and AX.25 must carry his real callsign as `MYCALL`. Keep command counters, CRC, and any HMAC over readable commands; encrypted-payload Challenges remain software-only. This is the project operating policy for the bench. The first software demo requires no RF hardware.

**Scenario.** An Icom IC-9700 transmits at its lowest power setting on 2 m (144-148 MHz) or 70 cm (420-450 MHz), through coax and an attenuator, directly into an Icom IC-R8600 receiver. Nothing is meant to radiate. Mode is AX.25 packet at AFSK 1200 baud (Direwolf) carrying project-defined command frames with sequence counters, CRCs, and later possibly an HMAC/signature. Control operator is Diab, Technician class. Location is UCF, Orlando, Florida.

**Sources.** All rule text is from the current eCFR, Title 47, Part 97 (https://www.ecfr.gov/current/title-47/chapter-I/subchapter-D/part-97), retrieved 2026-09-17. Where a definition lives outside Part 97 (47 CFR 2.1, 2.106) or in an FCC order, that is cited explicitly. ARRL material is cited only where it quotes or characterizes the rule.

**One-paragraph summary.** Part 97 never defines "transmit" and has no dummy-load exception, so the safe assumption is that keying the IC-9700 on an amateur frequency is a transmission and every Part 97 operating rule applies, even though the RF is confined to coax. That is not a problem for this project: a Technician may use data modes on all of 2 m above 144.1 MHz and all of 70 cm; AX.25 carries the callsign in every frame, which is the accepted way packet stations identify; there is no logging requirement; counters, CRCs, and an authentication tag are not "encoding to obscure meaning" as long as the command payload itself stays readable. The rules that actually bite are: Diab must be at the computer whenever it transmits (or the setup must qualify as an automatically controlled digital station), an unlicensed teammate may only act under his direct, continuous supervision, and the space-telecommand exception does not apply because there is no real space station.

---

## 1. Transmitting into a dummy load or closed cable path

### What the rules say

Part 97 does not define "transmit" or "transmission." The definitions section (§ 97.3) defines an amateur station as "a station in an amateur radio service consisting of the apparatus necessary for carrying on radiocommunications" (§ 97.3(a)(5)), but leaves "transmit" undefined. There is no exemption anywhere in Part 97 for transmissions into a dummy load, a closed cable, or a shielded enclosure.

The general definitions in Part 2 give the only textual hook:

> "Radiocommunication. Telecommunication by means of radio waves." (47 CFR 2.1)
> "Radio Waves or Hertzian Waves. Electromagnetic waves of frequencies arbitrarily lower than 3,000 GHz, propagated in space without artificial guide." (47 CFR 2.1)

A literal reading is that RF energy confined to coax (an "artificial guide") is not "radio waves" and therefore not "radiocommunication." No FCC order or Part 97 rule adopts that reading for amateur stations, and every real cable, attenuator, connector, and radio chassis leaks a little; § 97.307(c) explicitly treats "chassis or power line radiation" as an emission the station is responsible for.

Two rules make short, informational-content-free transmissions clearly legal even when they are one-way:

> "In addition to one-way transmissions specifically authorized elsewhere in this part, an amateur station may transmit the following types of one-way communications: (1) Brief transmissions necessary to make adjustments to the station; ..." (§ 97.111(b)(1))
> "A station may transmit a test emission on any frequency authorized to the control operator for brief periods for experimental purposes ..." (§ 97.305(b))

The 2 m band (144.1-148 MHz) and the entire 70 cm band list "test" as an authorized emission type (§ 97.305(c)(4)(iii), (c)(5)(i)).

**Logging.** Part 97 contains no requirement to keep a station log. The FCC eliminated routine amateur logging in a 1982-83 rulemaking (PR Docket 82-726, FCC 82-456, "Elimination of logging requirements in the Amateur Radio Service"). What remains is: the licensee "must make the station and the station records available for inspection upon request by an FCC representative" (§ 97.103(c)); the FCC presumes the licensee is the control operator "unless documentation to the contrary is in the station records" (§ 97.103(b)); and a Regional Director may, case by case, order a station using an unspecified digital code to "maintain a record, convertible to the original information, of all digital communications transmitted" (§ 97.309(b)(3)).

### Practical reading

- Safest assumption: treat every keyed transmission on an amateur frequency as a Part 97 transmission, whether or not an antenna is connected. Follow identification, control-operator, and content rules exactly as if it were on the air. Under that assumption the bench test is a perfectly ordinary amateur activity; the rules do not make it harder because it is on a bench.
- The "artificial guide" argument in 47 CFR 2.1 is a reasonable fallback position, not a licence to ignore the rules. Nobody should rely on it.
- No log is required. Keeping a simple session log (date, time, frequency, power setting, who was control operator, what was sent) is good engineering practice under § 97.101(a) and it also serves the project's own reproducibility needs, so keep one anyway.
- Some hams argue that RF into a dummy load is not "on the air" and so needs no ID. That is a convention, not a rule. The conservative convention (ID anyway) costs nothing here because AX.25 IDs in every frame (section 2).

---

## 2. Station identification for digital modes, and how AX.25 satisfies it

### What the rules say

> "(a) Each amateur station, except a space station or telecommand station, must transmit its assigned call sign on its transmitting channel at the end of each communication, and at least every 10 minutes during a communication, for the purpose of clearly making the source of the transmissions from the station known to those receiving the transmissions. No station may transmit unidentified communications or signals, or transmit as the station call sign, any call sign not authorized to the station." (§ 97.119(a))
> "(b) The call sign must be transmitted with an emission authorized for the transmitting channel in one of the following ways: ... (3) By a RTTY emission using a specified digital code when all or part of the communications are transmitted by a RTTY or data emission; ..." (§ 97.119(b)(3))

The "specified digital codes" are Baudot, AMTOR, and ASCII (§ 97.309(a)(1)-(3)). ASCII is "the 7-unit, International Alphabet No. 5, code defined in ITU-T Recommendation T.50" (§ 97.309(a)(3)).

The "telecommand station" exemption in § 97.119(a) does not apply: a telecommand station is "an amateur station that transmits communications to initiate, modify or terminate functions of a space station" (§ 97.3(a)(45)), and a space station is one "located more than 50 km above the Earth's surface" (§ 97.3(a)(41)). The IC-R8600 on the bench is neither.

### How AX.25 fits

Every AX.25 frame carries a source address and a destination address, each consisting of an amateur callsign (upper-case ASCII letters and digits, up to six characters) plus a 4-bit SSID (AX.25 v2.2 spec § 3.12). Direwolf puts the configured callsign (e.g. `MYCALL`) into the source field of every frame it transmits. So each frame transmits the callsign in ASCII, which is a specified digital code, and the "at the end of each communication and at least every 10 minutes" requirement is satisfied automatically as long as the callsign is real.

Two details to be aware of:

- **Bit shifting.** AX.25 stores the callsign characters shifted left one bit so the low bit can be used as an address-extension flag (AX.25 v2.2 § 3.12). Any packet decoder reverses this and displays the plain callsign. Packet stations have identified this way since the 1980s and the amateur community treats it as compliant; there is a long-running debate on the edges (APRS digipeaters, tactical callsigns) but none of it touches a station that puts its own callsign in the source field of every frame it originates.
- **Indicators.** If an indicator (e.g. `/AG`) were ever required, § 97.119(c) says it must be separated by a slant mark, and the AX.25 address field has no room for one; it would have to go in the information field. Nothing in this project requires an indicator (Diab is both licensee and control operator, so § 97.119(e) does not apply).

### Practical reading

- Set Direwolf `MYCALL` to Diab's exact FCC callsign (optionally with an SSID such as `-1`). Never use a made-up or project-themed callsign in the source field: § 97.119(a) forbids transmitting "any call sign not authorized to the station," and § 97.113(a)(4) forbids "false or deceptive ... identification."
- The **destination** field is a different matter. In real packet operation it holds the other station's callsign; in APRS it holds a software identifier. Putting a project label like `SAT1` in the destination is conventional and is not identification; identification is the source field. If the "satellite" side ever transmits back (it currently does not; the IC-R8600 is receive only), its source field must also be Diab's callsign.
- If the team ever adds a non-AX.25 test emission (an unmodulated carrier, a raw AFSK tone), § 97.119 still applies to it. Keep those bursts brief (§ 97.111(b)(1), § 97.305(b)) and send an AX.25 frame or a CW/voice ID before stopping.

---

## 3. "Encoded to obscure meaning" vs. counters, CRCs, and an HMAC

### What the rules say

> "(a) No amateur station shall transmit: ... (4) ... messages encoded for the purpose of obscuring their meaning, except as otherwise provided herein; ... or false or deceptive messages, signals or identification." (§ 97.113(a)(4))

The digital-code rules say the same thing from the other side:

> "Only a digital code of a type specifically authorized in this part may be transmitted." (§ 97.3(c)(2), definition of a "data" emission)
> "(b) Where authorized by §§ 97.305(c) and 97.307(f), a station may transmit a RTTY or data emission using an unspecified digital code, except to a station in a country with which the United States does not have an agreement permitting the code to be used. RTTY and data emissions using unspecified digital codes must not be transmitted for the purpose of obscuring the meaning of any communication." (§ 97.309(b))

On 2 m and 70 cm, unspecified digital codes are expressly allowed:

> "(5) A RTTY, data or multiplexed emission using a specified digital code listed in § 97.309(a) may be transmitted. The symbol rate must not exceed 19.6 kilobauds. A RTTY, data or multiplexed emission using an unspecified digital code under the limitations listed in § 97.309(b) also may be transmitted. The authorized bandwidth is 20 kHz." (§ 97.307(f)(5), applies to 2 m via § 97.305(c)(4)(iii))
> "(6) ... The symbol rate must not exceed 56 kilobauds. A RTTY, data or multiplexed emission using an unspecified digital code under the limitations listed in § 97.309(b) also may be transmitted. The authorized bandwidth is 100 kHz." (§ 97.307(f)(6), applies to 70 cm via § 97.305(c)(5)(i))

So a project-defined binary packet format is legal as a matter of code choice; the only content test is the "purpose of obscuring meaning" language. (The ticket mentions § 97.117 in this context; that section is actually about international communications: "Transmissions to a different country, where permitted, shall be limited to communications incidental to the purposes of the amateur service and to remarks of a personal character." It does not concern digital codes and is irrelevant to a domestic bench test.)

The rules themselves carve out three places where encoding that hides meaning is permitted, which shows the FCC distinguishes control/authentication coding from message content:

> "(b) A telecommand station may transmit special codes intended to obscure the meaning of telecommand messages to the station in space operation." (§ 97.211(b))
> "(f) Space telemetry transmissions may consist of specially coded messages intended to facilitate communications or related to the function of the spacecraft." (§ 97.207(f))
> "(b) The control signals are not considered codes or ciphers intended to obscure the meaning of the communication." (§ 97.215(b), telecommand of model craft)

**Does § 97.211(b) apply to the bench test?** No. A telecommand station is defined by what it commands: "an amateur station that transmits communications to initiate, modify or terminate functions of a space station" (§ 97.3(a)(45)), and a space station is "an amateur station located more than 50 km above the Earth's surface" (§ 97.3(a)(41)). Eligibility also requires designation "by the licensee of a space station" (§ 97.211(a)). A simulated satellite on a bench is not a space station, so the project cannot claim § 97.211(b) and cannot encrypt command payloads under it. The rule is still useful as evidence of how the FCC thinks: it treats obscured telecommand codes as an exception that had to be written in, which implies ordinary stations may not do the same.

**Does an HMAC or signature count as obscuring meaning?** The rule turns on purpose ("for the purpose of obscuring their meaning"). An HMAC or digital signature is a fixed-length tag appended to a message whose content stays fully readable; its purpose is to let the receiver verify who sent the message and that it was not altered, not to hide what it says. A sequence counter and a CRC are plain integers. None of these hide the meaning of the command.

There is no FCC rule or order that says this in so many words. The closest primary statements are:

- The FCC's 2013 order dismissing a petition to allow encryption for emergency traffic (RM-11699, DA 13-1918, released 18 Sep 2013) restates the rule and its rationale: "To ensure that the amateur service remains a non-commercial service and self-regulates, amateur stations must be capable of understanding the communications of other amateur stations. The content of messages that are encoded, however, are known only to those stations that have the code used to encode the message." (DA 13-1918 at para. 6). The test the FCC articulates is whether another amateur listening can understand the content. A readable command with an appended tag passes that test; an encrypted command body does not.
- ARRL's comments in the same docket (filed 2013) report that "ARRL has previously advised members, following discussions with Commission Enforcement Bureau and Wireless Bureau staff, that encoding exclusively for authentication purposes does not violate Section 97.113(a)(4)," and that "encryption of control signals and coding for authentication of a transmission is not related to the message transmitted, which cannot be obscured intentionally" (ARRL Comments, RM-11699, paras. 10, 19-20). ARRL also concedes the rule "is somewhat subjective and it does not lend itself to a 'bright line' application" (para. 20). This is an ARRL characterization of informal staff views, not an FCC ruling.

### Safest reading for a student project

1. Keep the command payload itself in the clear. Anyone with Direwolf and the project's packet spec (which should be public in the repo) must be able to decode a frame and read what command it carries. That is exactly the "other amateurs can understand it" test from DA 13-1918.
2. Counters, CRCs, and an HMAC/signature tag are additions to a readable message, not a cipher over it. Document the field layout and the purpose of the tag in the packet spec, so the intent is evident from the design.
3. Do not encrypt the payload, even as a "realistic" challenge, on the amateur band. If a challenge needs an encrypted or obfuscated body, run that challenge over the software-only path (loopback / network), not over the IC-9700.
4. Do not rely on § 97.211(b). There is no space station.
5. Because the rule is purpose-based and ambiguous at the edges, and because the tag is a binary blob, also make sure the mode is squarely within what § 97.307(f)(5)/(6) allow (AFSK 1200 baud AX.25 is far below the 19.6 / 56 kilobaud limits and well inside 20 / 100 kHz), so the only remaining question is content, not code.

---

## 4. Technician privileges on 2 m and 70 cm; what "control operator" means with a scripted computer

### Frequency and mode privileges

> "(a) For a station having a control operator who has been granted a Technician, General, Advanced, or Amateur Extra Class operator license ...: 2 m 144-148 MHz ... 70 cm 420-450 MHz" (§ 97.301(a), ITU Region 2 column)

Technicians have full VHF/UHF band privileges; there is no Technician-specific sub-band on 2 m or 70 cm. Emission types (§ 97.305(c)):

- 2 m, 144.1-148.0 MHz: "MCW, phone, image, RTTY, data, test" (§ 97.305(c)(4)(iii)). The segment 144.0-144.1 MHz is not in the table, so only CW is authorized there (§ 97.305(a)). Stay above 144.1 MHz.
- 70 cm, entire band: "MCW, phone, image, RTTY, data, SS, test" (§ 97.305(c)(5)(i)).

Data standards: on 2 m, symbol rate up to 19.6 kilobaud and 20 kHz bandwidth (§ 97.307(f)(5)); on 70 cm, 56 kilobaud and 100 kHz (§ 97.307(f)(6)). AFSK 1200 baud in a ~3 kHz FM channel is far inside both.

Power: "An amateur station must use the minimum transmitter power necessary to carry out the desired communications" (§ 97.313(a)); the general ceiling is 1.5 kW PEP (§ 97.313(b)). Lowest power into an attenuator is the textbook case of § 97.313(a). One location-specific rule matters for Florida: on 70 cm, "No other station may transmit with a transmitter power exceeding 50 W PEP on the UHF 70 cm band from an area specified in § 2.106(c)(270)(i) of this chapter" (§ 97.313(f)), and footnote US270(a)(1) to 47 CFR 2.106 lists "Arizona, Florida and New Mexico" in full, plus a 322 km radius around Patrick AFB. The bench test at minimum power is nowhere near 50 W, but any future outdoor 70 cm test at UCF is capped at 50 W PEP.

Sharing: 70 cm is a secondary allocation. Amateur stations there "must not cause harmful interference to, and must accept interference from," US Government radiolocation (§ 97.303(b)) and foreign radiolocation in 430-450 MHz (§ 97.303(d)). On the bench this is moot; it matters if anything ever radiates.

Two Technician-only restrictions in § 97.307(f)(9)-(10) (CW-only / CW-and-SSB-only) apply only to HF segments and do not touch 2 m or 70 cm.

### Control operator

> "Control operator. An amateur operator designated by the licensee of a station to be responsible for the transmissions from that station to assure compliance with the FCC Rules." (§ 97.3(a)(13))
> "When transmitting, each amateur station must have a control operator. The control operator must be a person: (a) For whom an amateur operator/primary station license grant appears on the ULS consolidated licensee database ..." (§ 97.7)
> "(a) The station licensee is responsible for the proper operation of the station in accordance with the FCC Rules. ... (b) The station licensee must designate the station control operator. The FCC will presume that the station licensee is also the control operator, unless documentation to the contrary is in the station records." (§ 97.103(a)-(b))
> "(a) The control operator must ensure the immediate proper operation of the station, regardless of the type of control. (b) A station may only be operated in the manner and to the extent permitted by the privileges authorized for the class of operator license held by the control operator." (§ 97.105)
> "(a) The station apparatus must be under the physical control of a person named in an amateur station license grant ... before the station may transmit on any amateur service frequency ..." (§ 97.5(a))

The rules recognise three types of control (§ 97.3(a)(6), (31), (39); § 97.109):

- **Local control**: "The use of a control operator who directly manipulates the operating adjustments in the station to achieve compliance with the FCC Rules" (§ 97.3(a)(31)). "When a station is being locally controlled, the control operator must be at the control point. Any station may be locally controlled." (§ 97.109(b))
- **Remote control**: the same but via a control link (§ 97.3(a)(39), § 97.109(c), § 97.213). Not relevant unless the team wants to key the IC-9700 from another room.
- **Automatic control**: "The use of devices and procedures for control of a station when it is transmitting so that compliance with the FCC Rules is achieved without the control operator being present at a control point." (§ 97.3(a)(6)) "When a station is being automatically controlled, the control operator need not be at the control point. Only stations specifically designated elsewhere in this part may be automatically controlled." (§ 97.109(d))

For data on VHF/UHF, automatic control is one of the designated cases:

> "(b) A station may be automatically controlled while transmitting a RTTY or data emission on the 6 m or shorter wavelength bands ..." (§ 97.221(b))

### What this means for a scripted transmitter

- "Control operator present" is not about who typed the command; it is about who is responsible and where they are. A computer running a script is not a control operator (§ 97.7 requires a licensed person). While Diab sits at the bench with his hand able to reach the radio or kill the script, the station is under **local control**, he is at the control point, and the fact that software rather than a finger keys the radio does not change anything.
- If Diab leaves the room while the script keeps transmitting, the station is no longer under local control. It is only lawful as an **automatically controlled digital station** under § 97.221(b), which is available for data on 2 m/70 cm. Even then he remains fully responsible: "the control operator must ensure the immediate proper operation of the station, regardless of the type of control" (§ 97.105(a)), and automatic control "must cease upon notification by a Regional Director" of improper operation (§ 97.109(d)). Third-party content adds another restriction (section 5).
- Design implication: build in a hard stop. The script should not be able to transmit unattended by accident (a session time limit, a physical PTT enable, or running the script only when Diab has started it). This mirrors the safeguard the rules require for remotely controlled stations, "a period of no more than 3 minutes in the event of malfunction in the control link" (§ 97.213(b)), and is good practice even though § 97.213 does not strictly apply to local control.
- Station licence: § 97.5(a) requires the apparatus to be "under the physical control" of the licensee. Diab's operator/primary station licence covers this (§ 97.5(b)(1)); the IC-9700 does not need its own licence, and the FCC "equipment authorization program does not generally apply to amateur station transmitters" (FCC Amateur Radio Service page). The IC-R8600 is a receiver and is not regulated by Part 97 at all.
- Compensation: § 97.113(a)(3) forbids communications in which the control operator has a pecuniary interest, "including communications on behalf of an employer." Unpaid students doing coursework are not being compensated. If a paid faculty member ever acts as control operator, § 97.113(a)(3)(iii) covers "compensation as an incident of a teaching position ... as a part of classroom instruction at an educational institution."

---

## 5. An unlicensed teammate operating the software while Diab is present

### What the rules say

> "Third party communications. A message from the control operator (first party) of an amateur station to another amateur station control operator (second party) on behalf of another person (third party)." (§ 97.3(a)(47))
> "(b) The third party may participate in stating the message where: (1) The control operator is present at the control point and is continuously monitoring and supervising the third party's participation; and (2) The third party is not a prior amateur service licensee whose license was revoked ... [or] the subject of a cease and desist order which relates to amateur service operation and which is still in effect." (§ 97.115(b))
> "(c) No station may transmit third party communications while being automatically controlled except a station transmitting a RTTY or data emission." (§ 97.115(c))

And, restated from section 4: only a licensed person may be control operator (§ 97.7); the station apparatus must be under the physical control of a licensee (§ 97.5(a)); the control operator is responsible for every transmission (§ 97.3(a)(13), § 97.105(a)).

### What changes

- An unlicensed teammate can type commands, run the challenge script, or click "send" **only** while Diab is "present at the control point and is continuously monitoring and supervising" (§ 97.115(b)(1)). Diab does not have to touch the keyboard, but he must be there, paying attention, and able to stop the transmission. He, not the teammate, is legally responsible for whatever goes out (§ 97.105(a), § 97.103(a)).
- If Diab is not present, the teammate cannot operate the transmitter at all. There is no arrangement under which an unlicensed person becomes the control operator (§ 97.7), and the unattended-script route (automatic control under § 97.221(b)) still requires that the content be Diab's responsibility. § 97.115(c) does allow third-party content under automatic control for data emissions, so a queued command from a teammate that the script sends later is not prohibited by § 97.115(c) itself; but the whole arrangement then rests on Diab having set up and remaining responsible for an automatically controlled station, which is a heavier lift than simply being in the room.
- The definition in § 97.3(a)(47) is written around a message sent to "another amateur station control operator." On the bench there is no second control operator; the receiver is a passive IC-R8600. Whether teammate-originated commands into a receiver are "third party communications" at all is therefore ambiguous. The safe reading is to treat them as third-party participation and follow § 97.115(b) anyway, because that costs nothing (Diab is present regardless) and avoids the argument entirely.
- The disqualification in § 97.115(b)(2) (revoked/suspended former licensees, cease-and-desist subjects) will not apply to any student, but it is the one check the rule asks the control operator to make.
- Every frame still identifies as Diab's station (§ 97.119(a)); the teammate's name or a project handle never goes in the AX.25 source field. Putting a teammate's identifier elsewhere in the payload is fine (that is ordinary third-party content).

---

## Confirm with an experienced ham before the first session

Things the rules leave open or that depend on local practice. Bring this list to a club Elmer, the UCF amateur radio club, or an ARRL Technical Specialist.

1. **Dummy-load convention.** Confirm the local view: do experienced operators around here treat RF into a dummy load or closed coax as a transmission requiring ID (the conservative reading in section 1), and is the plan to ID via the AX.25 source field in every frame sufficient in their eyes?
2. **Leakage check.** Part 97 has no dummy-load exception partly because setups leak (§ 97.307(c) covers chassis radiation). Ask someone with a handheld or SDR to listen a few metres from the bench while transmitting at minimum power, to confirm nothing meaningful escapes. Also confirm the attenuator and connectors are rated for the IC-9700's true minimum output (the [bench research](rf-bench-attenuation.md) covers the numbers).
3. **HMAC / signature field.** Section 3's conclusion (tag on a readable payload is not "obscuring meaning") rests on the FCC's self-policing rationale in DA 13-1918 and ARRL's report of informal staff guidance, not on a rule or order that names authentication. Ask whether the ham agrees, and have the packet spec on hand to show that the command body stays in the clear.
4. **Automatic control.** Decide in advance whether the script will ever transmit without Diab in the room. If yes, confirm the § 97.221(b) automatically-controlled-digital-station reading and what safeguards (time limit, PTT interlock) an experienced operator would expect. If no, write "control operator present for all transmissions" into the test procedure and enforce it.
5. **Unlicensed teammates at the keyboard.** Confirm that having Diab present and watching (§ 97.115(b)(1)) is how local operators handle guests at a station, and agree on a simple rule (e.g. Diab starts and stops every session; teammates operate only between those points).
6. **Frequency choice.** Pick a specific 2 m frequency above 144.1 MHz (§ 97.305(c)(4)(iii)) or a 70 cm frequency, and ask which local simplex/packet frequencies to avoid (repeater inputs, APRS on 144.39 MHz, satellite sub-bands 145.8-146.0 and 435-438 MHz) so any leakage is harmless. Frequency coordination is a matter of "good amateur practice" (§ 97.101(a)-(b)), not a hard rule, but it is what an experienced operator will ask first.
7. **Florida 70 cm power cap.** Note for the record that § 97.313(f) / US270 caps 70 cm at 50 W PEP anywhere in Florida. Irrelevant on the bench, relevant if the project ever radiates on 70 cm.
8. **Licensee vs. control operator paperwork.** Diab is both licensee and control operator, so § 97.103(b)'s presumption holds and no record is needed. If anyone else licensed ever runs a session, note it in the session log (§ 97.103(b)).
9. **Simple session log.** Not required (section 1), but agree on a one-line-per-session log format and keep it with the project records so § 97.103(c) ("station records available for inspection") and the project's own reproducibility needs are covered.
10. **More than one licensed operator.** Encourage one or two more teammates to sit the Technician exam. It removes the § 97.115 question entirely, spreads the control-operator load, and is squarely within the stated purpose of the service ("expansion of the existing reservoir ... of trained operators, technicians, and electronics experts," § 97.1(d)).

---

## Section index

| Section | Topic | Used in |
| --- | --- | --- |
| 47 CFR 2.1 | Definitions of radiocommunication, radio waves | 1 |
| 47 CFR 2.106 footnote US270 | 70 cm 50 W areas (includes Florida) | 4 |
| § 97.1 | Basis and purpose | confirm list |
| § 97.3(a)(5), (6), (13), (31), (39), (41), (45), (47); (c)(2) | Definitions | 1-5 |
| § 97.5 | Station licence required; physical control | 4, 5 |
| § 97.7 | Control operator required; must be licensed | 4, 5 |
| § 97.101 | Good amateur practice | 1, confirm list |
| § 97.103 | Licensee responsibilities; station records | 1, 4 |
| § 97.105 | Control operator duties | 4, 5 |
| § 97.109 | Station control (local/remote/automatic) | 4 |
| § 97.111(b)(1) | Brief adjustment transmissions | 1 |
| § 97.113(a)(3), (a)(4) | Pecuniary interest; obscured meaning; false ID | 3, 4 |
| § 97.115 | Third party communications | 5 |
| § 97.117 | International communications (not relevant) | 3 |
| § 97.119 | Station identification | 2 |
| § 97.207(f) | Space telemetry coded messages | 3 |
| § 97.211 | Space telecommand station | 3 |
| § 97.213 | Telecommand of an amateur station (remote control) | 4 |
| § 97.215(b) | Model-craft control signals not "codes" | 3 |
| § 97.221(b) | Automatically controlled digital station | 4, 5 |
| § 97.301(a) | Technician bands incl. 2 m, 70 cm | 4 |
| § 97.303(b), (d) | 70 cm secondary to radiolocation | 4 |
| § 97.305(a), (b), (c)(4)(iii), (c)(5)(i) | Authorized emissions; test emissions | 1, 4 |
| § 97.307(c), (f)(5), (f)(6) | Spurious emissions; VHF/UHF data standards | 1, 3, 4 |
| § 97.309(a), (b) | Specified and unspecified digital codes | 2, 3 |
| § 97.313(a), (b), (f) | Minimum power; 1.5 kW; Florida 70 cm cap | 4 |
| FCC DA 13-1918 (RM-11699, 2013) | Encryption petition dismissal; self-policing rationale | 3 |
| FCC 82-456 (PR Docket 82-726) | Elimination of amateur logging | 1 |
| ARRL Comments, RM-11699 (2013) | Reports informal FCC staff view on authentication coding | 3 |
| AX.25 v2.2 § 3.12 | Address field encoding | 2 |

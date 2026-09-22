# CCSDS Space Packet framing and the Direwolf radio path

Research background; current product scope and implementation requirements are in the [documentation index](../specs/project-plan.md#build-specifications).

Protocol research for the packet format and later RF integration. Written for a reader who is not an RF or protocol
specialist; jargon is glossed on first use. Every factual claim carries an inline citation to
the primary source it was checked against. Where a claim is an engineering judgement rather than
a sourced fact, it is marked **[judgement]**. Where I could not verify something against a
primary source, it is marked **[unverified]**.

Related notes in this folder: `rf-bench-attenuation.md` (power levels, attenuator chain,
receiver protection) and `part-97-rules.md` (licensing). This note does not repeat them.

---

## 0. The proposal in one screen

**Packet format.** Use the [Packet format v1 specification](../specs/packet-format-v1.md) for the six-byte CCSDS primary header, binary command body, two APIDs, and CRC-16/CCITT-FALSE. The spec owns field assignments, message numbering, parsing rules, and worked examples. This note provides supporting protocol sources and the planned AX.25/Direwolf path for later RF work.

**Later radio-chain proposal.** Python builds the Space Packet, wraps it in an **AX.25 UI
frame** (the standard amateur packet-radio envelope, PID `0xF0`), wraps that in **KISS**
framing, and writes it to **Direwolf's TCP port 8001**. Direwolf turns the frame into AFSK
1200-baud audio, plays it out the IC-9700's USB sound device and keys the transmitter by
toggling the RTS (or DTR) line of one of the IC-9700's two virtual USB serial ports, which the
radio's `USB SEND` menu item maps to PTT. On the receive side the IC-R8600 puts demodulated
audio on its USB sound device; a second Direwolf decodes it and hands the AX.25 frame to the
Satellite Sim over KISS/TCP 8001. AX.25 UI frames are the right container: default payload
limit 256 bytes (Direwolf accepts up to 2048), and it is what most amateur-band CubeSats and
the AFSK1200 challenges in the HTB satellite track use. Details in §3.

---

## 1. Sources consulted

| Short name used below | Document | Where |
|---|---|---|
| **SPP** | CCSDS 133.0-B-2, *Space Packet Protocol*, Recommended Standard Issue 2, June 2020, with Editorial Changes 1 (Oct 2020) and 2 (Sep 2024). This is the current issue; Issue 1 (2003) is superseded. [SPP Document Control, p. v] | https://ccsds.org/Pubs/133x0b2e2.pdf |
| **TC-SDLP** | CCSDS 232.0-B-4 Cor. 1, *TC Space Data Link Protocol*, Issue 4, Oct 2021 / Cor. 1 Oct 2023 | https://ccsds.org/Pubs/232x0b4e1c1.pdf |
| **PUS** | ECSS-E-ST-70-41C, *Telemetry and telecommand packet utilization*, 15 April 2016 (clause 7.4.4.2 checked; Annex B body was not in the copy I could access) | https://ecss.nl (registration required for the PDF) |
| **AX.25** | *AX.25 Link Access Protocol for Amateur Packet Radio*, Version 2.2, TAPR/ARRL, July 1998 revision | https://ax25.net/AX25.2.2-Jul%2098-2.pdf (the tapr.org URL is currently 404) |
| **KISS** | Chepponis & Karn, *The KISS TNC: A simple Host-to-TNC communications protocol*, 1987 | https://www.ax25.net/kiss.aspx |
| **DW-UG** | Dire Wolf User Guide, Version 1.8, October 2025 (`doc/User-Guide.pdf` in the wb2osz/direwolf repo, master) | https://github.com/wb2osz/direwolf |
| **DW-RIG** | Dire Wolf Radio Interface Guide (in the separate wb2osz/direwolf-doc repo) | https://github.com/wb2osz/direwolf-doc |
| **DW-SRC** | Direwolf source: `src/ax25_pad.h`, `src/kiss_frame.h` (master) | same repo |
| **9700-B** | Icom IC-9700 Basic Manual (local PDF, `Documentation/Radios/`) | local |
| **9700-A** | Icom IC-9700 Advanced Manual (local PDF) | local |
| **9700-CIV** | Icom IC-9700 CI-V Reference Guide (local PDF) | local |
| **R8600-IM** | Icom IC-R8600 Instruction Manual (local PDF) | local |
| **R8600-CIV** | Icom IC-R8600 CI-V Reference Guide (local PDF) | local |
| **Hamlib** | `include/hamlib/riglist.h`, Hamlib master | https://github.com/Hamlib/Hamlib |
| **HTB** | Hack The Box blog, "Hack the Orbit: The HTB Satellite Exploitation Track Is Live", 13 Aug 2026 | https://www.hackthebox.com/blog/hack-the-orbit-satellite-exploitation-track |
| **gr-sat** | gr-satellites documentation, "Components" page, v5.1.1 | https://gr-satellites.readthedocs.io/en/v5.1.1/components.html |

Page numbers for the Icom manuals are the printed page numbers (e.g. "p. 8-15" = chapter 8,
page 15), which is how the manuals cross-reference themselves.

---

## 2. Part 1: CCSDS Space Packet Protocol

### 2.1 What a Space Packet is, in plain language

A *Space Packet* is the standard envelope that spacecraft and ground systems use to carry one
command or one telemetry report. It is nothing more than a fixed 6-byte header followed by a
variable-length payload. The header says "who this is for / who it came from" (the APID),
"which one in the series this is" (sequence count), and "how long the payload is". The
standard deliberately says almost nothing about what goes *inside* the payload, and nothing at
all about how the packet gets from A to B; that is left to lower layers (radio framing) and to
the project. [SPP §2.4, p. 2-8: "SPP itself has no networking capabilities and fully relies on
the services provided by the applicable subnetworks."]

Size limits: the whole packet is between 7 and 65,542 bytes; the header is exactly 6 bytes;
the payload ("Packet Data Field") is 1 to 65,536 bytes. [SPP §4.1.2.1 and §4.1.2.2, p. 4-1]

Bit numbering: CCSDS numbers bits from 0 at the left; bit 0 is the first bit transmitted and
is the most significant bit of any numeric field. Fields are grouped into 8-bit octets (bytes),
so in practice every field below is **big-endian** ("network byte order"). All spare bits
must be 0. [SPP §1.6.3 and Figure 1-1, p. 1-5]

### 2.2 Primary header, bit by bit

The header is four fields, 48 bits total, in this order. [SPP §4.1.3.1 and Figure 4-2, p. 4-2]

| Bits | Width | Field | Value / meaning | Source |
|---|---|---|---|---|
| 0-2 | 3 | Packet Version Number | Must be `000` ("Version 1 CCSDS Packet"). Reserved so other packet structures could be introduced later. | SPP §4.1.3.2, p. 4-2 |
| 3 | 1 | Packet Type | `0` = telemetry (reporting), `1` = telecommand (requesting). "The exact definition of 'telemetry Packets' and 'telecommand Packets' needs to be established by the project." | SPP §4.1.3.3.2, p. 4-3 |
| 4 | 1 | Secondary Header Flag | `1` if a Packet Secondary Header is present, `0` if not. Must be constant for a given APID for a whole mission phase. Must be `0` for Idle Packets. | SPP §4.1.3.3.3, p. 4-3 |
| 5-15 | 11 | APID (Application Process Identifier) | Names the "managed data path", in practice the on-board application or subsystem a command is for (or a report is from). Any value 0-2046 may be used; they need not be consecutive. `0x7FF` (all ones) is reserved for Idle Packets. | SPP §4.1.3.3.4, pp. 4-3 to 4-4 |
| 16-17 | 2 | Sequence Flags | `11` = unsegmented (whole message in one packet); `01` = first segment; `00` = continuation segment; `10` = last segment. "The use of the Sequence Flags is not mandatory for the users of the SPP." If the Octet String service is used anywhere, must always be `11`. | SPP §4.1.3.4.2, pp. 4-4 to 4-5 |
| 18-31 | 14 | Packet Sequence Count *or* Packet Name | For telemetry (Type 0): always a sequence count. For telecommand (Type 1): either a sequence count or a "Packet Name" (any 14-bit pattern the project likes). See §2.4. | SPP §4.1.3.4.3, p. 4-5 |
| 32-47 | 16 | Packet Data Length | Number of bytes in the Packet Data Field **minus one**. See §2.3. | SPP §4.1.3.5, p. 4-6 |

Bytes 0-1 (version, type, flag, APID) are together called the *Packet Identification Field*;
bytes 2-3 (flags + count) are the *Packet Sequence Control Field*. [SPP §4.1.3.3.1, §4.1.3.4.1]

The Packet Data Field that follows the header consists of an optional Packet Secondary Header
and/or a User Data Field; at least one must be present and the field must contain at least one
byte. [SPP §4.1.4.1, p. 4-6] If a secondary header is used it must be an integral number of
bytes, its contents are project-defined (typically a time code and/or "ancillary data"), and the
choice must stay fixed for a given APID. [SPP §4.1.4.2.1, pp. 4-6 to 4-7]

### 2.3 The "minus one" rule for Packet Data Length

The 16-bit length field holds `C = (number of bytes in the Packet Data Field) - 1`. [SPP
§4.1.3.5.2 and §4.1.3.5.3, p. 4-6] So:

- A packet whose payload is 4 bytes has length field `3`.
- A packet whose payload is 1 byte (the minimum) has length field `0`.
- The maximum value `65535` means a 65,536-byte payload.
- Total packet length in bytes = `6 + C + 1 = C + 7`.

This is the single most common interoperability bug in student implementations, and it is a
good candidate for a deliberate "malformed packet" challenge (§2.6). Note that the length
counts *everything after the 6-byte header*, including any secondary header and, in our
proposal, the appended CRC.

### 2.4 How the sequence count is meant to behave

- It is a per-APID counter: "Packet Sequence Counts are unique and independent per each user
  application as identified by the APID and are not shared across multiple APIDs." [SPP
  §4.1.3.4.3.3, p. 4-5]
- It increments by one per packet and wraps modulo 16384 (14 bits). Resetting it before it
  reaches 16383 "shall not take place unless it is unavoidable." [SPP §4.1.3.4.3.4, p. 4-5]
- Its purpose is to let the receiver re-order packets and detect missing ones. [SPP §4.1.3.4.3
  NOTE 1] The standard's receive-side "Packet Extraction Function checks the continuity of the
  Packet Sequence Count to determine if one or more Packets have been lost" and raises an
  optional "Data Loss Indicator". [SPP §4.3.2.2, p. 4-11; §3.4.2.4, p. 3-6]
- After an unavoidable reset "the completeness of a sequence of Packets cannot be determined."
  [SPP §4.1.3.4.3 NOTE 2]
- For telecommands the field may instead hold a "Packet Name", an arbitrary 14-bit identifier
  the ground uses to tag a command so a later acknowledgement can refer to it. [SPP §4.1.3.4.3.2
  and NOTE 4, p. 4-5]

For this project: use a true sequence count on both directions (it gives the Satellite Sim a
natural replay-detection hook, which is exactly what HTB's "Echoes in Orbit" challenge exercises:
"sync a packet counter, then get it to hand you the flag" [HTB]). Keep one counter per APID on
each side.

### 2.5 CRC / checksum: what the standard says, and what real systems do

**The Space Packet Protocol defines no CRC, checksum, or any other integrity field.** The word
"CRC" does not appear in the document. Integrity is delegated:

- To the link layer below it: SPP "relies on the Packet Services provided by the Space Data Link
  Protocols (i.e., TM, TC, AOS, Proximity-1, and USLP)". [SPP §2.4, p. 2-8]
- For security (authentication, confidentiality, integrity), to the Space Data Link Security
  protocol at the link layer or Bundle Security Protocol at the network layer: "The SPP does
  not provide any security function." [SPP Annex B1, p. B-1]

What real systems do, from most standard to most common-in-CubeSats:

1. **CRC-16 at the CCSDS frame layer.** The TC Space Data Link Protocol's transfer frame has an
   optional 2-byte *Frame Error Control Field* (FECF) computed by CRC with generator polynomial
   `G(X) = X^16 + X^12 + X^5 + 1` and the shift register **preset to all ones**; the FECF covers
   the whole frame before it. [TC-SDLP §4.1.4.1 and §4.1.4.2.2, pp. 4-9 to 4-10] In modern
   CRC-catalogue terms that is **CRC-16/CCITT-FALSE** (also called CRC-16/IBM-3740): poly
   `0x1021`, init `0xFFFF`, no bit reflection, no final XOR, check value for ASCII "123456789"
   = `0x29B1`. Its presence "shall be established by management", i.e. it is a per-mission
   choice. [TC-SDLP §4.1.4.1.1]
2. **CRC-16 appended to the packet itself (ECSS PUS).** The European PUS standard, which most
   ESA-heritage missions and many university CubeSats follow, ends every telecommand packet with
   a 16-bit *packet error control* field, computed over the whole packet once all other fields
   are complete; the mission chooses between "the ISO standard 16-bits checksum or the CRC
   standard 16-bits checksum" (Annex B.1). [PUS §7.4.4.2 a, d, e; Figure 7-10] The CRC option
   in PUS Annex B is the same polynomial/preset as the CCSDS FECF **[unverified against Annex B
   text; I could only access clause 7]**; the actively maintained `spacepackets` Python library
   implements PUS TC with `fastcrc.crc16.ibm_3740`, which is CRC-16/CCITT-FALSE
   [`src/spacepackets/ecss/tc.py` lines 313-315, us-irs/spacepackets-py master].
3. **CRC at the radio frame layer (amateur CubeSats).** Satellites that transmit AX.25 get a
   16-bit *Frame Check Sequence* on every radio frame for free; see §3.3. The receiver discards
   any frame whose FCS does not match. [AX.25 §3.7 and §4.4.6] gr-satellites' AX.25 deframer
   likewise "performs NRZ-I decoding, frame boundary detection, bit de-stuffing, and CRC-16
   checking". [gr-sat, "AX.25 deframer"]
4. **HTB's challenges** model option 1/2: challenge 4 "No Errors" is "Same target, protected
   transmission mode. Build a fully valid frame, CRC and all, per the actual CCSDS spec." [HTB]

So the honest summary is: *the Space Packet standard has no CRC; every real system that cares
puts a CRC-16 with poly 0x1021 / init 0xFFFF either just below the packet (frame FECF) or just
after it (PUS packet error control).* Our proposal (§2.9) picks the second.

### 2.6 "Malformed" faults a receiver is expected to detect

The Blue Book itself is nearly silent on receiver-side validation: the only checks it names are
sequence-count continuity [SPP §4.3.2.2] and demultiplexing by APID [SPP §4.3.3.2]. The
following list is therefore **[judgement]**, derived from each "shall" in §4.1 plus the CRC
convention. The Satellite Sim should reject (and, for the challenge platform, log with a
distinct reason code) a packet that:

| # | Fault | Rule violated |
|---|---|---|
| 1 | Fewer than 6 bytes received (no complete header) | SPP §4.1.2.1 |
| 2 | Version != `000` | SPP §4.1.3.2.2 |
| 3 | Length field disagrees with bytes actually received (`len(packet) != C + 7`). Both "too short" (truncated) and "too long" (trailing garbage) cases. | SPP §4.1.3.5 |
| 4 | Packet Data Field empty (impossible to encode: `C=0` means 1 byte) but a payload of 0 bytes could be *attempted* by a buggy sender producing a 6-byte packet | SPP §4.1.4.1.2 |
| 5 | Unknown / unrouted APID, or the Idle APID `0x7FF` arriving as a command | SPP §4.1.3.3.4.4, §4.3.3.2 |
| 6 | Packet Type wrong for the direction (a Type-0 telemetry packet arriving on the uplink) | SPP §4.1.3.3.2, project rule |
| 7 | Secondary Header Flag differs from the fixed value declared for that APID | SPP §4.1.3.3.3.3 |
| 8 | Sequence Flags != `11` when the project does not segment | SPP §4.1.3.4.2.3, project rule |
| 9 | Sequence count discontinuity (gap = lost packets; repeat = replay; backwards = replay or reorder) | SPP §4.1.3.4.3.4, §4.3.2.2 |
| 10 | CRC mismatch (our convention, §2.9) | project rule |
| 11 | Body shorter than the command's declared argument structure (only checkable after 1-10 pass) | project rule |

Order matters for a security platform: check length and CRC *before* interpreting anything
else, so a fuzzed length field cannot cause an out-of-bounds read. The AX.25 layer will already
have thrown away radio frames with a bad FCS before the Satellite Sim sees them (§3.3), so faults
10 and 9 are the ones a challenger can actually inject once the radio bench is in use.

### 2.7 Python libraries that encode/decode Space Packets

Checked on PyPI and GitHub on 2026-09-17.

| Library | Repo | Licence | Latest release | Activity | What it is good for |
|---|---|---|---|---|---|
| **spacepackets** | github.com/us-irs/spacepackets-py | Apache-2.0 | 0.32.0, 2026-05-03 | pushed 2026-05 | Space Packet primary header (`SpacePacketHeader`, pack/unpack, `MAX_APID`, `MAX_SEQ_COUNT` constants), plus ECSS PUS TC/TM with the CRC-16 already wired in, CCSDS time codes, CFDP. Python >= 3.9, depends on `fastcrc`. Best fit for this project. |
| **ccsdspy** | github.com/CCSDSPy/ccsdspy | BSD-3-Clause | 2.0.1, 2026-08-13 | pushed 2026-08 | NumPy-based *decoder* of fixed/variable-length telemetry packet streams ("used in flight missions"). Oriented to science-data pipelines, not to building commands. |
| **space_packet_parser** | github.com/lasp/space_packet_parser | BSD-3-Clause (per PyPI) | 6.2.0, 2026-09-13 | pushed 2026-09 | XTCE-driven telemetry decoding (you describe the packet in an XML dictionary). Heavyweight for our needs. |

**Implementation reference [judgement]:** the six-byte header can be encoded as three 16-bit words with Python `struct`; the [packet specification](../specs/packet-format-v1.md) owns the required output. These library comparisons are research, not required dependencies. The MVP supplies an encoder/decoder; Players do not implement one.

### 2.8 Project packet format

[Envelope and limits](../specs/packet-format-v1.md#envelope-and-limits) defines header fields and per-Attempt sequence counts. [Command body](../specs/packet-format-v1.md#command-body) defines the binary commands. Use the [worked Hello, Satellite exchange](../specs/packet-format-v1.md#worked-hello-satellite-exchange) for example bytes.

### 2.9 CRC convention and rationale

The project’s [Checksum and minimum parsing](../specs/packet-format-v1.md#checksum-and-minimum-parsing) rules define CRC presence and validation. Script delivery uses the [raw WebSocket endpoint](../specs/backend-and-simulation.md#packet-delivery-and-helper).

**Convention.** Append a 16-bit CRC to every Space Packet, as the final two bytes of the Packet
Data Field, big-endian, computed over every preceding byte of the packet (primary header,
optional secondary header, body). Algorithm: **CRC-16/CCITT-FALSE**: polynomial `0x1021`
(`X^16 + X^12 + X^5 + 1`), initial value `0xFFFF`, input and output not reflected, no final XOR.
Test vector: `crc(b"123456789") == 0x29B1`. Verification shortcut: CRC over the whole packet
including its CRC equals `0x0000`.

**Rationale.**

1. It is the same CRC the CCSDS telecommand frame layer uses (generator `X^16+X^12+X^5+1`,
   register preset to all ones) [TC-SDLP §4.1.4.2.2], and the same one ECSS PUS packets carry as
   their packet error control field [PUS §7.4.4.2; spacepackets `tc.py`]. Students who go on to
   HTB's "No Errors" challenge or to real mission software will meet exactly this CRC.
2. Putting it *inside* the Space Packet (PUS style) rather than in a separate frame means the
   same packet bytes can be used over the Software Link and the later RF Link, with no
   second framing layer of our own to design.
3. The AX.25 FCS on the radio bench uses a *different* CRC-16 variant (reflected, final XOR
   `0xFFFF`, catalogue name CRC-16/X-25, check value `0x906E`) [AX.25 §3.7 references ISO 3309
   HDLC], and Direwolf silently drops frames that fail it (§3.3). Because the modem hides that
   layer, **our appended CRC is the only integrity check a challenger can see, corrupt, or be
   asked to recompute.** That makes it the right place for "malformed packet" challenges.
4. Cost is 2 bytes per packet; irrelevant at our sizes.

Reference implementation (pure Python, ~10 lines, no dependency):

```python
def crc16_ccitt_false(data: bytes) -> int:
    crc = 0xFFFF
    for b in data:
        crc ^= b << 8
        for _ in range(8):
            crc = ((crc << 1) ^ 0x1021) & 0xFFFF if crc & 0x8000 else (crc << 1) & 0xFFFF
    return crc
```

Cross-check against `fastcrc.crc16.ibm_3740` or `spacepackets` in tests.

---

## 3. Part 2: bytes to radio via Direwolf

### 3.1 The layer cake, plainly

```text
Ground Sim (Python)                                   Satellite Sim (Python)
  Space Packet  (6-byte header + body + CRC)            Space Packet
  |                                                     ^
  AX.25 UI frame (addresses + control + PID + payload)  AX.25 UI frame
  |                                                     ^
  KISS framing (C0 00 ... C0) over TCP :8001            KISS over TCP :8001
  v                                                     |
Direwolf (software modem, "TNC")                     Direwolf
  |  AFSK 1200-baud audio out                            ^  audio in
  |  PTT via RTS/DTR on virtual COM port                 |
  v                                                     |
IC-9700 (FM-D mode, USB audio in, lowest power)      IC-R8600 (FM, USB audio out)
  |                                                     ^
  coax + attenuator chain (see rf-bench-attenuation.md) |
  +-----------------------------------------------------+
```

Glossary: a **TNC** ("terminal node controller") is the historical name for the box between a
computer and a radio that turns bytes into tones and back; Direwolf is a TNC in software.
**AFSK 1200** means each bit is sent as one of two audio tones (1200 Hz and 2200 Hz) at 1200
bits per second, fed into an ordinary FM voice transmitter. [DW-UG §7.1, p. 39, startup line:
"1200 baud, AFSK 1200 & 2200 Hz"] **PTT** ("push to talk") is the signal that switches the
transmitter on. **AX.25** is the amateur packet-radio link protocol that defines the frame
around your bytes. **KISS** is the trivially simple byte protocol between a computer program
and a TNC.

### 3.2 Handing bytes to Direwolf: KISS over TCP (and AGW)

Direwolf listens for KISS clients on TCP port **8001** by default (`KISSPORT 8001`); up to 3
clients may connect at once, and setting the port to 0 disables it. [DW-UG §4.5, p. 18 and §9.5.2,
p. 83] At start-up it prints `Ready to accept KISS TCP client application 0 on port 8001 ...`.
[DW-UG §7.1, pp. 39-40] Since 1.7 you can bind separate ports to separate radio channels
(`KISSPORT 8001 1`). [DW-UG §9.5.2.1, p. 84] Serial-port KISS (`SERIALKISS`) and, on Linux, a
pseudo-terminal are also available but unnecessary here. [DW-UG §9.5.3-9.5.5]

The KISS wire format [KISS §2-4]:

- Each frame is bracketed by `FEND` = `0xC0`. Two consecutive `FEND`s do not mean an empty frame.
- Inside a frame, a literal `0xC0` is sent as `0xDB 0xDC` (`FESC TFEND`) and a literal `0xDB`
  as `0xDB 0xDD` (`FESC TFESC`).
- The first byte after `FEND` is a type byte: high nibble = TNC port (0 for a single radio),
  low nibble = command. Command `0` = "data frame: the rest of the frame is data to be sent on
  the HDLC channel". Commands 1-5 set TXDELAY, persistence, slot time, TX tail and full-duplex;
  Direwolf honours these (`KISS_CMD_*` in `src/kiss_frame.h`) and warns that a client setting
  TXTAIL to 0 "is asking for trouble" because PTT may drop before the last bits leave. [DW-UG
  §9.2.15, p. 77] Do not send them; use the config file.
- "No CRC or checksum is provided" in KISS itself. [KISS §2] The AX.25 FCS, flags and bit
  stuffing are added by the TNC (Direwolf), so **the KISS payload is the AX.25 frame from the
  first address byte to the last payload byte, with no FCS**.
- Direwolf's buffer is `MAX_KISS_LEN 2048` bytes ("Spec calls for at least 1024"). [DW-SRC
  `kiss_frame.h`]

Minimal Python sender (no library needed):

```python
import socket
FEND, FESC, TFEND, TFESC = 0xC0, 0xDB, 0xDC, 0xDD

def kiss_wrap(ax25_frame: bytes, port: int = 0) -> bytes:
    body = ax25_frame.replace(bytes([FESC]), bytes([FESC, TFESC])) \
                     .replace(bytes([FEND]), bytes([FESC, TFEND]))
    return bytes([FEND, (port << 4) | 0x00]) + body + bytes([FEND])

with socket.create_connection(("127.0.0.1", 8001)) as s:
    s.sendall(kiss_wrap(ax25_frame))
```

The receive side reads the stream, splits on `0xC0`, drops empty pieces, strips the type byte,
un-escapes, and gets back the AX.25 frame (again without FCS: Direwolf has already verified and
removed it). Existing Python KISS libraries exist (`kiss3` 8.0.0, 2022; `pyham_kiss` 1.0.0,
2024; `aioax25` 0.0.11, GPL-2.0, 2023) but none is clearly maintained and the protocol is four
special bytes, so **[judgement]** write it yourself.

**AGW.** Direwolf also offers the "AGW TCPIP Socket Interface" on port **8000** (`AGWPORT`).
It is only needed if you want Direwolf's AX.25 *connected* mode (acknowledged, retransmitted
sessions): "KISS is adequate for APRS which uses only UI frames. If you want to access the
connected mode implementation within direwolf, you need to use the AGW network interface."
[DW-UG §9.5.1, p. 83] We use UI frames, so AGW is not relevant; disable it (`AGWPORT 0`) to
reduce the attack surface on the bench computers.

### 3.3 Are AX.25 UI frames the right container?

Yes, for these reasons.

**What a UI frame is.** AX.25's *Unnumbered Information* frame "contains PID and information
fields and passes information along the link outside the normal information controls... Because
these frames cannot be acknowledged, if one such frame is obliterated, it cannot be recovered."
[AX.25 §4.3.3.6] It is the connectionless datagram of packet radio; APRS is built on it. Layout
[AX.25 Figure 3.1b, §3.1-3.7]:

```text
Flag | Dest addr (7) | Src addr (7) | [digipeaters, 0-8 x 7] | Control (1) | PID (1) | Info (0..N) | FCS (2) | Flag
```

- Addresses are 6 upper-case letters/digits, space-padded, plus an SSID byte; the last address
  has its extension bit set. Direwolf supports up to 8 digipeater addresses (`AX25_MAX_ADDRS 10`).
  [DW-SRC `ax25_pad.h`]
- Control byte for UI = `0x03` (P/F bit clear). [AX.25 §4.3.3, Figure 4.4]
- PID `0xF0` = "No Layer 3 Protocol": the payload is raw bytes for the application. [AX.25 §3.4,
  Figure 3.2] This is what to use for a Space Packet payload.
- FCS: 16-bit CRC "calculated in accordance with recommendations in the HDLC reference document,
  ISO 3309", transmitted MSB first while everything else goes LSB first. [AX.25 §3.7-3.8] A frame
  with an FCS error "is discarded with no further action taken." [AX.25 §4.4.6] Direwolf does
  exactly this unless you turn on its experimental bit-flipping `FIX_BITS`, which its author says
  "I don't recommend using". [DW-UG §9.2.7, pp. 65-66]

**Payload limit.** The AX.25 information field "defaults to a length of 256 octets", negotiable
in connected mode (parameter N1). [AX.25 §3.5, §6.7.2.1] Direwolf's compiled limit is
`AX25_MAX_INFO_LEN 2048` [DW-SRC `ax25_pad.h`] and its connected-mode `PACLEN` is "Default 256.
Maximum 2048." [DW-UG §10, p. 144]. A 256-byte payload at 1200 baud takes about 1.7 s on air
(256 x 8 / 1200) plus ~0.4 s of TXDELAY/TXTAIL, so **[judgement]** keep every Space Packet
(header + body + CRC) at or under 256 bytes; our commands will be tens of bytes.

**What real CubeSats do.** AFSK 1200 with unscrambled AX.25 UI frames is the de-facto standard
for amateur-band CubeSat telemetry and the mode gr-satellites, SatNOGS and Direwolf all decode;
gr-satellites documents that "G3RUH scrambling is typically used for faster baudrates, such as
9k6 FSK packet radio, but not for slower baudrates, such as 1k2 AFSK packet radio." [gr-sat,
"AX.25 deframer"] Some CubeSats put CCSDS Space Packets *inside* those AX.25 frames (PicSat, from
Observatoire de Paris, is a documented example: "PicSat's packets are based on the CCSDS Space
Packet Protocol" carried in 1k2 AX.25 frames; Daniel Estevez, destevez.net, Jan 2018
**[secondary source]**). So our stack (Space Packet in AX.25 UI, AFSK 1200) is a faithful
miniature of a real small-satellite link, not an invention.

**What HTB does.** The HTB Satellite Exploitation Track's challenge 8 "Signal from Space" has
you "Demodulate a real AFSK1200 transmission captured off a live overpass", and the capstone
"Space Ops" has you "Build the frame, modulate it yourself, transmit it, receive the response,
demodulate it, complete the handshake. The SDR modem is exposed raw over ZMQ endpoints." [HTB]
HTB does not publish the framing, but "a real AFSK1200 transmission captured off a live
overpass" is, by the practice above, almost certainly AX.25. Our bench replaces HTB's ZMQ
software modem with real radios and Direwolf; the byte-level skills transfer directly.

**Two things AX.25 does not give us.** No acknowledgement (UI frames are fire-and-forget) and no
authentication. Both are features for a security challenge platform: the Satellite Sim can
implement its own ack/verification packets (PUS service 1 style), and "no authentication" is
the whole point of the exercise.

### 3.4 Direwolf -> IC-9700 (transmit side)

**How the radio appears to the computer.** One USB type-B cable. The radio's `[USB]` port
carries: "Outputting the demodulated AF signal or 12 kHz IF signal. Inputting the modulation AF
signal. Interface for remote control by CI-V commands." [9700-B p. 13-2] On the computer it
enumerates as one USB audio device plus **two virtual serial (COM) ports**, "USB (A)" and
"USB (B)": "When you connect the transceiver to a PC with a USB cable, 2 COM ports are recognized
on the PC. To confirm USB (A)/USB (B), open the COM port properties, and confirm the 'Value' of
the 'Details' tab." [9700-B p. 8-15] "USB (A) is used for programming, or CI-V operation." [9700-B
p. 8-17] A Windows USB driver is downloadable from Icom [9700-B p. 13-2]; Linux needs none
(the ports appear as `/dev/ttyUSB0` and `/dev/ttyUSB1`, exactly as Direwolf describes for
"transceivers, like the IC-7100, that have a USB-to-serial converter built in" [DW-UG §9.2.11.1,
p. 68]). The audio device name is not stated in the Icom manual; it appears as **"USB Audio
CODEC"** on Windows/Linux/macOS **[unverified for the 9700 specifically; this is the generic TI
PCM29xx codec name, and Direwolf's own examples show `08bb 2904 USB Audio CODEC`, DW-UG §9.1.3
p. 56]**; confirm from Direwolf's start-up device listing [DW-UG §7.1, p. 39].

**Radio settings (all under MENU > SET > Connectors unless noted):**

| Setting | Value | Why | Source |
|---|---|---|---|
| Operating mode | `FM-D` (FM with the DATA key) on 144 or 430 MHz | Data mode routes the PC's USB audio to the modulator and can mute the mic. | 9700-B p. 3-3 ("Selecting the Data mode") |
| `MOD Input > DATA MOD` | `USB` (default is `ACC`) | Selects which connector supplies the modulation audio when data mode is ON. | 9700-B p. 8-15 |
| `MOD Input > USB MOD Level` | start at default 50 %, then adjust for ~3-3.5 kHz deviation | Direwolf: "The optimal peak FM deviation is somewhere around 3 or 3.5 kHz... Too high will cause distortion." Use `direwolf -x m` for a test tone. | 9700-B p. 8-15; DW-RIG §3 p. 9 |
| `USB SEND/Keying > USB SEND` | `USB (B) RTS` (recommended; `USB (B) DTR`, `USB (A) RTS/DTR` also possible) | Maps a serial control line to PTT: "You can control transmit, receive... from the PC through the USB port... assigned to the DTR/RTS terminals in the virtual port." Choosing port B leaves port A free for CI-V. | 9700-A p. 7-7; 9700-B p. 8-15 |
| `USB SEND/Keying > Inhibit Timer at USB Connection` | leave `ON` (default) | Suppresses a spurious key-up when a PC connects or opens the port: "The transceiver transmits after a few seconds have passed, to prevent unintentional transmission." | 9700-B p. 8-16 |
| `USB SEND/Keying > USB Keying (CW)` and `(RTTY)` | `OFF` | You cannot assign the same line to SEND and Keying. | 9700-B p. 8-15 |
| `USB (B)/DATA Function > USB (B) Function` | `OFF` (default) | Keeps port B a plain serial port for the PTT line. | 9700-B p. 8-17 |
| `CI-V > CI-V USB Port` | `Unlink from [REMOTE]` (default) | Port A is then an independent CI-V port at address `A2h`, `CI-V USB Baud Rate` Auto. | 9700-B p. 8-16 |
| `USB AF/IF Output > Output Select` / `AF Output Level` / `AF SQL` | `AF`, 50 %, `OFF (Open)` | Only matters if the ground Direwolf also listens on the 9700 (for its DCD); leave squelch open per Direwolf's advice. | 9700-B p. 8-15; DW-UG §3.1 p. 9 |
| RF POWER (Multi-function menu) and `TX PWR LIMIT` | minimum (0.5 W on 144/430 MHz) | See `rf-bench-attenuation.md`. | 9700-B p. 3-10, p. 11-2 |

**PTT options in Direwolf, compared.** Direwolf lists "up to five different methods": serial
port control lines; GPIO pins (Linux only); parallel printer port (Linux only); hamlib (optional,
Linux only); CM108/CM119 USB-audio GPIO (optional, Linux only); and VOX. [DW-UG §9.2.11, p. 67]

| Method | Config line | Works on | Verdict for the IC-9700 |
|---|---|---|---|
| **RTS/DTR on the IC-9700's own virtual COM port** | Linux: `PTT /dev/ttyUSB1 RTS`; Windows: `PTT COM4 RTS` (whichever is USB (B)) | Linux and Windows | **Use this.** Single cable, no extra hardware, matches the radio's `USB SEND` feature. Polarity: "Normally the higher voltage is used for transmit. Prefix the control line name with '-' to get the opposite polarity." [DW-UG §9.2.11.1, p. 68] |
| CAT / CI-V via hamlib | `PTT RIG 3081 /dev/ttyUSB0` or, better, run `rigctld -m 3081 -r /dev/ttyUSB0` and use `PTT RIG 2 localhost:4532` | Linux only, and only if Direwolf was built with hamlib (`Includes optional support for: ... hamlib`) | Works: Hamlib model `3081` = IC-9700 (`RIG_MAKE_MODEL(RIG_ICOM, 81)`, `RIG_ICOM 3`, 1000 models per backend) [Hamlib `riglist.h`]; the underlying CI-V command is `1C 00` with data `01` = TX, `00` = RX, sent as `FE FE A2 E0 1C 00 01 FD` [9700-CIV p. 3 data format; command table `1C 00`]. Slower than a control line and unavailable on Windows builds of Direwolf. Keep CI-V for setting frequency/mode/power from a script instead. |
| GPIO / CM108 | `PTT GPIOD 0 25` / `PTT CM108` | Linux SBCs; CM108-based USB audio dongles | Not applicable: the 9700 has its own codec, not a CM108, and the bench PCs are not Raspberry Pis. |
| VOX (radio's own) | none | any | "Using VOX built in to a transceiver is generally a bad idea... keep the transmitter on about a half second after the transmit audio has ended." [DW-UG §9.2.11] The 9700 has no data VOX anyway. |
| ACC socket SEND pin (pin 3) via an external interface | `PTT /dev/ttyUSBx RTS` on a USB-serial adapter wired to ACC pin 3 | any | Works electrically ("When this pin goes to ground, the transceiver transmits" [9700-B p. 13-1]) but needs a cable/interface; pointless when the USB SEND path exists. |

**Ground-side `direwolf.conf` (Linux example):**

```text
ADEVICE  plughw:1,0            # the IC-9700 "USB Audio CODEC"; check `aplay -l` / Direwolf startup list
ARATE    48000
ACHANNELS 1
CHANNEL 0
MYCALL   GROUND                # any legal AX.25 callsign-shaped string; nothing goes over the air
MODEM    1200
PTT      /dev/ttyUSB1 RTS      # IC-9700 USB (B); radio menu USB SEND = "USB (B) RTS"
TXDELAY  30                    # 300 ms, Direwolf's recommended default
TXTAIL   10
KISSPORT 8001
AGWPORT  0
```

Windows differs only in device naming: `ADEVICE USB` (a unique substring of the device
description) or the device numbers Direwolf prints at start-up, and `PTT COM4 RTS`. [DW-UG
§9.1.2, pp. 54-55; §9.2.11.1, p. 68] Direwolf's defaults for timing are `SLOTTIME 10`, `PERSIST 63`,
`TXDELAY 30`, `TXTAIL 10`, `FULLDUP OFF`, and the author "strongly recommend[s] that you use the
default values". [DW-UG §9.2.15, pp. 75-77] On a closed one-way bench `FULLDUP ON` would skip the
listen-before-talk wait; the guide says it "is appropriate only when transmitting and receiving
on different frequencies" [DW-UG §9.2.15, p. 77], and since the 9700's receiver hears only its own leakage this
is arguably that case **[judgement]**; try defaults first.

### 3.5 IC-R8600 -> Direwolf (receive side)

**How the radio appears.** The IC-R8600 has two USB ports, mini-B on the front and type-B on the
rear; each "Outputs the demodulated signal or 12 kHz IF signal" and carries CI-V remote control;
each has its own AF/IF, level and squelch settings. [R8600-IM p. 16-1] Use the **rear** port for a
permanent bench. The receiver's CI-V address is `96h` by default. [R8600-IM p. 11-6] A Windows
USB driver is downloadable from Icom [R8600-IM p. 16-1]; Linux needs none.

**Radio settings (MENU > SET > Connectors > USB (Rear)):**

| Setting | Value | Why | Source |
|---|---|---|---|
| Operating mode | `FM` on the same frequency as the 9700 | AFSK rides on ordinary narrow FM. | R8600-IM p. 3-1 |
| `USB (Rear) > Output Select` | `AF` (default) | Demodulated audio, not 12 kHz IF. | R8600-IM p. 11-5 |
| `USB (Rear) > AF Output Level` | 50 % default = "200 mV (RMS)"; adjust so Direwolf's `-a 10` report shows receive audio level in the 30-150 range | Direwolf: "set audio input gain so this is somewhere in the 30 to 150 range." | R8600-IM p. 11-5; DW-UG §7.3, p. 43 |
| `USB (Rear) > AF SQL` | `OFF (OPEN)` (default) | Direwolf: "Leave squelch open. Squelch delay will cut off the start of transmissions." | R8600-IM p. 11-6; DW-UG §3.1, p. 9 |
| `USB (Rear) > AF Beep/Speech... Output` | `OFF` (default) | Keeps UI beeps out of the decoder. | R8600-IM p. 11-6 |
| `USB (Rear) > Serial Function` | leave `FSK Decode` (default), irrelevant | The R8600's serial port is not needed for receive-only. | R8600-IM p. 11-6 |
| Front-panel `SQL`, `ATT`, `P.AMP` | squelch open; attenuator and preamp per `rf-bench-attenuation.md` | Level management is an RF question. | R8600-IM p. 5-1 |

**Satellite-side `direwolf.conf`:**

```text
ADEVICE  plughw:1,0            # the IC-R8600 "USB Audio CODEC"
ARATE    48000
ACHANNELS 1
CHANNEL 0
MYCALL   SAT
MODEM    1200
# no PTT line: receive only
KISSPORT 8001
AGWPORT  0
```

Direwolf will print `Note: PTT not configured for channel 0. (Ignore this if using VOX.)`, which
is expected here. [DW-UG §9.2.11, p. 67] Run with `direwolf -a 10 -t 0` during bring-up to get
the periodic audio-level report. [DW-UG §7.3, p. 43]

### 3.6 Step-by-step: the software chain on both sides

**Ground computer (transmit).**

1. One-time radio setup on the IC-9700 per the §3.4 table (FM-D, DATA MOD = USB, USB SEND =
   USB (B) RTS, minimum power, External P.AMP off per the attenuation note).
2. Plug in USB. Identify the sound device (`aplay -l` / Direwolf's start-up list) and the two
   serial ports (`ls /dev/serial/by-id/` on Linux; Device Manager > COM port > Details on Windows
   [9700-B p. 8-15]). Determine which is USB (B): the manual says check the port properties;
   empirically, toggle RTS on each and watch the radio's TX indicator with the antenna port on
   the attenuator chain, or use `rigctl -m 3081 -r <port> f` on each and the one that answers
   with a frequency is USB (A) **[judgement]**.
3. Start Direwolf with the ground config. Confirm the start-up lines: audio device, `Channel 0:
   1200 baud, AFSK 1200 & 2200 Hz`, `Ready to accept KISS TCP client application 0 on port 8001`.
   [DW-UG §7.1, pp. 39-40]
4. Set transmit audio level once: `direwolf -x m` sends a continuous mark tone; adjust `USB MOD
   Level` for ~3-3.5 kHz deviation as heard/seen on the R8600. [DW-RIG §3, p. 9]
5. Ground Sim / Link code: build the Space Packet (§2.8), compute and append the CRC (§2.9),
   build the AX.25 UI frame (dest `SAT`, src `GROUND`, control `0x03`, PID `0xF0`, info = the
   Space Packet), KISS-wrap it (§3.2), and `sendall()` it to `127.0.0.1:8001`. Direwolf adds
   flags, bit-stuffing and FCS, waits for a clear channel, asserts RTS (PTT), sends 300 ms of
   flags (TXDELAY), the frame, 100 ms of flags (TXTAIL), and drops PTT. [DW-UG §9.2.15, p. 75]
6. Optionally read from the same socket: Direwolf echoes anything it *receives* on the 9700's
   audio (its own leakage will not decode, so expect silence).

**Satellite computer (receive).**

1. One-time radio setup on the IC-R8600 per the §3.5 table (FM, same frequency, USB (Rear) AF,
   squelch open, ATT/P.AMP per the attenuation note).
2. Plug in the rear USB. Identify the sound device.
3. Start Direwolf with the satellite config; watch `-a 10` reports for a sane sample rate
   ("44.1 k"/"48 k", 0 errors) and receive level 30-150 while the channel is idle. [DW-UG §7.3,
   p. 43]
4. Satellite Sim connects to `127.0.0.1:8001`, reads the KISS stream, un-frames (§3.2), parses
   the AX.25 header (skip 14 address bytes, or more if digipeater addresses are present: stop at
   the address byte whose low bit is 1; then 1 control byte and 1 PID byte), and hands the
   remaining bytes to the existing Space Packet parser. From there phase-1 code runs unchanged:
   length check, CRC check, APID routing, sequence check, command execution.
5. Any frame Direwolf prints in its monitor window but the Satellite Sim does not receive means
   the KISS client is not connected; any frame the 9700 sends that Direwolf does not print means
   audio level, frequency, mode or attenuation is wrong (check the R8600 S-meter/dBm meter and
   Direwolf's level report first).

**Return path.** The IC-R8600 is a receiver only. Telemetry from the Satellite Sim back to the
Ground Sim cannot use this bench as drawn; phase 2 either keeps the downlink on the phase-1
TCP path (asymmetric, still realistic: many exercises only contest the uplink) or adds a second
transmitter/receiver pair. Resolve before implementing the later RF Link.

### 3.7 Gotchas

**Protocol / Direwolf, either OS**

- The KISS payload must *not* include the AX.25 FCS; Direwolf computes it. Including your own
  two bytes just becomes payload and will confuse the far end's length check. [KISS §4; AX.25
  §3.7]
- Bad-FCS frames vanish silently at the receiving Direwolf (unless `FIX_BITS`, not recommended
  [DW-UG §9.2.7]). Your Space Packet CRC (§2.9) is the only corruption a challenger can present
  to the Satellite Sim over the air; design the "corrupted packet" challenges at that layer.
- Direwolf's TCP KISS accepts at most 3 clients [DW-UG §9.5.2]; a stray monitoring script can
  lock out the Sim.
- KISS clients can override TXDELAY/TXTAIL/etc. by sending command frames; a client that sets
  TXTAIL 0 "is asking for trouble". [DW-UG §9.2.15, p. 77] Never send KISS command frames.
- Direwolf's KISS/AGW ports are unauthenticated TCP listeners designed so "Dire Wolf and the
  client application can be running on different computers" [DW-UG §9.5.2, p. 83]; on a shared
  lab network set `AGWPORT 0` for the unused port and firewall 8001, otherwise anyone who can
  reach the port can key the transmitter. The guide itself notes for AGW: "In a system exposed to
  the Internet, you might want to disable the port for security reasons." [DW-UG §9.5.1, p. 83]
- Keep AX.25 addresses to upper-case letters and digits, 1-6 characters, SSID 0-15 [AX.25
  §3.12.1 via the address-field encoding rules]; Direwolf validates addresses on frames coming
  from KISS clients **[unverified: observed behaviour, not found in the guide]**.
- Direwolf decodes both AX.25 and, if enabled, FX.25/IL2P forward-error-corrected frames
  simultaneously; do not enable `FX25TX`/`IL2PTX` on the ground side unless the Sat side is also
  Direwolf 1.7+ (it is, but HTB-style tooling will not decode them). [DW-UG §9.2.9-9.2.10]
- Use `TXDELAY 30` (300 ms) not less: "In my testing, I found 200 mS was too short for a
  typical major brand 2 meter transceiver." [DW-UG §9.2.15, p. 76]
- Sample rate: 44100 default; "A higher audio sample rate provides no benefit for 1200 bps."
  [DW-UG §9.1.5, p. 58] 48000 is fine and matches most USB codecs' native rate. If Direwolf's
  `-a` report shows a rate far from nominal (e.g. 42.8 k) "many samples were getting lost", often
  a USB hub problem; plug the radio straight into the PC. [DW-UG §7.3, p. 43]
- Turn off any AGC ("automatic gain control") on the recording device: "You want any auto gain
  control to be off." [DW-UG §7.3, p. 44]
- RF getting back into the PC can latch PTT on or crash the machine; ferrites, distance, and
  the attenuator chain (rather than an antenna) make this unlikely on our bench. [DW-RIG §9.4-9.5,
  p. 59]

**Icom-specific**

- Both radios enumerate as the *same* audio device name. If both are ever plugged into one PC
  (e.g. a single-machine demo), select by ALSA card number (`plughw:N,0`) on Linux, and by the
  numeric device index on Windows, and expect the numbers to move when devices are re-plugged:
  "The numbers can change as USB devices are added and removed." [DW-UG §9.1.2, p. 55; §9.1.3,
  pp. 55-57, with a link to persistent USB mapping on p. 56]
- The 9700 gives you *two* COM ports; only the one selected in `USB SEND` keys the radio. If
  PTT does nothing, you have the other port. [9700-B p. 8-15]
- When a program opens a serial port, the OS may briefly assert RTS/DTR. The 9700's `Inhibit
  Timer at USB Connection` (default ON) exists precisely to swallow this glitch; do not turn it
  off. [9700-B p. 8-16] Also, do not set `USB Keying (CW)`/`(RTTY)` to the same line as SEND;
  the menu forbids it. [9700-B p. 8-15]
- Icom warns that connecting a second Icom radio to the same PC can emit a short SEND/Keying
  pulse from the first: "we recommend that you do not connect a second transceiver to a USB port
  of the same PC. Or, always turn OFF the transceiver power before you connect a USB cable."
  [9700-A p. 7-7] With the 9700 on the attenuator chain a brief key-up is harmless, but keep the
  9700 and R8600 on separate computers as planned.
- `DATA MOD` defaults to `ACC`, not USB: with the default, the radio transmits silence when
  Direwolf keys it. [9700-B p. 8-15]
- Data mode must actually be selected (`FM-D`, shown as "FM-D" on screen); in plain FM the
  `DATA OFF MOD` setting (default `MIC,ACC`) governs and USB audio is ignored. [9700-B p. 3-3,
  p. 8-15]
- The 9700's CI-V `Transceive` function is ON by default and broadcasts status changes on the
  CI-V port; harmless for Direwolf (it never opens the CI-V port when using RTS), but a
  `rigctld` instance on port A will see unsolicited traffic. [9700-B p. 8-16]
- The R8600 rear and front USB ports have *separate* AF output settings; configure the one you
  are plugged into. [R8600-IM p. 11-5]
- Icom's manuals say "Icom does not guarantee performance of the application, PC, or network
  device." [9700-A p. 1-17] Expect to tune levels empirically.

**Linux**

- Audio device permissions: the user must be in the `audio` group; Direwolf's udev rule
  `99-direwolf-cmedia.rules` only covers CM108 HID devices, not serial ports. For the 9700's
  serial ports the user needs the `dialout` group (Debian/Ubuntu) **[standard Linux practice, not
  from the Direwolf guide]**.
- PulseAudio/PipeWire can grab the USB codec and resample it. The guide: "If the user has
  PulseAudio installed, the installing of pavucontrol is mandatory to make sure the right audio
  routing is done... there are reports that pavucontrol can create blocking issues." Use
  `plughw:N,0` and `alsamixer` to unmute and set gain; `sudo alsactl store` to persist. [DW-UG
  §9.1.3, p. 57] On a new system "you might find the audio input device initially muted"
  (`MM` in alsamixer). [DW-UG §9.1.3, p. 57]
- `/dev/ttyUSB0` vs `/dev/ttyUSB1` ordering is not stable across reboots; use
  `/dev/serial/by-id/...` paths in `direwolf.conf` **[standard practice, not from the guide]**.
- hamlib PTT requires a Direwolf built with hamlib (`libhamlib-dev` at build time); the packaged
  binary may not include it. [DW-UG §5.1.1, p. 25]

**Windows**

- Install Icom's USB driver first (both radios) or the COM ports and possibly the audio device
  will not appear correctly. [9700-B p. 13-2; R8600-IM p. 16-1]
- Direwolf on Windows supports only serial RTS/DTR PTT (and VOX). GPIO, parallel port, hamlib
  and CM108 PTT are all "Linux only". [DW-UG §9.2.11, p. 67] This is fine for the 9700 (RTS on
  the virtual COM port) but rules out CI-V PTT via Direwolf on Windows.
- COM port numbers move when you change USB sockets; confirm USB (A)/(B) in Device Manager as
  the manual describes. [9700-B p. 8-15]
- Do not let Windows make the radio codec the *default* playback/recording device, or system
  sounds will be modulated onto the transmitter the next time PTT is asserted and Windows may
  apply "enhancements"/AGC to the microphone input **[judgement; Direwolf's general "AGC off"
  advice applies, DW-UG §7.3 p. 44]**. Select devices in `direwolf.conf` by a unique substring of
  their description, e.g. `ADEVICE USB`, rather than by number. [DW-UG §9.1.2, p. 55]
- Serial-port KISS via com0com virtual null-modem pairs is documented but unnecessary; use TCP
  8001. [DW-UG §11.1, pp. 149-150]

---

## 4. Current contracts and later RF questions

The [packet specification](../specs/packet-format-v1.md#envelope-and-limits) now defines APIDs, body layout, sequence counts and CRC; those are settled, not open research tickets. [Catch and Log](../specs/ground-station-training.md#implementation-defaults-practice-pass-and-evidence) uses synthetic packet telemetry, without this radio chain.

Later RF integration still needs a return-path choice because the IC-R8600 is receive-only (§3.6), checked bench-PC operating systems/PTT configuration, measured levels under the [bench research](rf-bench-attenuation.md), and a decision on Direwolf FULLDUP. Check actual codec device names, USB serial-port identity and handling of non-callsign addresses before adopting the proposal as a station procedure. No live RF integration is required for the operations MVP.

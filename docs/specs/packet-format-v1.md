# Packet format v1

This specification defines the accepted packet format to build. It does not describe an existing implementation.

## Plain-language contract

A packet contains an envelope (the CCSDS header), an instruction or reply, and a checksum (CRC). The command also states a claimed Mission role, like a name written on an envelope; this is not proof of identity.

- Commands use compact binary fields. The Platform explains their meaning in readable text alongside the original bytes.
- Commands use one application identifier (APID); replies and periodic telemetry use another.
- The checksum is always present. Its Defense decides whether an incorrect checksum causes rejection.
- An interpretable command gets a reply reporting success or a reason for rejection, with current Satellite Sim state. Unreadable packets get feedback in Platform history but no reply packet.
- A new Attempt starts fresh message numbering. `REBOOT` stays inside the Attempt and preserves message numbering and replay history.

## Envelope and limits

All multi-byte integers use unsigned big-endian encoding (most significant byte first). One complete packet is one Link message. Maximum total size is 256 bytes, including the six-byte header and two-byte CRC. Link adapters preserve submitted bytes; they must not repair headers, renumber packets, or recalculate a Player's checksum. Delimiting messages on the raw endpoint belongs to [Script access](backend-and-simulation.md#script-access).

| Header field | Width | v1 value |
| --- | --- | --- |
| Version | 3 bits | `000` |
| Type | 1 bit | `1` command uplink, `0` reply/telemetry downlink |
| Secondary header present | 1 bit | `0`; all project fields live in the body |
| APID | 11 bits | `0x001` commands, `0x002` replies and telemetry |
| Sequence flags | 2 bits | `11`, one complete unsegmented packet |
| Sequence count | 14 bits | `0..16383` |
| Packet data length | 16 bits | Number of bytes after the header minus one, including CRC |

The first 16-bit word combines version, type, secondary-header flag and APID. The second combines sequence flags and count. The third is the data-length field. Thus total bytes = data-length field + 7. A v1 command starts with `10 01`, not `18 01` (which would set the secondary-header flag).

The Ground Sim's generated command count starts at zero per Attempt. The Satellite Sim's downlink count also starts at zero per Attempt, shared by replies and periodic telemetry. Each sender advances its own count for each newly generated packet, modulo 16384. Player-crafted packets retain the supplied count; retransmitting captured bytes retains their old count. Counts are not account identities or authorization credentials.

`REBOOT` does not reset either count, accepted-command history, or the replay Defense's memory. A new Attempt resets them. The receiver's policy for accepting wrapped or out-of-order counts is later Defense design and is out of MVP; sender wrap does not imply receiver acceptance.

## Command body

| Offset within body | Field | Size |
| --- | --- | --- |
| 0 | Command code | 1 byte |
| 1 | Claimed Mission role | 1 byte: `0` guest, `1` operator |
| 2 | Argument byte count | 1 byte |
| 3 | Arguments | As many bytes as the preceding count |
| 3 + argument count | Authorization-data byte count | 1 byte |
| 4 + argument count | Authorization data | As many bytes as the preceding count |

For MVP commands the authorization-data count is zero. The length-delimited space allows a later proof of identity without changing the six command layouts. Its contents, validation, and any role permissions are reserved for later authorization design and are out of MVP. Do not interpret the role byte or reserved data as authenticated identity. Authorization is off in the MVP.

| Code | Command | Arguments |
| --- | --- | --- |
| `0x01` | `PING` | None |
| `0x02` | `GET_TELEMETRY` | None |
| `0x03` | `SET_MODE` | One byte: `0` SAFE, `1` NOMINAL |
| `0x04` | `DEPLOY_ANTENNA` | None |
| `0x05` | `SET_HEATER` | One byte: `0` off, `1` on |
| `0x06` | `REBOOT` | None |

`PING` replies successfully without an actuator change; `GET_TELEMETRY` returns the same full-state reply. `SET_MODE` and `SET_HEATER` set their named values. `DEPLOY_ANTENNA` sets deployed to true and increments the deployment count each time it is accepted, including a replay when that Defense is off. The [simulation defaults](backend-and-simulation.md#implementation-defaults-simulation-and-storage) define the remaining MVP reboot effects; this spec owns its encoding.

## Telemetry state layout

Every state snapshot occupies 18 bytes, in this order:

| Offset | Field | Size and encoding |
| --- | --- | --- |
| 0 | `mode` | 1 byte: `0` SAFE, `1` NOMINAL |
| 1 | `antenna_deployed` | 1 byte: `0` false, `1` true |
| 2 | `deploy_count` | 4 bytes |
| 6 | `heater_on` | 1 byte: `0` false, `1` true |
| 7 | `battery_pct` | 1 byte, integer `0..100`; round down if internal simulation uses fractions |
| 8 | `uptime_s` | 4 bytes, whole seconds |
| 12 | `commands_received` | 4 bytes; count of commands accepted for execution in this Attempt |
| 16 | `last_seq_seen` | 2 bytes; sequence of most recently accepted command, `0xFFFF` before any |

Rejected/unreadable submissions are recorded in history, not counted in these two accepted-command fields. `last_seq_seen` is an observation, not by itself the replay Defense's entire memory or highest accepted count. Four-byte counters saturate at `0xFFFFFFFF` rather than rolling over. These choices define wire representation; initial state and battery evolution belong to Challenge design.

## Replies and periodic telemetry

Both use APID `0x002` and Type `0`. The first body byte identifies the message:

| Kind | Body after kind byte |
| --- | --- |
| `0x10` periodic telemetry | 18-byte state snapshot |
| `0x11` command reply | Command code (1 byte), echoed command sequence (2 bytes), status (1 byte), 18-byte state snapshot |

Hello emits periodic telemetry once per second; command replies are additional. Ready emits no packets. Catch uses the [finite practice-pass source](ground-station-training.md#implementation-defaults-practice-pass-and-evidence) with the same telemetry encoding, and accepts no uplinks. Every interpretable command generates one reply, including `PING`, `GET_TELEMETRY`, and rejected commands. Replies never themselves request acknowledgement. Their envelope sequence is the Satellite Sim's own count; the echoed sequence identifies the submitted command. A replay may produce the same echoed sequence again: it is not a globally unique submission identifier.

| Status | Meaning / suggested readable text |
| --- | --- |
| `0x00` | Accepted; e.g. “Heater turned on” |
| `0x01` | Bad envelope/header |
| `0x02` | Bad checksum |
| `0x03` | Replay detected |
| `0x04` | Unauthorized |
| `0x05` | Unknown command |
| `0x06` | Invalid arguments/body values |
| `0x07` | Command unavailable in current state |

Replies carry the state after successful execution or the unchanged state on rejection. Normal time evolution can continue; rejection itself does not mutate Satellite Sim state. Human-readable wording belongs to the Platform; packet status codes remain stable. Which optional Defense wins when several would reject remains later design.

The Platform distinguishes a submission receipt (“sent to the Link”), a Satellite Sim reply, and a local parsing explanation. A submission receipt is not command acceptance. For example, history may say “Cannot read packet: command body is truncated; no reply generated.” It must not fabricate a Satellite Sim reply. The Software Link and RF Link use the same packet meanings, although packets can be lost on a Link.

## Checksum and minimum parsing

Append a two-byte, big-endian CRC-16/CCITT-FALSE over every byte of the header and body, excluding the CRC itself: polynomial `0x1021`, initial value `0xFFFF`, no reflection, final XOR `0x0000`. Test input ASCII `123456789` produces `0x29B1`. This is the project's packet convention; it is not a CRC mandated by the CCSDS Space Packet Protocol. It is separate from the radio envelope's AX.25 checksum.

The format-validation Defense may reject header constants, declared packet length, and CRC. With it off, those checks do not stop an otherwise interpretable command. The last two bytes still occupy the CRC slot; the receiver uses the actual complete Link message boundary rather than trusting the declared header length. The supplied Hello tool generates a valid checksum. A manually changed, interpretable packet can execute with a bad checksum while this Defense is off, but cannot satisfy Hello’s conformance goal.

Minimum parsing always applies: enforce the 256-byte bound, require a complete six-byte header, body prefix, bounded argument/authorization sections and two-byte CRC slot, and reject ambiguous/truncated layouts without execution. The two length-delimited body sections must consume exactly the actual body bytes before the CRC; trailing unexplained bytes are ambiguous. Minimum command size is 12 bytes. No unchecked allocation or reading outside the supplied bytes. A fully bounded body with an unknown command or invalid argument value gets the corresponding reply; an unreadable body produces only a local history explanation. Disabling a Defense never makes missing instructions executable.

The exact optional-Defense validation order and rejection matrix remain later design. The MVP can reserve all status codes while implementing no optional Defenses.

## Worked Hello, Satellite exchange

This is a deterministic example fixture, not a requirement for every Attempt's initial values. The Ground Sim sends its first command with guest role, no arguments, and no authorization data:

```text
10 01 c0 00 00 05  01 00 00 00  c0 3e
----------------  -----------  -----
6-byte header     PING body    CRC
```

Header: command APID 1, unsegmented, sequence zero, data-length field 5 (four body bytes + two CRC bytes − 1). Total: 12 bytes.

Assume the Satellite Sim is SAFE, antenna undeployed, deployment count zero, heater off, battery 100%, uptime zero, and this is the first accepted command. Assume no periodic telemetry has yet been emitted, so its reply sequence is also zero:

```text
00 02 c0 00 00 18
11 01 00 00 00
00 00 00 00 00 00 00 64 00 00 00 00 00 00 00 01 00 00
e1 36
```

Lines are header, reply fields, state, CRC. Status zero means accepted. The state now reports one accepted command and last command sequence zero. Total: 31 bytes; data-length field `0x0018` means 25 bytes after the header. If telemetry was emitted first or state/time differs, the reply sequence, state, and CRC differ too.

The Platform renders this as “PING accepted” with the current readings. Receipt and [saved task completion](backend-and-simulation.md#task-completion) are separate outcomes. MVP downlinks have only the two kinds above; no Flag packet is generated.

## Follow-through

- Challenge design supplies initial state, captures, goals and Briefing examples. “Hello, Satellite” still asks for a correctly formed packet even though the optional Defenses are off.
- Defense design uses these stable fields and reply codes to settle wrap handling, validation order and the Defense interface.
- Later authorization design may use the claimed role and reserved authorization-data space. It is not an MVP implementation dependency.
- [CCSDS and Direwolf research](../research/ccsds-space-packet-and-direwolf-path.md) provides the supporting header/CRC sources and the planned AX.25/Direwolf path for later RF work. This spec owns the packet bytes.

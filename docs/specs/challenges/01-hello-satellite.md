# Hello, Satellite!

Status: confirmed as the first Basic operations Challenge in the MVP. The PING/reply foundation and task-based completion are agreed. The main learning route is running a supplied command in the integrated terminal and inspecting its real reply; no Python writing or editing is required.
Availability and prerequisites: [Challenge progression](../project-plan.md#challenge-progression).

## What the Player learns

Run a prepared command-line tool, send a structured message and recognize a real reply. A **packet** is a small message with an envelope, some contents, and a checksum. No radio experience is required. The Platform assumes basic Python familiarity; this first activity uses a supplied tool and does not require writing a script. Allow roughly 10–15 minutes for a first-time Player; this is a planning estimate, to be checked with beginners.

## Briefing — text shown to the Player

> KnightSat is a fictional student CubeSat testing a simple communications payload. You are working with its Satellite Sim before any real flight. Your first task is to check that it understands your messages. Use the supplied terminal command to send PING—an instruction that means “please reply”—then inspect the reply. Check that the reply corresponds to the message you sent. The Platform records completion when the required exchange succeeds and then explains what happened.

## What the Player receives

A prepared terminal environment, the supplied command with a plain-language explanation, readable message history and the live state panel. Explain what the tool will send before asking the Player to run it. Nothing is sent automatically when the page or terminal opens.

The Player runs the provided command and inspects the reply. They do not need to install packages, write Python, fill packet-header fields, calculate a checksum or copy connection credentials for this first exercise. Keep the code editor and shared files available for inspection and later use.

The tool constructs a correctly formed guest-role PING under [Packet format v1](../packet-format-v1.md), sends it through the current Attempt's authenticated internal script connection, and displays the actual response. A PING button or browser packet-building form is not the required learning route. Packet fields and original bytes remain available in a supporting guide/history view, without turning packet construction into the first task.

Raw helper submissions still preserve their supplied bytes; the prepared tool's valid-packet construction is not a rule to repair arbitrary Player input. External scripts on a Player's own computer remain outside the MVP.

### Implementation default

Reuse the prepared `ping_example.py` and Python helper already specified for the Workspace. Show `python ping_example.py` as the command from the Hello folder. The example reads the current Attempt connection details from managed resources so no live credentials are embedded in downloaded code. One invocation sends one PING, waits for the matching reply and prints understandable feedback; it never retries transmission automatically. The [Workspace helper contract](../terminal-workspace.md#implementation-defaults-terminal-and-helper-contracts) defines the connection file and five-second total reply timeout.

## Walkthrough and feedback

1. Read the Briefing and the explanation of the supplied terminal command.
2. Enter the command in the terminal and run it once.
3. History first reports “Sent to the Link.” This is only a submission receipt.
4. When the Satellite Sim replies, the terminal and history show the real PING reply and distinguish it from unrelated telemetry. The Player checks that it answers the command they sent.
5. The Platform verifies the successful exchange, saves completion and shows the accomplished task and Debrief. There is no Flag packet or submission step.

A normal PING does not change the heater, mode, or antenna. The accepted-command count increases; the last accepted message number changes. Receiving periodic status messages by itself does not solve the Challenge.

## Setup, Defenses and exact goal

Use the Software Link. All three optional Defenses—format validation, replay detection, and authorization—are off. The two-byte CRC slot remains present. Minimum safe parsing still applies: an unreadable message cannot be executed.

Start SAFE, antenna not deployed, deployment count 0, heater off, battery 100%, uptime 0, accepted-command count 0, and no last accepted sequence (`0xFFFF`). Send no automatic PING. Packet generation and periodic telemetry follow the shared packet specification. Time-dependent readings are not part of this goal.

The Sim Service records a conforming-PING candidate only after a Player-origin packet has been accepted and is a correctly formed PING under Packet format v1: correct header constants, actual/declared length agreement, valid CRC, recognized role, no arguments, and no authorization data. Any valid sequence count is allowed. This is a goal check, not an enabled rejection Defense. It needs only a PING-conformance check, not the later general Defense framework.

An interpretable PING with a wrong CRC may execute while format validation is off. Show “PING accepted, but the Challenge asks for a correctly formed packet. Check the checksum.” Do not record completion for that packet. Never fabricate a rejection Verdict. Another command, an automatic setup event, or telemetry alone cannot reach the goal. A later correct PING can still succeed in the same Attempt.

The Backend exposes `goal_reached` and attempts to save completion only after the required valid PING has been accepted and its real reply has arrived through the Software Link. Goal detection alone, a submission receipt or unrelated periodic telemetry is insufficient. Commit completion before showing it as saved. Further successful PINGs create no duplicate completion record. Restart resets the Attempt; saved completion remains. Completion follows the shared [task-completion rules](../backend-and-simulation.md#task-completion).

## When a beginner gets stuck

| Situation | What the Platform shows |
| --- | --- |
| Player cannot identify the first action | Show the exact supplied command beside the terminal and explain how to run it. |
| Tool or connection is unavailable | Identify the environment/connection problem and offer recovery. Do not ask a beginner to install dependencies or present a fabricated reply. |
| Raw input is truncated | A Platform explanation such as “This message is incomplete; no Satellite Sim reply was generated.” Preserve the bytes in history. |
| A different command was accepted | “Command accepted. This Challenge asks you to send PING.” |
| Packet executed but is not correctly formed | Identify the first mismatched goal requirement in plain language; retain the actual success reply. |
| Nothing came back | “Sent, waiting for a reply.” Connection loss gets a separate notice. Do not claim success from sending alone or automatically resend. |
| Exchange succeeds but progress cannot be saved | Show that the reply arrived but completion could not be saved. Preserve the received evidence and offer recovery without resending PING automatically. |

No points, hint currency, or progressive hint system. The readable labels, guide and error messages are the teaching support.

## Debrief — shown after task completion

> You sent a message and received the Satellite Sim's reply. Sending alone does not prove that a spacecraft received or understood a command: operators check the returned evidence. PING checks communication without changing the heater, operating mode or antenna. This exercise used a software connection to a fictional satellite; it was not a radio contact with a real spacecraft.

## Acceptance examples

Before calling the implementation complete, demonstrate: a newcomer can run the supplied terminal command without writing Python or configuring credentials; a valid PING and its received reply save completion without a Flag; submission alone cannot complete; wrong-CRC PING can execute without solving; unrelated commands cannot solve; a valid follow-up exchange can solve; refresh retains received history; repeated success does not duplicate completion; failed persistence does not report saved completion. These are future acceptance checks, not tests already executed.

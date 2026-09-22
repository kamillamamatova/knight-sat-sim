# Project glossary

Knight Sat Sim teaches satellite cybersecurity and ground-station operation, including the PSB station at UCF. Use these names consistently in specs. Explain them in plain language in Player-facing material.

| Term | Meaning |
| --- | --- |
| Platform | The whole product: browser interface, backend, simulation, cybersecurity Challenges, station lessons, and saved progress. |
| Player | A person practicing activities in any of the three tracks. |
| Track | A learning path grouping related Challenges: Basic operations, Offensive, or Defensive. Only Basic operations is in the MVP. |
| Satellite Sim | Software that receives commands, holds simulated spacecraft state, sends telemetry and replies, and supplies packet evidence for Challenge completion. |
| Ground Sim | The software that handles the ground side of messages sent to and received from the Satellite Sim. It is not the real PSB station or a complete station operations trainer. |
| Ground station | The real equipment and software used to track satellites and receive or send signals. Our training focuses on UCF's Physical Sciences Building (PSB) station. |
| Station lesson | Ground-station instruction and practice within Basic operations. A lesson may support a Challenge. The three MVP activities use the shared task-based completion model. |
| Link | The message path between Ground Sim and Satellite Sim. The Software Link is used for the first demo. The later RF Link would carry messages through station radios on a cabled bench. |
| Packet | A structured message with a header, body, and checksum. The [packet spec](specs/packet-format-v1.md) defines the bytes. |
| Telemetry | A status report from the Satellite Sim, such as battery level or antenna state. |
| Packet decoder | A tool that interprets packet bytes as named fields. The MVP supplies this tool; interpreting packets is distinct from recovering them from a radio signal. |
| Challenge | A practical learning activity within a track, with a Briefing, an observable goal and a Debrief. Basic operations Challenges use task-based completion, without Flag submission; exact success criteria belong to each spec. |
| Briefing | The story and goal shown before a Challenge. |
| Attempt | One Player's run of a Challenge, including simulation state, task evidence and packet history. Restart creates a new Attempt. |
| Practice pass | An on-demand simulated reception scenario shared by Ready for the Pass and Catch and Log. Scenario time is fictional, not the current real-world clock. |
| Recording | A Player-saved file of the practice pass’s received packets. It is packet data, not radio audio or IQ data. |
| Session log | Saved context, recording filename, readings and an outcome note for a practice pass. |
| Workspace | The Player's scripts, recordings, logs, notes, and supplied Challenge resources. Saved authored files survive refresh, Attempt Restart, and Challenge switching during the current session. Stop and expiry clear personal work. |
| Completion | Saved confirmation that the Player met a Challenge’s required goal. For Basic operations, the Platform verifies the task result and records completion without a Flag. |
| Debrief | The explanation shown after Challenge completion: what happened, what it teaches, and any relevant Defense. The three MVP activities teach operational foundations. |
| Defense | An optional command check for format, repetition, or authorization. The first demo leaves these off. Minimum parsing and Challenge goal checks still apply. |
| Verdict | A Defense's accept or reject decision for a message. |
| Mission | The fictional KnightSat story used for the Challenges, separate from real KSC spacecraft and the PSB station. |
| Mission role | The authority claimed inside a command, such as guest or operator. The claim alone does not authenticate the sender or grant Platform access. |
| Sandbox | Planned goal-free play with switchable Defenses. It is separate from the required Challenge Workspace. |
| MVP | The first software demo: Hello, Satellite!, Ready for the Pass, and Catch and Log in Basic operations, all with task-based completion. Offensive and Defensive Challenges are deferred. |

See the [project plan](specs/project-plan.md) for scope and the [station training plan](specs/ground-station-training.md) for PSB lessons. Definitions of future features do not make those features part of the first demo.

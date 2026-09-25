# Knight Sat Sim: project plan

## Goal

Build a Platform where people can practice satellite cybersecurity and learn to operate a ground station, including our station at UCF’s Physical Sciences Building (PSB).

Players will learn how the station works, practice operating it in software, and try cybersecurity Challenges against the Satellite Sim.

The product has three tracks: **Basic operations**, **Offensive**, and **Defensive**. The MVP is three Basic operations Challenges, starting with **Hello, Satellite!** Offensive and Defensive Challenges, including redesigned Replay, come later. Basic operations teaches ground-station fundamentals and builds toward the PSB equipment and workflow.

## Learner background

Assume some basic Python familiarity. This is a ground-station operations and satellite-security Platform, not a Python course. Give concise instructions for the supplied tools, file formats and project-specific interfaces; do not add lessons on general Python syntax. Station/RF knowledge is still taught from the basics. Running or editing Python is appropriate where it serves an operational task, without forcing code into every interaction.

## First software demo

The MVP contains exactly three Basic operations Challenges:

| Order | Challenge | Agreed scope |
| --- | --- | --- |
| 1 | **Hello, Satellite!** | Run a supplied terminal command, send PING and recognize the Satellite Sim's reply. Hello does not require writing a script. |
| 2 | **Ready for the Pass** | Check satellite, pass time, receiving frequency and receiving mode using graphical controls; enable automatic tracking. Use Check setup feedback to correct the mistaken setting before starting. |
| 3 | **Catch and Log** | Adapt a short Python starter to receive telemetry and save a recording. Use the supplied decoder to extract readings, then complete and save the session-log template. |

These three learning goals are agreed. Ready for the Pass and Catch and Log share one practice pass, with either repeatable from a prepared starting point. The simulated pass starts on demand after preparation, rather than requiring a real-time wait. The [operations curriculum](ground-station-training.md#mvp-operations-curriculum) owns the interactions and success criteria. Offensive and Defensive activities are outside this MVP.

Previously agreed delivery constraints remain: browser-based practice, current stable desktop Google Chrome, one active Player, local teammate development and a shared team-only hosted URL. Use Cloudflare Access email codes for approved teammates; individual Platform accounts are later work and progress remains shared. Use Diab's existing HP ProDesk on his home network with Cloudflare Tunnel/Access; Diab will purchase a domain through Cloudflare later. The [deployment spec](backend-and-simulation.md#ownership-and-deployment) owns that foundation.

Hello uses the Software Link, not a real spacecraft. Ready for the Pass and Catch and Log use a fictional pass brief and supplied synthetic telemetry source; the Player creates a packet recording. The [pass defaults](ground-station-training.md#implementation-defaults-practice-pass-and-evidence) define the fixture, files and checks. Decoding packet fields does not teach RF demodulation. No live radio or station-hardware integration has been added to the MVP. The integrated terminal, code editor and shared files remain in the MVP. Hello uses the terminal; Ready for the Pass uses graphical station controls with the terminal available alongside them. Catch and Log uses the editor and terminal for a short Python script and a saved log template. Scripting runs inside the website; external Player scripts remain deferred.

## Challenge progression

Organize activities into the three tracks rather than one security ladder. Basic operations is the only playable MVP track. Offensive and Defensive tracks have no required MVP Challenges or prerequisite completion.

Each Challenge has a Briefing, a practical learning goal and a Debrief. Operations completion is task-based: the Platform checks the required actions/results, shows what the Player accomplished and saves completion. No copying or submitting Flags in the MVP. Unlock Hello → Ready for the Pass → Catch and Log using shared demo progress. Completed Challenges stay available to repeat; a repeat of Catch and Log starts with a prepared station setup. The operations spec defines handoff and repeat defaults. Availability and completion are enforced by the server.

KnightSat remains a fictional student CubeSat, separate from any real KSC spacecraft and the PSB station. Explain technical terms in plain language. There are no points, leaderboards or badges in this course year. Completing simulated training records practice; it does not certify a Player to operate real equipment alone.

## Ground-station training

We want someone new to ground stations to understand the equipment and practice a station session before using the real PSB station.

The lessons should cover:

- What the antennas, pointing controls, radios, and computers do.
- How to prepare for a satellite pass and choose the right equipment and settings.
- How to follow a pass, receive and decode a signal, save the results, troubleshoot problems, and finish the session.
- How those steps work at PSB, using its actual equipment names, photos, software screens, and station guide.

We’ll build toward a full receive-only station session, including decoding, using simulated or recorded data for practice. The PSB guide is still being written. Diab will help write and check its procedures with someone familiar with the station. Players should be able to explain their choices and recognize common mistakes. Completing a software lesson records practice; using the real equipment still involves a station walkthrough with an experienced operator.

The [ground-station training plan](ground-station-training.md) lists the lessons, source material, and remaining questions. This adds training content and practice workflows. Connecting the Platform to live station hardware is a separate piece of work.

## Who owns what

We have two frontend roles and two backend roles. One frontend teammate will also help with the database. Diab is the project manager, handles satellite/RF work, and helps with frontend, backend, and database work as needed.

Sydney Lalah and Lily MacInnis handle frontend work. Sydney also helps with the database. Kamilla Mamatova and Denzel Galang handle backend work. The table gives each person a starting focus; we can move tasks as needed.

| Person | Main work | What they should have working |
| --- | --- | --- |
| Diab | Manage the project, build the simulation, handle satellite/RF work, write and check PSB training material, and help wherever needed | Working simulation, accurate lessons for the PSB station, and the pieces working together |
| Sydney | Challenge and station lesson pages, instructions, station setup controls, history, status, and progress. Help Kamilla with database work | Players can follow the lessons and Challenges and see their progress |
| Lily | Code editor, terminal, file controls, and the station practice screens | Players can run Python, save recordings/logs, and work through station practice |
| Kamilla | Attempts, browser API, access checks, task verification, database, and shared progress | Challenges work and lesson progress is saved; software practice and real-station sign-off are kept separate |
| Denzel | Running Player code, terminal connections, file storage, cleanup, and station practice data | Python and files work correctly; practice lessons can load their recorded or simulated data |

Sydney and Kamilla work together on Challenge pages and saved progress. Lily and Denzel work together on the editor and terminal. The backend teammates coordinate with Diab to connect everything to the simulation. Kamilla leads database work, with help from Sydney.

Everyone tests and documents their own work. We track tasks in the Knight Sat Sim (KSAT) Jira project, with one accountable owner per task. Request a non-author teammate's review when a pull request is ready; do not preassign a fixed reviewer. For external setup without a PR, another teammate verifies the recorded result when ready. RF hardware work comes later.

## How we’ll build it

The repository is a scaffold, not the implemented product. The implementation defaults are agreed; use these delivery stages for Jira tasks. Track actual implementation prerequisites with Jira blocking links; the Player's unlock order does not prevent development with isolated persisted-completion fixtures.

1. **Shared foundation:** packet encoder/decoder and Software Link; FastAPI models, SQLite migrations/completion, ownership and event snapshots; React shell/shared controls/generated types. Use the contracts below without choosing new frameworks.
2. **Workspace and Hello:** isolated runtime, terminal/editor/files and prepared PING tool; complete the real browser → helper → Sim/Link → saved completion journey. Prove cleanup, access isolation and stale-save handling at this boundary.
3. **Ready for the Pass:** scenario fixture, graphical setup, field feedback and completion; prove current setup validation and the handoff into Catch.
4. **Catch and Log:** receive stream, Python starter/decoder, pass start, saved recording/log and evidence checks; prove repeat, interruption and persistence behavior.
5. **Shared hosted demo:** implement Access-gated deployment alongside the feature work; verify all three activities at the team URL and have a newcomer complete them.

The agreed baseline was published to the official repository, `kamillamamatova/knight-sat-sim`, at [c7e742a](https://github.com/kamillamamatova/knight-sat-sim/commit/c7e742a0b663a97694f5691bf1a3773fdca3ba07). KSAT-5 also requires a teammate's fresh-clone/startup verification; publication alone does not establish that result.

Use one shared backlog and a weekly plan-and-demo routine. Keep the four epics for foundation, Workspace, Challenges and hosting; use area labels to find UI, server, simulation, hosting and documentation work. Build tickets have a practical starting point, one owner, true prerequisites, an explicit handoff and observable acceptance checks. Split independently deliverable behaviors while keeping their required failure handling and tests together. The [contribution rules](../../CONTRIBUTING.md#planning-and-board) own the board workflow, ticket format and review procedure. Dates and task status belong in Jira. PSB procedure research can proceed separately; unverified real-station instructions do not block the simulated MVP.

## Completion checks

These are implementation acceptance requirements, not completed results.

- [ ] All three Basic operations Challenges work in order, with server-enforced unlocks and independent repeats of completed activities.
- [ ] Hello verifies a real PING/reply; Ready verifies current setup; Catch verifies a usable saved recording and factual session log.
- [ ] Completion is committed before success/unlock is shown, with a Debrief and no Flag step.
- [ ] Terminal, editor, notes, files and downloads work with the ownership, lifecycle and recovery rules below.
- [ ] A newcomer with basic Python familiarity can complete the activities in Chrome and explain their operational purpose.
- [ ] Approved teammates can use the hosted demo with shared progress, access isolation and safe updates.
- [ ] Simulation is clearly labelled; no unverified PSB procedure or later security activity is required.

The primary acceptance boundary is browser interaction against the real Backend, Sim/Link, SQLite and runtime. Use the [Workspace acceptance suite](terminal-workspace.md#primary-acceptance-suite) and each activity's checks, with focused persistence/access/runtime faults. Use Playwright with Google Chrome for browser acceptance checks; pytest/Vitest cover focused behavior where useful.

## Tools and later work

We’re building a custom web Platform and a small Python Satellite Sim. The frontend uses React and TypeScript; the backend uses Python and FastAPI; SQLite stores progress; Docker runs Player code. The Web Backend and Sim Service share one server process. The [Sim Service contract](backend-and-simulation.md#ownership-and-deployment) defines their responsibilities, and the [Workspace contract](terminal-workspace.md#execution-storage-and-endpoint-isolation) defines Player execution. The map gives fictional mission context; it never blocks a command based on position.

Later work includes the Offensive and Defensive tracks, real radios, orbital physics, more Challenges, free-play Sandbox, switchable Defenses, accounts, multiple Players at once, and leaderboards. Installing extra packages and unrestricted internet access from the terminal are also outside the first version.

### Later tracks

The Offensive track may include redesigned Replay and unauthorized commanding. The Defensive track may include detecting suspicious commands and demonstrating protections. Specific activities, their sequence and whether they share scenarios are deferred. A meaningful consequence and a visible Defense remain the direction for a future Replay redesign; its previous antenna-counter exercise is not approved as that redesign.

Real radios, live hardware integration and advanced station lessons need their own requirements. The [station training plan](ground-station-training.md) supplies the broader operations goals; the first three Challenges need not cover the entire station workflow.

## Build specifications

| Area | Read this |
| --- | --- |
| Shared UI components, styling, and desktop support | [Frontend](frontend.md) |
| State, commands, Attempts, access, task completion and hosting | [Backend and simulation](backend-and-simulation.md) |
| Packet bytes, encoding, and parsing | [Packet format](packet-format-v1.md) |
| Terminal execution, editor/files, access, and cleanup | [Terminal Workspace](terminal-workspace.md) |
| Ready for the Pass, Catch and Log, and broader PSB lessons | [Ground-station training](ground-station-training.md#mvp-operations-curriculum) |
| Guided PING instructions, goal, and checks | [Hello, Satellite](challenges/01-hello-satellite.md) |

The [glossary](../glossary.md) defines shared terms, and [Contributing](../../CONTRIBUTING.md) covers tests and review. Record agreed choices in the spec that owns them; dates and task assignments belong in Jira.

## Implementation handoff

The requirements and implementation defaults in the linked specs are agreed. Record changes in the owning spec; do not add ADRs or duplicate explanations. Package versions belong in lockfiles, running/setup instructions in README, and task status/dates in Jira. Use this plan as the reading map and the individual specs for contracts and acceptance behavior.

External setup still needs teammate verification of the published baseline, setup of the selected HP home host, the domain and approved-email list. Shared setup can proceed alongside implementation. Runtime sizing and hosted access must be proven before the shared demo. Checked PSB source material and [later station lesson design](ground-station-training.md#work-still-to-specify) remain separate follow-on work.

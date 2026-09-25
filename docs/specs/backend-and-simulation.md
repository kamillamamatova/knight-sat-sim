# Backend and simulation

This specification owns simulation state and commands, the Backend/Sim interface, Attempt lifecycle, script access, station practice, and task completion. These are build requirements, not implemented code. The requirements and implementation defaults below record agreed choices.

This contract covers all three Basic operations Challenges. The [operations curriculum](ground-station-training.md#mvp-operations-curriculum) owns their tasks and evidence; broader PSB lessons remain later work. No Flag generation, packet or submission route is part of this MVP.

## Ownership and deployment

The first demo requires a shared, team-only hosted URL as well as local Docker development. Hosted Players need only a browser; execution runs on the host. Use one dedicated Linux server running Docker Compose for the hosted demo, with one active Player. Keep the built React interface, FastAPI/Sim, SQLite storage, and trusted runtime controller on that server; the controller manages isolated Player execution containers.

Use Diab's existing HP ProDesk 600 G2 (Intel Core i3-6100T, 16 GB RAM, 1 TB SSD) on his personal home network as the initial hosted-demo server, with Ubuntu LTS x86-64 and Docker Compose. Diab owns host operation and Cloudflare Tunnel/Access administration. Cloudflare Tunnel and approved-email Access provide the team entrance. This replaces the Azure preference; Azure credit verification and a cost estimate are not prerequisites for this host decision. Keep Compose portable to another Linux host if needed. Diab will purchase a project domain through Cloudflare later; the exact name is not selected yet. Availability depends on the HP remaining powered on and connected to home internet. Host setup, runtime validation, the domain and access configuration remain delivery work.

This is the architecture to build, not evidence of a deployed service or a purchased domain. The Compose setup described below is the current local development scaffold.

One Python server process contains FastAPI and the Sim Service; one Player holds the sole active Attempt. The Sim Service owns Satellite Sim state, packet handling, Ground Sim, Link, scenario playback and packet-goal evidence. FastAPI owns Attempt lifecycle, station-setup/file evidence verification, unlocks, and saved progress. FastAPI calls an asynchronous Python interface and consumes events; it never edits Satellite Sim state directly. These calls return data, not references to mutable simulation objects.

Browser actions use HTTP to FastAPI; live updates use WebSocket. Generate frontend HTTP request and response types from the FastAPI OpenAPI schema using `openapi-typescript`; the [Frontend HTTP client rules](frontend.md#http-api-types-and-client) define their use. WebSocket event schemas require a separate explicit contract. Player scripts enter through the Sim Service's raw endpoint. The terminal PING tool uses that packet path through Ground Sim/Link, preserving supplied bytes. Browser station controls change station configuration, not packet bytes; the separate internal receive stream carries the practice pass delivered through the Software Link. The internal service interface does not replace the Software Link. A future separate-process adapter is deferred and will need its own failure/reconnection handling.

Docker Compose starts `ui` (React development server) and `server` (FastAPI plus Sim Service). Run one server worker for this in-memory MVP. SQLite in a persistent volume stores the shared demo profile and solved Challenges. Satellite Sim state and Attempt packet history are in memory. No separate database server is required. The integrated terminal adds a replaceable Docker execution container, independently retained Workspace storage, and an authenticated terminal gateway. The current two-service scaffold does not yet provide this execution infrastructure. The [Workspace contract](terminal-workspace.md#application-and-responsibility-boundaries) defines how it connects to the combined Backend/Sim process.

## Updating the hosted demo

Provide a manually triggered GitHub Actions workflow labelled **Update website**. Merging code runs checks but does not automatically update the hosted demo. Deploy a selected commit from `main` only after its required checks pass; deploy that exact revision rather than a moving branch reference.

Refuse the update while a Player session owns the Platform, leaving the running version and that Player's work intact. The operator can try again after the session ends. Check ownership and block admission of new Players atomically, so a new session cannot start between the idle check and the update. Do not force-stop a Player or wait indefinitely for their session to end.

Serialize updates so a second deployment cannot overlap or cancel one in progress. While updating, show that the demo is temporarily unavailable and retain the SQLite progress volume. Reopen admission only after migrations and application health checks succeed. A failed preflight leaves the current deployment running; a failure during replacement requires recovery before reopening the demo.

These are requirements for a deployment workflow that is still to be built.

## Hosted team access

Use Cloudflare Access with emailed one-time codes for the MVP. Allow only explicitly approved email addresses; possession of any email address does not grant entry. The access gate covers the hosted browser interface, HTTP API, and browser WebSocket connections. The hosted origin must not provide an alternate path that bypasses the gate.

This gate controls entry to the demo, not individual Platform accounts. Keep the shared demo progress profile and the single active Player rule. Passing the gate does not grant control of another browser's Attempt: the existing ownership cookie, controlling-tab rules, and Attempt-bound script credentials remain required.

Player execution uses the restricted internal script bridge with its existing Attempt credential. Do not place Cloudflare administration credentials or an access-gate bypass credential in Player containers or distributed starters. MVP scripting runs inside the website's Workspace. Connecting scripts from a Player's own computer is deferred, so do not publish an external Player script route or build access-gate integration for local scripts in this release.

Individual Platform accounts and per-account progress belong to later work. Cloudflare Access supplies the login experience; the MVP does not implement signup, passwords, or account recovery. Use the [hosting defaults](#implementation-defaults-hosting) for routing, access sessions and deployment mechanics.

## Progress storage

Use Python's built-in `sqlite3` through one shared progress module. Use parameterized SQL and explicit transactions; no ORM or additional database-access package is required for the first demo. API handlers call named operations to read completed Challenges, save completion, and reset demo progress rather than implementing database handling independently.

- Store shared demo progress in the SQLite database configured by `SQLITE_PATH`, using the persistent volume described above. Live Attempt state and packet history remain in memory. All three MVP activities use the same Challenge completion schema; broader lesson schemas are deferred.
- Enforce completion uniqueness in the database so repeating a verified task completion cannot create duplicate completion records. Commit completion before reporting success. A database failure must not produce a successful completion response.
- The module owns connection and transaction lifetimes. Keep synchronous database work off the asynchronous event loop; create, use, and close a connection within the same worker operation. Do not share an arbitrary connection across request threads. Keep transactions short.
- Track database structure changes as numbered SQL migration files in the repository. Apply pending migrations in order before serving requests, recording successfully applied versions in the database. Apply each migration and its version record in one transaction; roll back that migration and fail startup if it fails. Repeated startup must not reapply completed migrations.
- Add a new migration when the schema changes instead of editing an already-applied migration. Preserve existing progress during upgrades. Reset demo progress clears the intended progress records; it does not recreate the database or erase migration history.

Automated backups, including scheduled and pre-update backups, are deferred beyond the MVP. Normal restarts and website updates must retain the SQLite progress volume. If the server's storage is lost, shared demo completion may need to be recreated; off-server recovery is not an MVP requirement. Workspace retention and cleanup rules are unchanged.

## State and commands

State fields are `mode` (SAFE or NOMINAL), `antenna_deployed`, `deploy_count`, `heater_on`, `battery_pct`, `uptime_s`, `commands_received`, and `last_seq_seen`. Hello publishes full-state telemetry once per second; Ready has no live packet feed, and Catch publishes only its finite scenario stream. Encoding, counter limits, and command replies are defined in [Packet format v1](packet-format-v1.md).

The six commands are `PING`, `GET_TELEMETRY`, `SET_MODE`, `DEPLOY_ANTENNA`, `SET_HEATER`, and `REBOOT`. Keep this existing packet vocabulary; only PING is required in the learner route. Ready/Catch are receive-only station practice, not additional command exercises. Individual Challenge specs define starting state and exact goals.

`DEPLOY_ANTENNA` marks the simulated antenna deployed and increments `deploy_count` each time it is accepted. An unchanged replay can execute again while replay detection is off. The antenna remains deployed; it does not physically unfold twice.

Message numbering and accepted-command history survive `REBOOT` within an Attempt. A fresh Attempt resets them. The [model defaults](#implementation-defaults-simulation-and-storage) define the remaining reboot effects.

## Defenses and model limits

The planned optional Defenses are format validation, replay detection, and authorization. They are off in the operations MVP; the general Defense framework is deferred. Minimum safe parsing and the exact Challenge goal checks are still required.

Receiver rules for wrapped or out-of-order message numbers, authorization, and the full rejection order belong to later Defense work. A claimed Mission role does not prove identity.

Hello success does not depend on battery or uptime. The MVP requires no realistic orbital or radio physics. Use the Hello model below and the [Catch fixture](ground-station-training.md#implementation-defaults-practice-pass-and-evidence), whose recorded values determine log answers. Ground-station tracking/settings are separate from Satellite Sim spacecraft state.

## Operations and data

| Internal operation | Input | Output / meaning |
| --- | --- | --- |
| `start_attempt` | `attempt_id`, `challenge_id`, server-owned scenario/configuration | `AttemptSnapshot`; initialize one fresh Attempt, or report busy. FastAPI checks unlocks first. No private Flag argument. |
| `stop_attempt` | `attempt_id`, `reason` | End the Attempt and timers, close script access. Repeated stop is harmless. Release ownership only after coordinated cleanup succeeds. |
| `send_packet` | `attempt_id`, original `packet_bytes`, server-derived `submission_id` and origin | Submission receipt, not proof of acceptance; preserve malformed bytes. Hello only; reject uplinks in Ready/Catch before Link delivery. Never retry automatically. |
| `start_pass` | `attempt_id`, verified server-owned setup | Start the finite Catch source once; return current pass state if already started. Requires the current receiver and valid setup. |
| `read_attempt` / `watch_attempt` | `attempt_id` | Snapshot / ordered event feed of state, original packets, pass status, goal evidence and termination. |

`AttemptSnapshot` contains identity, Challenge, lifecycle status, public spacecraft state, packet history and goal evidence. Events identify their Attempt. Station setup, the received-pass manifest and Backend file checks belong to Attempt context; they are not added to the binary spacecraft state. Return copies, not mutable service objects. The browser contract below defines the combined public view.

FastAPI handles browser ownership separately from shared progress: tabs in the active browser reconnect to the same Attempt; a different browser gets Platform busy. The script credential identifies that same Attempt. Within the owning browser, explicit Take control revokes old tab mutations without changing the Attempt or interrupting programs. This tab grant does not replace script authentication.

## Task completion

The [Hello goal](challenges/01-hello-satellite.md#setup-defenses-and-exact-goal) and [operations evidence](ground-station-training.md#implementation-defaults-practice-pass-and-evidence) define success. Hello needs an accepted conforming Player PING and its correlated reply actually received through the Link. Ready needs the explicit successful check of current settings. Catch needs a finished pass, matching saved recording and factual saved log. Neither an HTTP receipt, browser checkbox nor terminal success text is sufficient.

After verifying evidence, FastAPI writes the unique Challenge completion to SQLite. Only committed progress unlocks the next Challenge and shows saved completion/Debrief. **Completion representation:** keep `goal_reached` (verified evidence) distinct from `completion_state = in_progress|save_failed|saved`. A failed write retains evidence and reports that the task succeeded but progress could not be saved. Explicit Retry saving performs persistence again without resending a packet, rerunning Python or replaying the pass. Recheck Attempt identity and the current setup/file evidence on a Ready/Catch retry; changed or missing evidence clears the unsaved goal and returns to `in_progress`, requiring a new check. Setup edits also invalidate an unsaved successful setup check. Once saved, completion remains shared progress despite later edits, restarting or an unsuccessful repeat.

**Coordination:** use a per-Attempt operation lock to serialize setup changes, verification, completion persistence and lifecycle revocation; perform bounded file reads/database work in workers without blocking the event loop. A stale operation must not commit for a replacement Attempt. The unique database key makes repeated successful verification harmless. Shared progress and an individual repeat's completion state are separate: starting a completed Challenge creates a fresh local task state without relocking later Challenges.

## Browser API errors

Browser HTTP endpoints use one JSON error envelope with an appropriate non-success HTTP status:

```json
{
  "error": {
    "code": "FILE_CONFLICT",
    "message": "This file changed in the terminal. Your unsaved edits are preserved. Compare the versions or save under another name."
  }
}
```

`code` is a stable machine-readable identifier; frontend logic uses it rather than matching message text. `message` explains the failure in plain language. An optional `field_errors` array inside `error` contains `{ "field": "field_name", "message": "What to correct" }` entries for invalid input. Define this envelope in the Backend schema used to generate frontend HTTP types, including request-validation responses. Unexpected server failures use a safe generic message; diagnostic details stay in server logs.

Show failures beside the affected control or panel, with recovery instructions where applicable. Failures requiring Player action remain visible until resolved or dismissed. Preserve entered values and unsaved drafts after a rejected operation. The [Frontend error presentation rules](frontend.md#http-api-types-and-client) also cover network failures.

A missing response does not establish whether a mutation was applied. Do not turn connection uncertainty into a confirmed rejection or automatically retry the operation. These are Platform errors; Satellite Sim command replies and Defense results keep their existing packet semantics.

## Live updates and resynchronization

The browser Attempt event connection starts with an authoritative snapshot, then delivers numbered updates. On every reconnect, send a fresh snapshot of the current Attempt, including its current state, packet history, and goal status. Include completion checks and current pass/setup status; never invent task evidence after a disconnect.

- Associate each snapshot and update with its Attempt ID. A snapshot identifies the last included event revision; subsequent updates use consecutive increasing revisions within that Attempt. These revisions are browser-event ordering metadata, separate from packet sequence numbers.
- Coordinate snapshot capture and event subscription under the same serialization boundary. Send the snapshot first, then all later updates in order, so a change during connection setup cannot fall between the snapshot and the stream. A disconnected HTTP read followed by an unrelated subscription is insufficient.
- After installing a snapshot, ignore duplicate or older updates. Apply only the next expected revision for that Attempt. A revision gap triggers a fresh snapshot and stream synchronization rather than guessing the missing state. Ignore messages from superseded connections or Attempts, including late snapshots.
- During disconnection or resynchronization, label retained readings as stale. Mark them current only after the authoritative snapshot has been installed. If the Attempt has ended, report that outcome instead of restoring stale state or starting a replacement automatically.
- Reconnecting restores observation only. It never reruns a program, resends a packet, or repeats an uncertain HTTP mutation. Terminal attachment follows its separate Workspace contract.

The first demo does not require a durable browser-event replay log. This does not remove the required in-memory Attempt packet history. The [browser contract](#implementation-defaults-browser-contract) defines event payloads alongside the HTTP API.

## Lifecycle and recovery

Changing the viewed Challenge page does not change the active Attempt. Start/Switch actions explicitly request lifecycle changes; the [Frontend navigation rules](frontend.md#navigation-and-active-attempt) distinguish browsing from switching the active Challenge.

Stop ends the session: terminate Player execution, invalidate access, clear personal Workspace data, then release the slot. Explain the deletion consequence before Stop and provide script, recording and log/notes downloads. Restart or switching Challenges stops old programs and the old Attempt, retains Workspace-authored files, and creates a new ID, fresh state, and fresh packet history; replace supplied Attempt resources. Preserve exclusive ownership across these transitions. Old connections/events cannot control the replacement. Solved progress remains. Challenge settings come from server-owned configuration; arbitrary Defense toggles belong to the later Sandbox.

FastAPI coordinates these Workspace and execution transitions; they do not move filesystem or terminal responsibilities into the Satellite Sim. The [Workspace contract](terminal-workspace.md) specifies accepted behavior. Reset workspace requires confirmation, erases personal files, restores starters, and starts a new Attempt while retaining solved progress. Runtime-only failure recovery is explicit and leaves the current Satellite Sim state intact.

Reap after 30 minutes without Player activity, terminate execution, and clear personal Workspace data before releasing the slot. A live watching page, accepted terminal input/file saves, explicit setup/pass/check actions, or packet submissions through the raw endpoint keep the Attempt alive; autonomous telemetry, receive-only stream traffic, terminal output, and idle sockets do not. A receive-only pass lasts seconds; leaving an idle receiver open cannot hold ownership indefinitely. Closing one tab does not end it. Refresh reconnects, fetches current state/history, and resumes updates. The implementation must retain goal status in the snapshot so a missed event cannot lose success detection.

A Python server crash ends unfinished Attempts; personal Workspace recovery is not guaranteed and abandoned execution/data must be cleaned up before reuse. Show “Attempt ended—start again”; retain saved solved Challenges. Connection uncertainty never triggers an automatic packet resend, because retransmission can itself change the Challenge outcome. An explicit Reset demo progress action clears shared solved progress under the idle-only contract below.

## Script access

### Connecting and owning an Attempt

- The Challenge page shows the Workspace's script connection address and a copyable script access credential for its active Attempt. Use `/raw/attempts/{attempt_id}` for Hello or `/receive/attempts/{attempt_id}` for Catch through the restricted internal bridge. These connection details are for scripts running in the Workspace, not a publicly reachable endpoint for a Player's own computer. Hosted browser WebSockets use `wss://`; the private bridge transport follows the [Workspace isolation contract](terminal-workspace.md#execution-storage-and-endpoint-isolation). The raw handler belongs to the Sim Service in the existing Python server. No separate TCP port per Attempt or exposed modem interface is needed for the MVP.
- The Web Backend recognizes the browser through an opaque, server-issued ownership cookie, shared across tabs and refreshes. It is HttpOnly, SameSite=Lax, and Secure when hosted over HTTPS. Browser control requests and browser WebSocket connections validate ownership and origin. Shared demo progress does not grant ownership.
- A different browser receives “Platform busy.” Losing the ownership cookie does not let a browser take over the current Attempt; it waits for that Attempt to stop or expire under the existing lifecycle rules. Account-based recovery is deferred.
- The script credential is an independently generated, unpredictable secret bound to one Attempt, obtainable only by its owning browser. Send it as `Authorization: Bearer <credential>` in the WebSocket opening request, never in the URL or inside a Space Packet. The credential grants this Attempt’s internal script access; it is distinct from the packet’s claimed Mission role.
- Stop, Restart, Challenge switch, timeout, or server failure invalidates old access. Restart supplies a new Attempt ID and credential. Close old sockets and check that an Attempt is still active before delivering queued packets, so old traffic cannot affect its replacement.
- Browser terminal/editor control uses the [Workspace controlling-tab rules](terminal-workspace.md). Taking control in another tab does not invalidate the active Attempt's script credential or interrupt running programs. The browser tab grant never replaces the raw endpoint's bearer authentication.
- Browser controls and scripts operate on the same Attempt and packet history. Hello permits multiple raw observers with ordered command ingestion. Catch permits one receive-stream subscriber; additional receivers get `RECEIVER_BUSY`. Ready exposes no packet connection. A reconnect never automatically resends a command. Mere idle script connections do not count as Player activity; packet submissions do. A watching browser retains the keepalive behavior in [Lifecycle and recovery](#lifecycle-and-recovery).

### Packet delivery and helper

The binary-message rules below apply to Hello’s raw connection. Catch uses the receive-only JSON stream defined in the next section.

- After authentication, one binary WebSocket message contains exactly one complete Space Packet, including header and CRC. Use message boundaries, not the packet's declared length, to delimit submissions. WebSocket fragmentation does not create additional packets.
- Preserve supplied bytes through Ground Sim and the selected Link, including intentionally incorrect headers or CRCs. Enforce the existing 256-byte maximum. Oversize submissions are rejected before Link delivery; other malformed bytes follow the minimum parsing rules in Packet format v1. Never silently fix, renumber, or retry a packet.
- Each received Satellite Sim packet is a binary WebSocket message. Text messages are reserved for Platform feedback, such as a local parsing error, and must not be presented as Satellite Sim replies. Packet history retains original bytes, direction, and time. Delivery to the Link is not proof that the command was accepted.
- Ship a small Python helper under `player/python/` and examples under `player/examples/` in the planned project repository. Preinstall the helper and supply starter files and guidance in the embedded execution environment so Players need neither the private repository nor local installation. Catch and Log uses the embedded editor and general terminal. A separate local-installation helper bundle is not required for the MVP; recording, log, saved-script and note downloads remain available. [Workspace lifecycle rules](terminal-workspace.md) govern provisioning, access, files, and recovery.
- The helper connects, sends exact bytes, receives packets and Platform feedback, and provides the required PING builder and packet decoder. Its raw-send function never rewrites bytes. Examples cover connecting, sending a PING, and inspecting replies; Catch’s starter handles connection setup but leaves receiving/saving and selecting log readings to the Player. Byte construction utilities may be shared with the server, but no server-private configuration or internals are included in the download.

### Implementation defaults: receive-only pass stream

The [Workspace receive helper](terminal-workspace.md#implementation-defaults-terminal-and-helper-contracts) defines the JSON messages for Catch. Use the same Attempt-bound bearer opening header and revocation checks as raw access. The internal listener exposes only these two script routes; neither is mounted on the public browser app. Authenticate first, then validate Challenge and current Attempt. Reject Player uplink data on the receive stream. No Cloudflare bypass or infrastructure credential is supplied.

The Sim Service forwards Link-delivered telemetry, retaining original bytes and per-pass metadata. Browser history, helper output and grading manifest must describe the same actual deliveries. Publish receiver readiness only after authenticated subscription is installed. Serialize subscription/disconnection and Start pass so it cannot begin against a stale readiness flag. Bound the receive subscriber to 64 messages of at most 4 KiB; overflow or loss while running interrupts the pass rather than silently dropping records. The UI and stream report the interruption; no automatic stream replay.

## Challenge goals

Use [Challenge progression](project-plan.md#challenge-progression) for the three server-enforced IDs and prerequisites. Exact goals belong to the Hello and operations specs, not a separate security ladder.

**Correlation:** the raw handler assigns a fresh server `submission_id` and origin `raw_player` to every deliberate binary message. Generated downlinks use origin `satellite`; scenario packets also carry `pass_run_id` and index metadata. Never trust origin or internal correlation IDs supplied by a Player. These are internal history metadata, not extra Space Packet bytes. Correlate each reply to its actual ingestion event, not merely the reusable 14-bit wire sequence. Preserve original bytes, acceptance result and received reply evidence. A command can execute without meeting Hello's conformance goal; never fabricate a rejection Verdict. Raw retransmissions remain deliberate new commands, with no automatic deduplication or retry.

## Implementation defaults: browser contract

Use these interfaces for the first implementation slices. Browser routes use `/api`; the internal script endpoints are never mounted on that public application. JSON field names are snake_case. Attempt, Workspace, runtime, tab and pass IDs are UUID strings; Challenge/scenario IDs are the stable slugs specified by the curriculum. Times are UTC RFC 3339 strings, and packet bytes lowercase hexadecimal without spaces. Hex fields represent history and recordings; raw uplinks remain binary. Apply the 256-byte packet limit before delivery and validate hex when checking recordings.

**Shared records.** `Progress` is `{completed_challenge_ids: string[]}`. A `Challenge` is `{challenge_id, title, availability: available|locked, prerequisite_ids: string[], completed: boolean}`; include exactly the three MVP IDs and compute availability on the server. `Control` is `{tab_id: string|null, generation: integer}`. `Check` is `{field, passed: boolean, message}`. `PassView` is `{scenario_id, pass_run_id, status: not_started|running|finished|interrupted, receiver_ready: boolean, received_count: integer}`.

`AttemptView` contains `{attempt_id, workspace_id, challenge_id, status, state, goal_reached, completion_state, checks: Check[], station_setup: StationSetup|null, setup_verified: boolean, pass: PassView|null, runtime_status, runtime_id, control, last_activity_at}`. Lifecycle `status = preparing|active|failed|ending`; `runtime_status = starting|ready|failed|stopped`, with nullable `runtime_id`. `completion_state` follows Task completion above. `state` uses the model fields, string mode, booleans and integer counters; `last_seq_seen` is null before acceptance (binary `0xffff`). Hello has null station/pass; Ready has station setup only; Catch has both. Preparing ends only when required managed files and runtime are available. Station readiness, runtime readiness, and saved completion are distinct.

A `HistoryEntry` has `{entry_id, time, direction: uplink|downlink, packet_hex, origin: raw_player|satellite, submission_id: string|null, pass_run_id: string|null, index: integer|null, scenario_time: string|null, feedback: {code, message}|null}`. Use a per-Attempt increasing `entry_id`; retain original packets including unreadable submissions. Link replies to actual submissions; autonomous telemetry has no submission ID. Pass metadata is null outside Catch. Feedback is Platform explanation, never a fabricated reply or Verdict. `Snapshot` is `{view: AttemptView, history: HistoryEntry[], progress: Progress, revision: integer}`. No bearer credential belongs in these records.

| Method and path | Request → successful response |
| --- | --- |
| `GET /api/challenges` | `{challenges: Challenge[], progress: Progress}` |
| `GET /api/session` | `{status: idle\|owned\|busy\|maintenance, attempt_id: string\|null, workspace_id: string\|null}`; only the owner receives IDs. No personal data for busy browsers. |
| `POST /api/attempts` | `{challenge_id, tab_id}` → 201 `Snapshot`; acquire the idle slot and set the ownership cookie. Never replace a running Attempt through this route. |
| `GET /api/attempts/{id}` | Owner → `Snapshot`. |
| `POST /api/attempts/{id}/control` | Owner supplies `{tab_id}` → `Control`; explicit Take control atomically increments generation. |
| `POST /api/attempts/{id}/actions` | `{action: restart\|switch\|stop\|reset_workspace\|reopen_runtime, challenge_id?: string}` → `{snapshot: Snapshot\|null}`; `challenge_id` required only for switch; stop returns null after cleanup. |
| `PUT /api/attempts/{id}/station-setup` | Ready only, complete `StationSetup` → `Snapshot`; validates types/ranges, saves even semantically wrong training choices and clears `setup_verified`. |
| `POST /api/attempts/{id}/setup/check` | Ready only, `{}` → `Snapshot` with per-field checks; all passing verifies setup and attempts to save completion. |
| `POST /api/attempts/{id}/pass/start` | Catch only, `{}` → `Snapshot`; requires correct setup and attached receiver. Never restarts an existing pass. |
| `POST /api/attempts/{id}/work/check` | Catch only, `{recording_path, recording_version, log_path, log_version}` → `Snapshot` with evidence checks and completion state. Paths are relative to the authored Catch folder. |
| `POST /api/attempts/{id}/completion/retry` | `{}` → `Snapshot`; retry only previously verified unsaved completion, subject to current evidence validation. |
| `GET /api/attempts/{id}/script-access` | Owner → `{url, credential, protocol: raw\|receive}` or 409 `SCRIPT_UNAVAILABLE` for Ready; `Cache-Control: no-store`. Managed helper configuration uses the same values. |
| `POST /api/attempts/{id}/activity` | Visible owning page heartbeat → 204, at most once every 30 seconds; observers in that browser may keep it alive. |
| `POST /api/progress/reset` | `{confirm: "RESET DEMO PROGRESS"}` → `Progress`; allowed only while idle, atomically excluding new ownership. Available to any approved teammate; local development follows the same rule. |

Every owner-only route checks the cookie. HTTP reads reject a different Origin when present; mutations and WebSocket handshakes require the exact allowed Origin. Do not enable cross-origin credentialed access; every mutation of an active Attempt except explicit Take control and heartbeat also requires `X-Tab-Id` and `X-Control-Generation`. Start establishes generation 1. Restart, Switch and Reset workspace preserve the controlling tab but increment its generation. Validate a requested Challenge, its availability and any Ready → Catch handoff before revoking the current Attempt; an invalid Switch leaves the running session intact. Store tab ID and current grant in sessionStorage; a new tab must generate its own ID, including when opened from another tab. Recheck grants and Attempt identity when applying queued work. Use one lifecycle lock for ownership/admission/transitions; serialize Sim packet ingestion and snapshot/subscription together. File and terminal work use the same revocation boundary. Do not hold the event loop during Docker or filesystem operations.

A wrong setup/answer is successful evaluation with failed checks, not a transport error. A verified task whose database write failed returns 503 `STORAGE_UNAVAILABLE`; the subsequent snapshot exposes `save_failed` and retained checks/evidence. Each check reads the current server-owned setup. For Check work, use the Workspace's checked descriptor/path rules and per-file locks, read both bounded files into immutable buffers and match their content hashes to the supplied versions. Validate these buffers, never execute them. A different/moving file gives 409 `FILE_CONFLICT`; the shell is not a collaborative lock participant, so the check certifies the captured versions, not all future writes. Retain validated hashes/context for Retry saving; if either current file differs, require a new Check work. Hold the logical Attempt operation lock through validation and the database commit; no lifecycle replacement can overtake persistence.

Errors use the common envelope: 403 `NOT_OWNER`, `INVALID_ORIGIN` or `NOT_CONTROLLER`; 409 `PLATFORM_BUSY`, `STALE_CONTROL`, `CHALLENGE_LOCKED`, `FILE_CONFLICT`, `SETUP_REQUIRED`, `RECEIVER_REQUIRED`, `RECEIVER_BUSY`, `SCRIPT_UNAVAILABLE`, `WRONG_CHALLENGE`, `EVIDENCE_REQUIRED`, `PASS_ALREADY_STARTED` or `TRANSITION_IN_PROGRESS`; 410 `ATTEMPT_ENDED`; 413 `TOO_LARGE`; 422 `INVALID_INPUT`; 503 `RUNTIME_UNAVAILABLE`, `CLEANUP_FAILED`, `MAINTENANCE` or `STORAGE_UNAVAILABLE`. Unknown resources use 404. Grant loss is distinct from a stale file version. Disable affected mutation controls while stale/disconnected; always enforce the same restrictions server-side. No browser packet builder, packet-submission or Flag route is required.

**Attempt WebSocket:** `/api/attempts/{id}/events`, owner cookie and Origin required. Server first sends `{type: "snapshot", attempt_id, revision, data: Snapshot}` with matching inner revision. Each subsequent `{type: "update", attempt_id, revision, data: {view: AttemptView, history_append: HistoryEntry[], progress: Progress}}` replaces the small public view and appends only new packets. Increment revision for every published update, including control/readiness/progress changes; send no separate private event type. On termination send `{type: "ended", attempt_id, revision, reason}` then close normally. An HTTP read can recover current state but does not substitute for the atomic WebSocket snapshot/subscription. Reconnect with a fresh snapshot; back off 1, 2, 4, 8, then 15 seconds maximum while the page is visible. Never replay mutations.

Define these records once with Pydantic, including a discriminated union for event `type`. Explicitly include the event models in OpenAPI `components.schemas` for TypeScript generation; do not invent HTTP routes to make them appear. Check in generated TypeScript, and make CI export/regenerate and fail on a difference. The browser connection module checks event type, IDs, integer revision and required fields before dispatch; invalid messages trigger a visible protocol failure and fresh synchronization. See [OpenAPI TypeScript schema guidance](https://openapi-ts.dev/advanced).

## Implementation defaults: simulation and storage

- Use a deterministic Software Link with ordered delivery, no artificial latency, drops or contact windows. Process one command at a time. Use real reply delivery for Hello evidence. Ground Sim and downlink sender counts follow the packet spec; helper-generated raw input retains its supplied sequence as Player-crafted bytes; Catch uses its finite scenario source instead of the live timer.
- For Hello, keep battery at its initial 100%. Uptime is whole elapsed monotonic seconds since Attempt start or the latest REBOOT. REBOOT sets SAFE, heater off and uptime zero; preserves battery, physical antenna state, deployment/accepted-command counts, goal latch and all Attempt/sequence history. It is itself an accepted command. No battery-related command gating or asynchronous reboot downtime.
- First SQLite migration creates `schema_migrations(version INTEGER PRIMARY KEY)` and `challenge_completion(challenge_id TEXT PRIMARY KEY, completed_at TEXT NOT NULL)`. There is one shared profile, so no user/account table. Completion insert is conflict-ignore; preserve the first UTC completion timestamp. Reset deletes completion rows only. Each connection uses a 5-second busy timeout; serialize short writes through the shared module. Use SQLite's default rollback journal for this single-process MVP; add WAL only if measurements justify it.
- No future lesson tables, account migrations, background job system or general Defense framework are prerequisites for the operations MVP.

## Implementation defaults: hosting

- Initial validation target: the HP host selected above running Ubuntu LTS x86-64. Its stated hardware is not a capacity guarantee; validate actual runtime limits and builds for the one-active-Player demo. Diab is the deployment/Access maintainer.
- Serve the built UI, `/api` and browser WebSockets on one origin through an Nginx Compose service. Proxy API/WebSocket traffic to the single server worker; use SPA fallback only for UI routes. Local Vite proxies `/api` (including WS) to the server for the same cookie behavior. Backend readiness is `/health`; return 503 until migrations and abandoned-resource cleanup finish.
- Use a Cloudflare Tunnel from that origin with no public application ports; keep SSH as a separate key-authenticated management path. Cloudflare Access gates the whole hostname with the approved-email list and an 8-hour session. Restrict origin ingress to the tunnel service and validate the Access JWT at the origin against the configured audience/issuer, signature and expiry, following [Cloudflare’s validation contract](https://developers.cloudflare.com/cloudflare-one/access-controls/applications/http-apps/authorization-cookie/validating-json/). Tunnel/Access configuration is operational setup, not a Player secret. If access expires, preserve drafts, explain reauthentication, then reconnect; never infer an expired Attempt or resubmit work from an access-gate redirect.
- Build revision-tagged images in GitHub Actions and pull them from GHCR. The manual workflow resolves an immutable `main` commit and checks that exact SHA's required checks. Serialize deployments with cancellation disabled; connect over SSH using GitHub Environment secrets. No self-hosted Actions runner or Docker administration endpoint is reachable from Player code.
- A host deployment command takes an exclusive deployment lock, calls an internal-only admission operation over a Unix socket (idle check and close admission are atomic), and places a persistent maintenance marker outside the containers before replacement. On startup that marker keeps new admission closed. Keep the database volume; run migrations and smoke-check the deployed revision, then explicitly clear maintenance and reopen. Only the deploy identity can access this control socket. Failure leaves maintenance visible; a maintainer repairs or redeploys deliberately rather than automatically rolling back an incompatible schema. Access-gated Nginx serves the maintenance page even if the API is restarting.
- Before the first shared demo, verify Access denial, origin bypass prevention, browser and terminal WebSockets, resource limits, an idle update, a busy refusal, and progress surviving a restart. Host setup and domain purchase are separate from these implementation defaults. KSAT-24 validates the selected host and access; KSAT-25 must establish a secure deployment management path to the home host.

## Acceptance checks

- An update request during an owned session leaves the running demo and Player work untouched. An idle update blocks new sessions, deploys only the selected tested revision, preserves saved completion, and reopens only when healthy. Concurrent update requests cannot interrupt an update already in progress.
- Verify fresh database initialization, repeated startup, upgrades preserving completion, and rollback/startup failure on a failed migration. Repeated valid completion creates one record; failed persistence never reports success. Reset progress preserves the database schema and migration history.
- Disconnect around an accepted PING and its reply, then reconnect: the snapshot restores current state and received history without a duplicate command. Verify snapshot/stream handoff during concurrent updates, duplicate revisions, revision gaps, and late messages from superseded connections or Attempts.
- Verify browser controls and script actions use the same Attempt; Hello replies and Catch packets in history match actual Link deliveries. Reconnect must not resend packets.
- Reject control from a different browser, invalid origins, and wrong or expired script credentials. Ended Attempts must reject queued traffic and cannot affect replacements.
- Reject false completion from submission alone, incorrect settings, missing/old recordings or wrong log facts. Failed persistence does not unlock the next Challenge; Retry saving never reruns a packet or pass.
- Refresh restores state, history, and goal status. Restart replaces state, pass identity and access while preserving solved progress. A server crash ends the unfinished Attempt but retains saved completion.
- Reset demo progress restores the initial Challenge progression; it is separate from Reset workspace, which preserves completion.
- Check every internal operation and raw delivery rule above, plus [Hello](challenges/01-hello-satellite.md#acceptance-examples), [operations activities](ground-station-training.md#mvp-acceptance-checks), and [Workspace lifecycle and fault checks](terminal-workspace.md#focused-integration-and-fault-checks).

# Integrated terminal and Workspace — implementation spec

Status: agreed operations-MVP requirements and implementation defaults, not implemented features.

Start with the [project plan](project-plan.md). This file owns terminal execution, shared files, tab control, retention, and recovery for the operations MVP. The [operations curriculum](ground-station-training.md#mvp-operations-curriculum) owns the three MVP learning activities; broader PSB lessons are later work.

## Problem Statement

The Player needs to read a Challenge, inspect its resources, edit and run a script, and understand the Satellite Sim's response without leaving the Platform or installing local tools.

## Solution

Provide a real Linux terminal and file editor inside the selected Challenge workspace, with Python, the public helper, and useful file-inspection tools prepared in advance. The terminal and editor share saved files. The Player still performs the scripting task; the Platform supplies the environment with connection boilerplate and a supplied decoder for Catch and Log.

Use replaceable Docker execution with independently retained Workspace storage and an authenticated terminal gateway. Keep the Web Backend and Sim Service together in their existing Python process. One browser owns the active Attempt; one tab within that browser controls terminal input and editing. Explicit Take control transfers that tab role without interrupting programs. Explicit Save and file-version checks protect against stale editor writes.

The first deployment is internal-team-only and must be available at a shared hosted URL, with Player execution on the host. Local Docker development remains supported. Hosting and access follow the [Backend deployment section](backend-and-simulation.md#ownership-and-deployment). The general shell has access only to required Platform endpoints. Hello uses a supplied terminal command to send PING and inspect its reply, without requiring the Player to write Python. Catch and Log uses a short Python starter to receive, record and interpret the practice pass. The mission map remains contextual and does not determine communication availability.

## Implementation Decisions

### Application and responsibility boundaries

- Use the selected desktop workspace layout: stable Challenge navigation, a central work area with the terminal dock available, and supporting Briefing/task-completion/telemetry views. Preserve readable labels, keyboard focus, tab semantics, and the distinction between submission, reply receipt, verified task evidence and committed completion. The contextual map adds no orbit physics; simulated pass timing follows the operations scenario.
- Integrate xterm.js into the React interface. A trusted terminal gateway authenticates browser access, transports interactive input/output and resize/interrupt operations, and attaches to a real shell. Reattaching is not executing a new command. Terminal output is untrusted display data.
- FastAPI retains browser ownership, Attempt lifecycle, access control, completion verification, and SQLite progress. The Sim Service retains packet processing, Ground Sim, Software Link, Satellite Sim state, and goal detection in the same Python process. Terminal and filesystem responsibilities do not move into the Satellite Sim.
- A trusted runtime controller creates, attaches, replaces, and destroys the Docker execution container and manages Workspace storage and managed Attempt resources. The controller placement below makes that responsibility concrete while preserving these authorization and failure boundaries. Player code receives no infrastructure-control interface.
- Frontend work includes layout, browsing files, terminal, editor/notes, save/conflict state, control transfer, downloads, keyboard access, and recovery feedback. Backend integration includes ownership enforcement, files, runtime control, provisioning, lifecycle coordination, and cleanup. The [project role table](project-plan.md#who-owns-what) assigns Lily and Denzel the main Workspace work, with Kamilla and Diab handling lifecycle and simulation integration. Revised estimates belong to ticket planning.

### Execution, storage, and endpoint isolation

- The MVP supports Player scripting inside the embedded Workspace only. External scripts on a Player's own computer are deferred. Keep the raw and receive endpoints behind the restricted internal bridge; downloads of recordings, saved scripts, and notes do not grant external access.
- Maintain one active Player execution environment. Run a prepared non-root Linux image with shell, Python, the public helper, and selected file-inspection tools. Use a read-only base image, dropped capabilities, no privilege escalation, and bounded writable storage. Player package installation and unrestricted internet access remain outside the MVP.
- Keep authored Workspace storage independently attached across execution-container replacement. The demonstrated quota-limited volatile storage fits the accepted lack of personal-work recovery guarantees after a server crash. Do not rely on the lifetime of the Player container to retain authored files.
- Keep managed Attempt resources separate from authored Challenge folders. Provisioning fresh credentials, pass briefs or templates must not overwrite authored scripts, recordings, logs or notes. Disposable home directories, temporary files, and shell state belong to execution and are included in cleanup.
- Expose only the permitted raw and receive endpoints through a narrow bridge or proxy. The prototype's no-external-network container with a fixed Unix-socket bridge is the implementation reference; an equivalent mechanism must prove the same endpoint restriction. A generally reachable internal network is insufficient. Direct access to the permitted bridge must still enforce Attempt-bound bearer authentication.
- Never mount server source, private configuration, progress storage, or Docker control access into Player execution. Treat file operations, managed-resource provisioning, and downloads as traversal- and symlink-sensitive operations within authorized storage.
- Continue using the raw endpoint's opening-header bearer credential, complete binary-message boundary, size limit, original-byte preservation, and server-derived submission origin. The terminal gateway does not replace the raw packet endpoint. Changing controlling tabs does not rotate the Attempt's script credential or interrupt authenticated running scripts.
- Resource and output limits must be configurable and tested. Start from the prototype's values below, then adjust from measured workload requirements; these are initial engineering values, not immutable product limits or performance guarantees.

| Resource | Initial value to validate |
| --- | --- |
| Player CPU and memory | 0.5 CPU; 128 MiB memory, no additional swap |
| Processes and file descriptors | 64 processes; 256 file descriptors |
| Authored Workspace storage | 64 MiB |
| Disposable temporary/home storage | 16 MiB / 4 MiB |
| Managed Attempt resources | 1 MiB |
| Gateway terminal history / browser scrollback | 128 KiB / 1,000 lines |

Bound queues and apply backpressure; sustained output, slow readers, partial terminal escape sequences at reconnect, and overflow recovery must not produce unbounded buffers or rerun programs. A bounded visual history is not an authoritative replacement for Attempt message history.

### Browser ownership and controlling tab

- Reuse the existing server-issued browser ownership and Origin validation. Another browser cannot acquire an active Attempt, view personal files, or attach to its terminal. Shared completion provides no Workspace authorization.
- Within the owning browser, one tab controls terminal input and editing; other tabs observe and can explicitly Take control. Use server-enforced tab identity and a changing control generation, not merely disabled buttons. Reject mutations from a tab whose grant is stale at the time of application.
- Transfer control atomically. Old terminal input and editor saves queued under the revoked generation must not be applied afterward. Leave the shell, running programs, Workspace, and Attempt intact. Preserve an old tab's unsaved draft visibly as unsaved, but do not let it save until it reacquires control and passes file-version checks.
- A refresh may restore the same tab's control only if its grant is still current. It must not steal control from a newer tab. Concurrent Take control requests must result in one current grant. Closing a tab alone does not stop execution; another owning-browser tab can take control explicitly.
- Coordinate browser lifecycle mutations through the controlling tab so that pending edits and transition notices can be handled consistently. Runtime or server-driven expiry remains authoritative even if a browser has unsaved work. Tab control does not grant access to other browsers or add accounts.

### Editor, notes, and saves

- Use CodeMirror 6 for the editor. Include Python syntax highlighting, line numbers, indentation, find/replace, and explicit Save. Notes use plain-text editing. Autocomplete, diagnostics, and debugging are deferred. Run saved scripts in the existing terminal; the editor does not add a separate execution environment. Shared controls and styling follow the [Frontend spec](frontend.md).
- Use explicit Save for scripts, session logs and notes, with Saved, Unsaved, Saving, Conflict, and Failed feedback as appropriate. Mark a file saved only after an authoritative success response. Running a file uses saved contents; unsaved editor text is not silently executed.
- Reads return the content and a version derived from the actual file state. A save supplies the version it was based on, current Workspace/Attempt context, and current tab-control grant. Validate these before applying the mutation. Serialize editor saves per file, reject a mismatched version, and replace successfully saved content atomically.
- Stale-file checks must notice terminal-side changes already present when the save is validated, not just earlier API saves. A general shell may write files outside the editor's coordination; atomic editor replacement is not a collaborative lock on arbitrary shell writers. Test this boundary explicitly and document limitations of a program continuously rewriting a file.
- On conflict or save failure, keep the unsaved draft and the current stored file unchanged by that rejected save. Offer a way to reload/compare or save the draft under a different name; do not silently force an overwrite or automatically merge.
- Before user-initiated Restart, Challenge switch, Stop, or Reset workspace, require affected pending edits to be saved or explicitly discarded. Keep separate Reset workspace confirmation. Use an appropriate browser warning for unsaved navigation/refresh; retained-work guarantees cover successfully saved content, not drafts lost to a browser or server crash.
- Keep authored work in per-Challenge folders. Provide downloads of saved scripts, recordings, logs and notes. Stop explains deletion before proceeding. Downloads made by the Player are outside the Platform's cleanup responsibility; research logging/export remains deferred.

### Operations integration and readiness

Hello receives a prepared PING example. Ready receives the pass brief and graphical controls. Catch receives a connection starter, decoder reference and session-log template; the Player writes their own recording. The [operations defaults](ground-station-training.md#implementation-defaults-practice-pass-and-evidence) own scenario data, file formats and checks. Do not provision a completed recording or an automatic solve script.

Expose connection details in managed resources separate from authored/downloadable code. Replace these resources for each Attempt; retained scripts that cache stale credentials receive clear access errors. Hello's helper waits for an actual correlated reply; Catch's receiver readiness comes from an authenticated stream subscription, not an open terminal or a printed message.

Represent environment preparation, receiver readiness, pass progress and saved completion separately. A working shell or fixture echo cannot set a real goal. The Backend verifies Sim/Link evidence and saved files before committing completion. Downloading a saved recording/log preserves the bytes on disk; it does not silently save an editor draft.

### Lifecycle and ordering

Challenge switching below means explicitly replacing the active Attempt. Browsing another Challenge's page or using Back/Forward does not switch the active Attempt; see [Frontend navigation](frontend.md#navigation-and-active-attempt).

| Event or action | Execution and access | Personal work | Attempt and completion |
| --- | --- | --- | --- |
| Refresh or temporary disconnect | Keep programs running while Attempt is active; reattach without reexecution. | Retain saved files. | Restore the same Attempt and history. |
| Attempt Restart | Revoke old access and replace the entire runtime, including detached descendants. | Retain authored files; replace managed resources. | Fresh Attempt/history/pass identity; retain completion. |
| Change Challenge | End old execution/access and provision for an available Challenge. | Retain authored files in per-Challenge folders. | Fresh Attempt; retain completion. |
| Stop / end session | Revoke access, terminate runtime, verify cleanup, then release ownership. | Clear personal work after explaining the consequence and making downloads available. | End Attempt; retain completion. |
| Idle expiry | Terminate execution/access under the existing inactivity policy. | Clear personal work before reuse. | End Attempt; retain completion. |
| Reset workspace | Confirm, revoke access, and replace execution. | Erase authored scripts/recordings/logs/notes and restore starters. | Fresh current-Challenge Attempt; retain completion. |
| Execution-only failure | Require explicit reopening; never rerun commands. | Retain saved work independently of failed runtime. | Preserve active Attempt, Satellite Sim state, and history. |
| Python server failure | End old access and reconcile abandoned runtime before admitting a new Player. | Recovery not guaranteed; remove abandoned personal data before reuse. | Unfinished Attempt ends; SQLite completion survives. |

For replacement, coordinate a revoke → terminate entire old runtime → verify cleanup → prepare new Attempt resources → attach new runtime transition while retaining browser ownership. Ready → Catch handoff must validate and copy the prepared context before revocation; publishing a log template requires the new pass ID. Bind queued packet delivery, file mutations, and terminal input to their current Attempt/control/execution generations; recheck at application time. Never automatically retry packet submission when completion is uncertain.

Stop and expiry clean authored storage, managed resources, temporary/home data, terminal buffers, shell history, and stored credentials in Platform-controlled storage. If a required cleanup step fails, keep ownership unavailable to another Player and show an actionable failure. Startup reconciliation must clean abandoned resources before handing out ownership. Destroy only resources owned by this Platform deployment.

Retain the 30-minute inactivity policy. A watching owning-browser page and accepted Player actions such as terminal input, file saves, and raw packet submissions count as activity. Idle sockets, receive-only telemetry, and autonomous terminal output do not. Explicit setup/pass/check actions count; receiving alone cannot indefinitely retain ownership. Disconnect alone does not trigger immediate cleanup. Reset workspace and Reset demo progress remain distinct; only the latter clears shared completion.

### Interface responsibilities

| Interface | Required contract |
| --- | --- |
| Workspace read | Owning-browser authorization; current Workspace/Attempt status, managed-resource readiness, file versions, and tab-control state; never server-private configuration. |
| Take control | Owning-browser authorization and atomic generation change; previous tab loses mutation rights; execution continues. |
| File read/save/download | Authorized storage, safe path resolution, actual-file versions, current control validation on saves, atomic success or explicit conflict/failure, bounded sizes. |
| Terminal attach/input/interrupt | Browser ownership and Origin validation; current runtime identity; controller-only mutations; bounded stream and no command replay. |
| Lifecycle coordination | Idempotent infrastructure cleanup, serialized transitions, retained ownership until safe reuse, stale access rejection, and explicit recovery. |
| Script packet access | Existing raw authentication, exact binary bytes, current Attempt validation at delivery, authoritative origin, real Sim replies and pass telemetry. |

## Implementation defaults: runtime and files

The requirements and implementation defaults are agreed. The prototype supports feasibility, not completed production acceptance.

**Controller and storage.** Put the trusted controller and terminal gateway inside the single FastAPI process. Use the Python Docker SDK from a worker thread for blocking operations; pin it in the server lockfile. Only the trusted server mounts Docker's control socket. Give every managed container/volume a deployment label and every execution a fresh `runtime_id`; cleanup uses those labels and recorded IDs, never broad Docker prune commands. A lifecycle transition revokes grants before awaiting Docker cleanup; no long Docker operation holds the event loop.

Use Docker named volumes backed by bounded Linux tmpfs: authored files 64 MiB, managed resources 1 MiB, raw bridge 1 MiB. Keep them mounted in the trusted server so replacing Player execution does not discard files. Mount authored storage read/write at `/workspace`, resources read-only at `/attempt`, bridge read-only at `/bridge` in Player execution. Use UID/GID 1000 for Player files. Host restart may lose these volatile volumes; startup cleans them before reuse. Persistent SQLite lives separately. [Docker volume options](https://docs.docker.com/reference/cli/docker/volume/create/) document the tmpfs-backed named-volume mechanism.

Run Player execution with network `none`, read-only root, all capabilities dropped, no-new-privileges, and the limits above. Prepare Bash, Python, `ls`, `cat`, `cp`, `mv`, `rm`, `mkdir`, `head`, `tail`, `xxd`, `od`, `sha256sum`, the helper, and `socat` in the image. Use a fixed loopback listener at `127.0.0.1:8766` forwarding only to `/bridge/raw.sock`. A second, script-only ASGI listener in the existing Python process serves that Unix socket; expose only `/raw/attempts/{attempt_id}` and `/receive/attempts/{attempt_id}` there, with bearer authentication on every opening handshake. It shares the actual Sim instance. Public API, progress, filesystem and Docker operations never appear on this listener. The Player can reach the socket directly, so security belongs in its handler rather than the forwarding command.

Create one Bash PTY per runtime, starting in `/workspace/{challenge_id}`; Ctrl-C uses the PTY, not a special Python runner. Closing a terminal tab only detaches. A shell exit or container failure marks runtime failed and offers explicit Reopen terminal; it does not automatically launch or rerun anything. Reopen replaces the entire runtime and preserves the current Attempt and authored volume. Runtime destruction always removes detached descendants as well as the visible shell.

**Files.** `workspace_id` identifies the owning session's authored storage: it survives Restart/Switch, changes on Reset workspace, and disappears on Stop/expiry. Provision only missing authored starters: Hello gets `ping_example.py` and `notes.txt`; Ready gets `notes.txt`; Catch gets `main.py` (connection boilerplate with task comments) and `notes.txt`. Files stay in `/workspace/{challenge_id}`. User-created scripts, recordings and logs are never overwritten by provisioning.

Managed, read-only resources: `/attempt/connection.json` holds `{attempt_id, url, credential, protocol: raw|receive}` for Hello/Catch; Ready has no connection file. The URL uses the loopback listener and the current route. Ready/Catch receive `/attempt/pass-brief.json`; Catch also receives `/attempt/log-template.json` with current scenario/pass/setup metadata and null result fields. On each Catch run, instruct the Player to copy this fresh template to a chosen authored log filename before filling it in; if that filename exists, offer a new name or explicit overwrite. Never silently update an old authored log to pretend it belongs to a new pass. The template and brief formats are owned by the operations spec. No completed recording is supplied.

The helper reads the current managed connection file at invocation, so distributed starters contain no live credentials. The owning browser may inspect managed resources, but the normal UI never logs or exports credentials with a starter/download bundle. Personal files that a Player deliberately puts credentials into remain their own authored content; explain that credentials expire with the Attempt.

All file routes require ownership, a current Attempt and matching `workspace_id`; writes additionally require the current tab grant. Use the API conventions and errors in the [Backend contract](backend-and-simulation.md#implementation-defaults-browser-contract).

| Method and path | Request → response |
| --- | --- |
| `GET /api/attempts/{id}/files` | Query `workspace_id`, `root=authored\|resources`, `path` (relative directory, empty for root), `offset=0` → `{entries: [{path, kind: file\|directory\|symlink, size_bytes, read_only}], next_offset: integer\|null}`; 200 entries/page, lexical order. Symlinks are listed as unavailable. |
| `GET /api/attempts/{id}/file` | Query `workspace_id`, `root`, relative `path` → `{path, size_bytes, version, editable, text: string\|null}`. UTF-8 text at most 1 MiB is editable; binary/larger files show metadata and download. Resources are always read-only. |
| `PUT /api/attempts/{id}/file` | `{workspace_id, path, base_version: string\|null, text}` → `{path, version, size_bytes}`; authored root only, UTF-8 at most 1 MiB. Null version means create only if absent, including Save as. |
| `GET /api/attempts/{id}/download` | Same query as file read → attachment with original bytes, bounded to 64 MiB. Read through a checked descriptor; do not accept an arbitrary host path or serve active HTML inline. |

File versions are opaque SHA-256 digests of actual bytes read from disk, never an API-only counter. Under the per-file save lock, read current content again, compare the supplied version, validate current lifecycle/grant, then write a temporary file and atomically replace within the same checked directory. No success response before the replace succeeds. Quota failure preserves the existing file and removes the temporary file. A concurrent shell writer remains outside that lock, as described above.

Resolve relative paths by walking directory descriptors beneath the selected volume with no symlink following. Reject absolute paths, `..`, NUL, symlinks, non-regular-file reads and special devices/FIFOs; do not rely on a string prefix check or a preflight `resolve()` followed by an unchecked open. Anchor creates/replaces to the checked parent descriptor, recheck context before committing, and close all descriptors. A short shared apply gate serializes the final grant check and replace with control revocation; awaiting a worker must not let an already-revoked save commit later. Rename/delete/new directories are terminal operations for MVP; the editor includes Save as without a separate file-management framework. Refresh the listing on panel focus and after saves; re-read an open file on editor focus to detect shell edits without replacing a dirty draft.

## Implementation defaults: terminal and helper contracts

Browser terminal socket: `/api/attempts/{id}/terminal`, authenticated by cookie and Origin. First client message is `{type: "attach", runtime_id, tab_id, generation, after_offset: integer|null}`. Observers may attach/read; every input, resize and interrupt verifies the current grant/runtime again at application time. Payloads are JSON; terminal byte strings use base64, not Unicode conversion of arbitrary output.

| Direction | Message |
| --- | --- |
| Server → browser | `{type: "attached", runtime_id, start_offset, end_offset, history_lost: boolean}` |
| Server → browser | `{type: "output", runtime_id, offset, data_base64}`; offset counts raw bytes from runtime start, independent of Attempt-event revision. |
| Browser → server | `{type: "input", runtime_id, tab_id, generation, data_base64}`; no retry/replay, including after reconnect. |
| Browser → server | `{type: "resize", runtime_id, tab_id, generation, cols, rows}`; bounds 20–300 columns and 5–120 rows. Ctrl-C is ordinary input byte `0x03`. |
| Server → browser | `{type: "error", code, message}` or `{type: "closed", runtime_id, reason}`. No terminal text becomes an HTML fragment or application event. |

Use xterm's fit addon to compute rows/columns; the controlling tab alone resizes the shared PTY. Limit decoded input to 8 KiB/message, output chunks to 8 KiB and each subscriber queue to 128 KiB. Retain the agreed 128 KiB gateway ring and 1,000-line browser scrollback. Replay output only, with offsets preventing duplicate display. If a retained xterm instance can resume at an available offset, send the exact missing bytes, even across split escape sequences. A fresh xterm can replay safely from byte zero only. If that history is gone or a gap appears, reset its parser, show “Earlier terminal output is unavailable,” and attach at the current tail without replaying a ring that starts mid-sequence; the Player may redraw manually. Do not send a shell command to reconstruct output. Disconnect slow subscribers on queue overflow; continue draining the PTY into the bounded ring. Bound client rendering queues with the same overflow behavior; output floods cannot accumulate unbounded browser memory. Disable automatic terminal hyperlinks, clipboard access and unsupported terminal escape extensions.

Use a small synchronous Python module named `kss_client` backed by pinned `websockets`. Pin compatible helper/packet versions into the Player image. Document imports, methods, formats and the exact terminal command beside each starter; assume basic Python knowledge.

- `load_connection()` reads `/attempt/connection.json` and returns its documented fields. `connect(url, credential)` is a context manager for the raw endpoint, using the bearer opening header. `send(packet: bytes)` preserves bytes. `receive(timeout_s=5)` returns either Packet (`data: bytes`) or Feedback (`code`, `message`), and raises TimeoutError if no message arrives. Raw text feedback is `{type: "feedback", code, message}`. No implicit reconnect or retransmission.
- `build_ping(sequence, role)` constructs the documented packet; the Hello example uses guest role and sequence 0 per invocation. Repeated invocations are deliberate transmissions and may reuse sequence 0 with Defenses off. `decode_packet(packet)` verifies supported downlink header/length/CRC and returns `{kind, sequence, state, reply}`: kind is `telemetry` or `reply`; `state` uses the Backend model fields, and `reply` is null or `{command_code, command_sequence, status}`. Invalid packets raise a readable `ValueError`; decoding never repairs or sends bytes.
- `python ping_example.py` sends exactly one PING and waits up to five seconds total for the matching successful PING reply, displaying actual fields and ignoring periodic telemetry as completion evidence. On timeout/connection failure it says no reply was observed; it never resends. Server goal correlation uses the actual ingestion event, not just the reusable wire sequence. Raw downlinks are live only; reconnect history is available in the browser, not silently replayed by the helper.
- `receive_pass(url, credential)` is a context manager for the receive endpoint. The starter opens it and prints “Receiver ready; use Start pass in the website.” Connection establishment consumes the server's initial ready event. `stream.records()` yields dictionaries in the [recording format](ground-station-training.md#implementation-defaults-practice-pass-and-evidence), blocking until the Player starts the pass and then until each packet arrives. The Player adds the loop, file writing and decoding with `decode_packet(bytes.fromhex(record["packet_hex"]))`. End of the pass finishes iteration. Interruption/connection loss raises a readable error and leaves a partial file for inspection. Waiting before Start has no short receive timeout; activity/expiry still follow Backend rules.

**Receive wire contract.** An authenticated connection to a `not_started` Catch pass receives `{type: "ready", pass_run_id}` only after subscription is registered. Subsequent JSON messages are `{type: "packet", record: {pass_run_id, index, scenario_time, packet_hex}}`, then `{type: "end", pass_run_id, count: 12}` and normal close; errors use `{type: "error", code, message}` and close. No client data messages are allowed. Connections during running/finished/interrupted passes are rejected with `PASS_ALREADY_STARTED`; use explicit Restart for another reception. A receiver disconnect before Start merely clears readiness; after Start it interrupts the pass unless the source already finished. A successful pass is marked finished before normal close, so closing after `end` is not an interruption. The helper checks message shape/ID/index and reports gaps or invalid frames without filling them in. Connection-only starter code is fully supplied; recording and extracting readings remain the task.

The Catch terminal runs the saved script with `python main.py`. Explicitly explain that saving a file, running a program, receiving all packets and passing Check work are different steps. Editor and terminal use the same authored files.

## Testing Decisions

The user confirmed one primary acceptance boundary: the Player journey through the browser and real Backend/Sim, supported by focused runtime-cleanup, isolation, and stale-save tests. Test observable outcomes rather than mirroring internal helpers or counting implementation calls.

### Primary acceptance suite

1. Run the Hello terminal command and receive its real reply; verify committed completion unlocks Ready. Correct the setup, enable tracking and check it; continue to Catch, adapt/save/run Python, explicitly start the pass, save its recording and complete/save the log. Check work and verify SQLite completion. Use the real Sim/Link, runtime and filesystem.
2. Reject false success from Hello submission alone, wrong setup, incomplete/stale recordings or incorrect log facts. Check field-level recovery, receiver-before-start behavior, pass interruptions and failed persistence without automatic reruns. Exercise independent repeats and reject direct starts of locked Challenges.
3. Refresh a running program and verify the same execution continues with no duplicate invocation or submission. Recover reply/pass history and completion state without retransmitting.
4. Restart and switch Challenges with saved scripts/recordings/logs/notes and background processes: work remains, descendants are terminated, old credentials and queued traffic cannot affect the replacement, and new resources belong to the new Attempt.
5. Exercise multi-tab control transfer: observers cannot mutate, stale requests are rejected, programs continue, refresh cannot reclaim a newer grant, and unsaved drafts are not silently submitted or discarded.
6. Exercise explicit saves, saved/unsaved run behavior, terminal-side prior edits, concurrent API saves, conflict handling, quota failure, and lifecycle actions with dirty editors. Rejected saves must not destroy draft or stored content.
7. Stop, expire, cancel/confirm Reset workspace, and start as a different browser. Verify clean handover, retained completion, and the separate semantics of Reset demo progress.
8. Fail the runtime alone, then the server: explicit runtime recovery preserves real Sim state; server restart ends unfinished Attempts, removes abandoned execution/personal work, and retains SQLite completion. No automatic rerun or resend occurs.
9. Verify keyboard access, labels, selected-layout behavior, downloads with inspected file contents, and distinguishable setup, terminal, packet-delivery, and acceptance errors.

### Focused integration and fault checks

Use the existing browser/API boundary where possible. Add targeted tests around the runtime controller and storage operations only where fault injection or real host-resource inspection cannot be expressed reliably through ordinary browser interaction.

- Verify whole-container termination including detached children, controller failure during each transition, interrupted provisioning, cleanup retry, and startup reconciliation. Check actual old-resource absence, not just a success response.
- Verify denied filesystem traversal/symlinks, server/control/progress mounts, non-permitted endpoint/network access, wrong browser/Origin, and expired credentials. Use real containers for isolation claims.
- Exercise memory/process/storage limits, bounded output and backpressure, slow terminal readers, disconnect during output, and consistent recovery. Broader security assessment remains outside this prototype's evidence; document the tested internal deployment conditions.
- Test inactivity rules with a controlled clock at the lifecycle boundary and a real execution environment for cleanup; no 30-minute wall-clock sleep is needed to cover timing edges. Test queued packets and tab/file mutations around revocation explicitly.
- Test receiver readiness and recording checks against actual Sim/Link deliveries, not a fixture's ready flag. Infrastructure test doubles may help isolate failures, but they cannot satisfy the final real-Sim suite.

Existing test prior art is limited to a FastAPI health test and a React title-render test. The retained prototype provides useful experiment scripts and browser observations, not a production regression suite. Build production tests around these public contracts and failure outcomes. Do not require the prototype's internal gateway or exact Docker command structure.

## Out of Scope

Unrestricted public access (the required shared hosting remains team-only); individual Platform accounts or simultaneous Players; collaborative editing or automatic merging; durable personal-work recovery after server failure; Player package installation and unrestricted outbound networking; a general hosted IDE; goal-free Sandbox; real orbital contact windows or radio operation; working Defense comparisons; research logging/export; and importing the entire throwaway prototype as production architecture.

A small contextual map, saved script/recording/log/note downloads, original-byte inspection, and basic resource/error handling remain included. A public-security readiness claim and service-level timing guarantees are not inferred from local prototype measurements.

## Further Notes

### Primary references and prototype evidence

- [Project plan](project-plan.md) and [glossary](../glossary.md).
- [Backend and simulation](backend-and-simulation.md) owns Attempts, script access, and completion. The [operations curriculum](ground-station-training.md#mvp-operations-curriculum) owns the learning tasks, and [Packet format](packet-format-v1.md) owns the packet bytes.
- [Selected visual handoff](</Users/diab/ksc/PSB GS/mockups/HANDOFF.md>) defines the approved layout. Its fixed packet/terminal output remains illustrative.
- [Terminal findings](</Users/diab/ksc/PSB GS/prototypes/kss-terminal-feasibility-2026-09-18/server/terminal-prototype/FINDINGS.md>) and [run/cleanup instructions](</Users/diab/ksc/PSB GS/prototypes/kss-terminal-feasibility-2026-09-18/server/terminal-prototype/README.md>) are the retained primary execution evidence.
- Prototype branch: `prototype/terminal-feasibility-2026-09-18`. Source/evidence commit: `9bd1f30872774ea09fa657b27665bb67e5183cbc`; cleanup-documentation HEAD: `d4407d9f8925b031e8d49da41550caaabed6022d`; base: `7ed045a736ee4d4679df7fae7f76a3af0790fd2f`. Retained locally; not published.

The prototype demonstrated execution, shared files, authenticated byte preservation, reconnect, container replacement, cleanup, and the specific isolation/limit probes in its findings. It used a raw echo fixture, plain HTML, a separate throwaway gateway, interleaved multi-tab terminal input, and last-writer-wins file saves. Actual Sim Service acceptance, operations task verification, completion persistence, and React integration were not demonstrated.

The prototype is infrastructure evidence. Production must satisfy [tab control](#browser-ownership-and-controlling-tab), [file save rules](#editor-notes-and-saves), and the [real Backend/Sim acceptance suite](#primary-acceptance-suite).

The prototype's warm local transition measurements are observations, not estimates or guarantees. Cold deployment and active-workload sizing remain implementation validation. Provide teammates access to the retained evidence before relying on its local references in published tickets.

The acceptance requirements and architecture are agreed. The sections above define the agreed endpoint, controller, storage and terminal defaults. Limit tuning still requires measurements. Team ownership and build order are in the project plan. Local prototype paths are provenance only; no teammate task depends on accessing them. No product implementation or published Jira issue is implied by this document.

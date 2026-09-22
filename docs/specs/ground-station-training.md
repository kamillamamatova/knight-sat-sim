# Ground-station training

## Goal

Teach people to operate a ground station and prepare them to use the PSB station at UCF. A Player should understand what each piece of equipment does and be able to work through a station session: prepare, track, receive, decode, save results, and shut down.

This belongs to the Basic operations track, which now owns the three-Challenge MVP. The agreed activities are Hello, Satellite!, Ready for the Pass, and Catch and Log. The list below describes the broader training path, not a requirement to cover every topic in three Challenges. Offensive and Defensive tracks come later.

## Learner background

Assume basic Python familiarity, following the [project audience](project-plan.md#learner-background). Explain station concepts and the provided tools/API, not Python itself. Starter code may handle connection and setup details; editing or writing a short script is allowed when it supports the operation being practiced. Catch and Log requires adapting a short Python starter to receive telemetry, save a recording and extract useful readings.

## MVP operations curriculum

The following activity goals and task-based completion are agreed. Use a fictional pass brief and a supplied synthetic telemetry source. The Player creates the saved recording. Concrete data and evaluation rules are under Implementation defaults below. These are learning requirements, not completed features or a checked PSB operating procedure.

| Challenge | Player task | Intended practical result |
| --- | --- | --- |
| Hello, Satellite! | Run a supplied terminal command to send PING and recognize the Satellite Sim's reply; no Python writing required. | Distinguish sending a message from receiving confirmation. Detailed packet handling is in the [Hello spec](challenges/01-hello-satellite.md). |
| Ready for the Pass | Read the supplied pass brief, select the satellite, check timing, configure the simulated receiving station and correct one mistaken setting. | A station setup appropriate for the intended reception. |
| Catch and Log | Adapt and run a short Python starter to receive telemetry, save a recording and extract useful readings; complete and save the supplied log template. | A usable recording/result with enough information for another operator to locate and understand it. |

Completion follows successful performance of the required task. Show the accomplished steps and a short Debrief; no Flag generation, copy/paste or Flag-submission step is required for these operations activities. A browser checkbox alone is not evidence that reception, a saved result or a correct setting actually exists. The checks below define the evidence required for each interaction.

Keep the integrated terminal, code editor and shared files in the MVP. Their existing execution, storage and access guarantees remain requirements; Hello uses a supplied terminal command. Ready for the Pass uses graphical station controls to select the satellite and receiving settings, while the terminal remains available for inspecting configuration and running tools. Catch and Log uses the code editor, terminal and shared files, followed by an editor-based session log.

Ready for the Pass and Catch and Log follow the same practice pass: preparation establishes the context for reception and logging. Each exercise can also be repeated from a prepared starting point without requiring the Player to redo the entire sequence. The practice pass starts on demand once preparation is complete. The Player checks the scheduled time in the supplied brief, but does not wait for a real-world clock or miss a pass while reading instructions. Label scenario time separately from real time. Unlock Hello → Ready for the Pass → Catch and Log using shared demo progress. Completed Challenges remain available to repeat. Repeating Catch and Log supplies the prepared station configuration without requiring another completion of Ready for the Pass. The pass defaults below define handoff, pacing and recovery.

### Ready for the Pass: agreed interaction

The Player checks four items against the supplied pass brief: selected satellite, scheduled pass time, receiving frequency and receiving mode. Explain what each means beside its control. The Player enables automatic antenna tracking; manual antenna steering is deferred. Any tracking display is a teaching simulation, not evidence of live pointing or an orbit-physics model.

Include the agreed mistaken setting in the starting setup. A **Check setup** action identifies the incorrect field, explains why it matters and allows correction before the pass starts. Recheck the whole current configuration after an edit; an earlier successful check must not leave a changed, incorrect setup marked ready. Preparation succeeds only with correct settings and automatic tracking enabled. Do not make an unexplained reception failure the teaching mechanism in this first setup exercise.

The pass starts on demand after preparation, following the timing rules above. The scenario below supplies the values. Use shared Select/Input controls, a tracking toggle and an explicit Check setup button.

### Catch and Log: agreed interaction

Provide a short Python starter that handles connection setup, plus concise documentation for the supplied API and data format. The Player adapts it to receive telemetry, save a recording and extract useful readings. They run their saved script in the integrated terminal. Assume basic Python knowledge; do not require one prescribed coding solution or turn the exercise into a Python tutorial.

Judge the operational result: a usable saved recording and correct interpretation of its contents. Running a script or displaying a success message alone is insufficient. Supply a decoder that turns packet bytes into readable fields. The Player writes the receiving/saving code and selects the readings needed for the log; building a packet decoder is not an MVP task. Document the format for reference. Use the recording format below.

Provide a short session-log template in the editor. Prefill the satellite, pass time and receiving settings from the practice session. The Player adds the recording filename, two observed readings (battery level and operating mode), and a brief outcome note, then saves the log. Check the referenced recording, factual entries and saved log; do not grade prose style or prescribe the wording of the outcome note. The log format below defines the fields and which observation to report.

### Implementation defaults: practice pass and evidence

These agreed implementation and content defaults cover the MVP only; actual PSB procedures remain separate.

**Scenario.** Version the server-owned fixture as `knightsat-training-v1`. The brief names `KNIGHTSAT`, scheduled time `2026-09-21T18:00:00Z`, receive frequency `145825000` Hz (display 145.825 MHz), receiving mode `FM`, and automatic tracking. These are fictional training values, not a real satellite prediction or operating instruction. Frequency/mode/tracking are configuration checks; no RF demodulation, Doppler or orbit physics is simulated. Ground-antenna tracking is separate from the spacecraft's `antenna_deployed` telemetry field.

**Preparation.** `StationSetup` is `{satellite_id, pass_start_utc, receive_frequency_hz, receive_mode, tracking_enabled}`: strings for satellite/time/mode, positive integer Hz and a boolean for tracking. `/attempt/pass-brief.json` is `{scenario_id, expected_setup: StationSetup}` with the values above; the UI renders it as readable instructions. Start Ready for the Pass with only frequency wrong (`145800000` Hz); tracking starts disabled as an explicit action still to perform. Other fields match the brief. Allow satellite choices KNIGHTSAT / PRACTICE-B, an editable UTC time/frequency, and FM / AM modes. Check setup returns one result per field, including tracking, with corrective explanations. Every setup edit invalidates the current check. A fully correct check saves completion; saved course progress remains completed even if the Player subsequently changes settings, but the current setup must be revalidated before continuing to reception.

**Handoff and repeat.** Explicitly switching from Ready for the Pass to Catch and Log copies a verified setup and scenario version into the new Attempt before ending the old context. If that setup has changed, refuse the switch with field feedback and leave the current Attempt intact. When Catch and Log is already unlocked and started from any other context, or restarted, the server supplies the correct prepared setup. This permits independent repeats without requiring a surviving previous Attempt. No extra lesson-progress table or browser-stored readiness is needed. Restarting Ready resets its mistaken setting; switching/restarting retains authored files under the Workspace rules.

**Reception.** Catch starts with pass status `not_started` and a fresh `pass_run_id`. The Player saves/runs their script, which connects as the single receiver; the UI shows Receiver ready. Then they click Start pass. The server requires correct setup and an attached receiver, atomically marks the pass `running`, freezes setup and emits 12 telemetry packets at one-second intervals, first packet immediately. Display scenario timestamps separately from elapsed wall time. End after packet index 11 with status `finished`; duplicate Start requests return the current pass without starting it again. A disconnected receiver, interrupted source or failed runtime during a running pass marks it `interrupted`; never silently fill a partial recording. Browser disconnection alone leaves a connected script running. Repeat reception requires explicit Attempt Restart, a new pass ID and a newly saved recording; no automatic playback resume or rerun.

**Source fixture.** The Sim Service generates the 12 packets with the production Packet v1 encoder and delivers them through the Software Link. Each is kind `0x10`, downlink sequence/index 0–11, with NOMINAL mode, antenna deployed, deployment count 1, heater off, battery 80%, uptime 600–611, commands received 0 and last sequence `0xffff`. Initial deployment is a scenario state, not a sent setup command. Scenario timestamps are scheduled start plus index seconds. These values model a short, stable teaching pass, not battery physics. Ready uses Hello’s initial spacecraft state without advancing uptime or emitting packets. Catch starts with the first fixture state and updates the displayed spacecraft state on each delivered sample; its uptime is scenario data, not a wall-clock timer. There is no background telemetry or uplink commanding during Ready/Catch; Hello retains its normal live feed. Publish packets in Attempt history and forward exactly those delivered bytes to the receiver. The server retains the emitted packet manifest for evidence checks. No completed Player recording is pre-provisioned.

**Recording.** Use UTF-8 JSON Lines, one record per received packet: `{pass_run_id, index, scenario_time, packet_hex}`. `packet_hex` is lowercase hex without spaces, containing the complete original packet including CRC. The helper yields this record; the Player chooses a filename and writes records. A usable MVP recording contains all 12 records in index order, with no duplicates or extra records, matching the current pass ID, timestamps and actual Link-delivered bytes. Limit grading input to 1 MiB; report missing/incorrect records clearly. This is a packet recording, not audio or IQ data.

**Log.** Use editable JSON so the supplied decoder and Backend can check named fields without parsing prose. The managed template contains `scenario_id`, `pass_run_id`, `satellite_id`, `pass_start_utc`, `receive_frequency_hz`, `receive_mode`, and `tracking_enabled`, prefilled from this Attempt. Blank result fields are `recording_path`, `battery_pct`, `mode`, and `outcome_note`. The Player copies it to their authored Catch folder, completes it and explicitly saves it. `recording_path` is relative to that folder. The two readings must match the last recording entry (index 11): battery 80 and mode NOMINAL for this fixture. Require a nonblank outcome note of at most 1,000 characters, without grading its wording. Compare prefilled context against server-owned context; edited metadata does not redefine the expected answer.

**Check work.** The explicit Check work action selects a saved recording and saved log. Backend checks the current completed pass, both files, their context and factual readings, then commits completion. It does not execute submitted Python or grade source style; evidence establishes the operational result, not proof of a particular language or coding method. Missing/partial/stale files, wrong readings or an interrupted pass produce individual corrective checks, never success. Preserve scripts, recordings and logs for correction/download; saving files alone does not complete the activity. The [Backend contract](backend-and-simulation.md#implementation-defaults-browser-contract) owns request/version/concurrency rules; the [helper contract](terminal-workspace.md#implementation-defaults-terminal-and-helper-contracts) owns imports and messages.

### MVP acceptance checks

- Ready starts with the specified frequency mistake and tracking disabled. Check setup explains both; changing any field invalidates readiness. Only a correct current setup saves completion and can hand off to Catch.
- Catch cannot start while unprepared or without a receiver. Opening a page never starts a pass. A successful pass delivers the fixture through the real Sim/Link and a Player script saves it.
- Check work accepts a full current recording and correct saved log, irrespective of implementation style. Reject missing, altered, duplicate, stale-pass or incomplete records; wrong facts/context; empty outcome note; unsaved-only changes; and unsafe file paths.
- Refresh/browser reconnect preserves an active receiver/pass, history and saved work without resending or restarting. A receiver loss marks the pass interrupted. Explicit Restart creates a fresh pass and keeps authored work; old recordings cannot solve it.
- Repeated completion is unique; failed SQLite persistence shows that the task succeeded but saving failed and offers retry without rerunning the pass. Unlocks depend on committed progress.
- Debriefs explain why checking settings prevents avoidable reception failures, and why a recording needs both readable observations and session context. Neither completion asserts real RF decoding or qualification to use PSB alone.

## Later PSB curriculum

The sections below describe the broader training roadmap. They are not MVP prerequisites or checked operating procedures.

## What people should learn

| Topic | What the Player should be able to do |
| --- | --- |
| Know the station | Identify the radios, antennas, pointing controls, computers, and connections. Explain what each does. |
| Prepare for a pass | Choose a suitable pass and explain the required equipment and settings. Check the time, station location, and tracking information. |
| Set up at PSB | Follow the checked station procedure, use the right computer and software, and recognize the actual equipment and controls. |
| Track and receive | Follow the pass, observe the signal, and make a recording. Recognize when tracking or reception is not working. |
| Decode | Use the appropriate software on a supported recording, inspect the decoded result, and distinguish a successful decode from noise or an error. |
| Save the result | Keep the recording and decoded output with enough notes for someone else to understand what was done. |
| Troubleshoot | Work through common problems such as the wrong input, mode, frequency, pointing, file, or decoder settings. |
| Finish the session | Save work, follow the checked shutdown procedure, and leave useful notes for the next person. |

The training should connect these steps into a full session. Players should get feedback on their choices and have a chance to correct mistakes.

## Make it specific to PSB

Use photos of the station, its equipment names, and examples from the software installed there. Show how the simulated controls relate to the real controls. Use recordings that the team has checked, with known results for the decoding lessons.

Existing station notes list IC-9700 and IC-R8600 radios, antenna pointing equipment, and two computers labelled PC1-RF and PC2-Pointing. The computer notes say their software and connections still need checking. Confirm these details at the station before turning them into instructions.

The station guide is a work in progress. Writing and checking it is part of this work. Each lesson should identify the procedure it uses and when someone last checked that procedure at PSB. Unchecked steps stay marked as drafts.

## Later PSB build plan

1. **Document the station.** Diab and someone familiar with PSB check the equipment, software, connections, and current operating steps. Add photos and record what is still unknown.
2. **Write and check a full session.** Work through setup, a receive pass, decoding, saving, and shutdown at PSB. Choose a supported signal and decoder. Keep a sample recording and the expected result for practice.
3. **Build the lessons.** Sydney builds the lesson pages. Lily builds the practice screens. Kamilla saves lesson progress, with database help from Sydney. Denzel supplies practice data and any processing needed. Diab writes the content and checks the simulated behavior.
4. **Add mistakes to practice.** Include a few common setup, reception, and decoding problems. Explain what happened and let the Player try again.
5. **Try it with a newcomer.** Have someone follow the lessons, then walk through the real station with an experienced operator. Record where they need help and improve the lesson and guide.

Start with a receive-only session. Exact lesson order, supported signals and software, and delivery dates still need to be agreed. Any later transmitting exercises need their own checked station procedure. The software lessons can use simulated or recorded data without connecting to live hardware.

## When the PSB curriculum is ready

- [ ] Lessons cover the full session, including preparation, reception, decoding, saving, troubleshooting, and shutdown.
- [ ] PSB equipment names, photos, software, and procedures match a checked station setup.
- [ ] The Player can complete a practice session and explain the important choices.
- [ ] The decoding lesson uses a checked recording and recognizes the expected result.
- [ ] Common mistakes produce useful feedback and allow another try.
- [ ] Lesson progress is saved. Completing a software lesson means practice is complete; it does not grant access to the station or certify someone to operate alone.
- [ ] A newcomer has tried the lessons and a station walkthrough, and the team has addressed the problems found.

## Work still to specify

The broader PSB learning goals remain planned. These later choices do not block the three simulated MVP Challenges:

- The supported signal, recording format, decoder, and expected decoded result for the first full-session lesson.
- Which controls need simulation and which steps can use checked recordings or guided choices. No realistic orbit or radio model is required by the MVP.
- The lesson API, completion checks, progress records, and how a lesson resumes. Reuse task-based completion where appropriate; specify extensions beyond the three MVP Challenges before building them.
- How lesson activity uses the single-player slot and retained files, if it needs the execution environment. The existing Challenge switching contract does not yet define switching into a station lesson.
- Lesson dates, the reviewed PSB procedures, and access to checked training material for teammates.

Diab leads station content and procedure checks. Sydney and Lily build the lesson pages and practice screens. Kamilla leads lesson progress and APIs with Sydney helping on database work; Denzel handles practice files and processing. Their [main responsibilities](project-plan.md#who-owns-what) also cover the operations MVP.

Catch and Log decodes fields in simulated telemetry packets. It does not satisfy the later station lesson about receiving and decoding an RF signal. A cabled RF security Challenge is also separate from the full PSB operating session.

## Starting material

These local wiki pages are work in progress. The receive-pass draft explicitly says recording, file locations, antenna resting position, and shutdown still need checking.

- [PSB station overview](</Users/diab/ksc/PSB GS/kscwiki/projects/ground-stations/physics-ground-station/README.md>)
- [Receive-pass draft](</Users/diab/ksc/PSB GS/kscwiki/projects/ground-stations/physics-ground-station/receive-pass.md>)
- [Station computers](</Users/diab/ksc/PSB GS/kscwiki/projects/ground-stations/physics-ground-station/station-computers.md>)
- [Hardware and manuals](</Users/diab/ksc/PSB GS/kscwiki/projects/ground-stations/physics-ground-station/hardware.md>)

For the team’s submission and lessons, make the checked guide and approved training material accessible to everyone; these local paths are research references.

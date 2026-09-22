# Frontend

This spec owns the shared UI foundation and browser conventions for the operations MVP. Start with the [project plan](project-plan.md). The foundation, behavior and implementation defaults below are agreed requirements. The current UI remains a scaffold.

## Shared UI foundation

| Area | Use |
| --- | --- |
| Application | React, TypeScript, and Vite |
| Components | shadcn/ui with Base UI primitives |
| Styling | Tailwind with centrally defined colors, typography, and spacing |
| Icons | Lucide |
| Navigation | React Router, with shareable main-page URLs and a persistent workstation shell |
| Shared Attempt state | React Context and `useReducer`, accessed through shared hooks |
| HTTP API | Types generated from FastAPI OpenAPI with `openapi-typescript`; one shared client using browser `fetch` |
| Editor and terminal | CodeMirror 6 and xterm.js, following the [Workspace spec](terminal-workspace.md#editor-notes-and-saves) |

Configure the shared components once to match the approved dark workstation design. Keep reusable controls in `ui/src/components/ui/`; feature components import them rather than creating separate button, dialog, tab, form, or menu implementations. Use one shared set of style tokens for colors, typography, spacing, and control states.

Provide a small developer component example page showing the supported controls and their intended usage. It is a reference for teammates, not part of the Player's Challenge workflow. Package versions belong in the package manifest and lockfile.

## Layout and screen support

The supported MVP browser is current stable desktop Google Chrome. Run the required browser acceptance checks in Chrome and record the tested version with the results. Other browsers are outside the MVP support commitment; do not deliberately block them solely by browser name.

Preserve the approved full-height workstation: left navigation, central work area with a terminal dock, and right supporting panels. Use the existing dark visual direction, readable labels, visible keyboard focus, and text alongside status colors. The [Workspace spec](terminal-workspace.md#application-and-responsibility-boundaries) owns workspace interaction requirements.

The first demo supports desktop use, targeting widths of 1280px and above. Below the supported width, show a clear message explaining that the demo requires a wider desktop window. Phone/tablet layouts and collapsing the workstation into a mobile interface are deferred. Keep keyboard access and readable text at browser zoom; do not shrink text to force the layout to fit.

## Operations interaction choices

Hello, Satellite! introduces a supplied terminal command. Ready for the Pass uses graphical controls for station preparation; keep the terminal/editor/files available alongside them. The pass begins on demand after preparation, with scenario time clearly distinguished from wall-clock time. Preparation includes satellite, pass time, receiving frequency, receiving mode, an automatic-tracking control and Check setup feedback. Use the [agreed setup behavior](ground-station-training.md#ready-for-the-pass-agreed-interaction); Catch and Log uses the editor and terminal to adapt/run a Python starter, then an editor-based log template with prefilled session details and Player-entered results. Use the [scenario/file defaults](ground-station-training.md#implementation-defaults-practice-pass-and-evidence) for exact context, readings and evidence. Catch exposes Receiver ready, Start pass, pass progress, saved-file selection and Check work; completion feedback distinguishes verified work from successfully saved progress. Downloads use saved file contents. Do not add a PING button, packet-building form or Flag entry box.

## Navigation and active Attempt

Hosted entry uses [Cloudflare Access email codes](backend-and-simulation.md#hosted-team-access) for approved addresses. Do not add custom signup, password, or profile screens for the MVP. Individual Platform accounts are later work; completion remains shared.

Give Challenge pages shareable URLs and working browser Back/Forward navigation. Keep the workstation shell mounted during in-app navigation; individual workspace panels remain ordinary tabs.

The viewed Challenge and the Challenge running in the active Attempt are separate. Opening Hello's page while Catch and Log runs shows Hello's instructions without replacing the reception Attempt or stopping its programs. Clearly label the Challenge that owns the terminal and active Attempt tools, including telemetry, history, station controls, Start pass and Check work, so browsing another page cannot mislabel live data or target the wrong Attempt.

Starting or switching the active Attempt requires an explicit Start or Switch action. A URL change alone never starts, stops, or replaces an Attempt. Explicit switching follows the [Workspace lifecycle and save/discard rules](terminal-workspace.md#editor-notes-and-saves), and the server still enforces Challenge availability. In-app browsing must preserve unsaved drafts; navigation that would discard a draft follows the existing warning rules.

## Shared state and connections

Use React Context and `useReducer` for shared Attempt state. No additional state-management library is required for the first demo. Mount one Attempt provider in the persistent workstation shell so in-app navigation does not recreate it.

Keep state updates in a pure reducer with named, typed actions. Expose state and dispatch through separate contexts and shared hooks, giving feature components one consistent way to read state and request updates. The Backend remains authoritative for ownership, simulation state, command acceptance, and completion; browser state reflects its responses and events.

One shared connection module owns the Attempt HTTP/WebSocket integration and feeds updates into the reducer. Individual panels do not open their own Attempt event connections. The terminal gateway connection is separate and follows the Workspace contract. Use an authoritative snapshot on connection/reconnect, followed by numbered updates, under the [Backend synchronization rules](backend-and-simulation.md#live-updates-and-resynchronization). Mark disconnected readings stale, ignore obsolete updates, and resynchronize on gaps without automatically resending commands.

Keep ordinary form input and temporary panel state local. CodeMirror owns editor documents and unsaved drafts; xterm.js owns terminal output. Do not route every keystroke or terminal byte through shared Attempt state. Preserve drafts during in-app navigation as required above.

## HTTP API types and client

Generate the frontend's HTTP request and response types from the FastAPI OpenAPI schema using `openapi-typescript` as a development dependency. Backend API definitions own these shapes; frontend features import generated types instead of maintaining separate copies. Do not hand-edit generated types. Regenerate them when API definitions change.

Use one shared HTTP client built on the browser's `fetch`. Feature code uses that client so ownership-cookie handling, response handling, and errors follow one convention. Follow the existing ownership and no-automatic-command-retry rules in the [Backend spec](backend-and-simulation.md).

Use the [shared Backend error envelope](backend-and-simulation.md#browser-api-errors): a stable code, readable message, and optional field errors. Show errors beside the affected control or panel and preserve entered values and drafts. Failures requiring action stay visible until resolved or dismissed; a disappearing notification is not sufficient.

The shared client normalizes network failures into the same presentation without pretending they are server responses. If an operation's outcome is unknown, say so and reconcile with authoritative state where possible; do not claim rejection, completion, or save success without evidence. Never automatically repeat an uncertain command or mutation.

Generated TypeScript types provide development-time checks; they do not validate incoming data at runtime. WebSocket events need an explicit shared contract and are not automatically covered by the HTTP OpenAPI schema. Use the [browser wire contract](backend-and-simulation.md#implementation-defaults-browser-contract) for event payloads and their generated types.

## Acceptance checks

- Feature screens reuse the shared controls and styling; the developer example page demonstrates their intended usage.
- The workstation follows the selected layout at supported desktop widths, with readable labels, visible focus, and keyboard-operable controls.
- Narrower viewports show an understandable desktop-width requirement.
- Python editing and notes meet the [editor requirements](terminal-workspace.md#editor-notes-and-saves).
- Opening another Challenge page or using Back/Forward leaves the active Attempt and its execution intact. Active tools remain labelled with their owning Challenge; only explicit Start/Switch actions change the active Challenge.
- Panels share the same authoritative Attempt view through the provider. Navigation does not create duplicate Attempt event connections; terminal output and editor keystrokes do not update the shared Attempt reducer.
- HTTP types regenerate from the Backend OpenAPI schema and the UI type-checks against them; features use the shared `fetch` client.
- Field errors appear beside the relevant inputs; actionable failures remain visible. A rejected save preserves the draft, and a lost response is reported as an uncertain outcome without automatically repeating the operation.

## Implementation defaults: screens and styling

These defaults carry the approved visual direction into a self-contained build reference; teammates do not need the local mockup to implement the shell.

- Routes: `/` redirects to `/challenges/hello-satellite`; `/challenges/:challenge_id` displays a Briefing without starting an Attempt. MVP IDs are `hello-satellite`, `ready-for-the-pass` and `catch-and-log`. Unlock them in that order using server-owned shared demo progress; completed activities remain repeatable. Offensive and Defensive tracks are labelled Coming later with no playable routes. Unknown IDs show Not found. `/dev/components` is available only in development.
- Use a full-height CSS grid: left rail 204px, flexible center with minimum width 640px, right inspector 296px. At 1500px and wider, use 224px and 320px side columns. Regions scroll independently; the center dock starts at 276px high. Resizable splitters and saved layout preferences are deferred.
- Left: Challenge navigation and files. Center: contextual map and Challenge tools (Hello terminal instructions or the current operations exercise), then Terminal / Editor / Notes tabs. Right: Briefing, Satellite Sim state, task-completion feedback and Debrief. Bottom: connection and command-delivery status. Display the active Challenge beside every live tool when it differs from the viewed page.
- Narrow windows and browser zoom show a nonblocking width notice above the workstation; preserve the session, drafts, keyboard access and horizontal scrolling. Never stop execution or hide the only Stop control because the viewport shrank.
- Start with the tokens below, system sans-serif for prose and system monospace for bytes/code. Body text is 14–16px; secondary labels at least 12px. Use 4px spacing increments and 6px control corners. Verify contrast and focus in the rendered UI; the mockup's tiny labels are not requirements.

| Token | Initial value |
| --- | --- |
| Background / panel / input | `#10161c` / `#141c24` / `#141e27` |
| Text / secondary text / border | `#e0e7ed` / `#a0afbc` / `#2a3540` |
| Accent / text on accent | `#bce387` / `#15200c` |

Use shared Button, Input, Label, Tabs, Dialog, AlertDialog, Select, Tooltip and Alert controls. Native tables render packet fields/history; CodeMirror and xterm own their specialist surfaces. Stop, Reset workspace and Reset demo progress use explicit destructive-action dialogs. Prefer inline status to toasts, CSS grid to a layout package, and native form state to an additional form library. Dark appearance only for MVP.

Implementation references: [shadcn/ui for Vite](https://ui.shadcn.com/docs/installation/vite) and [CodeMirror](https://codemirror.net/docs/guide/).

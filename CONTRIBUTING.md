# Contributing

Work is tracked in the **Knight Sat Sim (KSAT)** Jira project. Use the [shared board](https://seniordesign-g20.atlassian.net/jira/software/projects/KSAT/boards/1) and [backlog](https://seniordesign-g20.atlassian.net/jira/software/projects/KSAT/boards/1/backlog). Pick a ready ticket, branch, open a pull request, pass CI, request review, then merge.

The official repository is [kamillamamatova/knight-sat-sim](https://github.com/kamillamamatova/knight-sat-sim). Do not change repository visibility as part of implementation work.

## Branch and pull request

- Use the exact key on the assigned Jira issue; the repository name does not determine the Jira project key.
- Branch name includes that key: `codex/KSAT-10-packet-crc`.
- Pull request title starts with the same key: `KSAT-10 Add CRC-16 helper`.
- Fill in [`.github/PULL_REQUEST_TEMPLATE.md`](.github/PULL_REQUEST_TEMPLATE.md).

## Planning and board

Use one shared backlog with continuous flow. The four epics group the shared foundation, terminal/editor/Workspace, Basic operations Challenges, and team-only hosted demo. Area labels help find work: `ui`, `server`, `sim`, `hosting`, and `docs`. Apply more than one when the ticket spans areas; an area is not a separate team or an ownership restriction.

The workflow is **Backlog → Ready → In progress → Review → Done**. Backlog holds work not yet selected or ready. Ready means the scope and acceptance checks are clear, the owner has a supported starting point, and true prerequisites are met. Flag blocked work, state what it needs, and link the actual prerequisite. Player unlock order is not automatically implementation order; use isolated test fixtures where the specs allow them.

Start with one active implementation ticket per person. Help finish reviews and unblock teammates before starting more work. One person owns the outcome and can ask others to help. Request approval from one teammate other than the author when the pull request is ready; do not assign a fixed reviewer in the ticket description.

At the start of each week, select a small demonstrable goal based on available time. At the end, show working behavior, discuss blockers, and choose one process improvement. Report blockers as they occur. Fixed sprints and story-point estimates are not required for this workflow; retain course-required reporting separately.

## Ticket format

Use a short title that names the deliverable, such as **UI: Build workstation navigation** or **SERVER: Save shared Challenge progress**. Use the primary area when useful; use **FEATURE:** for a complete behavior spanning areas, with all relevant area labels. Jira supplies the `KSAT` identifier separately. Name the developer's deliverable rather than giving the Player an instruction.

Each build ticket contains:

1. **Result:** the observable capability or recorded decision.
2. **Starting point:** what the prerequisites supply, the interfaces to extend, and the development/test setup.
3. **Scope and handoff:** behavior and essential contract details, with adjacent work identified by Jira key and descriptive title.
4. **Acceptance checks:** setup → action → expected result, including relevant failure/recovery cases and evidence to record.
5. **References:** exact owning spec sections and these shared contribution rules.

Keep assignee, status, epic and blocking relationships in Jira fields. Descriptions explain why a dependency or handoff exists. Aim for work that can be completed and reviewed in a few work sessions. Split independent behaviors while retaining each behavior's required authorization, cleanup, error handling and tests. Do not defer these protections to a generic hardening ticket.

Every ticket has a finite completion boundary. Name the ticket that owns later integration checks instead of writing “when available.” Infrastructure tickets verify the capabilities available at their boundary; KSAT-26 owns final hosted feature acceptance. Summarize useful contract values without duplicating whole specifications.

## How to run

See the root [`README.md`](README.md). Use `docker compose up --build` unless you have a reason to run `server/` and `ui/` separately. Keep a single Python server worker.

## How to test

- New behaviour needs tests in the same change: `pytest` under `server/`, Vitest under `ui/`.
- Lint: `ruff check` for Python, `npm run lint` for TypeScript.
- Do not push a change that fails those commands locally if you can run them.

## Definition of done

A change is done when all of the following are true:

1. The Jira issue's acceptance checks are met, and the relevant `docs/specs/` rules are followed.
2. New behaviour has tests.
3. CI is green on the pull request.
4. One teammate (not the author) has reviewed and approved.
5. The Jira key is in the branch name and the pull request title.

Then merge to `main` and move the Jira issue.

For external setup without a code change, meet the ticket's acceptance checks, record the decision or evidence in the agreed team location, and obtain verification from another teammate when ready. Code tests, CI and a pull request are required only when that setup also changes repository code or configuration. Do not put account secrets in tickets or the repository.

## Updating the shared website

Merging code does not automatically update the hosted demo. The planned **Update website** workflow is manually triggered and refuses to run while someone owns a Player session. See the [deployment rules](docs/specs/backend-and-simulation.md#updating-the-hosted-demo). This workflow has not been implemented yet.

## Specs and language

Start with the [project plan](docs/specs/project-plan.md) and use the terms in [`docs/glossary.md`](docs/glossary.md). Record decisions directly in the owning spec; do not add ADRs or duplicate rationale. Implementation defaults in the specs are agreed; build tickets should use them. Research and local prototype links are supporting evidence, not extra acceptance requirements. Do not invent synonyms the glossary tells you to avoid. If a spec and a ticket disagree, stop and ask; do not silently pick.

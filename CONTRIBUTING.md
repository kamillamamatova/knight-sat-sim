# Contributing

Work is tracked in **Jira** Pick an issue, branch, pull request, CI, review, merge.

The official repository is [kamillamamatova/knight-sat-sim](https://github.com/kamillamamatova/knight-sat-sim). Do not change repository visibility as part of implementation work.

## Branch and pull request

- Use the exact key on the assigned Jira issue; the repository name does not determine the Jira project key.
- Branch name includes that key: `codex/PROJECT-12-packet-crc` (replace the placeholder).
- Pull request title starts with the same key: `PROJECT-12 Add CRC-16 helper`.
- Fill in [`.github/PULL_REQUEST_TEMPLATE.md`](.github/PULL_REQUEST_TEMPLATE.md).

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

## Updating the shared website

Merging code does not automatically update the hosted demo. The planned **Update website** workflow is manually triggered and refuses to run while someone owns a Player session. See the [deployment rules](docs/specs/backend-and-simulation.md#updating-the-hosted-demo). This workflow has not been implemented yet.

## Specs and language

Start with the [project plan](docs/specs/project-plan.md) and use the terms in [`docs/glossary.md`](docs/glossary.md). Record decisions directly in the owning spec; do not add ADRs or duplicate rationale. Implementation defaults in the specs are agreed; build tickets should use them. Research and local prototype links are supporting evidence, not extra acceptance requirements. Do not invent synonyms the glossary tells you to avoid. If a spec and a ticket disagree, stop and ask; do not silently pick.

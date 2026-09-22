# Knight Sat Sim

A **Platform** for practicing satellite cybersecurity and learning ground-station operation, including the station at UCF’s Physical Sciences Building (PSB). **Players** practice with simulated systems and learn the equipment and workflow used at PSB.

This repository is **Knight Sat Sim (KSS)**, the UCF CS Senior Design implementation of that Platform. The product has three tracks: **Basic operations**, **Offensive**, and **Defensive**. The first version to build (MVP) focuses on **three Basic operations Challenges**, **Hello, Satellite!**, **Ready for the Pass**, and **Catch and Log**. Completion is based on performing the tasks, without submitting Flags. The specs define the agreed learning flow and implementation defaults. Offensive and Defensive Challenges come later. Practice remains browser-based, without live station-hardware integration. See the [ground-station training plan](docs/specs/ground-station-training.md).

Start with the [project plan](docs/specs/project-plan.md) for scope, responsibilities, progression, and completion checks. Its [build specifications](docs/specs/project-plan.md#build-specifications) table links to the implementation and learning documents. Each spec owns its decisions; research provides supporting sources, not additional MVP requirements.

## Repository layout

| Path | Contents |
| --- | --- |
| `server/sim/` | Sim Service, packet handling, Ground Sim, and Software Link |
| `server/api/` | FastAPI Web Backend |
| `ui/` | React and TypeScript frontend |
| `docs/specs/` | Current scope, architecture, behavior, and acceptance checks |
| `docs/research/` | Supporting sources and technical findings |

The repository currently contains a scaffold. Running it starts the frontend and backend health endpoint; it does not provide the planned Challenges, terminal, database behavior, or station lessons yet.

## Run locally

You need Docker. From the repository root:

```bash
docker compose up --build
```

- Browser UI: http://localhost:5173
- Web Backend health check: http://localhost:8000/health

Compose starts the React development server and **one** Python process that will contain FastAPI and the Sim Service. The `sqlite-data` volume is reserved for the planned SQLite progress database. Do not run extra server workers: the MVP Attempt lives in memory in that single process.

Without Docker:

- Server: Python 3.12+, from `server/`: `uv sync` then `uv run uvicorn api.main:app --reload --host 0.0.0.0 --port 8000` (still one worker).
- UI: Node 22+, from `ui/`: `npm install` then `npm run dev`.

## Tests and lint

```bash
# server
cd server && uv sync --extra dev && uv run ruff check . && uv run pytest

# ui
cd ui && npm ci && npm run lint && npm test
```

Pull requests run the same checks in GitHub Actions.

## How we work

Jira holds build tickets. Open a branch named with the Jira key, open a pull request whose title starts with that key, wait for CI, and get one teammate review before merge. Details: [`CONTRIBUTING.md`](CONTRIBUTING.md).

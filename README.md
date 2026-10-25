# PipelineDoctor

AI CI/CD failure explainer. Paste a failed pipeline log (or connect a GitHub repo) and get the root cause,
the exact log lines that prove it, and a fix you can apply. No claim is shown without evidence.

## What it does

| Module | What you get |
|---|---|
| **Failure Explainer** | Category, root cause, confidence, evidence lines, fix steps, suggested diff, "check next" list |
| **Flaky Test Detector** | Tests that pass and fail on the same commit, or flip between runs, with minutes wasted |
| **Deploy Risk Scorer** | 0-100 pull request score, broken down by change size, files, CI config, dependencies, missing tests, recent failures |
| **Reliability Coach** | Weekly report: recurring causes ranked by time lost, flaky tests, failure rate |

Also included: secret and PII redaction before analysis, role-based access (developer, lead, admin),
an audit log, a sandbox with sample logs, a GitHub webhook that comments on pull requests, and a REST API.

## How a diagnosis works

1. **Redact.** Tokens, keys, passwords, emails, IPs and private keys are replaced. Line numbers don't change.
2. **Rules.** About 18 deterministic parsers recognise common failures (missing module, npm conflict, permissions,
   out of memory, timeout, Docker build, missing secret, and more). Each match records the lines it matched.
3. **AI (optional).** If no rule is confident (below 80%), the log excerpt goes to an LLM with a strict JSON schema.
   The log is treated as untrusted data. Evidence line numbers from the model are checked against the real log,
   the evidence text is always copied from the log, and an answer with no valid evidence is capped at 40% confidence.
4. **Merge.** Rules and AI are combined. Anything under 50% is labelled *uncertain* instead of presented as fact.

## Quick start

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt
cp .env.example .env                                    # all values are optional
uvicorn app.main:app --app-dir backend --reload
```

Open http://localhost:8000, pick a sample log, and press **Diagnose**. API docs are at `/docs`.

With Docker: `docker compose up --build`.

## Configuration (`.env`)

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Enables AI analysis for unclear logs. Without it, rules only. |
| `LLM_MODEL` | Model name for the AI step (default `claude-sonnet-4-6`). |
| `API_KEYS` | `key:role,key:role`. Roles: `developer`, `lead`, `admin`. Empty means open (local dev only). |
| `GITHUB_TOKEN` | Token used to download run logs and post PR comments. |
| `GITHUB_WEBHOOK_SECRET` | Required to enable `/webhooks/github`. |
| `DB_PATH` | SQLite file (default `data/pipelinedoctor.db`). |
| `MAX_LOG_BYTES` | Largest log accepted (default 2,000,000). |

## API

| Method and path | Role | Description |
|---|---|---|
| `POST /api/diagnose` | developer | Body: `log`, optional `repo`, `commit_sha`, `branch`, `duration_s`, `use_llm` |
| `GET /api/diagnoses`, `/api/diagnoses/{id}` | developer | History and detail |
| `POST /api/risk` | developer | Body: `files`, `additions`, `deletions`, `repo` |
| `GET /api/flaky?repo=` | developer | Flaky test list |
| `GET /api/report?repo=&days=` | lead | Reliability report |
| `GET /api/admin/stats` | admin | Repos, runs, diagnoses, average confidence, top causes |
| `GET /api/samples` | none | Sandbox logs |
| `POST /webhooks/github` | signature | `workflow_run` events |

Send the key as `X-API-Key`. Wrong role returns `403 Access denied`.

```bash
curl -s localhost:8000/api/diagnose -H 'Content-Type: application/json' \
  -d '{"log": "ModuleNotFoundError: No module named '"'"'pandas'"'"'", "use_llm": false}'
```

## Connect GitHub

1. Set `GITHUB_TOKEN` (needs Actions read and Pull requests write on the repo) and `GITHUB_WEBHOOK_SECRET`.
2. In the repository, go to Settings > Webhooks > Add webhook.
3. Payload URL: `https://<your-host>/webhooks/github`, content type `application/json`, same secret,
   event: **Workflow runs**.
4. When a run fails, PipelineDoctor downloads the logs, diagnoses them, stores the result, and comments on the pull request.

## Tests

```bash
pytest -q
ruff check .
```

## Project layout

```
backend/app/    redact, parsers, llm, diagnose, flaky, risk, report, github, db, main
backend/tests/  unit and API tests
frontend/       single-page dashboard served by FastAPI (no build step)
.github/        CI: lint, tests, Docker build
```

## Known limits

- Flaky detection learns from test results found in analysed logs. It gets better as more failed runs are recorded.
  Test names are read from pytest-style output (`path::test PASSED/FAILED`).
- Rules cover common patterns; unusual failures rely on the AI step or are marked uncertain.
- SQLite is the default store. `db.py` is the only module that touches storage, so moving to Postgres or Supabase
  means rewriting that one file.
- No rate limiting yet. Put it behind a proxy before exposing it publicly.

## Roadmap

GitLab CI and Jenkins support, auto-opened fix pull requests with human approval, chat over failure history,
team-level cost analytics.

## License

MIT

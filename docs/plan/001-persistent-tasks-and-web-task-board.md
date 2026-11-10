---
plan_id: 001-persistent-tasks-and-web-task-board
spec: 001-persistent-tasks-and-web-task-board
feature: Tasks persist in an in-cluster Postgres database and are listed and added on the web board
gitops_app: ika100-demo/taskboard10-gitops
status: draft
repos:
  - id: taskboard10-gitops
    shape: gitops-app
    summary: Declare the system's Postgres database, give it to the task API and point the web board at the API
    acs: [AC-001.1, AC-001.2, AC-001.3]
    gitops:
      - {resource: postgres}
      - {uses: postgres, component: taskboard10-api}
      - {env: {TASKS_API_URL: 'http://taskboard10-api'}, component: taskboard10-web}
    depends_on: []
    done: false
  - id: taskboard10-api
    shape: service-python
    summary: Store tasks in Postgres so they survive restarts and every instance sees the same list
    acs: [AC-001.1, AC-001.2, AC-001.7, AC-001.8]
    criteria:
      - "Given tasks \"A\" and then \"B\" were created through `POST /tasks`, when the service is stopped and started again (or a new app instance is started) against the same database, then `GET /tasks` returns 200 with \"A\" before \"B\", each with the same `id`, `title` and `due_date` the create responses returned (system AC-001.1)"
      - "Given a database with no `tasks` table, when the service starts and a client sends `GET /tasks`, then the service has created its table itself and answers 200 with `[]`; the `POST /tasks` 201 body, the title trimming and the 422 answers of spec 001 are unchanged against the database (system AC-001.1)"
      - "Given two instances of the service run against the same database and a task was just created through `POST /tasks` on one of them, when a client sends `GET /tasks` 10 times alternating between the two instances, then every one of the 10 responses contains that task, and tasks created through either instance are listed in creation order, oldest first (system AC-001.2)"
      - "Given the database is unreachable (stopped, wrong host, or `DATABASE_URL` unset), when a client sends `GET /tasks` or a valid `POST /tasks`, then the service answers 503 with the body `{\"detail\": \"Task store is unavailable\"}` within 3 seconds, no task is stored, and `/health` and `/ready` still answer 200 (system AC-001.7, AC-001.8)"
      - "Given the database was unreachable and the service answered 503, when the database becomes reachable again, then the next `GET /tasks` and `POST /tasks` answer 200 and 201 without a restart of the service (system AC-001.7)"
    arguments: |
      Replace the in-memory TaskStore behind get_task_store (design of spec 001) with a Postgres store; keep the
      in-memory class only as a unit-test double. Connection from DATABASE_URL (the platform also sets PG*);
      psycopg 3 with a small pool. Table `tasks`: `seq bigserial primary key`, `id text unique not null` (uuid4 string,
      as today), `title text not null`, `due_date date null`; list ORDER BY seq. Create the table on first successful
      connection with CREATE TABLE IF NOT EXISTS inside a transaction holding pg_advisory_xact_lock, because two replicas
      start at once. The service must start and stay ready while the database is down (503 on /tasks, never a crash
      loop). Acceptance tests need a real Postgres: add `postgresql` to devbox.json (or testcontainers) and a devbox
      recipe for it; CI needs the same. Existing smoke tests keep working once dev has the database.
    depends_on: [taskboard10-gitops]
    done: false
  - id: taskboard10-web
    shape: web-nextjs
    summary: Task board start page that lists tasks and adds a task through the task API
    acs: [AC-001.1, AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.7, AC-001.8]
    criteria:
      - "Given `GET /tasks` of the task API returns tasks, when a person opens the start page `/`, then it lists every task in the order the API returned (oldest first), each showing its `title` and its `due_date` exactly as `YYYY-MM-DD`, or the text \"No due date\" when `due_date` is null; the page asks the API on every load (no cached list), so tasks that exist after an API restart or were created elsewhere appear on the next load (system AC-001.1, AC-001.3)"
      - "Given `GET /tasks` returns `[]`, when a person opens the start page, then it shows the text \"No tasks yet\" and the add form with a title field, a date field and a submit button (system AC-001.4)"
      - "Given the board is open, when a person enters the title \"Buy milk\", picks the due date 2026-12-01 (or leaves the date empty) and submits, then `POST /tasks` reaches the task API with `{\"title\": \"Buy milk\", \"due_date\": \"2026-12-01\"}` (or `\"due_date\": null`), the task the API returned appears at the end of the list without a page reload, \"No tasks yet\" is gone, and both form fields are empty (system AC-001.5)"
      - "Given the board is open, when a person submits an empty or whitespace-only title, then \"Enter a title\" is shown next to the title field; when the title is longer than 200 characters after trimming, then \"Title must be at most 200 characters\" is shown there; in both cases the entered title and date stay in the form and no request reaches `POST /tasks` (system AC-001.6)"
      - "Given `GET /tasks` answers 503, any status other than 200, a body that is not a JSON array of tasks, or no answer within 5 seconds, when a person opens the start page, then it shows \"Tasks could not be loaded. Try again later.\" instead of the list and instead of \"No tasks yet\" (system AC-001.7)"
      - "Given `POST /tasks` answers 503, any status other than 201 and 422, or no answer within 5 seconds, when a person submits a valid task, then the board shows \"Task could not be saved. Try again.\", the entered title and due date stay in the form, and the list is unchanged (system AC-001.8)"
    arguments: |
      The page fetches the API on the server only: TASKS_API_URL (server-side env, no NEXT_PUBLIC_, default
      http://localhost:8080 for local dev); the browser never calls the task API directly, so no CORS. Render `/`
      dynamically (no static generation, `cache: "no-store"`). The add form is a client component posting to the web
      app's own route handler `POST /api/tasks` (see Contract), which forwards to the API. Show due_date as the string
      the API returned; never parse it into a Date (time zones shift the day). Replace the template's landing content
      on `/`; keep /api/health, /api/ready and /api/metrics. Document TASKS_API_URL in docs/env-vars.md. Keep a
      read-only smoke test (the board loads and shows a list, "No tasks yet" or the load error) and a @writes one
      that adds a `smoke-<uuid>` task in dev.
    depends_on: []
    done: false
gitops_pin: []
---

## taskboard10-gitops — Declare the system's Postgres database, give it to the task API and point the web board at the API

**Shape:** `gitops-app` · **Criteria:** AC-001.1, AC-001.2, AC-001.3 · **Depends on:** []

Persistence (AC-001.1) and one shared list across instances (AC-001.2) need a database both API pods reach: A7
picks a Postgres run in the cluster by the platform's database operator (CloudNativePG), declared as the system
resource `postgres`. `uses` gives `taskboard10-api` `DATABASE_URL` and `PG*` from the connection Secret. The board
(AC-001.3 onward) needs to know where the API lives: `TASKS_API_URL=http://taskboard10-api`, the API's in-namespace
Service (port 80). Both components already run in dev and are already exposed, so there is no `expose` operation.

This entry goes first (level 1): the API's next dev deploy reads `DATABASE_URL`, and its dev smoke tests
(`POST /tasks`, `GET /tasks`) answer 503 until the database exists. The in-memory API and the template web app
ignore the new variables, so landing this first is harmless.

AC-001.2 against the deployment needs two API replicas in dev. The plan's operation list has no `replicas`, so it
is a separate step: `cplat component set taskboard10-api --instance dev --replicas 2` (see gitops-app PR). The API
repo checks AC-001.2 itself with two app instances on one database.

## taskboard10-api — Store tasks in Postgres so they survive restarts and every instance sees the same list

**Shape:** `service-python` · **Criteria:** AC-001.1, AC-001.2, AC-001.7, AC-001.8 · **Depends on:** [taskboard10-gitops]

The API owns the persistence: the store behind `get_task_store` moves from process memory to Postgres, which fixes
both the lost tasks on restart (AC-001.1) and the per-instance lists (AC-001.2). Creation order across instances comes
from a database sequence, not from a pod's clock. It owns the 503 `Task store is unavailable` answer (AC-001.7) that
the board turns into its load and save messages (AC-001.8). The task model, the endpoints, the 201 body and the 422
rules of spec 001 do not change (spec Non-goals). Tasks held in memory at the switch are not carried over (A1).

Test concerns: the acceptance tests need a real Postgres (restart = new app instance on the same database; two
instances = two app objects or two processes on one database; unreachable = a closed port or a stopped server).
`/ready` must not depend on the database: a not-ready pod leaves the Service and the caller would get a connection
error instead of the 503 AC-001.7 requires.

## taskboard10-web — Task board start page that lists tasks and adds a task through the task API

**Shape:** `web-nextjs` · **Criteria:** AC-001.1, AC-001.3, AC-001.4, AC-001.5, AC-001.6, AC-001.7, AC-001.8 · **Depends on:** []

The board is the start page (A2): a server-rendered list from `GET /tasks` and one add form. It builds in parallel
with the API because the Contract below fixes every call; it works against the in-memory API as well. Its share of
AC-001.1 is to show what the API returns on every load, never a cached list. It checks the title rules before
sending (A4) and shows the spec's exact messages for the API being down (A5). A new task appears right after saving;
other people's tasks appear on the next load, no live updates (A6).

Test concerns: unit/component tests stub the task API (the server fetch and the `/api/tasks` route); the e2e tests
run against a stub API server on `TASKS_API_URL` that can answer 200, 201, 422, 503 or not at all. The date is shown
as the API's string, so a test with due date 2026-12-01 must show `2026-12-01` in any time zone.

## Contract

No authentication anywhere (spec Non-goals). Bodies are `application/json; charset=utf-8`. The task model and the
two task endpoints are those of taskboard10-api spec 001, unchanged; this spec adds the 503 answer and the web
app's own route.

### Task (the shape every endpoint returns)

| Field | Type | Notes |
|---|---|---|
| `id` | string | uuid4 assigned by the API; unique; never chosen by a client |
| `title` | string | trimmed, 1 to 200 characters |
| `due_date` | string `YYYY-MM-DD` or `null` | always present, never omitted |

### Task API (`taskboard10-api`, in-cluster `http://taskboard10-api`, from the web app `TASKS_API_URL`)

`GET /tasks`

- `200`: JSON array of Task, every stored task, creation order, oldest first. No cap, no pagination, no query parameters. Empty store: `[]`.
- `503`: `{"detail": "Task store is unavailable"}` when the database cannot be reached, a connection cannot be had within 2 s, or the query does not finish within 2 s.

`POST /tasks`

- Request: `{"title": "<string>", "due_date": "YYYY-MM-DD" | null}`; `due_date` may be omitted (= `null`); unknown fields are ignored.
- `201`: the created Task. It is committed to the database before the response is sent, so any instance's next `GET /tasks` contains it.
- `422`: FastAPI's default validation body `{"detail": [{"loc": ["body", "title" | "due_date"], "msg": "...", "type": "...", "input": ...}]}` for the spec-001 rules (title missing, not a string, empty after trimming, over 200 characters; due_date not a real `YYYY-MM-DD` date). Nothing is stored.
- `503`: `{"detail": "Task store is unavailable"}` as for `GET`; nothing is stored. Validation runs first: an invalid body gets 422 even when the database is down.

`GET /health` and `GET /ready` answer 200 whether or not the database is reachable (unchanged).

Database wiring: the API reads `DATABASE_URL` (set by the gitops `uses: postgres`; `PGHOST`, `PGPORT`, `PGDATABASE`,
`PGUSER`, `PGPASSWORD` are set too). Without `DATABASE_URL` the service starts and answers 503 on both task endpoints.

### Web board (`taskboard10-web`, exposed as `taskboard10-web.taskboard10-dev.localhost` in dev)

`GET /` (the board, server-rendered on every request)

- Calls `GET {TASKS_API_URL}/tasks` once per page load, uncached.
- 200 with tasks: an ordered list, one item per task in API order; each item shows the title and either the due date string `YYYY-MM-DD` or `No due date`.
- 200 with `[]`: the text `No tasks yet`.
- Anything else (see Errors): the text `Tasks could not be loaded. Try again later.`, no list and no `No tasks yet`.
- The add form is shown in all three cases: a text field labelled "Title", a date field (`<input type="date">`) labelled "Due date" and a submit button "Add task".

`POST /api/tasks` (the web app's own route handler, called only by the board's form)

- Request: `{"title": "<string>", "due_date": "YYYY-MM-DD" | null}` as typed by the person (the API trims).
- Forwards the body to `POST {TASKS_API_URL}/tasks` and answers:
  - `201` with the API's Task body when the API answered 201;
  - `422` with the API's body when the API answered 422;
  - `502` with `{"detail": "Task could not be saved"}` for every other outcome (API 503, other status, invalid body, connection error, timeout).

Board behaviour in the browser

- Before sending, the form trims the title: empty → `Enter a title`; more than 200 characters → `Title must be at most 200 characters`, shown next to the title field; nothing is sent and the fields keep their values. The message disappears on the next successful submit.
- On `201`: the returned Task is appended to the end of the list (replacing `No tasks yet`), both fields are cleared, no reload.
- On `422` from the route: the title message from the local rules is shown when `loc` names `title`; otherwise it is handled like a `502`.
- On `502`, any other status, or no route answer within 8 s: `Task could not be saved. Try again.` near the form, fields keep their values, list unchanged. The message is cleared on the next submit.
- The submit button is disabled while a request is in flight, so one click creates at most one task.

### Errors

- **Format.** The task API answers errors as `{"detail": <string>}` (503) or FastAPI's `{"detail": [<validation entries>]}` (422). The web route `/api/tasks` answers errors as `{"detail": <string>}` (502) or passes the API's 422 body through. A person never sees a raw error body: only the messages above.
- **Unlisted responses.** A caller treats any status the contract does not list for that call, a non-JSON body, or a body not matching Task / array of Task, as the other side being unavailable: the board's page load shows `Tasks could not be loaded. Try again later.`, the web route answers 502, the form shows `Task could not be saved. Try again.`. Nobody parses error text to decide; status codes decide.
- **No retries.** The web app does not retry: `POST` is not idempotent (a retry after a timeout could create a duplicate), and a failed page load is repeated by the person reloading. The API does not retry a failed database call within a request; it answers 503.
- **Logging.** The API logs a database failure once per request at `error` with the exception type (never `DATABASE_URL` or the password). The web app logs a failed API call at `error` with the call, status or error kind and duration.

### Timeouts

| Call | Timeout | When the other side is down or slow |
|---|---|---|
| task API → Postgres: get a pooled connection / connect | 2 s (`connect_timeout=2`, pool wait 2 s) | 503 `Task store is unavailable` |
| task API → Postgres: one statement | 2 s (`statement_timeout`) | 503; a `POST` whose insert did not commit stores nothing |
| task API overall, database down | answers within 3 s | 503 |
| web server → `GET {TASKS_API_URL}/tasks` (page render) | 5 s | page shows `Tasks could not be loaded. Try again later.` |
| web server → `POST {TASKS_API_URL}/tasks` (route `/api/tasks`) | 5 s | route answers 502; form shows `Task could not be saved. Try again.` |
| browser → web `POST /api/tasks` | 8 s | form shows `Task could not be saved. Try again.`, fields kept |

The timeouts nest (2 s < 3 s < 5 s < 8 s), so an unreachable database reaches the person as the spec's message and
not as a timeout one layer up. After the database comes back, the API's next request reconnects without a restart.

## gitops-app PR

One PR in this repo, before the API and web PRs merge: `/system:ship` runs the three operations in order
(`cplat resource add postgres`, `cplat component set taskboard10-api --uses postgres`,
`cplat component set taskboard10-web --env TASKS_API_URL=http://taskboard10-api`), then `devbox run render` and
`devbox run validate`. The orchestrator also runs `cplat component set taskboard10-api --instance dev --replicas 2`
for AC-001.2 in dev, because the plan's operations have no `replicas`.

No pins (`gitops_pin: []`): dev tracks `main`, and the database and the board stay in dev only (spec Non-goals: no
staging or prod in this spec; both components have only a `dev` instance, so the staging auto-promotion has nothing
to promote).

After Argo reconciles dev, verify:

- the CloudNativePG cluster `taskboard10-postgres` in `taskboard10-dev` is healthy and the Secret `taskboard10-postgres-app` exists;
- the `taskboard10-api` pods (2) have `DATABASE_URL` from that Secret, and the `taskboard10-web` pod has `TASKS_API_URL=http://taskboard10-api`;
- after the API and web merges: `POST /tasks` then 10 × `GET /tasks` on `taskboard10-api.taskboard10-dev.localhost` all contain the task (AC-001.2); `kubectl rollout restart deployment/taskboard10-api` keeps the list (AC-001.1); the board at `taskboard10-web.taskboard10-dev.localhost` shows it (AC-001.3) and the dev smoke runs of both components are green.

# 3-Tier Todo App — Docker & Kubernetes Practice

## 1. Architecture

A minimal 3-tier todo app for practicing container deployment:

- **Frontend**: Node/Express serving a static UI and proxying `/api/*` to the backend.
- **Backend**: Node/Express REST API for todos.
- **Database**: PostgreSQL, seeded by `database/init.sql`.

Every cross-service connection is driven by environment variables, so the
**only thing that changes between the Docker setup and the Kubernetes setup is
the values of those env vars** — not the application code.

```
 [ browser ]
      |
      v
 frontend (Node/Express, port 3000)
   - serves the static UI (public/)
   - proxies /api/* to BACKEND_URL
      |
      v
 backend (Node/Express, port 5000)
   - REST API for todos
   - connects to Postgres using DB_HOST / DB_PORT / DB_USER / DB_PASSWORD / DB_NAME
      |
      v
 database (PostgreSQL)
   - table "todos", seeded by database/init.sql
```

## Layout

```
frontend/       static UI + Express server that proxies /api to the backend
backend/        Express REST API (GET/POST/PUT/DELETE /api/todos)
database/       init.sql — schema + seed data for Postgres
.env.example    every env var used for service connectivity, documented
```

## Env vars (the important part)

| Var            | Used by  | Purpose                                          | Docker example        | Kubernetes example              |
|-----------------|----------|---------------------------------------------------|------------------------|----------------------------------|
| `BACKEND_URL`   | frontend | address of the backend service                    | `http://backend:5000`  | `http://backend:<global.backendPort>` |
| `DB_HOST`       | backend  | address of the database service                    | `database`              | `database`                       |
| `DB_PORT`       | backend  | database port                                       | `5432`                  | `5432`                           |
| `DB_USER`       | backend  | database user                                       | `todo_user`             | `global.postgresql.auth.username` |
| `DB_PASSWORD`   | backend  | database password                                   | `todo_pass`             | (from Secret)                    |
| `DB_NAME`       | backend  | database name                                       | `todo_db`               | `global.postgresql.auth.database` |
| `PORT`          | both     | port each service listens on                        | `3000` / `5000`         | same                              |
| `POSTGRES_USER/PASSWORD/DB` | db image | standard Postgres image init vars      | matches `DB_USER` etc.  | matches `DB_USER` etc.           |

See `.env.example` for a ready-to-copy reference.

## API endpoints (backend)

- `GET /health` — checks the DB connection
- `GET /api/todos` — list todos
- `POST /api/todos` `{ "text": "..." }` — create
- `PUT /api/todos/:id` `{ "completed": true }` — toggle
- `DELETE /api/todos/:id` — delete

## Quick local sanity check (no containers)

Useful once, just to confirm the app itself works before you containerize it.

1. Start a Postgres instance any way you like and run `database/init.sql` against it.
2. `cd backend && npm install && npm start` — export the `DB_*` vars first (or use `.env`).
3. `cd frontend && npm install && npm start` — export `BACKEND_URL=http://localhost:5000` first.
4. Open `http://localhost:3000`.

## 2. Docker deployment

Prerequisites: Docker, and a `.env` file in each of `database/`, `backend/`
and `frontend/` (they are git-ignored, and `docker/docker-compose.yaml`
reads them).

```
cd docker
docker compose up --build
```

`--build` rebuilds the frontend and backend images if the code changed.

Open the app at <http://localhost:3000>.

To stop it, run `docker compose down` (add `-v` to also delete the database volume).

## 3. Kubernetes deployment (Helm + minikube)

Prerequisites: Docker, [minikube](https://minikube.sigs.k8s.io/), `kubectl` and [Helm](https://helm.sh/).
Run the commands from the repo root. The chart lives in `helm_deployment/`.

1. Build the images. The chart expects the tags `frontend:latest` and `backend:latest`:
   ```
   docker build -t frontend:latest ./frontend
   docker build -t backend:latest ./backend
   ```
2. Start a minikube cluster:
   ```
   minikube start
   ```
3. Load the images into minikube. Minikube runs its own internal container runtime, separate from your host's Docker daemon, so it can't see locally built images otherwise:
   ```
   minikube image load frontend:latest
   minikube image load backend:latest
   ```
4. Resolve the chart's dependencies (downloads the Postgres chart, packages the local `backend`/`frontend` subcharts):
   ```
   helm dependency update ./helm_deployment
   ```
   Re-run this any time you edit `helm_deployment/charts/backend` or `helm_deployment/charts/frontend` — local
   chart dependencies are snapshotted into `.tgz` files at update time, so edits made afterward won't be picked
   up until you run it again.
5. Install (or upgrade) the release. `helm_deployment/values/` has per-environment override files
   (`values-dev.yaml`, `values-uat.yaml`, `values-prod.yaml`) — pick whichever fits:
   ```
   helm upgrade --install todo ./helm_deployment -f ./helm_deployment/values/values-dev.yaml
   ```
   `--install` makes this idempotent: it installs if the release doesn't exist yet, or upgrades it in place if it does.
6. Check that everything is running:
   ```
   kubectl get pods
   ```
7. Open the app:
   ```
   minikube service frontend --url
   ```
   On Windows/Mac with the `docker` driver, minikube's node runs inside an isolated container/VM, so its IP and
   NodePort aren't directly reachable from the host — `minikube service` sets up the tunnel for you. As an
   alternative, `kubectl port-forward service/frontend <local-port>:<service-port>` always works regardless of driver.

If a change only affects values read via a ConfigMap (e.g. `global.backendPort`), already-running pods won't
pick it up automatically — force a rollout after upgrading:
```
kubectl rollout restart deployment/backend deployment/frontend
```

To remove the release, run `helm uninstall todo`. To remove the cluster entirely, run `minikube delete`.

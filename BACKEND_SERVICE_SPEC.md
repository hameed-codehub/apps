# SkillCatalogService — Master Backend Service Specification

## 1. Purpose and status

This is the authoritative backend service specification for SkillCatalogService.
It consolidates the backend API contract, technical architecture and stack,
deployment and synchronization behavior, and the operational notes formerly
maintained separately. It distinguishes current implementation from required
or future behavior where applicable.

The checkout contains a React/TypeScript frontend and a FastAPI backend. Some
deployment information is documented but has not been verified against a
running production environment. There is no complete Docker Compose deployment
or verified end-to-end production configuration in this checkout.

## 2. Functional and access requirements

### User groups

- **Skill catalog users** browse, search, view, and download skills.
- **Skill catalog administrators** manage skills and sources, trigger syncs,
  and change administrative settings.
- Team accounts and shared workspaces are not implemented.

### API authorization

API access requires a user to have either the
`sfa_skillcatalog_user` or `sfa_skillcatalog_admin` entitlement. Protected
administrative operations must also enforce administrator authorization.
Entitlement identity must be established by the deployed authentication
integration or trusted gateway; it must not be inferred from UI visibility.

The current browser UI has no sign-in flow and does not attach an
administrator credential. In the current backend, write and administrative
routes use the configured `ADMIN_API_TOKEN`. The entitlement names are the
application access requirement; production integration must connect them to
the identity provider or trusted gateway.

### API surface

The API base path is `/api`; interactive local API documentation is available
at `http://127.0.0.1:8000/docs`.

| API group | Main operations |
|---|---|
| Health and operations | `GET /`, `GET /health`, `GET /ready`, `GET /metrics` |
| Skills | Search/filter/sort/page, top skills, details, create, update, delete, versions, files |
| Tags | List tags in use |
| Sources | List, add, update, archive, sync, sync status, and logs |
| Sync jobs | Queue all eligible sources and poll durable job status |
| System | Dashboard status and administrative settings |
| Bitbucket discovery | Search repositories in a project and explicitly add a selected repository as a source |

Protected mutations and repository discovery require administrator
authorization. Source deletion archives the source and retains its skills and
sync history.

## 3. End-to-end architecture

```text
Browser
  └─ React single-page application
       └─ Axios API client (/api by default)
            ├─ Local: Vite proxy → http://localhost:8000
            └─ Shared environments: HTTPS load balancer / DNS routing
                 └─ Any FastAPI API pod
                      ├─ Skills, tags, sources, settings, and API operations
                      ├─ Shared PostgreSQL database
                      │    ├─ Catalog and sync history
                      │    └─ Durable sync-job queue and claim leases
                      ├─ Bitbucket Data Center REST API
                      └─ Control-M schedules POST /api/sources/sync-all

Production frontend build:
Vite static output → Express server (src/server/index.ts)
```

In shared environments, all API pods must use the same PostgreSQL database
and compatible application configuration. Requests are not bound to a
particular pod: DNS/load-balancer routing can send each request to any healthy
pod. Catalog/API state that must be consistent across requests belongs in the
shared database or another shared service, not in pod-local memory.

This is the stateless-request property: clients do not need sticky sessions for
the API to route successive requests to different pods. It does not mean every
operational value is currently shared. Request rate-limit windows and basic
request counters are in memory per pod; apply shared rate limiting at the
gateway and aggregate metrics across pods. Do not rely on pod-local state for
cross-request application data.

The Cloudflare Pages/Fly.io hosting described in the product README is not
configured or verified here.

## 4. Synchronous API requests and asynchronous synchronization

Request routing and background work are separate concerns:

1. A Control-M HTTP request is routed to any healthy API pod.
2. `POST /api/sources/sync-all` validates the request, creates durable jobs in
   PostgreSQL, and returns `202 Accepted` with job identifiers and any skipped
   sources.
3. API-pod queue consumers claim jobs using short-lived database leases and
   perform the Bitbucket sync.
4. Control-M polls `GET /api/sync-jobs/{job_id}` until each job reaches
   `success` or `failed`.

The database queue is not required for DNS or load-balancer routing. It is
used by the current asynchronous sync workflow so the request can return
without holding the HTTP connection open for every repository, job status
survives API-process restarts, and a job with an expired lease can be claimed
by another pod after a failure. No separate worker-pod deployment is required:
the API pods also consume queued jobs.

An alternative design could run all sync work inline in the pod handling the
Control-M request, but then Control-M would wait for completion and a request
timeout or pod failure could interrupt the run. That would be a deliberate
change to the current asynchronous API contract, not a requirement for
stateless routing.

For DEV, SIT, UAT, and PROD:

- Configure Control-M to call the internal HTTPS API endpoint.
- Store administrator credentials in the Control-M credential store; do not
  place tokens in a URL or job script.
- Set `SYNC_ENABLED=false` on every API pod so each pod's APScheduler does not
  schedule duplicate runs.
- Poll the returned job IDs until terminal status.

APScheduler remains available for local development.

## 5. Technology stack

| Layer | Current technology | Implementation/status |
|---|---|---|
| Frontend | React 19, TypeScript 5.6 | Present under `src/client/src` |
| Routing | wouter | Present in frontend dependencies |
| UI | Radix UI, Tailwind CSS 4, Framer Motion | Present in frontend dependencies/configuration |
| Charts | Recharts | Dashboard visualizations |
| Markdown and archives | Streamdown, JSZip | Skill instructions and ZIP export |
| API client | Axios | Uses `VITE_API_BASE_URL` or `/api` |
| Localization | i18next and react-i18next | Seven locale bundles |
| Frontend tooling | Vite 7, pnpm, TypeScript, Vitest | Build, type check, and smoke-test tooling |
| Frontend server | Express 4 | Serves the built SPA from `src/server/index.ts` |
| Backend | FastAPI, Uvicorn | API routes in `backend/app/main.py` |
| Models and validation | SQLAlchemy 2, Pydantic | Database models and API schemas |
| Local database/search | SQLite, FTS5 | Local development default |
| Shared database/search | PostgreSQL, built-in full-text search | DEV/SIT/UAT/PROD target; also stores durable sync jobs |
| Repository provider | Bitbucket Data Center REST API via httpx | Repository discovery, file listing, raw content, and sync |
| Local scheduling | APScheduler | Optional local scheduling |
| Shared-environment scheduling | Control-M | Calls the protected sync-all endpoint |

No separate worker deployment is currently required. A complete deployment
blueprint, database migration framework, and verified production hosting setup
remain outside the implemented stack.

## 6. Bitbucket Data Center integration

### Configuration and credentials

- `BITBUCKET_URL` is the HTTPS base URL for the Bitbucket Data Center instance,
  including any configured context path.
- `BITBUCKET_TOKEN` is an optional server-side HTTP access token. A token saved
  through the protected settings API is encrypted at rest and takes precedence
  over the environment variable. The environment value is used if no saved
  token exists.
- The token needs read access to repositories that will be discovered or
  synchronized. Never expose it in browser bundles, API responses, logs, or
  metrics.
- The API validates submitted tokens before saving them.

### Discovery and synchronization

- Discovery searches repositories within a requested Bitbucket project and
  returns repositories visible to the configured token.
- Discovery is administrator-protected and requires an explicit action to add
  a selected repository as a source.
- Sources must be repository URLs on the configured Bitbucket instance and
  follow its expected project/repository URL structure.
- Sync reads repository files and supported skill Markdown content.
- Bitbucket Data Center does not provide the previous provider's repository
  star/fork counts or remaining-request quota. Imported star and fork values
  are zero; quota telemetry is not displayed.
- Discovery results and source sync must handle upstream pagination, bounded
  timeouts, retries, and errors without treating partial failures as success.

## 7. Data, schema, and migration

### Database selection

| Environment | Database |
|---|---|
| Local laptop development | SQLite with FTS5 |
| DEV, SIT, UAT, PROD | PostgreSQL with built-in full-text search |

Set `DATABASE_URL` for the target database. All API pods in a shared
environment must use the same PostgreSQL database so any pod can access the
catalog and durable sync-job state.

### Schema and migration limitations

- The application initializes tables using SQLAlchemy `create_all`; a
  versioned migration system is not currently included.
- `backend/sql/sqlite_schema.sql` and
  `backend/sql/postgresql_schema.sql` create fresh schemas; they are not
  migrations.
- Changing `DATABASE_URL` does not copy or migrate existing SQLite data.
  Seed the target database or perform a separate data migration.
- Existing databases rename the old repository URL column to
  `repository_url`, but legacy source URLs are not automatically converted.
  Replace or update source records so their URLs point to the configured
  Bitbucket Data Center instance before syncing.
- Plan and test backups, restore procedures, migration, and rollback before
  production schema changes.

## 8. Configuration and secret management

Configuration is loaded by `pydantic-settings` from the backend environment
and optional `.env` file. Important settings include:

| Setting | Purpose |
|---|---|
| `DATABASE_URL` | SQLite or PostgreSQL connection |
| `ADMIN_API_TOKEN` | Current backend credential for write/admin routes and Control-M |
| `BITBUCKET_URL` | Bitbucket Data Center base URL |
| `BITBUCKET_TOKEN` | Fallback server-side repository-read token |
| `SECRET_ENCRYPTION_KEY` | Encrypts tokens saved via API settings |
| `CORS_ORIGINS` | Allowed browser origins |
| `SYNC_ENABLED`, `SYNC_CRON` | Optional in-process scheduling control |
| `SYNC_CONCURRENCY` | Maximum simultaneous sync jobs |
| `SYNC_MAX_RETRIES`, `SYNC_TIMEOUT_SECONDS` | Sync retry and timeout bounds |
| `DISCOVERY_MAX_RESULTS` | Maximum repository discovery results |

Use the deployment secret store for deployed credentials. Do not commit
secrets or include them in frontend `VITE_*` variables. The intended API
access entitlements are `sfa_skillcatalog_user` and
`sfa_skillcatalog_admin`; connect entitlement validation to the identity
provider or trusted gateway as part of production authentication integration.

## 9. Reliability, security, and observability

- Keep API pods replaceable and avoid sticky-session requirements.
- Use PostgreSQL as the shared source of truth for catalog and sync-job data.
- Durable sync jobs must use bounded concurrency, claim leases, retries, and
  explicit terminal status. Duplicate active syncs for a source are prevented.
- Keep request rate limiting per-pod only as a local safeguard; apply shared
  limits at the gateway or ingress.
- Aggregate metrics and logs across pods. Current HTTP counters and rate-limit
  state are not globally shared.
- Restrict CORS, require TLS in shared environments, validate source URLs, and
  apply timeouts, response-size limits, retry ceilings, and concurrency limits
  to Bitbucket requests.
- Never return credentials or sensitive configuration from status/settings
  endpoints. Redact secrets from logs, errors, traces, and metric labels.
- Preserve prior valid catalog data on partial or failed syncs and record
  actionable job outcomes.

## 10. Repository and configuration map

| Concern | Location |
|---|---|
| Frontend routes and app providers | `src/client/src/App.tsx` |
| Frontend pages | `src/client/src/pages/` |
| Frontend API client and endpoint calls | `src/client/src/lib/api.ts` |
| Frontend API models | `src/client/src/lib/types.ts` |
| Localization | `src/client/src/i18n/` |
| Frontend package scripts and dependencies | `src/package.json` |
| Vite proxy and build config | `src/vite.config.ts` |
| Express static server | `src/server/index.ts` |
| Backend routes | `backend/app/main.py` |
| Backend configuration | `backend/app/config.py` |
| Bitbucket REST client | `backend/app/bitbucket.py` |
| Database models and setup | `backend/app/models.py`, `backend/app/database.py` |
| Backend schemas | `backend/app/schemas.py` |
| Sync implementation and queue | `backend/app/sync.py` |
| Fresh SQLite schema | `backend/sql/sqlite_schema.sql` |
| Fresh PostgreSQL schema | `backend/sql/postgresql_schema.sql` |
| Operational/API notes | `notes.md` |

## 11. Local development and verification

The local frontend uses Vite and proxies `/api` to
`http://localhost:8000`. Start the FastAPI backend on port `8000`; use
`backend/README.md` and the root README for environment setup, database
initialization, seed data, and frontend commands.

Verification should include:

- Backend unit and API tests for catalog behavior, source management,
  Bitbucket client behavior, settings, authorization, and sync jobs.
- Frontend type checking, smoke tests, and production build.
- Multi-pod tests confirming that requests work across pods sharing the same
  PostgreSQL database, and that queued work can be recovered after a pod exits.
- Control-M integration tests for sync-all submission, job polling, duplicate
  scheduling prevention, and terminal failure handling.
- Explicit validation of entitlement integration before enabling user API
  access in a deployed environment.

## 12. Stack replacement or deployment changes

When replacing a technology or changing deployment behavior:

1. Record the decision, rationale, and owner.
2. Preserve API contracts unless an intentional behavior change is approved.
3. Include frontend, backend, database, authentication, synchronization,
   localization, testing, and deployment dependencies in the impact analysis.
4. Define data migration and rollback before changing schemas or databases.
5. Keep credentials server-side and move them through the deployment secret
   store.
6. Validate stateless API routing independently from asynchronous job
   processing. A synchronous sync design would change response duration,
   timeouts, and failure recovery.
7. Update this specification, `notes.md`, setup instructions, and tests when
   implementation or operational behavior changes.

## 13. Detailed API contract

All paths below are under `/api` unless identified as process-level endpoints.
JSON is used except for documented downloads or metrics. Resource IDs are
positive integers. Timestamps are ISO 8601 UTC. Paginated collections use
`items`, `total`, `page`, and `page_size` where applicable. Page numbering
starts at 1; page size defaults to 20 and must be bounded (maximum 100).
Sort fields must be allow-listed.

### Skills and tags

| Method and path | Behavior and key details |
|---|---|
| `GET /api/skills` | Search and filter skills; supports `search`, comma-separated `tags`, `source_id`, `sort`, `page`, and `page_size`. Search covers name, description, tags, and indexed Markdown. |
| `GET /api/skills/top` | Return at most 20 ranked skills, optionally filtered by `tag`; use deterministic tie-breaking. |
| `GET /api/skills/{skill_id}` | Return skill metadata, Markdown content, files, star history, related skills, and versions. Empty detail collections are allowed; do not invent star-history data. |
| `POST /api/skills` | Create a manually authored skill from validated user-editable fields; administrator access required. Return `201 Created`. |
| `PUT /api/skills/{skill_id}` | Update editable fields, reindex search, and add version history for meaningful content/version changes; administrator access required. |
| `DELETE /api/skills/{skill_id}` | Soft-delete/hide a skill; administrator access required. Return `204 No Content`. |
| `GET /api/skills/{skill_id}/versions` | Return newest-first version records. |
| `GET /api/skills/{skill_id}/files` | Return relative file path, file type, and size; never expose server filesystem paths. |
| `GET /api/tags` | Return a stable de-duplicated list of tags. |

List skill fields include `id`, `name`, `description`, `stars`, `forks`,
`source`, `repository_url`, `tags`, timestamps, `language`, `license`,
`version`, `downloads`, `skill_md_preview`, and `platforms`. User input must
not set source-owned IDs, sync metadata, or privileged metrics.

### Sources and sync jobs

| Method and path | Behavior and key details |
|---|---|
| `GET /api/sources` | List active non-archived sources and their status, type, counts, branch, skills path, and last sync. Never return credentials. |
| `POST /api/sources` | Add a repository from the configured Bitbucket instance; validate and normalize URL/identity; administrator access required. |
| `PUT /api/sources/{source_id}` | Update source name, branch, or skills path; administrator access required. |
| `DELETE /api/sources/{source_id}` | Archive the source while retaining its skills and history; administrator access required; return `204`. |
| `POST /api/sources/{source_id}/sync` | Queue one source sync and return promptly with job/log IDs and `queued` status; administrator access required. |
| `POST /api/sources/sync-all` | Queue eligible sources, return job IDs and skipped source IDs; administrator access required; return `202 Accepted`. |
| `GET /api/sync-jobs/{job_id}` | Return durable job state (`queued`, `running`, `success`, or `failed`), attempts, lease expiry, and associated log; administrator access required. |
| `GET /api/sources/{source_id}/sync/status` | Return source status, skill count, last sync, current error, and latest log. |
| `GET /api/sources/logs` | Return newest-first paginated logs across sources. |
| `GET /api/sources/{source_id}/logs` | Return newest-first paginated logs for one source. |

Source states include `active`, `syncing`, `error`, and `pending`. Sync logs
include status, source, start/finish times, duration, added/updated/removed
counts, and a safe error summary. Prevent simultaneous active jobs for the
same source. Expired claim leases must permit recovery by another API pod.

### System settings, discovery, and process operations

| Method and path | Behavior and key details |
|---|---|
| `GET /api/system/status` | Dashboard totals, source/sync state, scheduler state, seven-day sync history, top tags, star distribution, and Bitbucket configuration booleans. Never include secrets. |
| `GET /api/system/settings` | Return safe scheduler settings and token-configured status; administrator access required. |
| `PUT /api/system/settings` | Validate optional Bitbucket token, cron expression, and concurrency; save token encrypted at rest; administrator access required. Never echo token. |
| `POST /api/discovery/search` | Search repositories in a requested Bitbucket project using bounded `project_key`, `query`, and `max_results`; administrator access required. |
| `GET /` | Return API identity/version and documentation link. |
| `GET /health` | Process liveness. |
| `GET /ready` | Database and required-dependency readiness without leaking internal configuration. |
| `GET /metrics` | Basic operational metrics in Prometheus text format. |

The settings token response reports validation status when a token is submitted.
A database-saved encrypted token takes precedence over `BITBUCKET_TOKEN`; the
environment token is the fallback if no saved token exists. Bitbucket
discovery returns candidates only; a maintainer must explicitly add a selected
repository as a source.

## 14. Persistence model and domain behavior

The logical relational model includes:

- **Skills:** name, description, Markdown content/preview, repository URL,
  language/license/version, counts, timestamps, optional soft-delete marker,
  and optional source relation. Search covers name, description, tags, and
  Markdown.
- **Sources:** normalized unique repository identity, Bitbucket URL, source
  type, branch, skills path, status, error, skills count, last-sync time, and
  archive marker.
- **Tags and skill-tags:** normalized tags with a unique skill/tag relation.
- **Skill files:** relative repository paths, type, and size.
- **Skill versions:** version, content snapshot/reference, changelog, and
  creation time.
- **Star history:** only genuine upstream observations. Bitbucket Data Center
  has no star/fork metrics in this integration, so imported values remain
  zero; do not fabricate observations.
- **Sync jobs and logs:** durable job state, source relation, attempts, claim
  lease, lifecycle timestamps, outcome counts, and safe error summary.
- **Settings and secrets:** scheduler settings persisted as application data;
  Bitbucket tokens encrypted at rest or held in deployment secret storage.
  Never store secrets in general logs or sync records.

Repository synchronization must be idempotent. Validate the source and reject
duplicate active jobs, create a durable queued job/log, then transition it to
running when claimed. Use explicit connect/read timeouts, bounded retries with
backoff, pagination, response-size limits, and bounded concurrency when
calling Bitbucket. Parse supported `SKILL.md` and `*.skill.md` files and safe
metadata only. Preserve existing catalog data if an upstream fetch is partial
or fails. On success, upsert skills, tags, files, and meaningful versions;
on failure, persist a safe actionable error and terminal status. Future
changes must continue to preserve manual edits during source sync.

## 15. API errors, validation, and security requirements

- Use typed Pydantic request/response schemas and request/body size limits.
- Use `400`/`422` for invalid input, `404` for missing records, `409` for
  conflicts, `401`/`403` for failed authorization, and explicit `429`/`502`/
  `503` responses for rate limiting or upstream/dependency failures.
- Maintain a consistent error response such as
  `{"detail":"Human-readable message","code":"stable_error_code"}`; do not
  expose stack traces, database details, local paths, or credentials.
- Validate names, tags, platform IDs, branch/path values, source URLs, cron
  expressions, and numeric bounds. Source URLs must match the configured
  HTTPS Bitbucket instance and expected project/repository path; do not provide
  a generic fetch-URL endpoint.
- Use parameterized SQL and allow-listed filter/sort fields.
- Apply CORS only for configured browser origins; use TLS and secure headers
  at the deployed edge.
- Keep Bitbucket credentials server-side and redact them from responses,
  logs, traces, and metric labels.
- Entitlement requirements are `sfa_skillcatalog_user` or
  `sfa_skillcatalog_admin` for API access; administrative operations require
  administrator authorization. Connect these entitlements to the identity
  provider or trusted gateway in the deployed authentication integration.
- In the current implementation, admin routes use `ADMIN_API_TOKEN`. The
  browser UI does not currently sign in or attach admin credentials. Treat
  this as an implementation gap, not as the intended final user experience.

## 16. Test and acceptance requirements

### Backend tests

- Search, filtering, sorting, pagination, validation, normalization, and
  deterministic result ordering.
- Skill create/update/delete, tag/file/version handling, and search indexing.
- Source URL validation, duplicate detection, archive behavior, and sync
  status transitions.
- Bitbucket pagination, response parsing, raw-content retrieval, token
  validation, timeouts, retries, and upstream errors.
- API success, malformed input, missing record, authorization failure,
  conflict, and secret non-disclosure.
- Successful and partial sync behavior, idempotency, concurrency limits,
  duplicate job prevention, lease renewal, and expired-lease recovery.

### Deployment acceptance

- Local laptop development works with SQLite/FTS5 and the documented Windows
  PowerShell setup.
- DEV/SIT/UAT/PROD use shared PostgreSQL, and all API pods can serve requests
  without sticky sessions.
- A request routed to any healthy pod can read/write shared state correctly.
- Control-M submits sync-all, receives job IDs, polls to terminal status, and
  does not depend on the request-handling pod remaining alive.
- Shared APScheduler duplication is prevented with `SYNC_ENABLED=false`.
- Entitlement integration, secrets handling, TLS, gateway rate limiting,
  monitoring aggregation, backups, restore, and rollback are validated before
  production reliance.
- Frontend type checks, smoke tests, and production build pass; live Bitbucket
  behavior must be separately validated against a configured instance.

# ClipForge AI Ultra — Render Deployment Notes

**Status: preparation only. No Render resources are created by this file.**

This repository can run on Render as a multi-service Docker deployment. Do not deploy a frontend-only service and call the clipping engine operational: the API, ARQ worker, PostgreSQL, Redis, and shared media storage are all required.

## Why this is not yet a one-click Blueprint

Render's current free service types include web services, PostgreSQL, and Key Value. Background workers are not available on the free tier. The ARQ worker is required to consume video jobs, so a complete Render deployment needs a paid worker (and normally persistent production database/storage). A Blueprint that silently creates those resources could incur charges. For that reason, this repository does **not** add an auto-deploying `render.yaml` until the owner approves the expected monthly cost.

Official Render references:
- [Blueprint YAML reference](https://render.com/docs/blueprint-spec)
- [Free instance limitations](https://render.com/docs/free)
- [Compute plan pricing](https://render.com/pricing)

## Required service topology

| Service | Render type | Docker build context | Dockerfile / start |
|---|---|---|---|
| Frontend | Web service | repository root | `frontend/Dockerfile`; listens on port `3107` |
| Backend API | Web service (private is preferred once frontend routing is confirmed) | `backend` | `Dockerfile`; default command runs Uvicorn on port `8000` |
| Main video worker | Background worker | `backend` | `Dockerfile`; command: `.venv/bin/arq src.workers.tasks.WorkerSettings` |
| PostgreSQL | Render Postgres | managed | One database shared by frontend and backend |
| Redis | Render Key Value | managed | Internal host/port used by backend and worker |
| Media storage | S3-compatible object storage | external | Required for reliable sharing between separately deployed API and worker |

The main worker must use the same database, Redis, LLM/transcription configuration, and media-storage configuration as the backend. Do not use separate local container volumes for uploads and clips: Render services do not share their local filesystems.

## Environment variable mapping

Configure values in Render's encrypted environment settings. Never commit real credentials.

### Database

- Frontend: `DATABASE_URL` = Render Postgres connection string.
- Backend and worker: `DATABASE_URL` must use SQLAlchemy's async driver form, `postgresql+asyncpg://...`. Render's standard connection string normally begins `postgresql://...`; confirm the backend URL is normalized to the asyncpg scheme before first deploy.
- Use the private/internal database connection string when all services are in the same Render region.

### Redis

The backend configuration reads:
- `REDIS_HOST`
- `REDIS_PORT`
- optional `REDIS_PASSWORD`

Use the Key Value instance's internal host and port. Do not expose Redis publicly.

### Media storage

Set these identically on the backend and worker:
- `MEDIA_STORAGE=s3`
- `MEDIA_S3_BUCKET`
- `MEDIA_S3_ENDPOINT`
- `MEDIA_S3_REGION`
- `MEDIA_S3_ACCESS_KEY_ID`
- `MEDIA_S3_SECRET_ACCESS_KEY`
- optional `MEDIA_S3_PREFIX` (defaults to `media`)

The application validates that a bucket is configured when S3 mode is enabled. Do not enable S3 mode until the bucket and credentials exist. Never put storage credentials in frontend variables.

### Application and AI configuration

- Generate strong, unique production values for `BACKEND_AUTH_SECRET` and `APP_SETTINGS_ENCRYPTION_KEY`; never reuse development defaults.
- Configure at least one supported LLM provider key and a matching `LLM` model setting before testing AI clip selection.
- The example environment uses AssemblyAI; if using it, configure `ASSEMBLY_AI_API_KEY`. YouTube caption-based transcription may work for eligible sources without this key, but it is not a replacement for upload transcription.
- Set `NEXT_PUBLIC_APP_URL` to the public frontend HTTPS URL.
- Set `NEXT_PUBLIC_API_URL` to the backend API URL required by the frontend build.
- Set backend `CORS_ORIGINS` to the exact frontend origin, including scheme.
- Keep social publishing disabled until official OAuth apps, callback URLs, and provider approvals are ready.

## Deployment and verification order

1. Confirm the approved monthly budget and choose paid worker/database/storage plans explicitly.
2. Create PostgreSQL, Redis Key Value, and the S3-compatible bucket in the same region where possible.
3. Deploy backend API and worker with matching environment variables.
4. Confirm backend health endpoints: `/health`, `/health/db`, `/health/redis`, `/health/worker`, and `/health/storage`.
5. Deploy frontend with its production API URL and app URL build arguments.
6. Test a short video end-to-end: ingest/upload, enqueue, worker processing, MP4 export, playback/download, and persistence after a service restart.
7. Only after the MP4 test passes, configure social OAuth and publishing.

A healthy API response alone does not prove that the worker can render a video.

## Budget and safety gate

- A free Render web service can sleep when idle and is not suitable for reliable long-running video jobs.
- Render's free tier does not provide a free background worker. The worker is essential, so a complete deployment is not zero-cost on Render.
- Do not click **Deploy Blueprint**, upgrade a service, or create a paid resource until the displayed price is understood and approved.
- Start with a short test video and strict duration/concurrency limits. AI API usage, object storage, bandwidth, and database hosting can add separate costs.

## Licensing

The source repository is licensed under AGPL-3.0. Preserve notices and review the license's source-disclosure requirements when modifying and offering the software as a network service.

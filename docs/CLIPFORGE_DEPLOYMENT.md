# ClipForge AI Ultra — Deployment Plan

This document records the practical path for running this repository as an independently operated video-clipping service. It is a deployment checklist, not a claim that production hosting or provider credentials are already configured.

## 1. Application components

The repository contains a Next.js frontend, a FastAPI backend, PostgreSQL, Redis, and an ARQ background worker. Video processing depends on FFmpeg and configured transcription/LLM providers. The worker is essential: the web interface alone cannot process videos.

## 2. Recommended deployment topology

- **Frontend:** deploy the Next.js application to a host that supports the repository's Next.js runtime.
- **Backend API:** run the FastAPI service as a persistent container or service.
- **Worker:** run the ARQ worker as a separate persistent process using the same application configuration as the backend.
- **Database:** use PostgreSQL with persistent storage and backups.
- **Queue/cache:** use Redis with persistence/availability appropriate to the workload.
- **Media storage:** use durable object storage or a persistent volume for uploaded source media and generated clips. Configure retention and access controls.

A frontend-only deployment (including a Vercel frontend) does not run the persistent Docker worker, FFmpeg jobs, PostgreSQL, or Redis by itself. Do not treat a successful frontend deployment as proof that clipping works.

## 3. Configuration and secrets

Start from `.env.example` and configure secrets in the hosting provider's encrypted environment-variable settings. Never commit real keys or passwords.

At minimum, review:
- Database connection settings for PostgreSQL.
- Redis connection settings.
- A strong, unique application/session secret; replace all development defaults.
- The selected transcription provider and its API key (the example configuration uses AssemblyAI).
- The selected LLM provider and its API key, plus the matching `LLM` provider/model setting.
- Public frontend URL and backend/API URL settings required by the current application configuration.
- Storage paths or object-storage credentials, if using external media storage.

Keep backend-only secrets out of browser-exposed variables (for example, variables prefixed with `NEXT_PUBLIC_`). Use separate credentials for development and production.

## 4. Initial deployment sequence

1. Review `QUICKSTART.md`, `DEPLOY_NOTES.md`, `.env.example`, and the Docker Compose file.
2. Provision PostgreSQL, Redis, durable media storage, and a container host capable of running FFmpeg.
3. Configure the backend and worker with the same database, Redis, provider, and storage settings.
4. Deploy the backend and worker; confirm both remain healthy.
5. Deploy the frontend and point it at the deployed backend.
6. Configure HTTPS, allowed origins, upload limits, and any required reverse-proxy timeouts.
7. Set up database backups, media retention, monitoring, and error logging.

The included `start.sh` / Docker Compose workflow is intended for a Docker-capable environment. It is not an Android-phone installation procedure.

## 5. Health and acceptance checks

The backend exposes health routes documented in the source, including:
- `/health`
- `/health/db`
- `/health/redis`
- `/health/worker`
- `/health/storage`

Before calling the service operational, verify:
- Frontend can reach the backend over HTTPS.
- Database and Redis health checks pass.
- Worker health is positive and queued jobs are consumed.
- A small permitted test video can be uploaded or ingested.
- A real processing job completes and produces a playable MP4 file.
- The MP4 can be downloaded after the job finishes and remains available after a service restart.
- Logs contain no exposed secrets or recurring processing errors.

A passing CI workflow is useful, but does not replace this live end-to-end video test.

## 6. Cost and provider constraints

Video processing consumes CPU/GPU time, storage, bandwidth, and potentially paid transcription/LLM usage. Free tiers have quotas and may sleep or terminate background work. Do not promise unlimited processing at zero cost. Start with short test videos and enforce upload, duration, concurrency, and retention limits.

Social publishing must use each platform's official API and approved permissions. Start by validating local MP4 generation; add account OAuth and publishing only after the core processing pipeline is reliable.

## 7. Security and licensing

- Keep secrets out of Git and rotate any credential that has been exposed.
- Restrict admin access and apply least-privilege database/storage credentials.
- Validate uploaded files and URLs; limit file size, duration, and processing concurrency.
- Keep dependencies and container images updated.
- This repository is licensed under AGPL-3.0. Preserve license and copyright notices and review the AGPL source-disclosure obligations before modifying and offering the software as a network service. This checklist does not replace legal advice.

## 8. Current readiness status

This file is a deployment plan only. It does not provision infrastructure, add provider credentials, deploy services, or demonstrate a successful MP4 render. Those steps must be completed and verified separately.

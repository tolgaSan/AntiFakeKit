# AntiFakeKit Tech Stack (Phase 1)

## Guiding Principles
- **Modern & pragmatic**: Use battle-tested, startup-friendly tooling with strong community support and fast iteration.
- **Separation of concerns**: Keep the UI, API, and detection pipeline modular with clear contracts.
- **Cloud-agnostic**: Default to self-hostable components; allow easy migration to managed services later.
- **GPU-ready**: Ensure the pipeline can leverage NVIDIA GPUs locally or in production without major refactors.
- **Developer productivity**: Favor TypeScript typing, code generation where helpful, and hot-reload dev loops.

## Selected Stack

### Frontend
- **Framework**: **Next.js 14 (React + TypeScript, App Router)** — strong ecosystem, SSR/ISR for authenticated dashboards, and first-class file-based routing.
- **Styling**: **Tailwind CSS** + **shadcn/ui** component primitives (Radix-based) for accessible, composable UI without vendor lock-in.
- **State & data fetching**:
  - **TanStack Query** for server state (uploads, job status polling, caching, optimistic updates).
  - **Zustand** for lightweight client state (UI preferences, transient UI flags) to avoid Redux boilerplate.
- **Form handling**: **react-hook-form** + **zod** schemas for type-safe validation shared with API DTOs.
- **Auth-ready**: Integrate **NextAuth.js** (bring-your-own provider) when authentication is required.

### Backend / API
- **Runtime**: **Python 3.11** with **FastAPI** — async-first, great typing, OpenAPI generation, and smooth interop with Python ML stack.
- **API style**: **REST** for simplicity in v1 (upload → analyze → poll), with OpenAPI docs; GraphQL can be added later for federated querying.
- **Background jobs**: **Celery** workers with **Redis** as broker (swappable to RabbitMQ if needed for higher throughput guarantees).
- **Packaging**: **Poetry** for dependency management and virtualenv isolation.
- **Testing**: **pytest** + **httpx** for API tests; **ruff** and **mypy** for linting/type checks.

### Workers / Detection Pipeline
- **Language**: **Python** for all ML/inference tasks to reuse ecosystem models.
- **Execution frameworks**:
  - **PyTorch** as default for model development and research-grade checkpoints.
  - **ONNX Runtime** for portable, optimized inference; optional **TensorRT** acceleration when GPUs are available.
- **Media processing**: **ffmpeg** for video/audio splitting; **pydub**/ **librosa** for audio features; **Pillow**/**opencv-python** for images.
- **Pipeline orchestration**: Celery task graph (ingest → preprocess → model ensemble → score aggregation → artifact export).
- **Model registry**: Simple local registry (e.g., `models/` with versioned folders and metadata YAML); pluggable to remote storage later.

### Infrastructure (Local Dev via Docker Compose)
- **API**: FastAPI service (uvicorn) with hot reload in dev.
- **Frontend**: Next.js dev server; production build served by Next.js standalone output.
- **Workers**: Celery worker(s) sharing codebase with API; optional GPU-enabled worker image (CUDA base) when hardware is present.
- **Message queue**: **Redis** (default) for broker + caching; can swap to RabbitMQ if SLA requires stronger delivery semantics.
- **Object storage**: **MinIO** (S3-compatible) for uploads, derived artifacts, and model weights.
- **Database**: **PostgreSQL** for jobs, results, audit trails, and user data.
- **Reverse proxy**: **Traefik** or **NGINX** (optional) for local TLS and consistent routing across services.

### Storage & Data Model (v1)
- **Tables** (PostgreSQL):
  - `users` (optional for auth-enabled deployments)
  - `jobs` (id, user_id, media_type, status, created_at, updated_at, priority)
  - `job_inputs` (job_id, source_type [upload/url], object_key, metadata JSON)
  - `job_results` (job_id, overall_score, per_signal_scores JSON, summary, completed_at)
  - `artifacts` (job_id, type [heatmap/report/log], object_key, mime_type)
  - `events`/`logs` (job_id, message, level, timestamp) for auditability
- **Object storage layout**:
  - `uploads/{job_id}/original/*`
  - `artifacts/{job_id}/{type}/*`
  - `models/{model_name}/{version}/*`

### API Surface (v1 REST)
- `POST /api/jobs` — initiate analysis; accepts media upload or URL reference; returns `job_id`.
- `GET /api/jobs/{job_id}` — fetch job metadata and current status.
- `POST /api/jobs/{job_id}/upload` — presigned URL flow for direct-to-MinIO upload (optional inline upload for small files).
- `GET /api/jobs/{job_id}/results` — aggregated scores, per-signal breakdown, artifact links.
- `GET /api/jobs/{job_id}/events` — processing log/events for transparency.
- `GET /health` — liveness/readiness probes.

### Repository & Folder Structure (proposed)
```
anti-fake-kit/
├─ apps/
│  ├─ web/                # Next.js app
│  └─ api/                # FastAPI service (shared schemas with workers)
├─ services/
│  └─ workers/            # Celery tasks, model loaders, pipelines
├─ packages/
│  ├─ shared-schemas/     # pydantic/zod schema parity via codegen
│  └─ ui/                 # Reusable UI components (optional)
├─ infra/
│  ├─ docker/             # Dockerfiles for api/web/workers (GPU + CPU variants)
│  └─ compose/            # docker-compose.yml for local dev
├─ models/                # Model weights + metadata (gitignored)
└─ docs/                  # Architecture docs (including this file)
```

### DevEx & CI/CD
- **Formatting & linting**: `ruff`, `black`, `isort` for Python; `eslint` + `prettier` for TypeScript; Husky pre-commit hooks.
- **Testing**: `pytest` for backend/workers; `vitest`/`jest` + `playwright` for frontend.
- **Type safety**: `mypy` for Python; strict TypeScript; shared OpenAPI schema generation for client types.
- **CI**: GitHub Actions with parallel jobs (lint, test, build). GPU-enabled workflow can be toggled via self-hosted runner.
- **Observability**: Structured logging (JSON) + **Prometheus** metrics endpoint from API/workers; **OpenTelemetry** instrumentation later.

### Rationale (Why This Stack)
- Aligns with modern, productivity-focused tooling favored by startups (Next.js, Tailwind, FastAPI, PyTorch).
- Keeps ML pipeline in Python to minimize impedance mismatch with model ecosystems.
- REST-first keeps the integration surface simple while preserving a path to GraphQL later.
- Docker Compose provides reproducible local dev while remaining deployable to K8s or serverless containers later.
- Redis/MinIO/PostgreSQL stack is self-hostable, scalable, and swap-friendly for managed services.

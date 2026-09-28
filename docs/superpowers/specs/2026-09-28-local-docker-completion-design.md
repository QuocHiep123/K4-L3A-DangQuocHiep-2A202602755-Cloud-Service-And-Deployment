# Local Docker Completion Design

## Goal

Complete the Day 12 cloud-service lab end to end using the documented local
Docker fallback, while targeting a final automated score of 100 through the
mandatory checkpoints, reflection answers, and CI/CD bonus.

## Scope and scoring

- Implement CP1 through CP4 completely.
- Package and run the service with Docker Compose using a real Redis service.
- Use `LOCAL_FALLBACK=true` for CP5 and accept its rubric cap of 9/15.
- Complete all ten reflection answers from verified local observations.
- Implement the GitHub Actions bonus so the capped final score can still reach
  100: 15 + 15 + 20 + 20 + 9 + 15 + 10 = 104, capped at 100.
- Do not fabricate a public URL, cloud output, image sizes, logs, or screenshots.

## Architecture

The FastAPI `agent` service remains stateless. Redis stores conversation
history, rate-limit events, and monthly spend so multiple service instances can
share state. The request pipeline is authentication, rate-limit check, budget
check, history read, mock-LLM execution, state persistence, cost recording, and
structured logging.

Docker Compose contains an `agent` service built from a multi-stage,
non-root Dockerfile and a persistent `redis:7-alpine` service. `/health` is a
dependency-free liveness endpoint. `/ready` checks Redis and refuses traffic
during graceful shutdown.

## Components

- `app/config.py`: typed environment configuration with a mandatory API key.
- `app/logging_utils.py`: one-line JSON logs with UTC timestamps.
- `app/auth.py`: constant-time API-key verification and user identity.
- `app/rate_limiter.py`: per-user 60-second sliding window in Redis ZSETs.
- `app/cost_guard.py`: per-user, per-month spend tracking in Redis.
- `app/store.py`: bounded, expiring conversation history in Redis lists.
- `app/lifecycle.py`: SIGTERM/SIGINT shutdown state with previous-handler
  forwarding.
- `app/main.py`: health, readiness, and protected ask endpoints.
- `Dockerfile`, `.dockerignore`, `docker-compose.yml`: local production-like
  packaging and execution.
- `.github/workflows/ci.yml`: tests first, image build second, guarded deploy job
  last, with secrets referenced through GitHub Secrets.

## Request and failure behavior

`/ask` rejects missing or invalid credentials with 401 before quota work. It
returns 429 when the sliding-window quota is exhausted and 402 when the monthly
budget is exhausted, both before calling the mock LLM. Empty questions are
rejected by Pydantic with 422. Redis failures make `/ready` return 503 but do
not make `/health` fail unless the process is shutting down.

## Configuration and secrets

`.env` is local-only and ignored by Git. Local Python tests use `fake://` where
appropriate; Docker Compose overrides `REDIS_URL` to `redis://redis:6379/0`.
The API key is passed through environment interpolation and is never embedded
in the image, Compose file, CI workflow, deployment documentation, or Git.

## Verification

Verification proceeds checkpoint by checkpoint with the repository tests, then
with the full suite. The Docker image must build, run as a non-root user, report
healthy, connect to the Compose Redis service, and serve `/health`, `/ready`,
and authenticated `/ask`. The final evidence consists only of output observed
from actual commands. If the environment cannot access the Docker daemon, code
and static tests will still be completed and the remaining runtime limitation
will be reported explicitly.

## Deliverables

- Passing CP1 through CP4 tests.
- Passing CP2 Docker build tests when Docker is available.
- Passing CP5 local-fallback tests when Docker is available.
- Ten original reflection answers grounded in observed results.
- Passing CI/CD bonus tests.
- Clean Git status with no tracked secret and no invented cloud evidence.

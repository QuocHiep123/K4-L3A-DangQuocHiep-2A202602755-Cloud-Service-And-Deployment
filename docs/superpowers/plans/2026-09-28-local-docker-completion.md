# Local Docker Completion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [x]`) syntax for tracking.

**Goal:** Complete every locally verifiable lab requirement and reach a capped final automated score of 100 by combining the Docker fallback score with the CI/CD bonus.

**Architecture:** Keep FastAPI stateless and use Redis for conversation history, rate-limit events, and monthly spend. Build a non-root multi-stage image and run it beside persistent Redis through Docker Compose, with separate liveness and readiness semantics.

**Tech Stack:** Python 3.12, FastAPI, Pydantic Settings, redis-py, fakeredis, pytest, Docker, Docker Compose, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-09-28-local-docker-completion-design.md`

## Global Constraints

- `AGENT_API_KEY` has no default and must fail fast when missing.
- `/health` must never contact Redis; `/ready` must check Redis.
- `/ask` must authenticate before rate limiting and must apply both guards before invoking the mock LLM.
- Redis state must be per-user, bounded, and expiring.
- The runtime container must use a slim multi-stage image and a non-root user.
- No secret value may be committed, copied into the image, or written into documentation.
- CP5 uses `LOCAL_FALLBACK=true`; no public URL or cloud evidence may be invented.
- Existing grading tests and `grade.py` remain unchanged.

## Review Focus

- Missing API key must prevent settings construction and return 401 at the HTTP boundary when the app is configured.
- Two requests with identical timestamps must remain distinct rate-limit entries.
- A Redis outage must produce `/ready` 503 while `/health` remains 200.
- Signal handling must forward to a callable previous handler without calling sentinel handlers such as `SIG_DFL`.
- Docker configuration must interpolate the API key without storing its value in tracked files.

---

### Task 1: Reproducible local Python environment

**Files:**
- Existing local-only: `.env`
- Existing local-only: `.venv/`
- Verify: `requirements.txt`

**Interfaces:**
- Consumes: Python 3.11+ and `requirements.txt`.
- Produces: `.venv\Scripts\python.exe` with all runtime and test packages; local environment values loaded from `.env`.

- [x] **Step 1: Install dependencies with UTF-8 console settings**

```powershell
$env:PYTHONUTF8 = '1'
$env:PYTHONIOENCODING = 'utf-8'
& '.\.venv\Scripts\python.exe' -m pip install -r requirements.txt
```

- [x] **Step 2: Verify critical imports**

```powershell
& '.\.venv\Scripts\python.exe' -c "import fastapi, pydantic_settings, redis, fakeredis, pytest; print('dependencies-ok')"
```

Expected: `dependencies-ok`.

- [x] **Step 3: Verify secret files are ignored**

```powershell
git check-ignore .env .venv
git ls-files .env
```

Expected: `.env` and `.venv` are ignored; `git ls-files .env` prints nothing.

### Task 2: CP1 configuration, structured logging, and liveness

**Files:**
- Modify: `app/config.py:15-45`
- Modify: `app/logging_utils.py:20-37`
- Modify: `app/main.py:76-90`
- Test: `tests/test_cp1.py`

**Interfaces:**
- Produces: `Settings`, `get_settings()`, `log_event(event, level="info", **fields) -> str`, and `health()`.
- Consumes: `lifecycle.shutting_down`, `SERVICE_NAME`, and `SERVICE_VERSION`.

- [x] **Step 1: Run CP1 to establish the failing baseline**

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/test_cp1.py -v
```

Expected: failures for missing settings fields, `log_event`, and `/health`.

- [x] **Step 2: Add typed settings with a required secret**

```python
port: int = 8000
agent_api_key: str
redis_url: str = "redis://localhost:6379/0"
rate_limit_per_minute: int = 10
monthly_budget_usd: float = 10.0
log_level: str = "INFO"
```

- [x] **Step 3: Implement one-line JSON logging**

```python
payload = {"event": event, "level": level.lower(), "timestamp": utc_now_iso(), **fields}
line = json.dumps(payload, ensure_ascii=False)
print(line, file=sys.stdout)
return line
```

- [x] **Step 4: Implement dependency-free liveness**

```python
if lifecycle.shutting_down:
    return JSONResponse(status_code=503, content={"status": "shutting_down"})
return {"status": "ok", "service": SERVICE_NAME, "version": SERVICE_VERSION}
```

- [x] **Step 5: Run CP1 and verify all tests pass**

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/test_cp1.py -v
```

Expected: 13 passed.

- [x] **Step 6: Commit CP1**

```powershell
git add app/config.py app/logging_utils.py app/main.py
git commit -m "feat: complete configuration health and logging"
```

### Task 3: CP3 authentication, rate limiting, budget guard, and ask pipeline

**Files:**
- Modify: `app/auth.py:18-37`
- Modify: `app/rate_limiter.py:30-59`
- Modify: `app/cost_guard.py:32-63`
- Modify: `app/main.py:111-148`
- Test: `tests/test_cp3.py`

**Interfaces:**
- Produces: `verify_api_key() -> str`, `RateLimiter.check()`, `CostGuard.check()`, `CostGuard.record()`, and the protected `/ask` endpoint.
- Consumes: `Settings.agent_api_key`, Redis string/ZSET operations, `ConversationStore`, `ask_llm`, and `log_event`.

- [x] **Step 1: Run CP3 to establish the failing baseline**

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/test_cp3.py -v
```

- [x] **Step 2: Implement constant-time authentication**

```python
expected = get_settings().agent_api_key
if x_api_key is None or not secrets.compare_digest(x_api_key, expected):
    raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="invalid or missing API key")
return x_user_id or ANONYMOUS_USER
```

- [x] **Step 3: Implement the Redis sliding window**

```python
now = now if now is not None else time.time()
key = self._key(user_id)
self.client.zremrangebyscore(key, 0, now - WINDOW_SECONDS)
count = int(self.client.zcard(key))
if count >= self.limit:
    raise HTTPException(status_code=429, detail="rate limit exceeded", headers={"Retry-After": str(WINDOW_SECONDS)})
self.client.zadd(key, {f"{now}:{uuid.uuid4().hex}": now})
self.client.expire(key, WINDOW_SECONDS)
```

- [x] **Step 4: Implement monthly spending operations**

```python
value = self.client.get(self._key(user_id, month))
return float(value) if value is not None else 0.0
```

```python
if self.spent(user_id, month) + estimated_cost > self.budget:
    raise HTTPException(status_code=402, detail="monthly budget exceeded")
```

```python
key = self._key(user_id, month)
total = self.client.incrbyfloat(key, cost)
self.client.expire(key, KEY_TTL_SECONDS)
return float(total)
```

- [x] **Step 5: Implement `/ask` in the required guard-before-cost order**

```python
limiter.check(user_id)
guard.check(user_id)
history = store.get_history(user_id)
result = ask_llm(payload.question, history)
store.append(user_id, "user", payload.question)
store.append(user_id, "assistant", result["answer"])
guard.record(user_id, result["cost_usd"])
log_event("ask_completed", user_id=user_id, tokens_in=result["tokens_in"], tokens_out=result["tokens_out"], cost_usd=result["cost_usd"])
return {"answer": result["answer"], "user_id": user_id, "history_length": len(history), "cost_usd": result["cost_usd"], "tokens": {"in": result["tokens_in"], "out": result["tokens_out"]}}
```

- [x] **Step 6: Run CP3 and verify all tests pass**

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/test_cp3.py -v
```

Expected: 22 passed.

- [x] **Step 7: Commit CP3**

```powershell
git add app/auth.py app/rate_limiter.py app/cost_guard.py app/main.py
git commit -m "feat: secure ask requests with quotas"
```

### Task 4: CP4 Redis history, readiness, and graceful shutdown

**Files:**
- Modify: `app/store.py:47-76`
- Modify: `app/lifecycle.py:25-59`
- Modify: `app/main.py:93-105`
- Test: `tests/test_cp4.py`

**Interfaces:**
- Produces: `ConversationStore.ping/append/get_history`, `Lifecycle.install/request_shutdown`, and `ready()`.
- Consumes: redis-py list/TTL operations and the shared `lifecycle` object.

- [x] **Step 1: Run CP4 to establish the failing baseline**

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/test_cp4.py -v
```

- [x] **Step 2: Implement bounded Redis history and resilient ping**

```python
try:
    return bool(self.client.ping())
except Exception:
    return False
```

```python
key = self._key(user_id)
self.client.rpush(key, json.dumps({"role": role, "content": content}, ensure_ascii=False))
self.client.ltrim(key, -HISTORY_MAX_MESSAGES, -1)
self.client.expire(key, HISTORY_TTL_SECONDS)
```

```python
return [json.loads(item) for item in self.client.lrange(self._key(user_id), 0, -1)]
```

- [x] **Step 3: Implement shutdown flagging and handler forwarding**

```python
self.shutting_down = True
previous = self._previous.get(signum)
if callable(previous):
    previous(signum, frame)
```

```python
for sig in (signal.SIGTERM, signal.SIGINT):
    self._previous[sig] = signal.getsignal(sig)
    signal.signal(sig, self.request_shutdown)
```

- [x] **Step 4: Implement readiness semantics**

```python
if lifecycle.shutting_down:
    return JSONResponse(status_code=503, content={"status": "shutting_down"})
if not store.ping():
    return JSONResponse(status_code=503, content={"status": "not ready", "redis": False})
return {"status": "ready", "redis": True}
```

- [x] **Step 5: Run CP4 and the combined Python checkpoints**

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/test_cp1.py tests/test_cp3.py tests/test_cp4.py -v
```

Expected: all selected tests pass.

- [x] **Step 6: Commit CP4**

```powershell
git add app/store.py app/lifecycle.py app/main.py
git commit -m "feat: add Redis state and graceful lifecycle"
```

### Task 5: CP2 production image and Compose stack

**Files:**
- Modify: `Dockerfile`
- Modify: `.dockerignore`
- Modify: `docker-compose.yml`
- Test: `tests/test_cp2.py`

**Interfaces:**
- Produces: a non-root `agent` image and a Compose stack with `agent` plus `redis`.
- Consumes: `$PORT`, `${AGENT_API_KEY}`, Redis hostname `redis`, and `/health`.

- [x] **Step 1: Run CP2 static tests**

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/test_cp2.py -v -m "not docker"
```

- [x] **Step 2: Replace the Dockerfile with a cached multi-stage build**

```dockerfile
FROM python:3.11-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.11-slim AS runtime
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 PORT=8000
RUN useradd --create-home --uid 10001 appuser
WORKDIR /app
COPY --from=builder /install /usr/local
COPY app ./app
COPY utils ./utils
USER appuser
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 CMD python -c "import os,urllib.request; urllib.request.urlopen('http://127.0.0.1:' + os.getenv('PORT','8000') + '/health', timeout=2)"
CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
```

- [x] **Step 3: Exclude secrets, caches, tests, and local artifacts**

```text
.git
.gitignore
.env
.env.*
!.env.example
.venv
__pycache__
*.py[cod]
.pytest_cache
tests
screenshots
docs
```

- [x] **Step 4: Add the Compose agent service**

```yaml
agent:
  build: .
  ports:
    - "8000:8000"
  environment:
    PORT: "8000"
    AGENT_API_KEY: ${AGENT_API_KEY}
    REDIS_URL: redis://redis:6379/0
    RATE_LIMIT_PER_MINUTE: ${RATE_LIMIT_PER_MINUTE:-10}
    MONTHLY_BUDGET_USD: ${MONTHLY_BUDGET_USD:-10.0}
    LOG_LEVEL: ${LOG_LEVEL:-INFO}
  depends_on:
    redis:
      condition: service_healthy
  healthcheck:
    test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=2)"]
    interval: 10s
    timeout: 3s
    retries: 5
```

- [x] **Step 5: Run CP2 static tests, build, and inspect the image user**

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/test_cp2.py -v
docker compose config
docker compose build agent
docker image inspect day12-agent:prod --format '{{.Config.User}}'
```

Expected: tests pass, Compose config resolves, and image user is non-root.

- [x] **Step 6: Commit CP2**

```powershell
git add Dockerfile .dockerignore docker-compose.yml
git commit -m "feat: containerize agent with Redis"
```

### Task 6: CP5 Docker fallback and runtime verification

**Files:**
- Modify local-only: `.env`
- Modify: `DEPLOYMENT.md`
- Create from actual evidence: `screenshots/health.png`
- Create from actual evidence: `screenshots/dashboard.png`
- Test: `tests/test_cp5.py`

**Interfaces:**
- Produces: a healthy local stack at `http://localhost:8000` and truthful local-fallback documentation.
- Consumes: the Compose stack and local API key.

- [x] **Step 1: Enable the documented fallback locally**

Set `LOCAL_FALLBACK=true` in `.env`; do not add `.env` to Git.

- [x] **Step 2: Start the stack and collect actual status output**

```powershell
docker compose up -d --build
docker compose ps
curl.exe -i http://localhost:8000/health
curl.exe -i http://localhost:8000/ready
curl.exe -i -X POST http://localhost:8000/ask -H "Content-Type: application/json" -d '{"question":"Hello"}'
```

Expected: 200 health, 200 readiness, and 401 unauthenticated ask.

- [x] **Step 3: Verify authenticated ask and rate limiting with the local key**

Use the `AGENT_API_KEY` from the local `.env` only in the shell environment and confirm a successful `/ask`, then repeated requests eventually return 429. Do not paste the key into any tracked file.

- [x] **Step 4: Update deployment documentation truthfully**

Record the local fallback platform, `http://localhost:8000`, the environment variable names, actual command output with secrets removed, and the reason cloud deployment was intentionally not used. Do not claim an HTTPS public deployment.

- [x] **Step 5: Capture real local evidence**

Capture the running Docker Compose service status and the actual `/health` response as PNG files. If GUI capture is unavailable, report the limitation instead of fabricating images.

- [x] **Step 6: Run CP5 fallback tests**

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/test_cp5.py -v
```

Expected: local fallback tests pass and CP5 is capped at 9/15 by the rubric.

- [x] **Step 7: Commit truthful CP5 evidence**

```powershell
git add DEPLOYMENT.md screenshots
git commit -m "docs: record local Docker deployment evidence"
```

### Task 7: Reflection answers grounded in observed behavior

**Files:**
- Modify: `exercises.md`

**Interfaces:**
- Consumes: actual logs, Docker image sizes/cache output, runtime requests, and failure observations from Tasks 2-6.
- Produces: ten original, explainable reflection answers.

- [x] **Step 1: Gather factual measurements**

Record a real JSON log line, image size, rebuild cache behavior, request-limit behavior, history-length behavior, and one actual deployment/runtime error. Do not invent measurements.

- [x] **Step 2: Replace all ten answer placeholders**

Answer each question in concise Vietnamese and include the collected measurements where requested.

- [x] **Step 3: Verify placeholder count**

```powershell
rg -n "Câu trả lời của bạn|\.\.\. MB" exercises.md
```

Expected: no matches.

- [x] **Step 4: Commit reflections**

```powershell
git add exercises.md
git commit -m "docs: complete deployment reflections"
```

### Task 8: CI/CD bonus

**Files:**
- Create: `.github/workflows/ci.yml`
- Modify: `README.md`
- Test: `tests/test_bonus_cicd.py`

**Interfaces:**
- Produces: push/PR test execution, Docker build after tests, and a main-branch deploy job gated by test success and GitHub Secrets.
- Consumes: `requirements.txt`, `Dockerfile`, and `secrets.RAILWAY_TOKEN` only at deployment time.

- [x] **Step 1: Run bonus tests to establish the failing baseline**

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/test_bonus_cicd.py -v
```

- [x] **Step 2: Create the workflow**

```yaml
name: CI
on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip
      - run: pip install -r requirements.txt
      - run: pytest tests --ignore=tests/test_cp5.py --ignore=tests/test_bonus_cicd.py -v

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t day12-agent:${{ github.sha }} .

  deploy:
    needs: [test, build]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    env:
      RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
    steps:
      - uses: actions/checkout@v4
      - run: npm install --global @railway/cli
      - if: env.RAILWAY_TOKEN != ''
        run: railway up --detach
```

- [x] **Step 3: Add the workflow status badge**

Add the repository-specific GitHub Actions badge for `.github/workflows/ci.yml` near the top of `README.md`.

- [x] **Step 4: Run bonus tests**

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests/test_bonus_cicd.py -v
```

Expected locally: all structural tests pass. The remote badge-status test can
only report `passing` after the committed workflow has been pushed and run on
the public GitHub repository; a missing remote run must be reported rather than
fabricated. Even 12/13 bonus tests yield enough bonus for the fallback score to
reach the final cap of 100.

- [x] **Step 5: Commit CI/CD**

```powershell
git add .github/workflows/ci.yml README.md
git commit -m "ci: add test build and deploy pipeline"
```

### Task 9: Full scoring and security audit

**Files:**
- Verify all modified files.

**Interfaces:**
- Consumes: all prior tasks.
- Produces: final score report and clean, secret-free repository state.

- [x] **Step 1: Run the complete test suite**

```powershell
& '.\.venv\Scripts\python.exe' -m pytest tests -v
```

- [x] **Step 2: Run the grader with UTF-8 output**

```powershell
$env:PYTHONUTF8 = '1'
$env:PYTHONIOENCODING = 'utf-8'
& '.\.venv\Scripts\python.exe' grade.py
```

Expected: mandatory score 94 with fallback cap, bonus 10, final capped score 100.

- [x] **Step 3: Audit tracked files for secrets and unfinished code**

```powershell
git ls-files | Select-String -Pattern '(^|/)\.env$|\.(pem|key)$'
rg -n "NotImplementedError|TODO \(CP" app Dockerfile docker-compose.yml .dockerignore
git diff --check
git status --short --branch
```

Expected: no tracked secret, no lab stubs, no whitespace errors, and only intentional changes.

- [x] **Step 4: Stop the local stack after verification**

```powershell
docker compose down
```

- [x] **Step 5: Commit any final documentation-only corrections**

```powershell
git add DEPLOYMENT.md exercises.md README.md
git commit -m "docs: finalize lab submission"
```

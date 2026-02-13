# DevOps Through Prompting

> CI/CD pipelines, infrastructure as code, monitoring, and observability — automated through AI collaboration.

← [Prompt Engineering](./prompt-engineering.md) | [Back to Index](./README.md) | [Workflows →](./workflows.md)

---

## The DevOps Prompting Principle

DevOps prompts are uniquely dangerous. A bad component renders wrong. A bad Dockerfile runs wrong. A bad IAM policy **opens your infrastructure to the internet.**

The principle:

> **Every DevOps prompt must include the threat model.**

"Generate a Dockerfile" is incomplete. "Generate a Dockerfile that runs as non-root, uses a minimal base image, and does not bake secrets into the image" is a specification.

---

## Prompting CI/CD Setup

### CI Pipeline Prompt Template

```
Generate a CI pipeline for [CI platform] with these stages:

PROJECT:
- Language: [Python/Node/Rust/Go]
- Framework: [FastAPI/Next.js/Axum]
- Monorepo: [yes/no]
- Test framework: [pytest/jest/cargo test]

STAGES:
1. Lint — [tool: ruff/eslint/clippy]
2. Type check — [mypy/tsc/none]
3. Unit tests — with coverage threshold [X%]
4. Integration tests — requires [postgres/redis] service containers
5. Security scan — [dependency audit: pip-audit/npm audit/cargo audit]
6. Build — [Docker image / static build]
7. Deploy — [staging on PR merge, production on tag]

REQUIREMENTS:
- Cache dependencies between runs
- Fail fast: stop on first stage failure
- Parallel test execution where possible
- Secret injection via CI platform secrets (never in config)
- Artifact: test results, coverage report, built image

SECURITY:
- Pin all action versions by SHA, not tag
- Minimize permissions (least privilege)
- No secrets in logs (mask sensitive values)
- Dependency vulnerability check blocks merge on critical
```

### GitHub Actions Example

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  PYTHON_VERSION: "3.12"
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
      - uses: actions/setup-python@82c7e631bb3cdc910f68e0081d67478d79c6982d # v5.1.0
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - run: pip install ruff
      - run: ruff check .
      - run: ruff format --check .

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      - uses: actions/setup-python@82c7e631bb3cdc910f68e0081d67478d79c6982d
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install mypy
      - run: mypy src/ --strict

  test:
    runs-on: ubuntu-latest
    needs: [lint, typecheck]
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test_db
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      - uses: actions/setup-python@82c7e631bb3cdc910f68e0081d67478d79c6982d
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          cache: pip
      - name: Install dependencies
        run: pip install -r requirements.txt -r requirements-test.txt
      - name: Run tests
        env:
          DATABASE_URL: postgresql+asyncpg://test:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379/0
          JWT_SECRET_KEY: test-secret-not-for-production
        run: |
          pytest tests/ \
            --cov=src \
            --cov-report=xml \
            --cov-report=term-missing \
            --cov-fail-under=80 \
            -x -v
      - name: Upload coverage
        uses: codecov/codecov-action@e28ff129e5465c2c0dcc6f003fc735cb6ae0c673 # v4
        with:
          files: ./coverage.xml
          fail_ci_if_error: true

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      - uses: actions/setup-python@82c7e631bb3cdc910f68e0081d67478d79c6982d
        with:
          python-version: ${{ env.PYTHON_VERSION }}
      - name: Dependency audit
        run: |
          pip install pip-audit
          pip-audit -r requirements.txt --strict

  build:
    runs-on: ubuntu-latest
    needs: [test, security]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
      - uses: docker/login-action@343f7c4344506bcbf9b4de18042ae17996df046d # v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@4a13e500e55cf31b7a5d59a38ab2040ab0f42f56 # v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## Infrastructure as Code

### Dockerfile Prompt

```
Generate a production Dockerfile for a [language/framework] application.

APPLICATION:
- Language: [Python 3.12 / Node 20 / Rust]
- Framework: [FastAPI / Next.js / Axum]
- Build step: [yes/no, describe]
- Static assets: [yes/no]
- Port: [8000]

REQUIREMENTS:
- Multi-stage build (builder + runtime)
- Minimal base image (alpine or distroless)
- Run as non-root user
- No secrets in image
- Layer ordering optimized for cache hits (dependencies before source)
- Health check instruction
- Labels: maintainer, version, description
- .dockerignore file

SECURITY:
- Pin base image by digest
- No unnecessary tools in runtime image (no curl, no bash in distroless)
- Read-only filesystem where possible
- Drop all Linux capabilities
```

### Production Dockerfile (Python/FastAPI)

```dockerfile
# ============================================================
# Stage 1: Builder
# ============================================================
FROM python:3.12-slim AS builder

WORKDIR /build

# Install build dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends gcc libpq-dev && \
    rm -rf /var/lib/apt/lists/*

# Install Python dependencies (cached layer)
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ============================================================
# Stage 2: Runtime
# ============================================================
FROM python:3.12-slim AS runtime

LABEL maintainer="team@example.com"
LABEL version="1.0.0"
LABEL description="Task Management API"

# Install only runtime dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends libpq5 && \
    rm -rf /var/lib/apt/lists/* && \
    # Create non-root user
    groupadd --gid 1001 appuser && \
    useradd --uid 1001 --gid 1001 --shell /bin/false appuser

# Copy installed packages from builder
COPY --from=builder /install /usr/local

WORKDIR /app

# Copy application code
COPY --chown=appuser:appuser src/ ./src/
COPY --chown=appuser:appuser alembic/ ./alembic/
COPY --chown=appuser:appuser alembic.ini .

# Switch to non-root user
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1

EXPOSE 8000

CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

```dockerfile
# .dockerignore
.git
.github
.vscode
__pycache__
*.pyc
.pytest_cache
.mypy_cache
.ruff_cache
.env
.env.*
docker-compose*.yml
Dockerfile*
README.md
docs/
tests/
*.md
```

### Docker Compose for Local Dev

```yaml
# docker-compose.yml
services:
  api:
    build:
      context: .
      target: runtime
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://dev:dev@postgres:5432/app_dev
      REDIS_URL: redis://redis:6379/0
      JWT_SECRET_KEY: local-dev-secret-not-for-production
      ENVIRONMENT: development
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./src:/app/src  # Hot reload in dev
    command: uvicorn src.main:app --host 0.0.0.0 --port 8000 --reload
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "1.0"

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app_dev
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: pg_isready -U dev
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: redis-cli ping
      interval: 5s
      timeout: 3s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 128M

  worker:
    build:
      context: .
      target: runtime
    environment:
      DATABASE_URL: postgresql+asyncpg://dev:dev@postgres:5432/app_dev
      REDIS_URL: redis://redis:6379/0
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    command: arq src.workers.WorkerSettings
    deploy:
      resources:
        limits:
          memory: 256M

volumes:
  postgres_data:
```

---

## Terraform / IaC Prompt

```
Generate Terraform configuration for deploying a container-based API to 
[cloud provider].

ARCHITECTURE:
- Container runtime: [ECS Fargate / Cloud Run / Azure Container Apps]
- Database: [RDS PostgreSQL / Cloud SQL / Azure Database]
- Cache: [ElastiCache Redis / Memorystore / Azure Cache]
- Load balancer: [ALB / Cloud Load Balancer]
- DNS: [Route53 / Cloud DNS]

REQUIREMENTS:
- Separate environments: staging, production
- Auto-scaling: min 2, max 10 instances
- Database: RDS PostgreSQL, encrypted at rest, automated backups
- Secrets: via secret manager (not environment variables in TF state)
- VPC: private subnets for DB/cache, public for ALB
- TLS: ACM certificate with auto-renewal

SECURITY:
- Security groups: principle of least privilege
- IAM roles: task-level permissions, not account-wide
- Encryption: in transit and at rest
- No public access to database
- Audit logging enabled

OUTPUT:
- Modular structure (modules for networking, compute, database, cache)
- Variable definitions with descriptions and validation
- Outputs for important endpoints and ARNs
- Backend configuration for S3 state storage
```

### Terraform Module Structure

```
infra/
├── environments/
│   ├── staging/
│   │   ├── main.tf          # Module calls with staging vars
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf       # S3 backend for staging state
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       └── backend.tf
├── modules/
│   ├── networking/
│   │   ├── main.tf           # VPC, subnets, security groups
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── database/
│   │   ├── main.tf           # RDS, parameter groups
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── cache/
│   │   ├── main.tf           # ElastiCache
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf           # ECS, task definitions, services
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── monitoring/
│       ├── main.tf           # CloudWatch, alerts
│       ├── variables.tf
│       └── outputs.tf
└── shared/
    └── backend.tf            # S3 bucket for TF state
```

---

## Deployment Pipelines

### Blue-Green Deployment Prompt

```
Design a blue-green deployment strategy for [platform] with:

REQUIREMENTS:
- Zero-downtime deployments
- Automated rollback on health check failure
- Database migration strategy (forward-compatible)
- Smoke test suite runs against new deployment before traffic switch
- Traffic switch is gradual (canary: 10% → 50% → 100%)

CONSTRAINTS:
- Database migrations must be backward-compatible (no dropping columns)
- WebSocket connections must drain gracefully
- Background job workers must finish current jobs before shutdown

PROVIDE:
1. Deployment flow diagram
2. Health check endpoint specification
3. Rollback trigger conditions
4. Migration strategy for breaking schema changes
```

### Deployment Flow

```mermaid
sequenceDiagram
    participant D as Developer
    participant CI as CI Pipeline
    participant R as Registry
    participant S as Staging
    participant C as Canary (10%)
    participant P as Production (100%)

    D->>CI: Push tag v1.2.0
    CI->>CI: Run tests, lint, security scan
    CI->>R: Push image v1.2.0
    CI->>S: Deploy to staging
    S->>S: Run smoke tests
    S-->>CI: ✅ Staging healthy
    
    CI->>C: Deploy canary (10% traffic)
    C->>C: Monitor error rate, latency, logs
    
    Note over C: 15 min observation window
    
    alt Error rate > 1% or p99 > 500ms
        C-->>CI: ❌ Canary failed
        CI->>C: Rollback canary
        CI-->>D: Alert: deployment rolled back
    else Metrics healthy
        C-->>CI: ✅ Canary healthy
        CI->>P: Promote to 50%
        Note over P: 10 min observation
        CI->>P: Promote to 100%
        CI-->>D: ✅ Deployment complete
    end
```

---

## Monitoring and Logging

### Monitoring Setup Prompt

```
Design a monitoring and alerting strategy for a [type] application.

STACK:
- Application: [FastAPI / Next.js]
- Database: [PostgreSQL]
- Cache: [Redis]
- Queue: [Redis / RabbitMQ / SQS]
- Container orchestration: [ECS / Kubernetes]

FOUR GOLDEN SIGNALS:
1. Latency — request duration (p50, p95, p99)
2. Traffic — requests per second by endpoint
3. Errors — error rate by status code and endpoint
4. Saturation — CPU, memory, DB connections, queue depth

CRITICAL ALERTS (page someone):
- Error rate > 5% for 5 minutes
- p99 latency > 2s for 5 minutes
- Database connection pool > 80% utilized
- Disk usage > 85%
- Health check failing for > 1 minute
- Certificate expiring within 14 days

WARNING ALERTS (Slack/email):
- Error rate > 1% for 15 minutes
- p95 latency > 1s for 15 minutes
- Memory usage > 70%
- Queue depth growing for > 10 minutes
- Background job failure rate > 5%

PROVIDE:
1. Prometheus metrics to expose from the application
2. Grafana dashboard JSON (key panels)
3. Alert rules configuration
4. Structured logging format with correlation IDs
```

### Structured Logging Standard

```python
# logging_config.py
import structlog
import logging

def setup_logging(environment: str = "production"):
    """Configure structured JSON logging."""
    
    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.StackInfoRenderer(),
            structlog.dev.set_exc_info,
            structlog.processors.TimeStamper(fmt="iso"),
            # Development: human-readable. Production: JSON.
            structlog.dev.ConsoleRenderer()
            if environment == "development"
            else structlog.processors.JSONRenderer(),
        ],
        wrapper_class=structlog.make_filtering_bound_logger(logging.INFO),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(),
        cache_logger_on_first_use=True,
    )

# Usage in middleware
@app.middleware("http")
async def logging_middleware(request: Request, call_next):
    request_id = request.headers.get("X-Request-ID", str(uuid4()))
    
    structlog.contextvars.clear_contextvars()
    structlog.contextvars.bind_contextvars(
        request_id=request_id,
        method=request.method,
        path=request.url.path,
        user_id=getattr(request.state, "user_id", None),
    )
    
    logger = structlog.get_logger()
    start_time = time.monotonic()
    
    try:
        response = await call_next(request)
        duration_ms = (time.monotonic() - start_time) * 1000
        
        logger.info(
            "request_completed",
            status_code=response.status_code,
            duration_ms=round(duration_ms, 2),
        )
        
        response.headers["X-Request-ID"] = request_id
        return response
        
    except Exception as exc:
        duration_ms = (time.monotonic() - start_time) * 1000
        logger.error(
            "request_failed",
            error=str(exc),
            error_type=type(exc).__name__,
            duration_ms=round(duration_ms, 2),
        )
        raise
```

```json
// Example structured log output
{
  "event": "request_completed",
  "request_id": "req_abc123",
  "method": "GET",
  "path": "/api/v1/tasks",
  "user_id": "usr_def456",
  "status_code": 200,
  "duration_ms": 42.7,
  "timestamp": "2026-02-13T06:00:00Z",
  "level": "info"
}
```

### Application Metrics (Prometheus)

```python
# metrics.py
from prometheus_client import Counter, Histogram, Gauge, Info

# Request metrics
REQUEST_COUNT = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status_code"],
)

REQUEST_DURATION = Histogram(
    "http_request_duration_seconds",
    "HTTP request duration in seconds",
    ["method", "endpoint"],
    buckets=[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0],
)

# Database metrics
DB_POOL_SIZE = Gauge(
    "db_pool_size",
    "Database connection pool size",
    ["state"],  # active, idle, overflow
)

DB_QUERY_DURATION = Histogram(
    "db_query_duration_seconds",
    "Database query duration",
    ["operation", "table"],
    buckets=[0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1.0],
)

# Background job metrics
JOB_COUNT = Counter(
    "background_jobs_total",
    "Total background jobs processed",
    ["job_type", "status"],  # status: success, failure, retry
)

JOB_DURATION = Histogram(
    "background_job_duration_seconds",
    "Background job processing time",
    ["job_type"],
)

QUEUE_DEPTH = Gauge(
    "job_queue_depth",
    "Current number of jobs in queue",
    ["queue_name"],
)

# App info
APP_INFO = Info("app", "Application metadata")
APP_INFO.info({
    "version": os.environ.get("APP_VERSION", "unknown"),
    "environment": os.environ.get("ENVIRONMENT", "unknown"),
})
```

---

## Scaling Infrastructure

### Scaling Decision Prompt

```
My application is hitting scaling limits. Help me identify the bottleneck 
and plan the scaling path.

CURRENT METRICS:
- Requests/sec: [X]
- p99 latency: [X ms]
- CPU utilization: [X%] on [N] instances
- Memory utilization: [X%]
- Database connections: [X/Y pool]
- Database CPU: [X%]
- Cache hit rate: [X%]
- Queue depth: [X messages, processing rate Y/s]

BOTTLENECK SYMPTOMS:
[Describe what you're observing — timeouts, 503s, slow queries, etc.]

CONSTRAINTS:
- Budget ceiling: $[X]/month
- Must maintain current uptime SLA
- Team cannot manage Kubernetes
- Migration window: [max downtime acceptable]

FOR EACH RECOMMENDATION:
1. What changes
2. Expected improvement
3. Implementation effort (hours/days)
4. Risk during migration
5. Monthly cost impact
```

---

## Observability Stack

```
┌───────────────────────────────────────────────────────────┐
│                    Observability Stack                      │
├───────────────────────────────────────────────────────────┤
│                                                            │
│  METRICS (Prometheus/Grafana)                             │
│  ├── Application: request rate, latency, error rate       │
│  ├── Database: query duration, pool usage, replication    │
│  ├── Cache: hit rate, memory, evictions                   │
│  ├── Queue: depth, processing rate, dead letters          │
│  └── Infrastructure: CPU, memory, disk, network           │
│                                                            │
│  LOGS (Structured JSON → ELK/Loki)                        │
│  ├── Application logs with request_id correlation         │
│  ├── Access logs (method, path, status, duration)         │
│  ├── Error logs with stack traces (no PII)                │
│  └── Audit logs (who did what when)                       │
│                                                            │
│  TRACES (OpenTelemetry → Jaeger/Tempo)                    │
│  ├── Request → API → Service → DB → Cache                 │
│  ├── Background job execution traces                      │
│  └── Cross-service traces (if microservices)              │
│                                                            │
│  ALERTS (PagerDuty/OpsGenie/Slack)                        │
│  ├── Critical: pages on-call                              │
│  ├── Warning: Slack notification                          │
│  └── Info: dashboard annotation                           │
│                                                            │
└───────────────────────────────────────────────────────────┘
```

---

## Common Failure Modes

| Failure | Symptom | Root Cause |
|---------|---------|------------|
| **Secrets in config** | API keys in terraform.tfvars or docker-compose.yml | Prompt didn't specify secrets manager. Model defaults to env vars in config |
| **Unpinned dependencies** | Build breaks after unchanged commit | `FROM python:3.12` changes under you. Always pin by digest or version |
| **No health checks** | Broken containers serve traffic | Dockerfile and orchestrator missing health check config |
| **Alerting too broad** | Every alert is critical, team ignores all | No severity tiers. Specify critical (page) vs warning (Slack) |
| **Log noise** | Millions of INFO lines, debugging impossible | No structured logging, no log levels. Model defaults to `print()` |
| **Missing rollback** | Bad deploy requires manual intervention | Pipeline has deploy but no automated rollback trigger |
| **Over-provisioned infra** | $3K/month for 50 users | Prompted for "production-ready" without budget constraint |
| **No resource limits** | Container OOMs, takes down host | Docker/K8s config missing memory/CPU limits |

---

## Production Checklist

- [ ] CI pipeline: lint, test, security scan, build on every PR
- [ ] Docker image: multi-stage, non-root, pinned base, no secrets
- [ ] Secrets: managed via secret manager, not environment files
- [ ] Infrastructure: defined as code (Terraform/Pulumi), version controlled
- [ ] Environments: staging mirrors production configuration
- [ ] Deployment: zero-downtime with automated rollback
- [ ] Database backups: automated, tested, encrypted
- [ ] Monitoring: four golden signals (latency, traffic, errors, saturation)
- [ ] Logging: structured JSON with request ID correlation
- [ ] Alerting: severity tiers with appropriate routing
- [ ] Scaling: auto-scaling configured with appropriate min/max
- [ ] Resource limits: CPU and memory limits on all containers
- [ ] TLS: enforced everywhere, certificates auto-renewed
- [ ] Disaster recovery: runbook documented and tested
- [ ] On-call rotation: documented escalation process

---

← [Prompt Engineering](./prompt-engineering.md) | [Back to Index](./README.md) | [Workflows →](./workflows.md)

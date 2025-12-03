---
name: ADK Deployment
description: Deployment patterns for Google ADK agents including local development, Cloud Run, Vertex AI Agent Engine, and GKE. Covers observability, tracing, and production best practices.
version: 1.0.0
---

# ADK Deployment Patterns (December 2025)

## Overview

ADK supports multiple deployment targets:
| Platform | Use Case | Scaling | Management |
|----------|----------|---------|------------|
| Local | Development/testing | Manual | Full control |
| Cloud Run | Production serverless | Auto | Managed |
| Vertex AI Agent Engine | Enterprise | Auto | Fully managed |
| GKE | Custom requirements | Custom | More control |

## Local Development

### ADK CLI

```bash
# Install ADK with CLI
pip install google-adk

# Start local web interface
adk web ~/path/to/agent/

# With specific port
adk web --port 8080 ~/path/to/agent/
```

### Custom FastAPI Server

From ss-agent pattern:

```python
# server.py
from fastapi import FastAPI, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from google.adk.runners import Runner
from google.adk.sessions import InMemorySessionService
from .agent import root_agent
import os

app = FastAPI(title="SoundSage Agent API")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["*"],
    allow_headers=["*"],
)

session_service = InMemorySessionService()
runner = Runner(agent=root_agent, session_service=session_service)

@app.get("/health")
async def health():
    return {"status": "healthy", "agent": root_agent.name}

@app.post("/api/mix")
async def mix_endpoint(request: MixRequest):
    try:
        session = session_service.get_or_create_session(
            app_name="soundsage",
            user_id=request.user_id or "default",
            session_id=request.session_id
        )

        # Inject state into session
        session.state.update(request.state or {})

        result = await runner.run_async(request.message, session=session)

        return {
            "success": True,
            "response": result.content,
            "actions": extract_actions(result)
        }
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(
        "server:app",
        host=os.getenv("HOST", "0.0.0.0"),
        port=int(os.getenv("PORT", 8000)),
        reload=True
    )
```

### Environment Setup

```bash
# .env file
GOOGLE_API_KEY=your-api-key  # For Google AI Studio
# OR for Vertex AI:
GOOGLE_CLOUD_PROJECT=my-project
GOOGLE_CLOUD_LOCATION=us-central1

PORT=8000
HOST=0.0.0.0
```

## Cloud Run Deployment

### Single Command Deploy

```bash
adk deploy cloud_run \
  --region us-central1 \
  --service_name soundsage-agent \
  --with_ui \
  ~/path/to/agent/
```

### CLI Options

| Option | Description | Default |
|--------|-------------|---------|
| `--region` | GCP region (required) | - |
| `--service_name` | Cloud Run service name | 'adk-default-service-name' |
| `--with_ui` | Deploy ADK web UI | false |
| `--port` | Container port | 8000 |
| `--trace_to_cloud` | Enable Cloud Trace | false |
| `--agent_engine_id` | Use Agent Engine sessions | - |

### What ADK CLI Does

1. Generates Dockerfile automatically
2. Builds container image
3. Pushes to Artifact Registry
4. Deploys to Cloud Run
5. Configures auto-scaling

### Custom Dockerfile

For more control:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PORT=8000
EXPOSE 8000

CMD ["uvicorn", "ss_agent.server:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Manual Deploy

```bash
# Build and push image
gcloud builds submit --tag gcr.io/PROJECT_ID/soundsage-agent

# Deploy to Cloud Run
gcloud run deploy soundsage-agent \
  --image gcr.io/PROJECT_ID/soundsage-agent \
  --region us-central1 \
  --allow-unauthenticated \
  --memory 2Gi \
  --cpu 2 \
  --min-instances 0 \
  --max-instances 10 \
  --set-env-vars "GOOGLE_API_KEY=your-key"
```

## Vertex AI Agent Engine

### Single Command Deploy

```bash
adk deploy agent_engine \
  --project my-project \
  --region us-central1 \
  --staging_bucket gs://my-bucket/staging \
  --enable_tracing \
  --display_name "SoundSage Agent" \
  ~/path/to/agent/
```

### Prerequisites

```bash
# Install Agent Engine dependencies
pip install google-cloud-aiplatform[adk,agent_engines]>=1.111

# Authenticate
gcloud auth application-default login
```

### CLI Options

| Option | Description | Required |
|--------|-------------|----------|
| `--project` | GCP project ID | Yes |
| `--region` | Deployment region | Yes |
| `--staging_bucket` | GCS bucket for artifacts | Yes |
| `--enable_tracing` | Enable Cloud Trace | No |
| `--display_name` | Agent display name | No |
| `--description` | Agent description | No |

### Agent Engine Benefits

- Fully managed runtime
- No infrastructure management
- Built-in scaling and security
- Session management included
- Framework-agnostic
- Model-agnostic
- Express mode (free trial, no GCP project)

### Programmatic Deploy

```python
from google.cloud import aiplatform

aiplatform.init(project="my-project", location="us-central1")

agent = aiplatform.Agent.create(
    display_name="SoundSage Agent",
    staging_bucket="gs://my-bucket/staging",
    agent_path="~/path/to/agent/",
    enable_tracing=True
)

print(f"Agent deployed: {agent.resource_name}")
```

## GKE Deployment

### ADK CLI Deploy

```bash
adk deploy gke \
  --project myproject \
  --cluster_name production \
  --region us-central1 \
  --with_ui \
  --log_level info \
  ~/path/to/agent/
```

### Custom Kubernetes Manifest

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: soundsage-agent
spec:
  replicas: 3
  selector:
    matchLabels:
      app: soundsage-agent
  template:
    metadata:
      labels:
        app: soundsage-agent
    spec:
      containers:
      - name: agent
        image: gcr.io/PROJECT_ID/soundsage-agent:latest
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "2000m"
        env:
        - name: GOOGLE_API_KEY
          valueFrom:
            secretKeyRef:
              name: agent-secrets
              key: api-key
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: soundsage-agent
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8000
  selector:
    app: soundsage-agent
```

## Observability

### OpenTelemetry Integration

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter

# Configure tracing
trace.set_tracer_provider(TracerProvider())
tracer_provider = trace.get_tracer_provider()

# Export to Cloud Trace
cloud_trace_exporter = CloudTraceSpanExporter(project_id="my-project")
tracer_provider.add_span_processor(BatchSpanProcessor(cloud_trace_exporter))

tracer = trace.get_tracer(__name__)

# Custom instrumentation
@tracer.start_as_current_span("process_mix_request")
async def process_request(request):
    with tracer.start_as_current_span("run_agent"):
        result = await runner.run_async(request.message)
    return result
```

### Cloud Trace with ADK CLI

```bash
# Cloud Run with tracing
adk deploy cloud_run --trace_to_cloud --region us-central1 ~/agent/

# Agent Engine with tracing
adk deploy agent_engine --enable_tracing --project my-project ~/agent/
```

### Logging

```python
import logging
from google.cloud import logging as cloud_logging

# Configure Cloud Logging
client = cloud_logging.Client()
client.setup_logging()

logger = logging.getLogger(__name__)

@app.post("/api/mix")
async def mix_endpoint(request: MixRequest):
    logger.info(f"Processing request: {request.message[:50]}...")

    try:
        result = await runner.run_async(request.message, session=session)
        logger.info(f"Request completed successfully")
        return {"success": True, "response": result.content}
    except Exception as e:
        logger.error(f"Request failed: {e}", exc_info=True)
        raise
```

### Third-Party Observability

**AgentOps:**
```python
import agentops
agentops.init(api_key="your-key")
# ADK automatically tracked
```

**Phoenix:**
```python
import phoenix as px
px.launch_app()
# Traces sent to Phoenix
```

## Production Best Practices

### 1. Environment Configuration

```python
# config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    google_api_key: str = ""
    google_cloud_project: str = ""
    google_cloud_location: str = "us-central1"
    port: int = 8000
    environment: str = "development"
    log_level: str = "INFO"

    class Config:
        env_file = ".env"

settings = Settings()
```

### 2. Session Service Selection

```python
def get_session_service():
    """Select session service based on environment."""
    env = os.getenv("ENVIRONMENT", "development")

    if env == "development":
        return InMemorySessionService()
    elif env == "production":
        return DatabaseSessionService(
            connection_string=os.getenv("DATABASE_URL")
        )
    elif env == "enterprise":
        return VertexAISessionService(
            project_id=os.getenv("GOOGLE_CLOUD_PROJECT"),
            location=os.getenv("GOOGLE_CLOUD_LOCATION")
        )
```

### 3. Health Checks

```python
@app.get("/health")
async def health():
    return {
        "status": "healthy",
        "agent": root_agent.name,
        "version": os.getenv("VERSION", "unknown"),
        "environment": os.getenv("ENVIRONMENT", "unknown")
    }

@app.get("/ready")
async def ready():
    # Check dependencies
    try:
        # Verify session service
        session_service.list_sessions(app_name="healthcheck", user_id="healthcheck")
        return {"status": "ready"}
    except Exception as e:
        raise HTTPException(status_code=503, detail=str(e))
```

### 4. Rate Limiting

```python
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

@app.post("/api/mix")
@limiter.limit("10/minute")
async def mix_endpoint(request: Request, data: MixRequest):
    ...
```

### 5. Security

```python
# Use Secret Manager for credentials
from google.cloud import secretmanager

def get_secret(secret_id: str) -> str:
    client = secretmanager.SecretManagerServiceClient()
    name = f"projects/{PROJECT_ID}/secrets/{secret_id}/versions/latest"
    response = client.access_secret_version(request={"name": name})
    return response.payload.data.decode("UTF-8")

# Never hardcode credentials
api_key = get_secret("google-api-key")
```

### 6. Graceful Shutdown

```python
import signal
import asyncio

async def shutdown(signal, loop):
    """Cleanup on shutdown."""
    logging.info(f"Received {signal.name}...")

    # Complete in-flight requests
    tasks = [t for t in asyncio.all_tasks() if t is not asyncio.current_task()]
    [task.cancel() for task in tasks]
    await asyncio.gather(*tasks, return_exceptions=True)

    loop.stop()

# Register handlers
for sig in (signal.SIGTERM, signal.SIGINT):
    asyncio.get_event_loop().add_signal_handler(
        sig, lambda s=sig: asyncio.create_task(shutdown(s, asyncio.get_event_loop()))
    )
```

## Deployment Checklist

- [ ] Environment variables configured
- [ ] Secrets in Secret Manager (not code)
- [ ] Health and readiness endpoints
- [ ] Logging configured for Cloud Logging
- [ ] Tracing enabled (Cloud Trace or third-party)
- [ ] Rate limiting configured
- [ ] Resource limits set (memory, CPU)
- [ ] Auto-scaling configured
- [ ] Error handling and retries
- [ ] Graceful shutdown handling
- [ ] CI/CD pipeline configured
- [ ] Monitoring alerts set up

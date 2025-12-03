---
name: deploy-agent
description: Deploy an ADK agent to Cloud Run, Vertex AI Agent Engine, or GKE
allowed_tools: [Read, Write, Edit, Glob, Bash]
---

# Deploy ADK Agent

You are deploying a Google ADK agent to production. Reference the deployment patterns and the project at `/Users/fiorante/Documents/sound-sage-finalboss/ss-agent`.

## Gather Information

Ask the user for:
1. **Agent Path** - Path to the agent directory
2. **Deployment Target**:
   - `cloud-run` - Serverless, auto-scaling (recommended)
   - `agent-engine` - Fully managed Vertex AI
   - `gke` - Kubernetes for advanced needs
3. **Region** - GCP region (default: us-central1)
4. **Service Name** - Name for the deployed service
5. **Options**:
   - Enable tracing?
   - Include web UI?
   - Min/max instances?

## Pre-Deployment Checklist

Before deploying, verify:

- [ ] `GOOGLE_CLOUD_PROJECT` environment variable set
- [ ] `gcloud auth login` completed
- [ ] Agent runs locally (`adk web .`)
- [ ] Tests pass (`pytest tests/`)
- [ ] Secrets configured in Secret Manager
- [ ] requirements.txt or pyproject.toml up to date

## Cloud Run Deployment

### Quick Deploy (Recommended)
```bash
# Set project
gcloud config set project YOUR_PROJECT_ID

# Deploy with ADK CLI
adk deploy cloud_run \
  --region us-central1 \
  --service_name {service-name} \
  --with_ui \
  --trace_to_cloud \
  {agent-path}/
```

### Custom Dockerfile Deploy
```bash
# Create Dockerfile if not exists
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PORT=8000
EXPOSE 8000

CMD ["uvicorn", "{package}.server:app", "--host", "0.0.0.0", "--port", "8000"]
EOF

# Build and deploy
gcloud builds submit --tag gcr.io/$PROJECT_ID/{service-name}

gcloud run deploy {service-name} \
  --image gcr.io/$PROJECT_ID/{service-name} \
  --region us-central1 \
  --allow-unauthenticated \
  --memory 2Gi \
  --cpu 2 \
  --min-instances 1 \
  --max-instances 10 \
  --set-secrets "GOOGLE_API_KEY=google-api-key:latest"
```

## Vertex AI Agent Engine Deployment

```bash
# Install Agent Engine dependencies
pip install google-cloud-aiplatform[adk,agent_engines]>=1.111

# Deploy
adk deploy agent_engine \
  --project $PROJECT_ID \
  --region us-central1 \
  --staging_bucket gs://{bucket}/staging \
  --enable_tracing \
  --display_name "{Agent Display Name}" \
  {agent-path}/
```

## GKE Deployment

```bash
# Deploy to existing cluster
adk deploy gke \
  --project $PROJECT_ID \
  --cluster_name {cluster-name} \
  --region us-central1 \
  --with_ui \
  {agent-path}/
```

## Post-Deployment Verification

```bash
# 1. Check service status
gcloud run services describe {service-name} --region us-central1

# 2. Get service URL
SERVICE_URL=$(gcloud run services describe {service-name} --region us-central1 --format 'value(status.url)')

# 3. Test health endpoint
curl $SERVICE_URL/health

# 4. Test agent endpoint
curl -X POST $SERVICE_URL/api/run \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello, are you working?"}'

# 5. Check logs
gcloud run services logs read {service-name} --region us-central1 --limit 50

# 6. Check traces (if enabled)
gcloud trace traces list --project $PROJECT_ID --limit 10
```

## Environment Variables

Set these secrets in Secret Manager:
```bash
# Create secret for API key
echo -n "your-api-key" | gcloud secrets create google-api-key --data-file=-

# Grant access to Cloud Run service account
gcloud secrets add-iam-policy-binding google-api-key \
  --member="serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

## Monitoring Setup

```bash
# Create uptime check
gcloud monitoring uptime-check-configs create {service-name}-health \
  --display-name="{Service Name} Health Check" \
  --resource-type=uptime-url \
  --resource-labels=host=$SERVICE_URL \
  --path=/health \
  --period=60s

# Create alert for errors
gcloud beta monitoring policies create \
  --display-name="{Service Name} Error Rate Alert" \
  --condition-display-name="Error rate > 5%" \
  --condition-filter='resource.type="cloud_run_revision" AND resource.labels.service_name="{service-name}" AND metric.type="run.googleapis.com/request_count" AND metric.labels.response_code_class!="2xx"' \
  --condition-threshold-value=0.05 \
  --notification-channels=CHANNEL_ID
```

## Rollback Procedure

```bash
# List revisions
gcloud run revisions list --service {service-name} --region us-central1

# Rollback to previous revision
gcloud run services update-traffic {service-name} \
  --region us-central1 \
  --to-revisions {previous-revision}=100
```

## CI/CD Integration

Create `.github/workflows/deploy.yml`:
```yaml
name: Deploy Agent

on:
  push:
    branches: [main]
    paths: ['{agent-path}/**']

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: google-github-actions/auth@v2
      with:
        credentials_json: ${{ secrets.GCP_SA_KEY }}
    - uses: google-github-actions/setup-gcloud@v2

    - name: Deploy to Cloud Run
      run: |
        gcloud builds submit --tag gcr.io/$PROJECT_ID/{service-name} ./{agent-path}
        gcloud run deploy {service-name} \
          --image gcr.io/$PROJECT_ID/{service-name} \
          --region us-central1
```

## Output

After deployment, provide:
1. **Service URL** - Endpoint for API calls
2. **Health Check URL** - For monitoring
3. **Logs Command** - How to view logs
4. **Rollback Command** - How to revert if needed
5. **Monitoring Dashboard** - Link to Cloud Console

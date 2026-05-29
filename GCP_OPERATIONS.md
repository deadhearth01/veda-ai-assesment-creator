# GCP Operations — VedaAI on Cloud Run

Operational runbook: redeploy after code changes, pause/resume to save cost, and tear down.

## Project facts

| Thing | Value |
|-------|-------|
| Project ID | `gen-lang-client-0648151309` |
| Region | `asia-south1` (Mumbai) |
| Cloud Run services | `vedaai-web`, `vedaai-api`, `vedaai-worker` |
| Artifact Registry | `asia-south1-docker.pkg.dev/gen-lang-client-0648151309/vedaai` |
| Redis (Memorystore) | `vedaai-redis` (Basic, 1 GB) |
| VPC connector | `vedaai-conn` |
| Secrets (Secret Manager) | `MONGODB_URI`, `REDIS_URL`, `GEMINI_API_KEY` |
| Live web | https://vedaai-web-439420707140.asia-south1.run.app |
| Live api | https://vedaai-api-439420707140.asia-south1.run.app |

Set once per shell:
```bash
export PROJECT_ID=gen-lang-client-0648151309
export REGION=asia-south1
export REPO=$REGION-docker.pkg.dev/$PROJECT_ID/vedaai
gcloud config set project $PROJECT_ID
```

---

## 1. Redeploy after code changes

`vedaai-api` and `vedaai-worker` share ONE image (`vedaai/api`). `vedaai-web` is its own image.

### Changed backend (`apps/api/**` or `packages/shared/**`)
```bash
# Build + push the api image
gcloud builds submit --config cloudbuild-api.yaml .
```
`cloudbuild-api.yaml` builds, pushes, and rolls out BOTH `vedaai-api` and `vedaai-worker`
automatically (its deploy steps redeploy each). Nothing else needed.

### Changed frontend (`apps/web/**` or `packages/shared/**`)
```bash
# Builds with the api URL baked in (NEXT_PUBLIC_*), pushes, deploys vedaai-web
gcloud builds submit --config cloudbuild-web.yaml .
```
If the api URL ever changes, pass it:
```bash
gcloud builds submit --config cloudbuild-web.yaml \
  --substitutions=_API_URL=https://NEW-API-URL.run.app .
```

### Manual single-service redeploy (image already built)
```bash
gcloud run deploy vedaai-api    --image $REPO/api:latest --region $REGION
gcloud run deploy vedaai-worker --image $REPO/api:latest --region $REGION \
  --command node --args dist/worker.js
gcloud run deploy vedaai-web    --image $REPO/web:latest --region $REGION
```

---

## 2. Pause (stop billing for compute)

`vedaai-api` and `vedaai-worker` run `--min-instances 1` (always warm) → they bill 24/7.
Set min to 0 to stop the warm instances.

```bash
gcloud run services update vedaai-api    --region $REGION --min-instances 0
gcloud run services update vedaai-worker --region $REGION --min-instances 0
```

What this does:
- **api** → still works, but cold-starts on first request (a few seconds of delay). Fine.
- **worker** → scales to zero. **Generation jobs will NOT process** while paused, because
  nothing is draining the BullMQ queue. Jobs sit in Redis until the worker comes back.
- **web** → already `min-instances 0`; nothing to change.

`vedaai-redis` and `vedaai-conn` keep billing even when services are paused (they are
separate always-on resources). To stop ALL cost, see section 4 (delete).

---

## 3. Resume

```bash
gcloud run services update vedaai-api    --region $REGION --min-instances 1
gcloud run services update vedaai-worker --region $REGION --min-instances 1 --no-cpu-throttling
```
Worker picks up any queued jobs once it boots. App fully live again.

---

## 4. Delete / tear down (stop ALL billing)

Order: services first, then Redis + connector (the costly always-on pieces).

```bash
# Cloud Run services
gcloud run services delete vedaai-web    --region $REGION --quiet
gcloud run services delete vedaai-api    --region $REGION --quiet
gcloud run services delete vedaai-worker --region $REGION --quiet

# Memorystore Redis (~biggest cost) + VPC connector
gcloud redis instances delete vedaai-redis --region $REGION --quiet
gcloud compute networks vpc-access connectors delete vedaai-conn --region $REGION --quiet

# Optional: images + secrets (cheap, can keep)
gcloud artifacts repositories delete vedaai --location $REGION --quiet
for S in MONGODB_URI REDIS_URL GEMINI_API_KEY; do gcloud secrets delete $S --quiet; done
```

MongoDB Atlas is NOT on GCP — delete/pause it from the Atlas dashboard separately.

---

## 5. Rebuild after a full delete

If everything was deleted, recreate from scratch — see `DEPLOY.md` (full step-by-step:
APIs, Artifact Registry, secrets, Redis, VPC connector, build, deploy, CORS).

---

## 6. Status & logs

```bash
# What's running + which image + min instances
gcloud run services list --region $REGION

# Health
curl https://vedaai-api-439420707140.asia-south1.run.app/health

# Logs (last 50)
gcloud run services logs read vedaai-api    --region $REGION --limit 50
gcloud run services logs read vedaai-worker --region $REGION --limit 50

# Redis state
gcloud redis instances describe vedaai-redis --region $REGION --format='value(state,host)'
```

---

## 7. Update a secret (e.g. rotate a key)

```bash
printf '%s' 'NEW_VALUE' | gcloud secrets versions add GEMINI_API_KEY --data-file=-
# then redeploy so the service picks up :latest
gcloud run deploy vedaai-api    --image $REPO/api:latest --region $REGION
gcloud run deploy vedaai-worker --image $REPO/api:latest --region $REGION \
  --command node --args dist/worker.js
```

---

## Cost notes

- Always-on cost = `vedaai-api` (min 1) + `vedaai-worker` (min 1, no-cpu-throttle) + Redis + VPC connector ≈ a few $/day.
- `vedaai-web` and Artifact Registry / Secret Manager are ~free.
- Demo for 10–15 days: pause (section 2) or delete (section 4) when done.

# LUXE — Full GCP Setup, 100% from the CLI

This is the same deployment as `commands.md`, but with **zero clicking in the
Google Cloud Console** and **no GitHub trigger** — every single step,
including creating the project itself, is one `gcloud` command. Builds are
submitted manually from your own machine (`gcloud builds submit`), so there's
nothing connected to GitHub at all.

**The one exception**, called out where it happens: the Google OAuth Client
ID for "Sign in with Google" still needs the Console (APIs & Services →
Credentials) — there's no stable `gcloud` command for the OAuth consent
screen. Everything else below is a real command.

> ⚠️ **This file contains your real billing account ID** (you pasted it in
> to get this file generated). That ID isn't a secret exactly, but it does
> uniquely identify your GCP billing account — treat this file like you would
> `.env`: keep it out of git. Don't commit/push this file to a shared repo;
> add it to `.gitignore` or just keep it as a local runbook.

Project name used throughout: **`gcp-fde-project`**
Region used throughout: **`asia-south1`** (Mumbai — matches MongoDB Atlas)
Billing account: **`REPLACE_WITH_YOUR_BILLING_ACCOUNT_ID`**

---

## 0. Authenticate the CLI

```bash
# Logs your user account into gcloud (opens a browser window)
gcloud auth login

# Sets up "Application Default Credentials" — lets local tools/libraries
# (not just gcloud itself) authenticate as you. Not strictly required for
# the steps below, but worth having.
gcloud auth application-default login
```

---

## 1. Create the project

```bash
# Creates a brand-new GCP project with this exact project ID
gcloud projects create gcp-fde-project --name="GCP FDE Project"

# Makes this the project every following gcloud command targets by default,
# so you don't have to pass --project=gcp-fde-project every time
gcloud config set project gcp-fde-project
```

## 2. Link it to your billing account

```bash
# Without this, none of the paid APIs below (Cloud Run, Cloud Build, etc.)
# will actually let you create resources
gcloud billing projects link gcp-fde-project \
  --billing-account=REPLACE_WITH_YOUR_BILLING_ACCOUNT_ID
```

## 3. Enable every API this project needs

```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  secretmanager.googleapis.com \
  iam.googleapis.com
```

| API | Why |
|-----|-----|
| `run.googleapis.com` | Cloud Run — hosts the container |
| `artifactregistry.googleapis.com` | Stores the built Docker image |
| `cloudbuild.googleapis.com` | Builds the image + runs the deploy step |
| `secretmanager.googleapis.com` | Holds `MONGO_URI`, `GROQ_API_KEY`, etc. |
| `iam.googleapis.com` | Lets you create the service account in step 5 |

## 4. Create the Artifact Registry repo

```bash
# Must be named "main-repo" and in "asia-south1" — cloudbuild.yaml in this
# repo already targets that exact name and region
gcloud artifacts repositories create main-repo \
  --repository-format=docker \
  --location=asia-south1 \
  --description="LUXE Docker images"

# Lets your local "docker"/"gcloud builds submit" push images here
gcloud auth configure-docker asia-south1-docker.pkg.dev
```

## 5. Create a dedicated service account for the deploy

```bash
# This is the identity that will actually run the build + deploy steps —
# not your own user account
gcloud iam service-accounts create fde-deployer \
  --display-name="FDE Deploy Service Account"
```

Grant it exactly the roles it needs — one command per role:

```bash
# Lets it deploy/update Cloud Run services
gcloud projects add-iam-policy-binding gcp-fde-project \
  --member="serviceAccount:fde-deployer@gcp-fde-project.iam.gserviceaccount.com" \
  --role="roles/run.admin"

# Lets it push the built image to Artifact Registry
gcloud projects add-iam-policy-binding gcp-fde-project \
  --member="serviceAccount:fde-deployer@gcp-fde-project.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.writer"

# Lets the deployed Cloud Run service run as itself (Cloud Run always
# executes as *some* service account — this lets fde-deployer authorize that)
gcloud projects add-iam-policy-binding gcp-fde-project \
  --member="serviceAccount:fde-deployer@gcp-fde-project.iam.gserviceaccount.com" \
  --role="roles/iam.serviceAccountUser"

# Lets it read MONGO_URI / GROQ_API_KEY / etc. out of Secret Manager at deploy time
gcloud projects add-iam-policy-binding gcp-fde-project \
  --member="serviceAccount:fde-deployer@gcp-fde-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Required because we pass a custom --service-account to `gcloud builds submit`
# in step 7 — that SA has to read the source tarball back out of the
# gs://gcp-fde-project_cloudbuild bucket that `gcloud` uploads it to. Without
# this you'll hit: "403 ... does not have storage.objects.get access ..."
gcloud projects add-iam-policy-binding gcp-fde-project \
  --member="serviceAccount:fde-deployer@gcp-fde-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"

# Lets Cloud Build write build logs (required when using a custom service
# account for a build — cloudbuild.yaml already sets logging: CLOUD_LOGGING_ONLY
# to match this)
gcloud projects add-iam-policy-binding gcp-fde-project \
  --member="serviceAccount:fde-deployer@gcp-fde-project.iam.gserviceaccount.com" \
  --role="roles/logging.logWriter"
```

## 6. Create the secrets

Replace the placeholder values with your real ones before running these.
`MONGO_URI` and `GROQ_API_KEY` are the only two the app truly requires to
start; `GOOGLE_CLIENT_ID` and `LOGFIRE_TOKEN` are optional (see `.env.example`).

```bash
echo -n "REPLACE_WITH_YOUR_MONGODB_URI" | gcloud secrets create MONGO_URI --data-file=-

echo -n "REPLACE_WITH_YOUR_GROQ_API_KEY" | gcloud secrets create GROQ_API_KEY --data-file=-

echo -n "REPLACE_WITH_YOUR_LOGFIRE_TOKEN" | gcloud secrets create LOGFIRE_TOKEN --data-file=-

# The one exception: the Google OAuth Client ID itself has to come from the
# Console first (APIs & Services -> Credentials -> Create Credentials ->
# OAuth client ID -> Web application), since the consent-screen setup has no
# stable gcloud command. Once you have it, storing it is a normal CLI step:
echo -n "REPLACE_WITH_YOUR_GOOGLE_CLIENT_ID" | gcloud secrets create GOOGLE_CLIENT_ID --data-file=-

# Not in the original 4 — you also handed over Portkey gateway credentials
# (client claims these are for guardrails/gateway routing to Groq). Same
# pattern: store them as secrets too if the app reads them at runtime.
echo -n "REPLACE_WITH_YOUR_PORTKEY_API_KEY" | gcloud secrets create PORTKEY_API_KEY --data-file=-

echo -n "REPLACE_WITH_YOUR_PORTKEY_GROQ_PROVIDER" | gcloud secrets create PORTKEY_GROQ_PROVIDER --data-file=-
```

*(`fde-deployer` already has `secretmanager.secretAccessor` at the project
level from step 5, so it can read all four of these — no extra per-secret
grant needed.)*

## 7. Build and deploy — no trigger, just run it

```bash
# Run this from the repo root (where cloudbuild.yaml lives). This is the
# manual equivalent of what a GitHub trigger would have done automatically —
# it builds the image, pushes it to Artifact Registry, and deploys it to
# Cloud Run, all in one command, using the service account from step 5.
gcloud builds submit --config cloudbuild.yaml . \
  --service-account="projects/gcp-fde-project/serviceAccounts/fde-deployer@gcp-fde-project.iam.gserviceaccount.com"
```

Run this same command again any time you want to ship a new change — there's
no automation watching your repo, so a deploy only happens when you run it.

## 8. Get the live URL

```bash
gcloud run services describe main-app \
  --region=asia-south1 \
  --format="value(status.url)"
```

---

## Cleanup — tear down what you built

### Option A — remove just the demo resources, keep the project

```bash
gcloud run services delete main-app --region=asia-south1 --quiet

gcloud artifacts repositories delete main-repo --location=asia-south1 --quiet

gcloud secrets delete MONGO_URI --quiet
gcloud secrets delete GROQ_API_KEY --quiet
gcloud secrets delete LOGFIRE_TOKEN --quiet
gcloud secrets delete GOOGLE_CLIENT_ID --quiet
gcloud secrets delete PORTKEY_API_KEY --quiet
gcloud secrets delete PORTKEY_GROQ_PROVIDER --quiet

gcloud iam service-accounts delete \
  fde-deployer@gcp-fde-project.iam.gserviceaccount.com --quiet
```

### Option B — delete the entire project (removes everything above in one shot)

```bash
gcloud billing projects unlink gcp-fde-project

gcloud projects delete gcp-fde-project --quiet
```

`gcloud projects delete` alone is enough to remove every resource inside it —
Option A is only useful if you want to keep the project around but wipe the
demo out of it.

# GCP setup

This project uses Google Cloud Build to build its Docker image and Artifact
Registry to store it.

Create a GCP project in the Google Cloud Console:

https://console.cloud.google.com/projectcreate

## Billing

Before enabling the required APIs, attach an open billing account to the GCP
project. The project billing page is:

https://console.cloud.google.com/billing/projects

## APIs and registry

Enable these APIs for the project:

```sh
gcloud services enable cloudbuild.googleapis.com artifactregistry.googleapis.com
```

Create an Artifact Registry Docker repository named `notely-ar-repo` in
`us-central1`. Use standard mode, a regional location, Google-managed
encryption, and disable vulnerability scanning.

## Build and publish

Set these shell variables for the current terminal. Replace the project
placeholder with the GCP Project ID.

```sh
export PROJECT_ID="YOUR_PROJECT_ID"
export REGION="us-central1"
export REPOSITORY="notely-ar-repo"
export IMAGE="notely"
```

Build the code in the current directory and push the image to Artifact
Registry:

```sh
gcloud builds submit \
  --tag "$REGION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY/$IMAGE:latest"
```

List the published images:

```sh
gcloud artifacts docker images list \
  "$REGION-docker.pkg.dev/$PROJECT_ID/$REPOSITORY"
```

## GitHub Actions authentication and publishing

The CD workflow runs on every push to `main`. After `npm run build`, it
authenticates to Google Cloud, sets up `gcloud`, and submits the Docker build
to Artifact Registry.

Create a service account named `Cloud Run Deployer` and grant it these roles:

- Cloud Build Editor
- Cloud Build Service Account
- Cloud Run Admin
- Service Account User
- Viewer

Create a JSON key for that account. Add the full JSON as a repository Actions
secret named `GCP_CREDENTIALS`. It must be a repository secret, not an
environment secret.

The workflow uses the current Google actions:

```yaml
- name: Authenticate to Google Cloud
  uses: google-github-actions/auth@v3
  with:
    credentials_json: ${{ secrets.GCP_CREDENTIALS }}

- name: Set up gcloud
  uses: google-github-actions/setup-gcloud@v3

- name: Build and push Docker image
  run: |
    PROJECT_ID=$(gcloud config get-value project)
    gcloud builds submit --tag "us-central1-docker.pkg.dev/${PROJECT_ID}/notely-ar-repo/notely:latest" .
```

Do not commit the JSON key or paste it into issues, pull requests, or chat.
For this setup, the key was uploaded as the `GCP_CREDENTIALS` repository
secret and the temporary local copy was deleted after use.

The workflow change is commit `48568e7`. Its rerun completed successfully on
October 4, 2026, including the Artifact Registry image push:

https://github.com/PedroMarianoAlmeida/learn-cicd-typescript-starter/actions/runs/37178450538

## Docker platform fix

Cloud Build workers use the `linux/amd64` platform. The Dockerfile previously
forced `linux/arm64`, which made the build fail with `exec format error` before
`npm ci` ran. The base image now lets Cloud Build use its native platform:

```dockerfile
FROM node:22-slim
```

`package-lock.json` was not the source of that failure.

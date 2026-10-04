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

## Docker platform fix

Cloud Build workers use the `linux/amd64` platform. The Dockerfile previously
forced `linux/arm64`, which made the build fail with `exec format error` before
`npm ci` ran. The base image now lets Cloud Build use its native platform:

```dockerfile
FROM node:22-slim
```

`package-lock.json` was not the source of that failure.

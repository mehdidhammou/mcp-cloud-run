# MCP Server on Google Cloud Run for AI agents

## Introduction

This project is part of a workshop at **Secure By Design** conference organized in Google Paris.

A cloud native MCP server on `Google Cloud Run` using `fastmcp` that exposes tools for querying a toy "zoo" dataset. The server is implemented in `src/server.py` and is packaged with `pyproject.toml`.

- **Python requirement**: `>=3.13` (as declared in `pyproject.toml`).
- **Main dependency**: `fastmcp==2.12.4`.

## Prerequisites

- Docker, Docker Compose (for local container testing)
- Google Cloud SDK (`gcloud` CLI) installed and configured (for Cloud Run deployment)
- uv
- Gemini CLI (for client interactions)

## Setup

### Server Setup

- **Create a venv and install dependencies**:

```bash
uv sync
```

- Configure GCP project and authentication:

```bash
gcloud projects create YOUR_PROJECT_ID
gcloud config set project YOUR_PROJECT_ID
gcloud auth login
```

- **Enable Cloud Run API**:

```bash
gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com
```

**Create a Service Account**

```bash
gcloud iam service-accounts create mcp-cloud-run-sa \
    --display-name="MCP Cloud Run Service Account"
```

- **Configure environment variables**:

```bash
cp .env.example .env
```

and edit `.env` as needed.

```bash
GOOGLE_CLOUD_PROJECT=
PROJECT_NUMBER=
ID_TOKEN=
```

- **Run locally** (default port 8080):

```bash
PORT=8080 python src/server.py
```

The server logs will indicate it started and which port it is listening on. Two tools are registered by the server:

- `get_animals_by_species(species: str)` — returns a list of animals of that species.
- `get_animal_details(name: str)` — returns details for a named animal.

You can interact with the server using a FastMCP client or your preferred HTTP client once the HTTP transport is running. The server binds to `0.0.0.0` by default so it is reachable from Docker and Cloud Run environments.

**Docker**

- Build the image:

```bash
docker build -t mcp-cloud-run .
```

- Run the container locally (map port 8080):

```bash
docker run --rm -p 8080:8080 -e PORT=8080 mcp-cloud-run
```

This repo includes a `Dockerfile` so you can build and deploy to Cloud Run via container image.

- Build and push with Cloud Build (replace `PROJECT_ID` and `REGION`):

```bash
gcloud run deploy zoo-mcp-server \
    --service-account=mcp-server-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
    --no-allow-unauthenticated \
    --region=europe-west1 \
    --source=.
```

After deployment, Cloud Run will provide a public HTTPS URL. Use that URL with a FastMCP client or your integration to call the registered tools.

### Client Setup

We'll use Gemini CLI for this project.

- Allow your service account to invoke the Cloud Run service:

```bash
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member=user:$(gcloud config get-value account) \
    --role='roles/run.invoker'
```

- Export PROJECT_NUMBER and ID_TOKEN environment variables:

```bash
export PROJECT_NUMBER=$(gcloud projects describe $GOOGLE_CLOUD_PROJECT --format="value(projectNumber)")
export ID_TOKEN=$(gcloud auth print-identity-token)
```

- Add these to your `.env` file.

the repo includes .gemini/settings.json for Gemini to be able to find the server.

- **Test the client**:

```bash
gemini
```

then enter and you should see the two registered tools.:

```bash
/mcp
```

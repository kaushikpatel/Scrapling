# Deploying the MCP server to Google Cloud Run

The `.github/workflows/deploy-cloud-run.yml` workflow builds the repository's `Dockerfile` and deploys the [MCP server](../ai/mcp-server.md) to [Cloud Run](https://cloud.google.com/run) with the 'Streamable HTTP' transport. It runs on every push to `main` and can be started by hand from the **Actions** tab.

GitHub signs in to Google Cloud through [Workload Identity Federation](https://github.com/google-github-actions/auth#workload-identity-federation-through-a-service-account), so no service account key is stored in GitHub. The server's bearer token lives in Secret Manager and never touches the workflow.

The workflow is skipped until you set the `GCP_PROJECT_ID` repository variable, so it does nothing until the setup below is done.

## One-time setup

Run these commands once with the [gcloud CLI](https://cloud.google.com/sdk/docs/install), signed in as a project owner. Replace the first three values with your own.

```bash
PROJECT_ID="your-project-id"
REGION="us-central1"
GITHUB_REPO="your-github-user/Scrapling"   # owner/name of the repository that runs the workflow

gcloud config set project "$PROJECT_ID"
PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format 'value(projectNumber)')

# 1. Turn on the services the deployment uses
gcloud services enable run.googleapis.com artifactregistry.googleapis.com \
  secretmanager.googleapis.com iamcredentials.googleapis.com

# 2. Create the Docker repository for the images
gcloud artifacts repositories create scrapling --repository-format docker --location "$REGION"

# 3. Store the MCP server's auth token in Secret Manager (save the printed token, clients need it)
TOKEN=$(openssl rand -hex 32); echo "Your MCP token: $TOKEN"
printf '%s' "$TOKEN" | gcloud secrets create scrapling-mcp-token --data-file=-

# 4. Let Cloud Run's runtime account read that secret
gcloud secrets add-iam-policy-binding scrapling-mcp-token \
  --member "serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role roles/secretmanager.secretAccessor

# 5. Create the service account GitHub Actions deploys with
gcloud iam service-accounts create github-deployer
DEPLOYER="github-deployer@${PROJECT_ID}.iam.gserviceaccount.com"
for ROLE in roles/run.admin roles/artifactregistry.writer; do
  gcloud projects add-iam-policy-binding "$PROJECT_ID" --member "serviceAccount:$DEPLOYER" --role "$ROLE"
done
gcloud iam service-accounts add-iam-policy-binding "${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --member "serviceAccount:$DEPLOYER" --role roles/iam.serviceAccountUser

# 6. Trust GitHub's OIDC tokens, limited to this repository
gcloud iam workload-identity-pools create github --location global
gcloud iam workload-identity-pools providers create-oidc github \
  --location global --workload-identity-pool github \
  --issuer-uri "https://token.actions.githubusercontent.com" \
  --attribute-mapping "google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition "assertion.repository == '${GITHUB_REPO}'"
gcloud iam service-accounts add-iam-policy-binding "$DEPLOYER" \
  --role roles/iam.workloadIdentityUser \
  --member "principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github/attribute.repository/${GITHUB_REPO}"

# 7. Print the values GitHub needs
echo "GCP_WORKLOAD_IDENTITY_PROVIDER = projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github/providers/github"
echo "GCP_SERVICE_ACCOUNT            = $DEPLOYER"
```

## GitHub settings

In the repository, open **Settings → Secrets and variables → Actions**.

**Secrets**

| Name | Value |
|---|---|
| `GCP_WORKLOAD_IDENTITY_PROVIDER` | The provider path printed in step 7 |
| `GCP_SERVICE_ACCOUNT` | The service account email printed in step 7 |

**Variables**

| Name | Required | Default | Meaning |
|---|---|---|---|
| `GCP_PROJECT_ID` | Yes | | The Google Cloud project; setting it turns the workflow on |
| `GCP_REGION` | No | `us-central1` | Region for Cloud Run and Artifact Registry |
| `CLOUD_RUN_SERVICE` | No | `scrapling-mcp` | Cloud Run service name |
| `ARTIFACT_REGISTRY_REPOSITORY` | No | `scrapling` | Docker repository created in step 2 |
| `MCP_TOKEN_SECRET_NAME` | No | `scrapling-mcp-token` | Secret created in step 3 |
| `MCP_ALLOWED_HOST` | No | | The service's host name (like `scrapling-mcp-abc123-uc.a.run.app`) to turn on DNS-rebinding protection; set it after the first deploy |
| `CLOUD_RUN_MAX_INSTANCES` | No | `1` | Browser sessions live in one instance's memory, so raise this only if you use the one-shot tools |
| `CLOUD_RUN_MEMORY` | No | `2Gi` | Memory per instance; Chromium needs at least 2 GiB |
| `CLOUD_RUN_CPU` | No | `2` | vCPUs per instance |

Then push to `main`, or run **Deploy MCP Server to Cloud Run** from the **Actions** tab. The run summary prints the endpoint.

## Connecting a client

```json
{
  "mcpServers": {
    "ScraplingServer": {
      "url": "https://<your-service-url>/mcp",
      "headers": {
        "Authorization": "Bearer <your-token>"
      }
    }
  }
}
```

With Claude Code:

```bash
claude mcp add --transport http ScraplingServer https://<your-service-url>/mcp \
  --header "Authorization: Bearer <your-token>"
```

## Notes

* The service scales to zero when idle, so you only pay while it handles requests, and the first request after a quiet period waits for a cold start of several seconds while the container starts.
* Every tool can fetch any URL from Google's network, so keep the token private. To rotate it, add a new version of the secret (`gcloud secrets versions add scrapling-mcp-token --data-file=-`) and re-run the workflow.
* Each request can run for up to 60 minutes, which Cloud Run allows for long streamed responses.

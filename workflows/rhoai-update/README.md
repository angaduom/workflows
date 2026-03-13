# RHOAI Nightly Update Workflow

Automated workflow for updating Red Hat OpenShift AI to latest nightly builds and managing test results.

## Overview

This workflow provides an AI-powered pipeline for:
- Updating RHOAI to latest nightly builds
- Running automated test suites
- Collecting and analyzing test results
- Updating JIRA issues with outcomes

## Structure

```
workflows/rhoai-update/
├── .ambient/
│   └── ambient.json            # Workflow configuration
├── .claude/
│   └── commands/
│       ├── oc-login.md          # OpenShift cluster login command
│       ├── rhoai-version.md     # RHOAI version detection command
│       ├── rhoai-update.md      # RHOAI update command
│       ├── rhoai-uninstall.md   # RHOAI uninstall command
│       ├── jenkins-trigger.md   # Jenkins build trigger command
│       ├── setup-pipelines.md   # AI Pipeline server setup command
│       └── setup-llamastack.md  # Llama Stack server setup command
└── README.md                    # This file
```

## Commands

### /oc-login

Login to OpenShift cluster using credentials from Ambient session.

**What it does:**
- Checks for required credentials (OCP_SERVER, OCP_USERNAME, OCP_PASSWORD)
- Verifies `oc` CLI is installed
- Executes login to the cluster
- Verifies connection and displays cluster info

**Usage:**
```
/oc-login
```

Or simply ask:
- "Login to my cluster"
- "Connect to OpenShift"
- "Login to OCP"

**Required Environment Variables:**
- `OCP_SERVER` - OpenShift cluster API URL (e.g., `https://api.cluster.example.com:6443`)
- `OCP_USERNAME` - Your OpenShift username
- `OCP_PASSWORD` - Your OpenShift password

### /rhoai-version

Detect RHOAI version and build information.

**What it does:**
- Checks RHOAI operator subscription and ClusterServiceVersion
- Reports DataScienceCluster status and component states
- Lists all component images with SHA256 digests
- Provides comprehensive version summary

**Usage:**
```
/rhoai-version
```

Or simply ask:
- "What version of RHOAI is installed?"
- "Check RHOAI version"
- "Show me RHOAI build info"

**Note:** You must be logged into the cluster first (use `/oc-login`)

### /rhoai-update

Updates RHOAI to the latest nightly build.

**What it does:**
- Checks current RHOAI version
- Updates the OLM catalog source to latest nightly
- Monitors the operator upgrade process
- Verifies component reconciliation
- Reports final status

**Usage:**
```
/rhoai-update
```

Or simply ask:
- "Update RHOAI to latest nightly"
- "Upgrade to RHOAI 3.4 nightly"
- "What version of RHOAI is installed?"

**Note:** You must be logged into the cluster first (use `/oc-login`)

### /rhoai-uninstall

Completely uninstall RHOAI from an OpenShift cluster.

**What it does:**
- Removes RHOAI operator and subscriptions
- Deletes custom resources (DataScienceCluster, DSCInitialization, etc.)
- Cleans up webhooks and finalizers
- Removes RHOAI namespaces
- Deletes CRDs (optional)
- Cleans up user data science projects (optional)

**Usage:**
```
/rhoai-uninstall              # Standard forceful uninstall
/rhoai-uninstall graceful     # Graceful uninstall followed by cleanup
/rhoai-uninstall keep-crds    # Keep CRDs installed
/rhoai-uninstall keep-all     # Keep CRDs and user resources
```

Or simply ask:
- "Uninstall RHOAI from the cluster"
- "Remove RHOAI completely"
- "Clean up RHOAI installation"

**Warning:** This will delete all RHOAI resources including user workbenches, models, and data. Backup important work first.

**Note:** You must be logged into the cluster first (use `/oc-login`) and have cluster-admin permissions.

### /jenkins-trigger

Trigger a Jenkins build job.

**What it does:**
- Validates Jenkins credentials (JENKINS_URL, JENKINS_API_TOKEN, JENKINS_USER)
- Triggers the build via Jenkins API
- Retrieves and saves the build number to `/tmp/.jenkins-last-build`
- Enables monitoring with `/jenkins-poll`

**Usage:**
```
/jenkins-trigger
```

Or simply ask:
- "Trigger Jenkins build"
- "Start Jenkins job"
- "Run Jenkins build"

**Required Environment Variables:**
- `JENKINS_URL` - Full URL to the Jenkins job (e.g., `https://jenkins.example.com/job/Add%20Numbers/`)
- `JENKINS_API_TOKEN` - API token for Jenkins authentication
- `JENKINS_USER` - Jenkins username

### /setup-pipelines

Setup AI Pipeline server on RHOAI with MinIO object storage.

**What it does:**
- Creates `minio-storage` and `ai-pipelines` namespaces
- Deploys MinIO S3-compatible object storage (20Gi)
- Creates S3 buckets for pipelines and data
- Deploys DataSciencePipelinesApplication with MariaDB (10Gi)
- Creates RHOAI data connections for dashboard integration
- Verifies all components are healthy and ready

**Usage:**
```
/setup-pipelines              # Full setup
/setup-pipelines verify       # Verify existing installation
```

Or simply ask:
- "Setup pipeline server"
- "Configure AI pipelines"
- "Deploy data science pipelines"

**Prerequisites:**
- RHOAI installed on the cluster
- Logged into cluster with admin permissions (use `/oc-login`)
- At least 50Gi available storage
- Default StorageClass configured (e.g., `gp3-csi`)

**What gets deployed:**
- **minio-storage namespace**: MinIO with 20Gi PVC, console route
- **ai-pipelines namespace**: Pipeline server with 7 components, MariaDB database
- **S3 buckets**: `pipelines` (artifacts), `data` (training data)
- **Data connections**: `pipeline-artifacts`, `pipeline-data`
- **Credentials**: Randomly generated and stored in Kubernetes secrets

**Access:**
- MinIO Console: `https://minio-console-minio-storage.apps.<cluster-domain>`
  - Credentials: `oc get secret minio-secret -n minio-storage -o jsonpath='{.data.accesskey}' | base64 -d`
- Pipeline API: `https://ds-pipeline-pipelines-ai-pipelines.apps.<cluster-domain>`
- RHOAI Dashboard: Navigate to Data Science Projects → ai-pipelines → Pipelines

**Note:** The command is idempotent - it checks for existing components and only creates what's missing.

### /setup-llamastack

Setup Llama Stack on RHOAI with Keycloak authentication and model endpoints.

**What it does:**
- Deploys PostgreSQL database for Keycloak (5Gi)
- Deploys Keycloak server with OAuth2 configuration
- Creates Keycloak realm, client, and user for authentication
- Configures Llama Stack server with LLM and embedding model endpoints
- Sets up OAuth2 token-based authentication
- Verifies end-to-end token generation and API access

**Usage:**
```
/setup-llamastack           # Full setup with interactive model configuration
/setup-llamastack verify    # Verify existing installation
```

Or simply ask:
- "Setup Llama Stack"
- "Configure Llama Stack server"
- "Deploy Llama Stack with Keycloak"

**Prerequisites:**
- RHOAI installed on the cluster
- Llama Stack Operator enabled in DataScienceCluster
- Logged into cluster with admin permissions (use `/oc-login`)
- At least one LLM model deployed (e.g., Llama 3.3 70B)
- At least one embedding model deployed (e.g., Granite Embedding)

**Note:** The command automatically installs Keycloak Operator if not already present.

**Required Model Information:**
You'll be prompted for:
- LLM model ID, URL, token, provider ID
- Embedding model ID, URL, token, provider ID, dimension
- Target namespace for Llama Stack deployment

**What gets deployed:**
- **keycloak-operator namespace**: Red Hat build of Keycloak Operator
- **postgresql namespace**: PostgreSQL database (5Gi) - service: `psql-keycloak`, database: `keycloak`
  - Credentials: Randomly generated and stored in secret `keycloak-db-secret`
- **keycloak namespace**: Keycloak server with OAuth2 configuration
  - Realm: `llamastack`
  - Client: `llama-stack-client` with direct access grants
  - User: `llama-user` (password randomly generated and stored in secret)
- **Target namespace** (e.g., llm-models): Llama Stack server with model configuration
- **ConfigMap**: Complete Llama Stack config with LLM/embedding endpoints and Keycloak auth
- **Credentials**: All passwords randomly generated and stored securely in Kubernetes secrets

**Access:**
- Keycloak Admin: Auto-detected from route
- Llama Stack API: `http://llama-stack-<service>.<namespace>.svc.cluster.local:8321`
- Token generation: Via OAuth2 password grant flow

**Authentication:**
```bash
# Retrieve credentials from secrets
KEYCLOAK_USER="llama-user"
KEYCLOAK_PASS=$(oc get secret llama-user-password -n keycloak -o jsonpath='{.data.password}' | base64 -d)
CLIENT_ID="llama-stack-client"

# Get Keycloak URL
KEYCLOAK_URL=$(oc get route -n keycloak -l app=keycloak -o jsonpath='{.items[0].spec.host}')

# Get access token
TOKEN=$(curl -k -d client_id=$CLIENT_ID \
  -d username=$KEYCLOAK_USER \
  -d password=$KEYCLOAK_PASS \
  -d grant_type=password \
  https://$KEYCLOAK_URL/realms/llamastack/protocol/openid-connect/token | jq -r .access_token)

# Call Llama Stack API
curl -H "Authorization: Bearer $TOKEN" \
  http://llama-stack-<service>.<namespace>.svc.cluster.local:8321/models/list
```

**Note:** The command auto-detects deployed models when possible, or prompts for manual input if not found.

## Prerequisites

- OpenShift cluster with RHOAI installed via OLM
- `oc` CLI installed
- Cluster credentials configured in Ambient session:
  - `OCP_SERVER` - OpenShift cluster API URL
  - `OCP_USERNAME` - Your OpenShift username
  - `OCP_PASSWORD` - Your OpenShift password
- Cluster admin permissions
- RHOAI installed from `rhoai-catalog-dev` catalog

## Output Artifacts

All artifacts are stored in `artifacts/rhoai-update/`:

- `reports/*.md` - Update reports and version changes
- `tests/*.log` - Test execution results
- `jira/*.md` - JIRA update summaries
- `logs/*.log` - Detailed execution logs

## GitHub Actions Integration

This workflow is designed to run via GitHub Actions with Ambient:

```yaml
- name: Update RHOAI to Latest Nightly
  uses: ambient-code/ambient-action@v0.0.2
  with:
    api-token: ${{ secrets.AMBIENT_API_TOKEN }}
    workflow: workflows/rhoai-update
    prompt: Update RHOAI to the latest nightly build
```

## Future Enhancements

- [ ] Automated test suite execution
- [ ] Test result parsing and analysis
- [ ] JIRA integration for issue updates
- [ ] Slack/email notifications
- [ ] Rollback capabilities
- [ ] Pre-upgrade validation checks

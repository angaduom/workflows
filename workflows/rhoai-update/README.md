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
│       ├── oc-login.md         # OpenShift cluster login command
│       ├── rhoai-version.md    # RHOAI version detection command
│       ├── rhoai-update.md     # RHOAI update command
│       └── jenkins-trigger.md  # Jenkins build trigger command
└── README.md                   # This file
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

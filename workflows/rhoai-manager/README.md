# RHOAI Manager

Comprehensive workflow for managing the complete lifecycle of Red Hat OpenShift AI: installation, updates, version detection, and uninstallation.

## Overview

This workflow provides an AI-powered pipeline for:
- Installing RHOAI from scratch on OpenShift clusters
- Updating RHOAI to latest nightly builds
- Detecting version and build information
- Completely uninstalling RHOAI when needed
- Managing cluster connections and authentication

## Structure

```
workflows/rhoai-manager/
├── .ambient/
│   └── ambient.json            # Workflow configuration
├── .claude/
│   └── commands/
│       ├── oc-login.md          # OpenShift cluster login command
│       ├── rhoai-install.md     # RHOAI installation command
│       ├── rhoai-version.md     # RHOAI version detection command
│       ├── rhoai-update.md      # RHOAI update command
│       └── rhoai-uninstall.md   # RHOAI uninstall command
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

### /rhoai-install

Install RHOAI from scratch on an OpenShift cluster.

**What it does:**
- Sets up OLM catalog source (dev or GA production)
- Creates operator namespace and subscription
- Waits for ClusterServiceVersion (CSV) to be ready
- Creates DataScienceCluster with component configuration
- Patches component states (Managed/Removed) as needed
- Verifies all components are healthy and reconciled
- Provides detailed installation summary

**Usage:**
```bash
# Development/Nightly builds (default)
/rhoai-install                                    # Latest dev catalog
/rhoai-install channel=beta                       # Dev catalog, beta channel
/rhoai-install image=quay.io/modh/rhoai-catalog:latest-release-3.5

# GA Production releases
/rhoai-install catalog=redhat-operators           # GA catalog, stable channel
/rhoai-install catalog=redhat-operators channel=fast     # GA catalog, fast channel
```

**Parameters:**
- `catalog` - Catalog source (`rhoai-catalog-dev` for nightly, `redhat-operators` for GA)
- `channel` - Subscription channel (`beta`, `fast`, or `stable`)
- `image` - Custom catalog image (only for `rhoai-catalog-dev`)

Or simply ask:
- "Install RHOAI on the cluster"
- "Deploy RHOAI from production catalog"
- "Set up RHOAI for testing"

**Prerequisites:**
- OpenShift cluster (version 4.12+)
- Logged into cluster with admin permissions (use `/oc-login`)
- No existing RHOAI installation

**What gets deployed:**
- **Operator namespace**: `redhat-ods-operator`
- **Application namespace**: `redhat-ods-applications`
- **Monitoring namespace**: `redhat-ods-monitoring`
- **DataScienceCluster**: Custom resource managing all RHOAI components
- **Component operators**: Dashboard, Workbenches, Model Serving, Pipelines, etc.

**Note:** Defaults to `rhoai-catalog-dev` for nightly builds. Use `catalog=redhat-operators` for GA production releases.

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
- Checks current RHOAI version and component states
- Updates the OLM catalog source to latest nightly
- Monitors the operator upgrade process
- Handles special scenarios:
  - **Scenario E**: Forced subscription reinstall when component images update without CSV version change
  - Channel preservation across updates
  - Component state preservation (Managed/Removed)
- Verifies component reconciliation
- Reports final status with before/after comparison

**Usage:**
```
/rhoai-update
```

Or simply ask:
- "Update RHOAI to latest nightly"
- "Upgrade to RHOAI 3.4 nightly"
- "Update RHOAI"

**Note:** You must be logged into the cluster first (use `/oc-login`)

**Advanced Features:**
- Detects when component images have newer builds without CSV version changes
- Automatically triggers forced reinstall in these cases
- Preserves DataScienceCluster component configuration across updates
- Waits for all components to reconcile before completing

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

## Prerequisites

- OpenShift cluster (version 4.12+)
- `oc` CLI installed (auto-installed if missing)
- Cluster credentials configured in Ambient session:
  - `OCP_SERVER` - OpenShift cluster API URL
  - `OCP_USERNAME` - Your OpenShift username
  - `OCP_PASSWORD` - Your OpenShift password
- Cluster admin permissions

## Output Artifacts

All artifacts are stored in `artifacts/rhoai-manager/`:

- `reports/*.md` - Installation and update reports with version changes
- `version/*.md` - Version detection summaries
- `logs/*.log` - Detailed execution logs

## Typical Workflows

### Fresh Installation
```
1. /oc-login              # Connect to cluster
2. /rhoai-install         # Install RHOAI from scratch
3. /rhoai-version         # Verify installation
```

### Regular Updates
```
1. /oc-login              # Connect to cluster
2. /rhoai-version         # Check current version
3. /rhoai-update          # Update to latest nightly
4. /rhoai-version         # Verify new version
```

### Decommissioning
```
1. /oc-login              # Connect to cluster
2. /rhoai-uninstall       # Remove RHOAI completely
```

## GitHub Actions Integration

This workflow is designed to run via GitHub Actions with Ambient:

```yaml
- name: Update RHOAI to Latest Nightly
  uses: ambient-code/ambient-action@v0.0.2
  with:
    api-token: ${{ secrets.AMBIENT_API_TOKEN }}
    workflow: workflows/rhoai-manager
    prompt: Update RHOAI to the latest nightly build
```

## Technical Details

### Catalog Source
- Uses `rhoai-catalog-dev` for nightly builds
- Image: `quay.io/modh/rhoai-catalog:latest-release-3.4`
- Updates trigger operator upgrades automatically

### Component Management
- DataScienceCluster manages component states (Managed/Removed)
- Component states preserved across updates
- Individual component image tracking with SHA digests

### Update Scenarios
The workflow handles several update scenarios:
- **Normal CSV upgrade**: Operator version changes
- **Forced reinstall**: Component images update without CSV version change
- **Channel migration**: Updates preserve subscription channel
- **Catalog refresh**: Forces OLM to re-evaluate available updates

## Future Enhancements

- [ ] Automated test suite execution after updates
- [ ] Test result parsing and analysis
- [ ] JIRA integration for issue updates
- [ ] Slack/email notifications
- [ ] Rollback capabilities
- [ ] Pre-upgrade validation checks
- [ ] Multi-cluster support

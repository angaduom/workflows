---
name: rhoai-update
description: Update Red Hat OpenShift AI (RHOAI) to nightly builds. Usage - /rhoai-update (updates to latest) or /rhoai-update 3.4 (updates to specific version).
---

# RHOAI Nightly Build Updater

This skill handles upgrading Red Hat OpenShift AI (RHOAI) operator installations to nightly/development builds.

## Command Usage

- `/rhoai-update` - Updates to the latest available nightly build (currently 3.4)
- `/rhoai-update 3.4` - Updates to RHOAI 3.4 nightly (latest image)
- `/rhoai-update 3.5` - Updates to RHOAI 3.5 nightly (if available)
- `/rhoai-update 3.3` - Downgrades to RHOAI 3.3 nightly
- `/rhoai-update 3.4@sha256:abc123...` - Updates to specific SHA digest (for reproducibility)

## When to Use This Skill

This skill is triggered when the user runs:
- `/rhoai-update` - Update to latest nightly build
- `/rhoai-update <version>` - Update to specific version (e.g., 3.4, 3.5)
- `/rhoai-update <version>@sha256:<digest>` - Update to specific SHA for reproducibility

## Prerequisites

Before running this skill, verify:
1. User has `oc` CLI access to their OpenShift cluster
2. Cluster is logged in with cluster-admin privileges
3. RHOAI is installed via OLM (Operator Lifecycle Manager)
4. The installation uses the `rhoai-catalog-dev` development catalog source

## How It Works

RHOAI nightly builds are distributed through catalog images:
- Catalog image format: `quay.io/rhoai/rhoai-fbc-fragment:rhoai-X.Y`
- Version examples: `rhoai-3.3`, `rhoai-3.4`
- Updates happen automatically when OLM detects new operator bundles

## Upgrade Process

### Step 1: Determine Target Version

Parse the command argument:
- If user runs `/rhoai-update` with NO argument → Default to **3.4** (current latest as of March 2026)
- If user runs `/rhoai-update 3.4` → Use **3.4** (latest image)
- If user runs `/rhoai-update 3.5` → Use **3.5** (if it exists)
- If user runs `/rhoai-update 3.3` → Use **3.3**
- If user runs `/rhoai-update 3.4@sha256:abc123...` → Use **3.4 with specific SHA digest**

**SHA Digest Format:**
When user provides a SHA digest (e.g., `3.4@sha256:352e38780ecee1298e927e3bc28b9d307dada2bb65ff0faa185ab15c455065a6`):
- Extract version: `3.4`
- Extract digest: `sha256:352e38780ecee1298e927e3bc28b9d307dada2bb65ff0faa185ab15c455065a6`
- Full image: `quay.io/rhoai/rhoai-fbc-fragment:rhoai-3.4@sha256:352e38780ecee1298e927e3bc28b9d307dada2bb65ff0faa185ab15c455065a6`

**Determining the Latest Version:**

To check if a newer version exists before defaulting to 3.4:

```bash
# Try to check if 3.5 catalog exists
oc patch catalogsource rhoai-catalog-dev -n openshift-marketplace \
  --type=merge \
  -p '{"spec":{"image":"quay.io/rhoai/rhoai-fbc-fragment:rhoai-3.5"}}' --dry-run=client

# Or check the catalog pod logs after attempting update
```

If the catalog fails to load with a newer version, fall back to 3.4.

**Current Known Versions:**
- 3.3 - Available ✅
- 3.4 - Available ✅ (Current Latest)
- 3.5 - Unknown (check availability first)
- 3.6+ - Future versions

**Important:** When no version is specified, use **3.4** as the default. Update this default when newer versions are confirmed to be available.

### Step 2: Locate Current Installation

Check the current RHOAI installation:

```bash
# Find the catalog source
oc get catalogsource -n openshift-marketplace | grep rhoai

# Get current version
oc get csv -n redhat-ods-operator | grep rhods-operator

# Check subscription channel
oc get subscription -n redhat-ods-operator -o yaml | grep -E "channel:|source:"
```

Expected output:
- Catalog source: `rhoai-catalog-dev`
- Namespace: Usually `redhat-ods-operator`
- Current CSV: `rhods-operator.X.Y.Z`

### Step 3: Update Catalog Source

Update the catalog source image to the target version.

**Option A: Use Version Tag (gets latest nightly for that version)**

```bash
# For RHOAI 3.4 (latest)
oc patch catalogsource rhoai-catalog-dev -n openshift-marketplace \
  --type=merge \
  -p '{"spec":{"image":"quay.io/rhoai/rhoai-fbc-fragment:rhoai-3.4"}}'

# For RHOAI 3.5 (latest)
oc patch catalogsource rhoai-catalog-dev -n openshift-marketplace \
  --type=merge \
  -p '{"spec":{"image":"quay.io/rhoai/rhoai-fbc-fragment:rhoai-3.5"}}'
```

**Option B: Use Specific SHA Digest (for reproducibility)**

```bash
# For RHOAI 3.4 with specific SHA
oc patch catalogsource rhoai-catalog-dev -n openshift-marketplace \
  --type=merge \
  -p '{"spec":{"image":"quay.io/rhoai/rhoai-fbc-fragment:rhoai-3.4@sha256:352e38780ecee1298e927e3bc28b9d307dada2bb65ff0faa185ab15c455065a6"}}'

# For RHOAI 3.5 with specific SHA
oc patch catalogsource rhoai-catalog-dev -n openshift-marketplace \
  --type=merge \
  -p '{"spec":{"image":"quay.io/rhoai/rhoai-fbc-fragment:rhoai-3.5@sha256:abc123def456..."}}'
```

**When to use SHA digest:**
- ✅ For reproducible deployments across environments
- ✅ For testing a specific nightly build
- ✅ To prevent automatic updates to newer nightlies
- ✅ For rollback to a known-good state

**How to get the current SHA digest:**

```bash
# Get current catalog image with digest
oc get catalogsource rhoai-catalog-dev -n openshift-marketplace \
  -o jsonpath='{.spec.image}'

# Or check the catalog pod image
oc get pod -n openshift-marketplace -l olm.catalogSource=rhoai-catalog-dev \
  -o jsonpath='{.items[0].spec.containers[0].image}'
```

### Step 4: Verify Catalog Update

Wait for the catalog to refresh (typically 15-30 seconds):

```bash
# Check catalog status
oc get catalogsource rhoai-catalog-dev -n openshift-marketplace \
  -o jsonpath='{.status.connectionState.lastObservedState}'

# Should return: READY
```

If status is not READY, wait longer or check pod logs:

```bash
oc get pods -n openshift-marketplace | grep rhoai-catalog-dev
oc logs -n openshift-marketplace <catalog-pod-name>
```

### Step 5: Monitor Operator Upgrade

The operator should automatically upgrade (if `installPlanApproval: Automatic`):

```bash
# Check for new operator version
oc get csv -n redhat-ods-operator | grep rhods-operator

# Check subscription status
oc get subscription rhoai-operator-dev -n redhat-ods-operator \
  -o jsonpath='{.status.currentCSV}'
```

You'll see:
- Old CSV in "Replacing" state
- New CSV in "Installing" → "Succeeded" state

Typical upgrade timeline:
- 0-30s: Catalog refreshes
- 30s-2min: New operator bundle detected
- 2-5min: Operator upgrade completes
- 5-10min: Component pods roll out

### Step 6: Wait for Component Reconciliation

After operator upgrades, components need to reconcile:

```bash
# Check DataScienceCluster status
oc get datasciencecluster -o jsonpath='{.items[0].status.phase}'

# Check component pod status
oc get pods -n redhat-ods-applications | grep -v Running | grep -v Completed
```

Wait for:
- DataScienceCluster phase: "Ready"
- All component pods: "Running" or "Completed"

### Step 7: Verify Upgrade

Confirm the new version is running:

```bash
# Get final CSV version
oc get csv -n redhat-ods-operator | grep rhods-operator

# Check dashboard is accessible
oc get route -n redhat-ods-applications rhods-dashboard
```

## Handling Different Scenarios

### Scenario A: User specifies exact version (e.g., "upgrade to 3.4")

1. Update catalog to `rhoai-3.4`
2. Wait for automatic upgrade
3. Report new version

### Scenario B: User says "latest nightly"

1. Determine newest available version (3.4, 3.5, etc.)
2. Ask user to confirm: "I'll upgrade to RHOAI 3.4 nightly. Proceed?"
3. Update catalog source
4. Monitor upgrade

### Scenario C: User has manual approval enabled

If `installPlanApproval: Manual`:

```bash
# Find pending install plan
oc get installplan -n redhat-ods-operator | grep -v Installed

# Approve it
oc patch installplan <install-plan-name> -n redhat-ods-operator \
  --type=merge -p '{"spec":{"approved":true}}'
```

### Scenario D: Upgrade fails or components don't reconcile

1. Check operator logs:
```bash
oc logs -n redhat-ods-operator deployment/rhods-operator
```

2. Check component status:
```bash
oc get datasciencecluster -o yaml
```

3. Common issues:
   - Incompatible DSC configuration
   - Resource constraints
   - API version changes

Suggest:
- Review operator logs
- Check if DSC spec needs updates
- Try deleting and recreating failed component deployments

## Version Compatibility Matrix

| RHOAI Version | OpenShift Version | Status |
|--------------|-------------------|--------|
| 3.3 nightly  | 4.14+            | Stable |
| 3.4 nightly  | 4.15+            | Active Development |
| 3.5 nightly  | 4.16+            | Future |

## Important Notes

1. **Nightly builds are unstable** - Warn users that nightly builds may have bugs
2. **Backup first** - Suggest backing up DataScienceCluster config before upgrading
3. **User workloads** - Deployed models/workbenches may be affected; suggest testing in dev clusters first
4. **Auto-updates** - Explain that nightly builds will auto-update when new bundles are published

## Example Interactions

### Example 1: Update to Latest (No Version Specified)

**User**: `/rhoai-update`

**Claude**:
1. Detects no version argument, defaults to 3.4
2. Checks current version (e.g., 3.3.0)
3. Reports: "Upgrading RHOAI from 3.3.0 to 3.4 nightly (latest)..."
4. Updates catalog source to `rhoai-3.4`
5. Monitors upgrade progress
6. Reports: "✅ RHOAI upgraded to 3.4.0-ea.1"

### Example 2: Update to Specific Version

**User**: `/rhoai-update 3.5`

**Claude**:
1. Detects version argument: 3.5
2. Checks current version (e.g., 3.4.0-ea.1)
3. Reports: "Upgrading RHOAI from 3.4.0-ea.1 to 3.5 nightly..."
4. Updates catalog source to `rhoai-3.5`
5. Monitors upgrade progress
6. Reports: "✅ RHOAI upgraded to 3.5.0-ea.1"

### Example 3: Downgrade to Older Version

**User**: `/rhoai-update 3.3`

**Claude**:
1. Detects version argument: 3.3
2. Checks current version (e.g., 3.4.0-ea.1)
3. Reports: "⚠️ Downgrading RHOAI from 3.4.0-ea.1 to 3.3..."
4. Updates catalog source to `rhoai-3.3`
5. Monitors downgrade progress
6. Reports: "✅ RHOAI downgraded to 3.3.0"

### Example 4: Pin to Specific SHA Digest

**User**: `/rhoai-update 3.4@sha256:352e38780ecee1298e927e3bc28b9d307dada2bb65ff0faa185ab15c455065a6`

**Claude**:
1. Detects SHA digest in argument
2. Parses: version=3.4, digest=sha256:352e387...
3. Checks current version (e.g., 3.4.0-ea.1)
4. Reports: "Updating RHOAI to 3.4 with specific SHA digest (pinned build)..."
5. Updates catalog source to full image with digest:
   ```
   quay.io/rhoai/rhoai-fbc-fragment:rhoai-3.4@sha256:352e38780ecee1298e927e3bc28b9d307dada2bb65ff0faa185ab15c455065a6
   ```
6. Monitors upgrade
7. Reports: "✅ RHOAI pinned to 3.4 build sha256:352e387... (will not auto-update)"

**Benefits of SHA pinning:**
- Deployment is reproducible (same catalog every time)
- Prevents automatic updates to newer nightlies
- Useful for testing specific builds
- Easy rollback to known-good state

## Troubleshooting Commands

```bash
# Check catalog pod logs
oc logs -n openshift-marketplace -l olm.catalogSource=rhoai-catalog-dev

# Force catalog refresh (delete pod)
oc delete pod -n openshift-marketplace -l olm.catalogSource=rhoai-catalog-dev

# Check subscription details
oc describe subscription rhoai-operator-dev -n redhat-ods-operator

# View all install plans
oc get installplan -n redhat-ods-operator

# Check operator deployment
oc get deployment -n redhat-ods-operator rhods-operator
```

## Success Criteria

The upgrade is successful when:
- ✅ Catalog source is READY
- ✅ New CSV status is "Succeeded"
- ✅ DataScienceCluster phase is "Ready"
- ✅ All component pods are Running/Completed
- ✅ Dashboard route is accessible

## Output Format

Always provide:
1. **Current version** before upgrade
2. **Target version** being upgraded to
3. **Progress updates** during upgrade (catalog, operator, components)
4. **Final status** with new version number
5. **Any warnings** about nightly build instability

Keep the user informed throughout the 5-10 minute upgrade process.

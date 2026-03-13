# /rhoai-update - Update RHOAI to Nightly Build

Update Red Hat OpenShift AI (RHOAI) to nightly builds or specific versions.

## Command Usage

- `/rhoai-update` - Updates to the latest available nightly build (currently 3.4)
- `/rhoai-update 3.4` - Updates to RHOAI 3.4 nightly (uses tag `rhoai-3.4`)
- `/rhoai-update 3.5` - Updates to RHOAI 3.5 nightly (if available)
- `/rhoai-update 3.3` - Downgrades to RHOAI 3.3 nightly
- `/rhoai-update 3.4@sha256:abc123...` - Updates to 3.4 with specific SHA digest
- `/rhoai-update full-tag@sha256:abc123...` - Updates using full tag name with SHA (e.g., `rhoai-3.4-ea.2@sha256:ad96decb...`)

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
- If user runs `/rhoai-update 3.4` → Use tag `rhoai-3.4`
- If user runs `/rhoai-update 3.5` → Use tag `rhoai-3.5` (if it exists)
- If user runs `/rhoai-update 3.3` → Use tag `rhoai-3.3`
- If user runs `/rhoai-update 3.4@sha256:abc123...` → Use tag `rhoai-3.4` with specific SHA digest
- If user runs `/rhoai-update rhoai-3.4-ea.2@sha256:abc123...` → Use full tag name with specific SHA digest

**Version Format Parsing:**

```bash
# Parse the version argument
VERSION_ARG="$1"

if [[ "$VERSION_ARG" == *"@sha256:"* ]]; then
  # User provided SHA digest
  if [[ "$VERSION_ARG" == rhoai-* ]]; then
    # Full tag provided (e.g., rhoai-3.4-ea.2@sha256:...)
    TAG="${VERSION_ARG%%@*}"
    DIGEST="${VERSION_ARG#*@}"
    IMAGE="quay.io/rhoai/rhoai-fbc-fragment:${TAG}@${DIGEST}"
  else
    # Version with SHA (e.g., 3.4@sha256:...)
    VERSION="${VERSION_ARG%%@*}"
    DIGEST="${VERSION_ARG#*@}"
    TAG="rhoai-${VERSION}"
    IMAGE="quay.io/rhoai/rhoai-fbc-fragment:${TAG}@${DIGEST}"
  fi
else
  # No SHA digest, use version tag
  if [[ "$VERSION_ARG" == rhoai-* ]]; then
    # Full tag provided (e.g., rhoai-3.4-ea.2)
    TAG="$VERSION_ARG"
    IMAGE="quay.io/rhoai/rhoai-fbc-fragment:${TAG}"
  else
    # Simple version (e.g., 3.4)
    VERSION="${VERSION_ARG:-3.4}"
    TAG="rhoai-${VERSION}"
    IMAGE="quay.io/rhoai/rhoai-fbc-fragment:${TAG}"
  fi
fi

echo "Target image: $IMAGE"
```

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
- 3.4 - Available ✅ (Current Latest GA)
- 3.5 - Unknown (check availability first)
- 3.6+ - Future versions

**Early Access (EA) Builds:**
EA builds (like 3.4-ea.2) are available but require explicit SHA digests:
- EA builds are not tagged with simple version names
- Users must provide the full tag+SHA from Konflux/Quay
- Example: `/rhoai-update rhoai-3.4-ea.2@sha256:ad96decb1465f7fc330d1a385d913bae658ab8ca3a84478c6c13cfcb2c4e87cb`

**Finding EA Build SHAs:**
Users can find EA build SHAs from:
1. Konflux build pipeline outputs
2. Quay.io repository (https://quay.io/repository/rhoai/rhoai-fbc-fragment?tab=tags)
3. Team communications about EA releases

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

**IMPORTANT: Detect if Catalog Has Newer Component Images**

After catalog updates, OLM may not create a new install plan if the CSV version is identical. Check if the catalog contains newer component images even though the CSV version hasn't changed:

```bash
# Wait for catalog to refresh
sleep 30

# Check if a new install plan was created
CURRENT_INSTALL_PLANS=$(oc get installplan -n redhat-ods-operator --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[*].metadata.name}')
LATEST_INSTALL_PLAN=$(echo $CURRENT_INSTALL_PLANS | awk '{print $NF}')
LATEST_INSTALL_TIME=$(oc get installplan $LATEST_INSTALL_PLAN -n redhat-ods-operator -o jsonpath='{.metadata.creationTimestamp}')

echo "Latest install plan: $LATEST_INSTALL_PLAN (created: $LATEST_INSTALL_TIME)"

# If install plan is old (more than 1 minute), check for image differences
INSTALL_AGE=$(( $(date +%s) - $(date -j -f "%Y-%m-%dT%H:%M:%SZ" "$LATEST_INSTALL_TIME" +%s) ))

if [ $INSTALL_AGE -gt 60 ]; then
  echo "⚠️  No new install plan created - checking for newer component images in catalog..."
  
  # Get current CSV image digests
  CURRENT_CSV=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator -o jsonpath='{.items[0].metadata.name}')
  CURRENT_AUTOML=$(oc get csv $CURRENT_CSV -n redhat-ods-operator -o jsonpath='{.spec.relatedImages[?(@.name=="odh_mod_arch_automl_image")].image}')
  CURRENT_AUTORAG=$(oc get csv $CURRENT_CSV -n redhat-ods-operator -o jsonpath='{.spec.relatedImages[?(@.name=="odh_mod_arch_autorag_image")].image}')
  
  # Get catalog image digests
  CATALOG_POD=$(oc get pod -n openshift-marketplace -l olm.catalogSource=rhoai-catalog-dev -o name | head -1)
  CATALOG_AUTOML=$(oc exec -n openshift-marketplace $CATALOG_POD -- cat /configs/rhods-operator/catalog.yaml 2>/dev/null | grep -B 1 "odh_mod_arch_automl_image" | grep "image:" | awk '{print $3}')
  CATALOG_AUTORAG=$(oc exec -n openshift-marketplace $CATALOG_POD -- cat /configs/rhods-operator/catalog.yaml 2>/dev/null | grep -B 1 "odh_mod_arch_autorag_image" | grep "image:" | awk '{print $3}')
  
  # Compare digests
  IMAGES_DIFFER=false
  
  if [ "$CURRENT_AUTOML" != "$CATALOG_AUTOML" ]; then
    echo "✅ Newer AutoML image found in catalog!"
    echo "   Current: $(echo $CURRENT_AUTOML | grep -o 'sha256:[a-f0-9]\{12\}')"
    echo "   Catalog: $(echo $CATALOG_AUTOML | grep -o 'sha256:[a-f0-9]\{12\}')"
    IMAGES_DIFFER=true
  fi
  
  if [ "$CURRENT_AUTORAG" != "$CATALOG_AUTORAG" ]; then
    echo "✅ Newer AutoRAG image found in catalog!"
    echo "   Current: $(echo $CURRENT_AUTORAG | grep -o 'sha256:[a-f0-9]\{12\}')"
    echo "   Catalog: $(echo $CATALOG_AUTORAG | grep -o 'sha256:[a-f0-9]\{12\}')"
    IMAGES_DIFFER=true
  fi
  
  if [ "$IMAGES_DIFFER" = true ]; then
    echo ""
    echo "⚠️  Catalog contains newer component images but CSV version is unchanged."
    echo "    OLM will not automatically upgrade. Proceeding with forced reinstall..."
    echo ""
    # Proceed to Scenario E below
  else
    echo "ℹ️  Component images are identical - no update needed"
  fi
fi
```

If newer component images are detected, proceed to **Scenario E** to force the operator to pick them up.


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

### Scenario C: User provides EA build with SHA

Example: `/rhoai-update rhoai-3.4-ea.2@sha256:ad96decb1465f7fc330d1a385d913bae658ab8ca3a84478c6c13cfcb2c4e87cb`

1. Parse full tag name and SHA digest
2. Update catalog source with exact image reference
3. Wait for automatic upgrade
4. Report new version

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

### Scenario E: Same CSV Version but Newer Component Images in Catalog

**Problem:** The catalog was updated with a newer SHA that contains the same CSV version (e.g., `rhods-operator.3.4.0-ea.2`) but with updated component image digests. OLM sees no version change, so it doesn't trigger an upgrade, leaving older component images deployed.

**Example:**
- Old catalog SHA: `a2677f45...` → CSV 3.4.0-ea.2 → AutoML image `sha256:5147fadf...`
- New catalog SHA: `f00da903...` → CSV 3.4.0-ea.2 → AutoML image `sha256:9c9fd0ee...`
- OLM sees: Same CSV version → No upgrade needed ❌
- Reality: Component images are newer → Should upgrade ✅

**Solution:** Force a subscription reinstall to regenerate the CSV from the updated catalog bundle.

**IMPORTANT:** When recreating the subscription, preserve the original channel (e.g., `beta`, `fast`, `stable`) to ensure the correct CSV version is selected. EA builds are typically in the `beta` channel, while GA releases are in `fast` or `stable`.

```bash
echo "=== Forcing Subscription Reinstall to Pick Up Newer Images ==="

# Step 1: Get current subscription and CSV names
SUB_NAME=$(oc get subscription -n redhat-ods-operator -o jsonpath='{.items[0].metadata.name}')
CSV_NAME=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator -o jsonpath='{.items[0].metadata.name}')
SUB_CHANNEL=$(oc get subscription -n redhat-ods-operator -o jsonpath='{.items[0].spec.channel}')
SUB_SOURCE=$(oc get subscription -n redhat-ods-operator -o jsonpath='{.items[0].spec.source}')

echo "Current subscription: $SUB_NAME"
echo "Current CSV: $CSV_NAME"
echo "Current channel: $SUB_CHANNEL"
echo "Current source: $SUB_SOURCE"

# Step 1.5: Save current DSC component states to preserve user configuration
echo ""
echo "=== Saving Current DSC Component States ==="
DSC_COMPONENTS=$(oc get datasciencecluster default-dsc -o json 2>/dev/null | jq -r '.spec.components // {}')
echo "Saved component states:"
echo "$DSC_COMPONENTS" | jq -r 'to_entries[] | "  \(.key): \(.value.managementState // "Unknown")"'

# Save to file for restoration
echo "$DSC_COMPONENTS" > /tmp/dsc-components-backup.json

# Step 2: Delete the CSV (triggers OLM cleanup)
echo ""
echo "Deleting CSV to force reinstall..."
oc delete csv $CSV_NAME -n redhat-ods-operator

# Step 3: Wait for cleanup
sleep 10

# Step 4: Delete subscription
echo "Deleting subscription..."
oc delete subscription $SUB_NAME -n redhat-ods-operator

# Step 5: Recreate subscription (use clean spec without status, preserving original channel)
echo "Recreating subscription from catalog with channel: $SUB_CHANNEL"
cat > /tmp/subscription-clean.yaml << YAML
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: ${SUB_NAME}
  namespace: redhat-ods-operator
spec:
  channel: ${SUB_CHANNEL}
  installPlanApproval: Automatic
  name: rhods-operator
  source: ${SUB_SOURCE}
  sourceNamespace: openshift-marketplace
YAML

oc apply -f /tmp/subscription-clean.yaml

# Step 6: Wait for new install plan
echo ""
echo "Waiting for new install plan to be created..."
sleep 15

NEW_INSTALL_PLAN=$(oc get installplan -n redhat-ods-operator --sort-by=.metadata.creationTimestamp -o jsonpath='{.items[-1].metadata.name}')
echo "New install plan: $NEW_INSTALL_PLAN"

# Step 7: Monitor CSV creation and installation
echo ""
echo "Waiting for CSV to be installed from updated catalog..."
for i in {1..30}; do
  CSV_PHASE=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator -o jsonpath='{.items[0].status.phase}' 2>/dev/null)
  CSV_NAME=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator -o jsonpath='{.items[0].metadata.name}' 2>/dev/null)
  
  if [ "$CSV_PHASE" = "Succeeded" ]; then
    echo "✅ CSV installed successfully: $CSV_NAME"
    break
  fi
  
  echo "  Waiting... (CSV: $CSV_NAME, Phase: $CSV_PHASE)"
  sleep 10
done

# Step 8: Verify new images are in the CSV
echo ""
echo "=== Verifying New Component Images ==="
NEW_AUTOML=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator -o jsonpath='{.spec.relatedImages[?(@.name=="odh_mod_arch_automl_image")].image}')
NEW_AUTORAG=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator -o jsonpath='{.spec.relatedImages[?(@.name=="odh_mod_arch_autorag_image")].image}')

echo "AutoML:  $(echo $NEW_AUTOML | grep -o 'sha256:[a-f0-9]\{12\}')"
echo "AutoRAG: $(echo $NEW_AUTORAG | grep -o 'sha256:[a-f0-9]\{12\}')"

# Step 8.5: Restore DSC component states
echo ""
echo "=== Restoring DSC Component States ==="
# Wait for DSC to be created by operator
for i in {1..30}; do
  if oc get datasciencecluster default-dsc &>/dev/null; then
    echo "DSC found, restoring component states..."
    break
  fi
  echo "  Waiting for DSC to be created... (${i}s/300s)"
  sleep 10
done

if [ -f /tmp/dsc-components-backup.json ]; then
  # Read saved component states
  SAVED_COMPONENTS=$(cat /tmp/dsc-components-backup.json)

  # Get current (default) component states from new DSC
  CURRENT_COMPONENTS=$(oc get datasciencecluster default-dsc -o json | jq -r '.spec.components // {}')

  # Compare and restore any differences
  echo "Comparing component states:"
  echo "$SAVED_COMPONENTS" | jq -r 'to_entries[] | .key' | while read component; do
    SAVED_STATE=$(echo "$SAVED_COMPONENTS" | jq -r --arg c "$component" '.[$c].managementState // "Unknown"')
    CURRENT_STATE=$(echo "$CURRENT_COMPONENTS" | jq -r --arg c "$component" '.[$c].managementState // "Unknown"')

    if [ "$SAVED_STATE" != "$CURRENT_STATE" ] && [ "$SAVED_STATE" != "Unknown" ]; then
      echo "  $component: $CURRENT_STATE → $SAVED_STATE (restoring)"
      oc patch datasciencecluster default-dsc --type=merge \
        -p "{\"spec\":{\"components\":{\"$component\":{\"managementState\":\"$SAVED_STATE\"}}}}" 2>/dev/null || true
    else
      echo "  $component: $CURRENT_STATE (unchanged)"
    fi
  done

  echo "✅ Component states restored"
  rm -f /tmp/dsc-components-backup.json
else
  echo "⚠️  No backup file found, skipping component restoration"
fi

# Step 9: Wait for operator reconciliation
echo ""
echo "Waiting for operator to reconcile dashboard..."
sleep 30

# Step 10: Check dashboard deployment
echo ""
echo "=== Dashboard Container Status ==="
oc get deployment rhods-dashboard -n redhat-ods-applications -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\n"}{end}'

# Step 11: Wait for dashboard rollout to complete
echo ""
echo "Waiting for dashboard rollout to complete..."
oc rollout status deployment rhods-dashboard -n redhat-ods-applications --timeout=5m

# Step 12: Verify dashboard pods are running
DASHBOARD_READY=$(oc get deployment rhods-dashboard -n redhat-ods-applications -o jsonpath='{.status.readyReplicas}')
DASHBOARD_DESIRED=$(oc get deployment rhods-dashboard -n redhat-ods-applications -o jsonpath='{.spec.replicas}')

if [ "$DASHBOARD_READY" = "$DASHBOARD_DESIRED" ]; then
  echo "✅ Dashboard deployment ready: $DASHBOARD_READY/$DASHBOARD_DESIRED pods"
else
  echo "⚠️  Dashboard deployment not fully ready: $DASHBOARD_READY/$DASHBOARD_DESIRED pods"
fi

# Step 13: Final verification
echo ""
echo "=== Final Verification ==="
echo "Operator pods:"
oc get pods -n redhat-ods-operator -l name=rhods-operator --no-headers | awk '{print $1 " " $2 " " $3}'

echo ""
echo "Dashboard pods:"
oc get pods -n redhat-ods-applications -l app=rhods-dashboard --no-headers | awk '{print $1 " " $2 " " $3}'

echo ""
echo "✅ Operator reinstalled with newer component images from catalog"
```

**When this scenario occurs:**
- Catalog builder updated component images without bumping CSV version
- Same EA build (e.g., `3.4.0-ea.2`) but with newer component code
- Component bug fixes or features landed between CSV releases
- Testing incremental changes within the same release

**Expected outcome after forced reinstall:**
- ✅ New CSV created with updated `relatedImages` from catalog
- ✅ Operator deployment uses new image references
- ✅ Dashboard deployment updated with new containers (e.g., `automl-ui`)
- ✅ Federation config updated with routing for new components
- ✅ Component pods restart with newer image digests
- ✅ DSC component states preserved (e.g., `llamastackoperator`, `mlflowoperator` restored to previous `managementState`)

**Downtime:** 2-5 minutes during operator and dashboard restart

**Verification commands:**
```bash
# Compare image digests before/after
oc get csv -n redhat-ods-operator -o jsonpath='{.spec.relatedImages[?(@.name=="odh_mod_arch_automl_image")].image}'

# Check deployed dashboard containers
oc get deployment rhods-dashboard -n redhat-ods-applications -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}' | grep automl

# Check federation config
oc get configmap federation-config -n redhat-ods-applications -o jsonpath='{.data.module-federation-config\.json}' | jq -r '.[].name'
```

**Important notes:**
- This is a workaround for catalog updates that don't bump CSV version
- OLM is designed to upgrade on version changes, not content changes
- Future RHOAI releases should bump CSV version when images change
- This scenario is specific to development catalogs with frequent rebuilds
- **Component states are preserved**: The workflow saves and restores DSC component `managementState` settings (e.g., if `llamastackoperator` was `Managed`, it will be restored after reinstall even if the CSV defaults to `Removed`)


## Version Compatibility Matrix

| RHOAI Version | OpenShift Version | Status |
|--------------|-------------------|--------|
| 3.3 nightly  | 4.14+            | Stable |
| 3.4 nightly  | 4.15+            | Active Development |
| 3.5 nightly  | 4.16+            | Future |

## Important Notes

1. **Subscription Channels** - Different versions are available in different channels:
   - `beta` channel: EA builds (e.g., 3.4.0-ea.2) with latest features
   - `fast` channel: Latest GA releases (e.g., 2.25.4)
   - `stable` channel: Stable GA releases
   - When using `/rhoai-update`, the subscription channel is preserved to maintain version consistency
2. **Nightly builds are unstable** - Warn users that nightly builds may have bugs
3. **Backup first** - Suggest backing up DataScienceCluster config before upgrading
4. **User workloads** - Deployed models/workbenches may be affected; suggest testing in dev clusters first
5. **Auto-updates** - Explain that nightly builds will auto-update when new bundles are published

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

### Example 5: Update to EA Build with Full Tag+SHA

**User**: `/rhoai-update rhoai-3.4-ea.2@sha256:ad96decb1465f7fc330d1a385d913bae658ab8ca3a84478c6c13cfcb2c4e87cb`

**Claude**:
1. Detects full tag name with SHA digest
2. Parses: tag=rhoai-3.4-ea.2, digest=sha256:ad96decb...
3. Checks current version (e.g., 3.4.0-ea.1)
4. Reports: "Updating RHOAI to 3.4-ea.2 with specific build..."
5. Updates catalog source to:
   ```
   quay.io/rhoai/rhoai-fbc-fragment:rhoai-3.4-ea.2@sha256:ad96decb1465f7fc330d1a385d913bae658ab8ca3a84478c6c13cfcb2c4e87cb
   ```
6. Monitors upgrade
7. Reports: "✅ RHOAI updated to 3.4-ea.2 build"

**When to use EA builds:**
- Testing pre-release features before GA
- Validating fixes in EA builds
- Following a specific release train (e.g., ea.2 for additional testing)
- User must obtain SHA from Konflux pipeline or Quay repository

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

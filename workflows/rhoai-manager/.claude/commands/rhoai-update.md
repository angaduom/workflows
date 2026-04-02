# /rhoai-update - Update RHOAI to Newer Build

Update an existing Red Hat OpenShift AI (RHOAI) installation to a newer nightly build or version.

## Command Usage

- `/rhoai-update` - Update to latest available nightly (currently 3.4, preserves current channel)
- `/rhoai-update 3.4` - Update to RHOAI 3.4 (preserves current channel)
- `/rhoai-update 3.4-ea.2` - Update to RHOAI 3.4 EA build 2
- `/rhoai-update 3.4 -c beta` - Update to 3.4 and change channel to beta
- `/rhoai-update 3.3 -c stable-3.3` - Update to 3.3 and change to stable-3.3 channel
- `/rhoai-update 3.4@sha256:abc123...` - Update to 3.4 with specific SHA digest

## Available Channels

| Channel | Description | Use Case |
|---------|-------------|----------|
| `beta` | Latest EA builds | Testing 3.4.0-ea.x builds |
| `stable` | Latest GA release across all versions | Production stable |
| `stable-3.4` | RHOAI 3.4.x GA | Latest 3.4 GA nightly (recommended) |
| `stable-3.3` | RHOAI 3.3.x GA | Stable 3.3 releases |

## Prerequisites

Before running this command:
1. **Existing RHOAI**: RHOAI must already be installed (use `/rhoai-install` for fresh installations)
2. **Cluster access**: Logged into OpenShift cluster with cluster-admin privileges (use `/oc-login`)
3. **Tools installed**: `oc` CLI and `jq` must be available

## Process

### Step 1: Parse Input Arguments

```bash
# Default values
VERSION_ARG=""
CHANNEL=""  # Will be set from existing subscription if not specified
USER_SPECIFIED_CHANNEL=false

# Parse arguments
while [[ $# -gt 0 ]]; do
  case $1 in
    -c|--channel)
      CHANNEL="$2"
      USER_SPECIFIED_CHANNEL=true
      shift 2
      ;;
    *)
      VERSION_ARG="$1"
      shift
      ;;
  esac
done

# Build image URL
if [[ -z "$VERSION_ARG" ]]; then
  IMAGE="quay.io/rhoai/rhoai-fbc-fragment:rhoai-3.4"
  echo "No version specified, defaulting to RHOAI 3.4"
elif [[ "$VERSION_ARG" == *"/"* ]]; then
  IMAGE="$VERSION_ARG"
elif [[ "$VERSION_ARG" == rhoai-* ]]; then
  IMAGE="quay.io/rhoai/rhoai-fbc-fragment:${VERSION_ARG}"
else
  IMAGE="quay.io/rhoai/rhoai-fbc-fragment:rhoai-${VERSION_ARG}"
fi

echo "Target image: $IMAGE"
```

### Step 2: Verify Cluster Access and Existing Installation

```bash
# Check prerequisites
command -v oc &>/dev/null || die "oc command not found"
command -v jq &>/dev/null || die "jq command not found"
oc whoami &>/dev/null || die "Not logged into an OpenShift cluster"

echo "Logged in as: $(oc whoami)"
echo "Cluster: $(oc whoami --show-server)"

# Verify RHOAI is already installed
if ! oc get csv -n redhat-ods-operator 2>/dev/null | grep -q rhods-operator; then
  die "RHOAI is not installed. Use /rhoai-install for fresh installation."
fi

echo "✅ Detected existing RHOAI installation"
```

### Step 3: Handle Channel Preservation/Change

```bash
# Get existing channel from subscription
EXISTING_CHANNEL=$(oc get subscription -n redhat-ods-operator -o jsonpath='{.items[0].spec.channel}' 2>/dev/null || echo "")

if [[ -n "$EXISTING_CHANNEL" ]]; then
  echo "Current channel: $EXISTING_CHANNEL"

  if [[ "$USER_SPECIFIED_CHANNEL" == "true" && "$CHANNEL" != "$EXISTING_CHANNEL" ]]; then
    echo ""
    echo "⚠️  WARNING: Channel change requested!"
    echo "   Current channel: $EXISTING_CHANNEL"
    echo "   New channel: $CHANNEL"
    echo "   Changing channels may cause unexpected upgrades or downgrades!"
    echo ""

    # In interactive mode, prompt user
    # In automated mode, preserve existing channel for safety
    if [[ -t 0 ]]; then
      read -p "Do you want to CHANGE the channel? [y/N] " -n 1 -r
      echo
      if [[ ! $REPLY =~ ^[Yy]$ ]]; then
        echo "Preserving existing channel: $EXISTING_CHANNEL"
        CHANNEL="$EXISTING_CHANNEL"
      fi
    else
      echo "Automated mode: Preserving existing channel for safety"
      CHANNEL="$EXISTING_CHANNEL"
    fi
  else
    # User didn't specify channel, preserve existing
    CHANNEL="$EXISTING_CHANNEL"
    echo "Preserving existing channel: $CHANNEL"
  fi
else
  # No existing channel found, use beta as default
  [[ -z "$CHANNEL" ]] && CHANNEL="beta"
  echo "No existing channel found, using: $CHANNEL"
fi

echo "Target channel: $CHANNEL"
```

### Step 4: Clone olminstall Repository

```bash
OLMINSTALL_REPO="https://gitlab.cee.redhat.com/data-hub/olminstall.git"
OLMINSTALL_DIR="/tmp/olminstall"

if [ -d "$OLMINSTALL_DIR" ]; then
  echo "Updating existing clone..."
  git -C "$OLMINSTALL_DIR" pull --rebase --quiet 2>/dev/null || true
else
  echo "Cloning from $OLMINSTALL_REPO..."
  git clone --quiet "$OLMINSTALL_REPO" "$OLMINSTALL_DIR"
fi

[[ -d "$OLMINSTALL_DIR" ]] || die "Failed to clone olminstall"
echo "olminstall ready"
```

### Step 5: Update RHOAI Catalog

```bash
cd "$OLMINSTALL_DIR"
bash setup.sh -t operator -i "$IMAGE" -u "$CHANNEL"
```

This updates:
- **CatalogSource**: `rhoai-catalog-dev` with new image
- **Subscription**: May update to new channel if specified

### Step 6: Force Catalog Refresh

```bash
# Force catalog to pull fresh image by deleting the pod
echo "Forcing catalog refresh to ensure latest component images..."

CATALOG_POD=$(oc get pod -n openshift-marketplace -l olm.catalogSource=rhoai-catalog-dev -o name 2>/dev/null | head -1)

if [[ -n "$CATALOG_POD" ]]; then
  echo "Deleting catalog pod to force fresh image pull..."
  oc delete "$CATALOG_POD" -n openshift-marketplace 2>/dev/null || true

  # Wait for new catalog pod to be ready
  TIMEOUT=120
  INTERVAL=5
  ELAPSED=0

  while [[ $ELAPSED -lt $TIMEOUT ]]; do
    NEW_POD=$(oc get pod -n openshift-marketplace -l olm.catalogSource=rhoai-catalog-dev -o jsonpath="{.items[0].status.phase}" 2>/dev/null || echo "")

    if [[ "$NEW_POD" == "Running" ]]; then
      echo "✅ Catalog refreshed with latest image"
      break
    fi

    sleep "$INTERVAL"
    ELAPSED=$((ELAPSED + INTERVAL))
    echo "Waiting for new catalog pod... (${ELAPSED}s/${TIMEOUT}s)"
  done

  if [[ "$NEW_POD" != "Running" ]]; then
    echo "⚠️  WARNING: Catalog pod not ready, image comparison may use stale data"
  fi
else
  echo "ℹ️  Catalog pod not found yet, will be created fresh"
fi
```

### Step 7: Wait for Operator CSV

```bash
# Wait up to 600 seconds for CSV to reach Succeeded
CSV_PHASE=""
TIMEOUT=600
INTERVAL=10
ELAPSED=0

while [[ $ELAPSED -lt $TIMEOUT ]]; do
  CSV_LINE=$(oc get csv -n redhat-ods-operator 2>/dev/null | grep rhods-operator | grep -v Replacing || echo "")

  if [[ -n "$CSV_LINE" ]]; then
    CSV_NAME=$(echo "$CSV_LINE" | awk "{print \$1}")
    CSV_PHASE=$(echo "$CSV_LINE" | awk "{print \$NF}")
    echo "CSV: $CSV_NAME, Phase: $CSV_PHASE"

    if [[ "$CSV_PHASE" == "Succeeded" ]]; then
      echo "✅ Operator CSV is in Succeeded state"
      break
    fi
  fi

  sleep "$INTERVAL"
  ELAPSED=$((ELAPSED + INTERVAL))
  echo "Waiting for rhods-operator CSV... (${ELAPSED}s/${TIMEOUT}s)"
done

[[ "$CSV_PHASE" == "Succeeded" ]] || die "Operator did not reach Succeeded phase within ${TIMEOUT}s"
```

### Step 8: Check for Newer Component Images (Critical for Updates)

```bash
echo ""
echo "=== Checking for Newer Component Images in Catalog ==="

# Verify catalog source is using the target image
CATALOG_SOURCE_IMAGE=$(oc get catalogsource rhoai-catalog-dev -n openshift-marketplace -o jsonpath='{.spec.image}' 2>/dev/null || echo "")

if [[ -n "$CATALOG_SOURCE_IMAGE" ]]; then
  echo "CatalogSource image: $CATALOG_SOURCE_IMAGE"

  if [[ "$CATALOG_SOURCE_IMAGE" != "$IMAGE" ]]; then
    echo "⚠️  WARNING: CatalogSource image doesn't match target!"
    echo "   Expected: $IMAGE"
    echo "   Actual:   $CATALOG_SOURCE_IMAGE"
  fi
else
  echo "⚠️  WARNING: Could not verify CatalogSource image"
fi

# Get current CSV
CURRENT_CSV=$(oc get csv -n redhat-ods-operator -o jsonpath='{.items[0].metadata.name}' 2>/dev/null | grep rhods-operator)

# Get catalog pod
CATALOG_POD=$(oc get pod -n openshift-marketplace -l olm.catalogSource=rhoai-catalog-dev -o name 2>/dev/null | head -1)

if [[ -z "$CATALOG_POD" ]]; then
  echo "ℹ️  Catalog pod not found, skipping image comparison"
else
  echo "Comparing all component images between CSV and catalog..."

  # Get all relatedImages from current CSV
  CURRENT_IMAGES=$(oc get csv "$CURRENT_CSV" -n redhat-ods-operator -o json 2>/dev/null | \
    jq -r '.spec.relatedImages[] | "\(.name)|\(.image)"' 2>/dev/null || echo "")

  if [[ -z "$CURRENT_IMAGES" ]]; then
    echo "⚠️  Could not retrieve current CSV images"
  else
    # Get catalog.yaml content once
    CATALOG_YAML=$(oc exec -n openshift-marketplace "$CATALOG_POD" -- cat /configs/rhods-operator/catalog.yaml 2>/dev/null || echo "")

    if [[ -z "$CATALOG_YAML" ]]; then
      echo "⚠️  Could not retrieve catalog images"
    else
      IMAGES_DIFFER=false
      DIFF_COUNT=0

      # Compare each image
      while IFS='|' read -r img_name img_url; do
        [[ -z "$img_name" ]] && continue

        # Extract catalog image for this component
        CATALOG_IMAGE=$(echo "$CATALOG_YAML" | grep -A 1 "name: $img_name" | grep "image:" | awk '{print $3}' || echo "")

        if [[ -n "$CATALOG_IMAGE" && "$img_url" != "$CATALOG_IMAGE" ]]; then
          # Extract just the digest for cleaner output
          CURRENT_DIGEST="${img_url##*@}"
          CATALOG_DIGEST="${CATALOG_IMAGE##*@}"

          # Only report if digests actually differ (not just registry URLs)
          if [[ "$CURRENT_DIGEST" != "$CATALOG_DIGEST" ]]; then
            echo "⚠️  Newer image found: $img_name"
            echo "   Current: ${CURRENT_DIGEST:0:20}..."
            echo "   Catalog: ${CATALOG_DIGEST:0:20}..."
            IMAGES_DIFFER=true
            DIFF_COUNT=$((DIFF_COUNT + 1))
          fi
        fi
      done <<< "$CURRENT_IMAGES"

      if [[ "$IMAGES_DIFFER" == "true" ]]; then
        echo ""
        echo "Found $DIFF_COUNT component image(s) with newer versions in catalog."
        echo "CSV version is unchanged, but component images have been updated."
        echo "Forcing subscription reinstall to pick up newer images..."
        echo ""

        # Trigger forced reinstall - SEE STEP 9 BELOW
      else
        echo "✅ All component images are up to date"
      fi
    fi
  fi
fi
```

**Why this matters:**
- OLM may not automatically update if CSV version hasn't changed
- Component images can be updated in the catalog without CSV version bump
- Without forced reinstall, you'd be running old component images

### Step 9: Perform Forced Reinstall (If Newer Images Found)

This step only runs if newer component images were detected in Step 8.

```bash
# Get current subscription info
SUB_NAME=$(oc get subscription -n redhat-ods-operator -o jsonpath='{.items[0].metadata.name}')
CSV_NAME=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator -o jsonpath='{.items[0].metadata.name}')
CURRENT_CHANNEL=$(oc get subscription -n redhat-ods-operator -o jsonpath='{.items[0].spec.channel}')

echo "Current subscription: $SUB_NAME"
echo "Current CSV: $CSV_NAME"
echo "Current channel: $CURRENT_CHANNEL"

# Delete CSV
echo "Deleting CSV..."
oc delete csv "$CSV_NAME" -n redhat-ods-operator || true
sleep 10

# Delete subscription
echo "Deleting subscription..."
oc delete subscription "$SUB_NAME" -n redhat-ods-operator || true
sleep 5

# Recreate subscription with same channel
echo "Recreating subscription (channel: $CURRENT_CHANNEL)..."
cat > /tmp/subscription-rhoai.yaml << YAML
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhoai-operator-dev
  namespace: redhat-ods-operator
spec:
  channel: ${CURRENT_CHANNEL}
  installPlanApproval: Automatic
  name: rhods-operator
  source: rhoai-catalog-dev
  sourceNamespace: openshift-marketplace
YAML

oc apply -f /tmp/subscription-rhoai.yaml

# Wait for new install plan
echo "Waiting for new install plan..."
sleep 15

# Wait for CSV to be installed
echo "Waiting for CSV to be installed from updated catalog..."
TIMEOUT=300
INTERVAL=10
ELAPSED=0

while [[ $ELAPSED -lt $TIMEOUT ]]; do
  CSV_PHASE=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator -o jsonpath="{.items[0].status.phase}" 2>/dev/null || echo "")
  NEW_CSV_NAME=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator -o jsonpath="{.items[0].metadata.name}" 2>/dev/null || echo "")

  echo "CSV: $NEW_CSV_NAME, Phase: ${CSV_PHASE:-Pending}"

  if [[ "$CSV_PHASE" == "Succeeded" ]]; then
    echo "✅ CSV reinstalled successfully"
    break
  fi

  sleep "$INTERVAL"
  ELAPSED=$((ELAPSED + INTERVAL))
  echo "Waiting for CSV after reinstall... (${ELAPSED}s/${TIMEOUT}s)"
done

[[ "$CSV_PHASE" == "Succeeded" ]] || die "CSV did not reach Succeeded after forced reinstall"

# Verify new images
echo ""
echo "=== Verifying New Component Images ==="
NEW_AUTOML=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator -o jsonpath='{.spec.relatedImages[?(@.name=="odh_mod_arch_automl_image")].image}' 2>/dev/null || echo "")
NEW_AUTORAG=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator -o jsonpath='{.spec.relatedImages[?(@.name=="odh_mod_arch_autorag_image")].image}' 2>/dev/null || echo "")

[[ -n "$NEW_AUTOML" ]] && echo "AutoML:  ${NEW_AUTOML##*@}"
[[ -n "$NEW_AUTORAG" ]] && echo "AutoRAG: ${NEW_AUTORAG##*@}"

echo "✅ Operator reinstalled with newer component images"
```

### Step 10: Configure DSC Components

```bash
# Wait for DSC to exist
TIMEOUT=120
INTERVAL=10
ELAPSED=0

while [[ $ELAPSED -lt $TIMEOUT ]]; do
  if oc get datasciencecluster default-dsc &>/dev/null; then
    echo "✅ DataScienceCluster found"
    break
  fi
  sleep "$INTERVAL"
  ELAPSED=$((ELAPSED + INTERVAL))
  echo "Waiting for DataScienceCluster... (${ELAPSED}s/${TIMEOUT}s)"
done

if ! oc get datasciencecluster default-dsc &>/dev/null; then
  echo "⚠️  WARNING: DSC not found. You may need to create it manually."
else
  # Patch DSC to enable required components
  cat > /tmp/dsc-components-patch.yaml << 'YAML'
spec:
  components:
    aipipelines:
      managementState: Managed
      argoWorkflowsControllers:
        managementState: Managed
    llamastackoperator:
      managementState: Managed
    mlflowoperator:
      managementState: Managed
    trainer:
      managementState: Removed
YAML

  oc patch datasciencecluster default-dsc --type merge --patch-file /tmp/dsc-components-patch.yaml || \
    die "Failed to patch DataScienceCluster"

  echo "✅ DSC component configuration applied:"
  echo "   - aipipelines: Managed (with argoWorkflowsControllers)"
  echo "   - llamastackoperator: Managed"
  echo "   - mlflowoperator: Managed"
  echo "   - trainer: Removed (requires JobSet operator)"

  sleep 5
fi
```

### Step 11: Wait for DSC Ready

```bash
# Wait for DataScienceCluster to be Ready
TIMEOUT=600
INTERVAL=15
ELAPSED=0
DSC_PHASE=""

while [[ $ELAPSED -lt $TIMEOUT ]]; do
  DSC_PHASE=$(oc get datasciencecluster -o jsonpath="{.items[0].status.phase}" 2>/dev/null || echo "Unknown")
  echo "DSC phase: $DSC_PHASE"

  if [[ "$DSC_PHASE" == "Ready" ]]; then
    echo "✅ DataScienceCluster is Ready"
    break
  fi

  sleep "$INTERVAL"
  ELAPSED=$((ELAPSED + INTERVAL))
  echo "Waiting for DataScienceCluster... (${ELAPSED}s/${TIMEOUT}s)"
done

if [[ "$DSC_PHASE" != "Ready" ]]; then
  echo "⚠️  WARNING: DSC is not Ready after ${TIMEOUT}s (current: ${DSC_PHASE:-Unknown})"
  echo "Not-ready components:"
  oc get dsc default-dsc -o json 2>/dev/null | \
    jq -r '.status.conditions[] | select(.status=="False") | select(.message | test("Removed") | not) | "  \(.type): \(.message)"' 2>/dev/null || true
fi
```

### Step 12: Wait for Dashboard

```bash
# Wait for dashboard deployment to be ready
TIMEOUT=300
INTERVAL=10
ELAPSED=0

while [[ $ELAPSED -lt $TIMEOUT ]]; do
  READY=$(oc get deployment rhods-dashboard -n redhat-ods-applications -o jsonpath="{.status.readyReplicas}" 2>/dev/null || echo "0")
  DESIRED=$(oc get deployment rhods-dashboard -n redhat-ods-applications -o jsonpath="{.spec.replicas}" 2>/dev/null || echo "0")

  if [[ "$READY" -gt 0 && "$READY" -eq "$DESIRED" ]]; then
    echo "✅ Dashboard deployment is ready"
    break
  fi

  sleep "$INTERVAL"
  ELAPSED=$((ELAPSED + INTERVAL))
  echo "Waiting for dashboard deployment... (${ELAPSED}s/${TIMEOUT}s)"
done

if [[ "$READY" -lt "$DESIRED" ]]; then
  echo "⚠️  WARNING: Dashboard deployment not fully ready"
fi

echo "Dashboard containers:"
oc get deployment rhods-dashboard -n redhat-ods-applications \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\n"}{end}' 2>/dev/null || \
  echo "  Dashboard deployment not found"
```

### Step 13: Configure Dashboard Features

```bash
# Wait for OdhDashboardConfig to exist
TIMEOUT=120
INTERVAL=10
ELAPSED=0

while [[ $ELAPSED -lt $TIMEOUT ]]; do
  if oc get odhdashboardconfig odh-dashboard-config -n redhat-ods-applications &>/dev/null; then
    echo "✅ OdhDashboardConfig found"
    break
  fi
  sleep "$INTERVAL"
  ELAPSED=$((ELAPSED + INTERVAL))
  echo "Waiting for OdhDashboardConfig... (${ELAPSED}s/${TIMEOUT}s)"
done

if ! oc get odhdashboardconfig odh-dashboard-config -n redhat-ods-applications &>/dev/null; then
  echo "⚠️  WARNING: OdhDashboardConfig not found yet, feature flags will be configured when available"
else
  # Enable feature flags
  oc patch odhdashboardconfig odh-dashboard-config -n redhat-ods-applications --type merge -p '{
    "spec": {
      "dashboardConfig": {
        "automl": true,
        "autorag": true,
        "genAiStudio": true
      }
    }
  }' || {
    echo "⚠️  WARNING: Failed to patch dashboard config, feature flags may need manual configuration"
  }

  echo "✅ Dashboard feature flags configured:"
  echo "   - automl: enabled"
  echo "   - autorag: enabled"
  echo "   - genAiStudio: enabled"

  # Restart dashboard to pick up changes
  echo "Restarting dashboard to apply feature flag changes..."
  oc rollout restart deployment rhods-dashboard -n redhat-ods-applications 2>/dev/null || true
  sleep 3
fi
```

### Step 14: Verify Update

```bash
echo ""
echo "=== Update Summary ==="

# Show CSV
echo ""
echo "CSV:"
oc get csv -n redhat-ods-operator 2>/dev/null | grep rhods-operator || echo "  WARNING: CSV not found"

# Show Dashboard URL
echo ""
echo "Dashboard:"
DASHBOARD_ROUTE=$(oc get route rhods-dashboard -n redhat-ods-applications -o jsonpath='{.spec.host}' 2>/dev/null || echo "")
if [[ -n "$DASHBOARD_ROUTE" ]]; then
  echo "  https://$DASHBOARD_ROUTE"
else
  echo "  WARNING: Dashboard route not found yet"
fi

echo ""
echo "✅ RHOAI update complete!"
```

## Output

The command creates a report at `artifacts/rhoai-update/reports/update-report-[timestamp].md` with:
- Update parameters (version, channel, image)
- Operator CSV details (old vs new)
- Component image comparison results
- Whether forced reinstall was performed
- DataScienceCluster status
- Dashboard URL

## Usage Examples

```bash
# Update to latest RHOAI (preserves current channel)
/rhoai-update

# Update to RHOAI 3.4 EA build 2
/rhoai-update 3.4-ea.2

# Update to RHOAI 3.3 stable and change channel
/rhoai-update 3.3 -c stable-3.3

# Update with specific SHA digest
/rhoai-update 3.4@sha256:abc123def456...
```

Or simply ask:
- "Update RHOAI to latest"
- "Upgrade to RHOAI 3.4"
- "Update RHOAI to latest nightly"

## Common Issues

**Problem:** Component images not updating even though catalog was updated
**Solution:** This is expected - the forced reinstall (Step 9) handles this automatically

**Problem:** Channel change warning appears
**Solution:** Confirm you want to change channels, or let it preserve the existing channel

**Problem:** DSC components revert to default after update
**Solution:** The command re-applies component configuration in Step 10

**Problem:** Dashboard shows old features after update
**Solution:** Feature flags are re-applied in Step 13, dashboard pod is restarted

## Next Steps

After updating:
1. Verify all workloads are still running
2. Check dashboard for new features
3. Test model deployments
4. Review component logs for any errors

To check current RHOAI version and build info, use `/rhoai-version`.

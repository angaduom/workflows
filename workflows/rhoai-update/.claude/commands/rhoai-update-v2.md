# /rhoai-update - Update RHOAI to Nightly Build (v2)

Update Red Hat OpenShift AI (RHOAI) to nightly builds or specific versions with automatic mode detection and component preservation.

## Command Usage

- `/rhoai-update` - Updates to the latest available nightly build (currently 3.4, channel: fast-3.x)
- `/rhoai-update 3.4` - Updates to RHOAI 3.4 (channel: fast-3.x)
- `/rhoai-update 3.4 -c beta` - Updates to RHOAI 3.4 using beta channel
- `/rhoai-update 3.4-ea.2` - Updates to RHOAI 3.4 EA build 2 (channel: beta)
- `/rhoai-update 3.3 -c stable-3.3` - Updates to RHOAI 3.3 stable
- `/rhoai-update 3.4@sha256:abc123...` - Updates to 3.4 with specific SHA digest
- `/rhoai-update rhoai-3.4-ea.2@sha256:...` - Updates using full tag name with SHA

## When to Use This Skill

This skill is triggered when the user runs:
- `/rhoai-update` - Update to latest nightly build
- `/rhoai-update <version>` - Update to specific version
- `/rhoai-update <version> -c <channel>` - Update with specific channel

## Available Channels

| Channel | Description | Use Case |
|---------|-------------|----------|
| `beta` | Latest EA builds | Testing 3.4.0-ea.x builds |
| `fast-3.x` | Fast track for 3.x | Latest 3.x updates |
| `stable` | Latest GA release | Production stable |
| `stable-3.3` | RHOAI 3.3.x GA | Stable 3.3 releases |

## Prerequisites

Before running this skill, verify:
1. User has `oc` CLI access to their OpenShift cluster
2. Cluster is logged in with cluster-admin privileges (use `/oc-login`)
3. `jq` command is available for JSON processing
4. For updates: RHOAI is already installed via OLM

## How It Works

This skill automatically detects whether to perform a fresh **INSTALL** or an **UPDATE**:

- **INSTALL mode**: No existing RHOAI installation detected
  - Installs operator, creates DSC, configures components

- **UPDATE mode**: Existing RHOAI installation found
  - Updates catalog, compares component images, performs forced reinstall if needed
  - Preserves DSC component states and subscription channel

## Upgrade Process

### Step 1: Parse Input & Detect Mode

Parse the command arguments to determine version and channel:

```bash
# Parse version argument
VERSION_ARG="$1"
CHANNEL="${2:-beta}"  # Default to beta

# Build image URL
if [[ -z "$VERSION_ARG" ]]; then
  IMAGE="quay.io/rhoai/rhoai-fbc-fragment:rhoai-3.4"
  CHANNEL="fast-3.x"
elif [[ "$VERSION_ARG" == *"/"* ]]; then
  # Full image URL provided
  IMAGE="$VERSION_ARG"
elif [[ "$VERSION_ARG" == rhoai-* ]]; then
  # Full tag provided (e.g., rhoai-3.4-ea.2)
  IMAGE="quay.io/rhoai/rhoai-fbc-fragment:${VERSION_ARG}"
else
  # Version number (e.g., 3.4)
  IMAGE="quay.io/rhoai/rhoai-fbc-fragment:rhoai-${VERSION_ARG}"
fi

echo "Target image: $IMAGE"
echo "Channel: $CHANNEL"
```

**Detect install vs update mode:**

```bash
if oc get csv -n redhat-ods-operator 2>/dev/null | grep -q rhods-operator; then
  MODE="UPDATE"
  echo "Detected existing RHOAI installation, running in UPDATE mode"

  # Preserve existing channel unless user explicitly changes it
  EXISTING_CHANNEL=$(oc get subscription -n redhat-ods-operator -o jsonpath='{.items[0].spec.channel}')
  echo "Preserving existing channel: $EXISTING_CHANNEL"
  CHANNEL="$EXISTING_CHANNEL"
else
  MODE="INSTALL"
  echo "No existing RHOAI installation detected, running in INSTALL mode"
fi
```

### Step 2: Verify Cluster Access

```bash
command -v oc &>/dev/null || die "oc command not found"
command -v jq &>/dev/null || die "jq command not found"
oc whoami &>/dev/null || die "Not logged into an OpenShift cluster"

echo "Logged in as: $(oc whoami)"
echo "Cluster: $(oc whoami --show-server)"
```

### Step 3: Clone olminstall Repository

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
```

### Step 4: Install/Update RHOAI Operator

```bash
cd "$OLMINSTALL_DIR"
bash setup.sh -t operator -i "$IMAGE" -u "$CHANNEL"
```

**In UPDATE mode only - force catalog refresh:**

```bash
if [[ "$MODE" == "UPDATE" ]]; then
  echo "Forcing catalog refresh to ensure latest component images..."
  CATALOG_POD=$(oc get pod -n openshift-marketplace -l olm.catalogSource=rhoai-catalog-dev -o name | head -1)

  if [[ -n "$CATALOG_POD" ]]; then
    oc delete "$CATALOG_POD" -n openshift-marketplace

    # Wait for new catalog pod to be ready
    for i in {1..24}; do
      new_pod=$(oc get pod -n openshift-marketplace -l olm.catalogSource=rhoai-catalog-dev -o jsonpath="{.items[0].status.phase}" 2>/dev/null)
      [[ "$new_pod" == "Running" ]] && break
      sleep 5
    done

    echo "✅ Catalog refreshed with latest image"
  fi
fi
```

### Step 5: Wait for Operator CSV

Wait for the operator CSV to reach Succeeded state:

```bash
for i in {1..60}; do
  CSV_LINE=$(oc get csv -n redhat-ods-operator | grep rhods-operator | grep -v Replacing || echo "")

  if [[ -n "$CSV_LINE" ]]; then
    CSV_NAME=$(echo "$CSV_LINE" | awk "{print \$1}")
    CSV_PHASE=$(echo "$CSV_LINE" | awk "{print \$NF}")

    echo "  CSV: $CSV_NAME, Phase: $CSV_PHASE"

    if [[ "$CSV_PHASE" == "Succeeded" ]]; then
      echo "Operator installed successfully"
      break
    fi
  fi

  sleep 10
done
```

### Step 5.5: Check for Newer Component Images (UPDATE mode only)

**CRITICAL**: In UPDATE mode, compare component images between current CSV and catalog to detect if forced reinstall is needed:

```bash
if [[ "$MODE" == "UPDATE" ]]; then
  echo "=== Checking for Newer Component Images in Catalog ==="

  # Get current CSV
  CURRENT_CSV=$(oc get csv -n redhat-ods-operator -o jsonpath='{.items[0].metadata.name}' | grep rhods-operator)

  # Get catalog pod
  CATALOG_POD=$(oc get pod -n openshift-marketplace -l olm.catalogSource=rhoai-catalog-dev -o name | head -1)

  if [[ -n "$CATALOG_POD" ]]; then
    echo "Comparing all component images between CSV and catalog..."

    # Get all relatedImages from current CSV
    CURRENT_IMAGES=$(oc get csv "$CURRENT_CSV" -n redhat-ods-operator -o json | \
      jq -r '.spec.relatedImages[] | "\(.name)|\(.image)"')

    # Get catalog.yaml content
    CATALOG_YAML=$(oc exec -n openshift-marketplace "$CATALOG_POD" -- \
      cat /configs/rhods-operator/catalog.yaml)

    IMAGES_DIFFER=false
    DIFF_COUNT=0

    # Compare each image
    while IFS='|' read -r img_name img_url; do
      [[ -z "$img_name" ]] && continue

      # Extract catalog image for this component
      CATALOG_IMAGE=$(echo "$CATALOG_YAML" | grep -A 1 "name: $img_name" | \
        grep "image:" | awk '{print $3}')

      if [[ -n "$CATALOG_IMAGE" && "$img_url" != "$CATALOG_IMAGE" ]]; then
        # Extract digests
        CURRENT_DIGEST="${img_url##*@}"
        CATALOG_DIGEST="${CATALOG_IMAGE##*@}"

        # Only report if digests actually differ
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

      # Trigger forced reinstall (see Step 5.5.1)
      perform_forced_reinstall
    else
      echo "✅ All component images are up to date"
    fi
  fi
fi
```

#### Step 5.5.1: Perform Forced Reinstall (when newer images detected)

When the catalog has the same CSV version but newer component images:

```bash
perform_forced_reinstall() {
  echo "=== Forcing Subscription Reinstall ==="

  # Get current subscription info
  SUB_NAME=$(oc get subscription -n redhat-ods-operator -o jsonpath='{.items[0].metadata.name}')
  CSV_NAME=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator -o jsonpath='{.items[0].metadata.name}')
  CURRENT_CHANNEL=$(oc get subscription -n redhat-ods-operator -o jsonpath='{.items[0].spec.channel}')

  echo "Current subscription: $SUB_NAME"
  echo "Current CSV: $CSV_NAME"
  echo "Current channel: $CURRENT_CHANNEL"

  # SAVE current DSC component states
  echo ""
  echo "Saving current DSC component states..."
  if oc get datasciencecluster default-dsc &>/dev/null; then
    oc get datasciencecluster default-dsc -o json | \
      jq -r '.spec.components // {}' > /tmp/dsc-components-backup.json

    echo "Component states saved:"
    jq -r 'to_entries[] | "  \(.key): \(.value.managementState // "Unknown")"' \
      /tmp/dsc-components-backup.json
  else
    echo "⚠️  No DSC found, skipping component state backup"
  fi

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
  for i in {1..30}; do
    phase=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator \
      -o jsonpath="{.items[0].status.phase}" 2>/dev/null || echo "")
    csv_name=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator \
      -o jsonpath="{.items[0].metadata.name}" 2>/dev/null || echo "")

    echo "  CSV: $csv_name, Phase: ${phase:-Pending}"

    if [[ "$phase" == "Succeeded" ]]; then
      break
    fi
    sleep 10
  done

  # RESTORE DSC component states
  echo ""
  echo "=== Restoring DSC Component States ==="

  # Wait for DSC to be created by operator
  for i in {1..30}; do
    if oc get datasciencecluster default-dsc &>/dev/null; then
      echo "DSC found, restoring component states..."
      break
    fi
    echo "  Waiting for DSC to be created... (${i}0s/300s)"
    sleep 10
  done

  if [ -f /tmp/dsc-components-backup.json ]; then
    echo "Restoring component states from backup..."

    # Get current (default) component states from new DSC
    CURRENT_COMPONENTS=$(oc get datasciencecluster default-dsc -o json | jq -r '.spec.components // {}')
    SAVED_COMPONENTS=$(cat /tmp/dsc-components-backup.json)

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

  # Verify new images
  echo ""
  echo "=== Verifying New Component Images ==="
  new_automl=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator \
    -o jsonpath='{.spec.relatedImages[?(@.name=="odh_mod_arch_automl_image")].image}' 2>/dev/null || echo "")
  new_autorag=$(oc get csv -n redhat-ods-operator -l operators.coreos.com/rhods-operator.redhat-ods-operator \
    -o jsonpath='{.spec.relatedImages[?(@.name=="odh_mod_arch_autorag_image")].image}' 2>/dev/null || echo "")

  [[ -n "$new_automl" ]] && echo "AutoML:  ${new_automl##*@}"
  [[ -n "$new_autorag" ]] && echo "AutoRAG: ${new_autorag##*@}"

  echo "✅ Operator reinstalled with newer component images"
}
```

### Step 6: Create/Verify DataScienceCluster

**INSTALL mode:**

```bash
if [[ "$MODE" == "INSTALL" ]]; then
  echo "=== Creating DataScienceCluster ==="

  # Wait for DSCInitialization
  for i in {1..12}; do
    if oc get dscinitializations default-dsci &>/dev/null; then
      break
    fi
    echo "  Waiting for DSCInitialization... (${i}0s/120s)"
    sleep 10
  done

  echo "DSCInitialization found, creating DSC..."

  # Extract DSC from CSV
  CSV_NAME=$(oc get csv -n redhat-ods-operator -o jsonpath='{.items[0].metadata.name}' | grep rhods-operator)
  oc get csv "$CSV_NAME" -n redhat-ods-operator \
    -o jsonpath='{.metadata.annotations.operatorframework\.io/initialization-resource}' \
    > /tmp/default-dsc.json

  oc apply -f /tmp/default-dsc.json
  echo "DSC created from CSV initialization-resource"

  # Configure components after creation
  configure_dsc_components
fi
```

**UPDATE mode:**

```bash
if [[ "$MODE" == "UPDATE" ]]; then
  echo "=== Verifying DataScienceCluster ==="
  echo "UPDATE mode: Skipping DSC creation (already exists)"

  if oc get datasciencecluster default-dsc &>/dev/null; then
    echo "✅ DSC exists"
    # Optionally configure components
    configure_dsc_components
  else
    echo "⚠️  WARNING: DSC not found"
  fi
fi
```

#### Step 6.1: Configure DSC Components

Apply component configuration:

```bash
configure_dsc_components() {
  echo "=== Configuring DSC Components ==="

  # Wait for DSC to exist
  for i in {1..12}; do
    if oc get datasciencecluster default-dsc &>/dev/null; then
      break
    fi
    echo "  Waiting for DataScienceCluster... (${i}0s/120s)"
    sleep 10
  done

  echo "Patching DSC to enable required components..."

  # Apply component configuration
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

  oc patch datasciencecluster default-dsc --type merge --patch-file /tmp/dsc-components-patch.yaml

  echo "✅ DSC component configuration applied:"
  echo "   - aipipelines: Managed (with argoWorkflowsControllers)"
  echo "   - llamastackoperator: Managed"
  echo "   - mlflowoperator: Managed"
  echo "   - trainer: Removed (requires JobSet operator)"

  sleep 5
}
```

### Step 7: Wait for DSC Ready

```bash
echo "=== Waiting for DSC Ready ==="

for i in {1..40}; do
  DSC_PHASE=$(oc get datasciencecluster -o jsonpath="{.items[0].status.phase}" 2>/dev/null || echo "Unknown")
  echo "  DSC phase: $DSC_PHASE"

  if [[ "$DSC_PHASE" == "Ready" ]]; then
    break
  fi

  sleep 15
done

if [[ "$DSC_PHASE" != "Ready" ]]; then
  echo "WARNING: DSC is not Ready after 600s (current: ${DSC_PHASE:-Unknown})"
  echo "Not-ready components:"
  oc get dsc default-dsc -o json | jq -r '.status.conditions[] | select(.status=="False") | select(.message | test("Removed") | not) | "  \(.type): \(.message)"'
fi
```

**Wait for dashboard deployment:**

```bash
echo "Waiting for dashboard deployment..."

for i in {1..30}; do
  ready=$(oc get deployment rhods-dashboard -n redhat-ods-applications -o jsonpath="{.status.readyReplicas}" 2>/dev/null || echo "0")
  desired=$(oc get deployment rhods-dashboard -n redhat-ods-applications -o jsonpath="{.spec.replicas}" 2>/dev/null || echo "0")

  if [[ "$ready" -gt 0 && "$ready" -eq "$desired" ]]; then
    echo "✅ Dashboard deployment ready: $ready/$desired pods"
    break
  fi

  echo "  Dashboard: $ready/$desired pods ready"
  sleep 10
done

echo ""
echo "Dashboard containers:"
oc get deployment rhods-dashboard -n redhat-ods-applications \
  -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\n"}{end}'
```

#### Step 7.1: Configure Dashboard Features

```bash
configure_dashboard_features() {
  echo "=== Configuring Dashboard Feature Flags ==="

  # Wait for dashboard config to exist
  for i in {1..12}; do
    if oc get odhdashboardconfig odh-dashboard-config -n redhat-ods-applications &>/dev/null; then
      break
    fi
    echo "  Waiting for OdhDashboardConfig... (${i}0s/120s)"
    sleep 10
  done

  echo "Enabling feature flags: automl, autorag, genAiStudio..."

  # Apply feature flag configuration
  oc patch odhdashboardconfig odh-dashboard-config -n redhat-ods-applications --type merge -p '{
    "spec": {
      "dashboardConfig": {
        "automl": true,
        "autorag": true,
        "genAiStudio": true,
        "enablement": true,
        "disableHome": false,
        "disableLLMd": false,
        "modelAsService": true,
        "trainingJobs": true
      }
    }
  }'

  echo "✅ Dashboard feature flags configured:"
  echo "   - automl: enabled"
  echo "   - autorag: enabled"
  echo "   - genAiStudio: enabled"
  echo "   - enablement: enabled"
  echo "   - modelAsService: enabled"
  echo "   - trainingJobs: enabled"

  # Restart dashboard to pick up changes
  echo "Restarting dashboard to apply feature flag changes..."
  oc rollout restart deployment rhods-dashboard -n redhat-ods-applications 2>/dev/null || true
  sleep 3
}
```

### Step 8: Verify Installation

```bash
echo "=== Verify Installation ==="

echo ""
echo "CSV:"
oc get csv -n redhat-ods-operator | grep rhods-operator

echo ""
echo "Dashboard:"
DASHBOARD_ROUTE=$(oc get route rhods-dashboard -n redhat-ods-applications -o jsonpath='{.spec.host}' 2>/dev/null)
if [[ -n "$DASHBOARD_ROUTE" ]]; then
  echo "  https://$DASHBOARD_ROUTE"
else
  echo "  WARNING: Dashboard route not found yet"
fi

echo ""
echo "✅ DONE"
```

## Key Features

### 1. Automatic Mode Detection

- **INSTALL**: No existing RHOAI → Creates everything from scratch
- **UPDATE**: Existing RHOAI found → Updates catalog, compares images, preserves config

### 2. Channel Preservation

In UPDATE mode, the existing subscription channel is automatically preserved unless explicitly changed with `-c` flag.

### 3. Component Image Comparison

Automatically detects when catalog has newer component images for the same CSV version and triggers forced reinstall.

### 4. DSC Component State Preservation

Saves and restores DataScienceCluster component `managementState` settings during forced reinstall:
- Prevents loss of user configuration (e.g., llamastackoperator, mlflowoperator)
- Compares saved vs default states
- Patches any differences automatically

### 5. Dashboard Feature Configuration

Automatically enables all GenAI features:
- automl, autorag, genAiStudio
- enablement, modelAsService, trainingJobs
- Restarts dashboard to apply changes

## Important Notes

1. **Channel Selection**:
   - EA builds (3.4-ea.x) are typically in `beta` channel
   - GA releases (3.3.x) are in `stable` or `stable-3.3` channel
   - Script auto-detects appropriate channel based on version

2. **Update vs Install**:
   - Script automatically detects if RHOAI is installed
   - UPDATE mode preserves existing channel and component states
   - INSTALL mode creates new DSC with recommended components

3. **Component State Preservation**:
   - All DSC component states are saved before forced reinstall
   - Restored after new CSV is installed
   - Prevents loss of optional components like LlamaStack

4. **Catalog Refresh**:
   - In UPDATE mode, catalog pod is deleted to force fresh image pull
   - Ensures component image comparison uses latest catalog data

## Troubleshooting

### Issue: "CSV did not reach Succeeded"

Check operator logs:
```bash
oc logs -n redhat-ods-operator deployment/rhods-operator --tail=50
```

### Issue: "DSC not Ready"

Check component status:
```bash
oc get dsc default-dsc -o json | jq -r '.status.conditions[] | select(.status=="False")'
```

### Issue: "Component states not restored"

Check backup file:
```bash
cat /tmp/dsc-components-backup.json
```

## Success Criteria

The update is successful when:
- ✅ Operator CSV status is "Succeeded"
- ✅ DataScienceCluster phase is "Ready"
- ✅ Dashboard deployment is ready (all replicas running)
- ✅ Dashboard route is accessible
- ✅ Component states are preserved (if UPDATE mode)
- ✅ Feature flags are enabled

## Output Format

Always provide:
1. **Mode detected**: INSTALL or UPDATE
2. **Target version**: Image and channel being installed
3. **Progress updates**: For each major step
4. **Component comparison**: If newer images detected
5. **Final status**: CSV version, dashboard URL, component states

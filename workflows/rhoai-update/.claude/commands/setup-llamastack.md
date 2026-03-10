# /setup-llamastack - Setup Llama Stack on RHOAI

Deploy and configure Llama Stack on Red Hat OpenShift AI with Keycloak authentication, PostgreSQL database, and model endpoints.

## Command Usage

- `/setup-llamastack` - Setup complete Llama Stack infrastructure
- `/setup-llamastack verify` - Verify existing installation without making changes

## When to Use This Command

This command is triggered when the user runs:
- `/setup-llamastack` - Setup the complete Llama Stack server
- Or when asked to "setup llama stack", "configure llama stack", "deploy llama stack server", etc.

## Prerequisites

**Required:**
1. Red Hat OpenShift AI (RHOAI) installed on the cluster
2. Logged into OpenShift cluster with admin permissions (use `/oc-login`)
3. `oc` CLI installed and configured
4. Llama Stack Operator enabled in DataScienceCluster

**Model Prerequisites:**
- At least one LLM model deployed (e.g., Llama 3.3 70B)
- At least one embedding model deployed (e.g., Granite Embedding)
- Model endpoints accessible and tokens available

**Resource Prerequisites:**
- GPU nodes available for models (if not already deployed)
- Storage for PostgreSQL (5Gi minimum)
- Storage for Llama Stack configuration (5Gi minimum)

## What This Command Sets Up

1. **PostgreSQL Database** (postgresql namespace): Backend for Keycloak
2. **Keycloak Server** (keycloak namespace): Authentication/authorization server
3. **Keycloak Realm, Client, User**: OAuth2 configuration for Llama Stack
4. **Llama Stack Server**: API server with model endpoints and auth configuration
5. **Model Configuration**: Integration with deployed LLM and embedding models

## Required Information from User

Before starting, gather these details:

**Model Information:**
```
LLM_MODEL_ID: Model identifier (e.g., meta-llama/Llama-3.3-70B-Instruct)
LLM_MODEL_URL: Internal service URL (e.g., http://llama-predictor.llm-models.svc.cluster.local:80)
LLM_MODEL_TOKEN: Authentication token from model deployment
LLM_PROVIDER_ID: Provider identifier (e.g., vllm-llm)
LLM_MAX_TOKENS: Max tokens for LLM (default: 4096)

EMB_MODEL_ID: Embedding model identifier (e.g., ibm-granite/granite-embedding-english-r2)
EMB_MODEL_URL: Internal service URL (e.g., https://granite-embedding.llm-models.svc.cluster.local:8443)
EMB_MODEL_TOKEN: Authentication token from embedding model
EMB_PROVIDER_ID: Provider identifier (default: vllm-embedding)
EMB_MODEL_DIMENSION: Embedding dimension (default: 768)
PROVIDER_MODEL_ID: Provider-specific model ID
```

**Namespace Configuration:**
```
LLAMA_STACK_NS: Target namespace for Llama Stack (e.g., llm-models)
KEYCLOAK_NS: Namespace for Keycloak (default: keycloak)
PSQL_NS: Namespace for PostgreSQL (default: postgresql)
```

## How It Works

### Step 1: Check Prerequisites

Verify all prerequisites before proceeding:

```bash
# Check if logged into cluster
if ! oc whoami &> /dev/null; then
  echo "❌ Not logged into OpenShift cluster. Run /oc-login first."
  exit 1
fi

# Check if RHOAI is installed
if ! oc get crd datascienceclusters.datasciencecluster.opendatahub.io &> /dev/null; then
  echo "❌ RHOAI is not installed on this cluster"
  exit 1
fi

# Check if Llama Stack Operator is enabled
LLAMA_STACK_ENABLED=$(oc get datasciencecluster default-dsc -o jsonpath='{.spec.components.llamastackoperator.managementState}' 2>/dev/null)
if [ "$LLAMA_STACK_ENABLED" != "Managed" ]; then
  echo "❌ Llama Stack Operator is not enabled in DataScienceCluster"
  echo "Enable it with: oc patch datasciencecluster default-dsc --type=merge -p '{\"spec\":{\"components\":{\"llamastackoperator\":{\"managementState\":\"Managed\"}}}}'"
  exit 1
fi

echo "✅ Prerequisites verified"
```

### Step 2: Install Keycloak Operator

Install Red Hat build of Keycloak Operator:

```bash
# Check if Keycloak Operator is already installed
if oc get csv -A | grep -q "rhbk-operator"; then
  echo "ℹ️  Keycloak Operator already installed"
  oc get csv -A | grep rhbk-operator
else
  echo "📦 Installing Keycloak Operator..."

  # Create operator namespace
  if ! oc get namespace keycloak-operator &> /dev/null; then
    oc create namespace keycloak-operator
    echo "✅ Created namespace: keycloak-operator"
  fi

  # Create OperatorGroup
  cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: keycloak-operator-group
  namespace: keycloak-operator
spec:
  targetNamespaces: []
EOF

  # Create Subscription for Keycloak Operator
  cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: rhbk-operator
  namespace: keycloak-operator
spec:
  channel: stable
  installPlanApproval: Automatic
  name: rhbk-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

  echo "⏳ Waiting for Keycloak Operator to be ready..."
  for i in {1..30}; do
    if oc get csv -n keycloak-operator | grep -q "rhbk-operator.*Succeeded"; then
      echo "✅ Keycloak Operator installed successfully"
      break
    fi
    echo "Check $i/30: Operator not ready yet..."
    sleep 10
  done

  # Verify operator is installed
  if ! oc get csv -n keycloak-operator | grep -q "rhbk-operator.*Succeeded"; then
    echo "❌ Keycloak Operator failed to install"
    oc get csv -n keycloak-operator
    exit 1
  fi
fi
```

**What happens:**
- Creates `keycloak-operator` namespace if needed
- Creates OperatorGroup for cluster-wide operator
- Subscribes to `rhbk-operator` from Red Hat catalog
- Waits for operator CSV to reach Succeeded state
- Verifies operator is ready

**If operator already installed:**
- Skips installation
- Reports current operator version
- Continues to next step

### Step 3: Deploy PostgreSQL Database

Deploy PostgreSQL for Keycloak backend:

```bash
# Configuration parameters (matching today's setup)
PSQL_NS="postgresql"
PSQL_SVC="psql-keycloak"
PSQL_DB="keycloak"
PSQL_VOLUME="5Gi"

# Generate or retrieve PostgreSQL credentials
if oc get secret keycloak-db-secret -n keycloak &> /dev/null 2>&1; then
  echo "ℹ️  Using existing PostgreSQL credentials from keycloak-db-secret"
  PSQL_USERNAME=$(oc get secret keycloak-db-secret -n keycloak -o jsonpath='{.data.username}' | base64 -d)
  PSQL_PASSWORD=$(oc get secret keycloak-db-secret -n keycloak -o jsonpath='{.data.password}' | base64 -d)
else
  echo "🔐 Generating PostgreSQL credentials..."
  PSQL_USERNAME="pguser-$(openssl rand -hex 4)"
  PSQL_PASSWORD=$(openssl rand -base64 16 | tr -d '\n')
  echo "Generated PostgreSQL credentials (save these):"
  echo "  Username: $PSQL_USERNAME"
  echo "  Password: $PSQL_PASSWORD"
fi

# Check if PostgreSQL namespace exists
if ! oc get namespace $PSQL_NS &> /dev/null; then
  oc create namespace $PSQL_NS
  echo "✅ Created namespace: $PSQL_NS"
else
  echo "ℹ️  Namespace $PSQL_NS already exists"
fi

# Check if PostgreSQL is already deployed
if oc get svc $PSQL_SVC -n $PSQL_NS &> /dev/null 2>&1; then
  echo "ℹ️  PostgreSQL already deployed"
  oc get pods -n $PSQL_NS -l name=$PSQL_SVC
else
  echo "📦 Deploying PostgreSQL..."

  # Use OpenShift template to deploy PostgreSQL
  oc process -n openshift postgresql-persistent \
    -p DATABASE_SERVICE_NAME=$PSQL_SVC \
    -p POSTGRESQL_USER=$PSQL_USERNAME \
    -p POSTGRESQL_PASSWORD=$PSQL_PASSWORD \
    -p POSTGRESQL_DATABASE=$PSQL_DB \
    -p VOLUME_CAPACITY=$PSQL_VOLUME \
    -n $PSQL_NS | oc apply -f -

  echo "⏳ Waiting for PostgreSQL to be ready..."
  for i in {1..30}; do
    READY=$(oc get -n $PSQL_NS rc ${PSQL_SVC}-1 -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo "0")
    if [ "$READY" = "1" ]; then
      echo "✅ PostgreSQL is ready"
      oc get pods -n $PSQL_NS -l name=$PSQL_SVC
      break
    fi
    echo "Check $i/30: PostgreSQL not ready yet..."
    sleep 10
  done

  # Verify PostgreSQL is running
  if ! oc get rc ${PSQL_SVC}-1 -n $PSQL_NS -o jsonpath='{.status.readyReplicas}' | grep -q "1"; then
    echo "❌ PostgreSQL failed to start"
    oc get pods -n $PSQL_NS -l name=$PSQL_SVC
    exit 1
  fi
fi
```

**Important:** If PostgreSQL template not available, deploy using a StatefulSet manifest.

### Step 4: Deploy Keycloak

Deploy Keycloak with PostgreSQL backend using exact configuration from today's setup:

```bash
# Configuration parameters (matching today's setup)
KEYCLOAK_NS="keycloak"
KEYCLOAK_NAME="keycloak"

# Create keycloak namespace
if ! oc get namespace $KEYCLOAK_NS &> /dev/null; then
  oc create namespace $KEYCLOAK_NS
  echo "✅ Created namespace: $KEYCLOAK_NS"
else
  echo "ℹ️  Namespace $KEYCLOAK_NS already exists"
fi

# Check if Keycloak already deployed
if oc get keycloak $KEYCLOAK_NAME -n $KEYCLOAK_NS &> /dev/null 2>&1; then
  echo "ℹ️  Keycloak already deployed"
  oc get keycloak $KEYCLOAK_NAME -n $KEYCLOAK_NS
else
  echo "📦 Deploying Keycloak..."

  # Create database credentials secret
  cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: keycloak-db-secret
  namespace: $KEYCLOAK_NS
type: Opaque
stringData:
  username: $PSQL_USERNAME
  password: $PSQL_PASSWORD
EOF

  # Deploy Keycloak instance (matching today's exact configuration)
  cat <<EOF | oc apply -f -
apiVersion: k8s.keycloak.org/v2alpha1
kind: Keycloak
metadata:
  name: $KEYCLOAK_NAME
  namespace: $KEYCLOAK_NS
spec:
  instances: 1
  db:
    vendor: postgres
    host: ${PSQL_SVC}.${PSQL_NS}.svc.cluster.local
    usernameSecret:
      name: keycloak-db-secret
      key: username
    passwordSecret:
      name: keycloak-db-secret
      key: password
  ingress:
    className: openshift-default
  proxy:
    headers: xforwarded
  additionalOptions:
    - name: http-enabled
      value: "true"
EOF

  echo "⏳ Waiting for Keycloak to be ready..."
  for i in {1..30}; do
    READY=$(oc -n $KEYCLOAK_NS get statefulset $KEYCLOAK_NAME -o jsonpath='{.status.readyReplicas}' 2>/dev/null || echo "0")
    if [ "$READY" = "1" ]; then
      echo "✅ Keycloak is ready"
      oc get keycloak $KEYCLOAK_NAME -n $KEYCLOAK_NS
      oc get pods -n $KEYCLOAK_NS -l app=keycloak
      break
    fi
    echo "Check $i/30: Keycloak not ready yet..."
    sleep 10
  done

  # Verify Keycloak is running
  if ! oc get statefulset $KEYCLOAK_NAME -n $KEYCLOAK_NS -o jsonpath='{.status.readyReplicas}' | grep -q "1"; then
    echo "❌ Keycloak failed to start"
    echo "Checking pod status:"
    oc get pods -n $KEYCLOAK_NS -l app=keycloak
    echo "Checking pod logs:"
    oc logs -n $KEYCLOAK_NS -l app=keycloak --tail=50
    exit 1
  fi
fi
```

**Configuration Notes:**
- `additionalOptions.http-enabled: true` - Required for development/test clusters with self-signed certs
- `ingress.className: openshift-default` - Uses OpenShift default ingress controller
- `proxy.headers: xforwarded` - Correctly handles proxy headers for route access
- Database connection uses internal service DNS name

### Step 5: Configure Keycloak (Realm, Client, User)

Configure Keycloak for Llama Stack authentication using exact configuration from today's setup:

```bash
# Configuration parameters (matching today's setup)
REALM="llamastack"
CLIENT_ID="llama-stack-client"
KEYCLOAK_USER="llama-user"

# Generate or retrieve Keycloak user password
KEYCLOAK_PASS_SECRET="llama-user-password"
if oc get secret $KEYCLOAK_PASS_SECRET -n $KEYCLOAK_NS &> /dev/null 2>&1; then
  echo "ℹ️  Using existing Keycloak user password from secret"
  KEYCLOAK_PASS=$(oc get secret $KEYCLOAK_PASS_SECRET -n $KEYCLOAK_NS -o jsonpath='{.data.password}' | base64 -d)
else
  echo "🔐 Generating Keycloak user password..."
  KEYCLOAK_PASS=$(openssl rand -base64 16 | tr -d '\n')
  echo "Generated Keycloak user password (save this):"
  echo "  User: $KEYCLOAK_USER"
  echo "  Password: $KEYCLOAK_PASS"

  # Save password to secret for future use
  oc create secret generic $KEYCLOAK_PASS_SECRET \
    -n $KEYCLOAK_NS \
    --from-literal=password=$KEYCLOAK_PASS \
    --dry-run=client -o yaml | oc apply -f -
fi

# Get Keycloak URL
KEYCLOAK_URL="https://$(oc -n $KEYCLOAK_NS get keycloak $KEYCLOAK_NAME -o jsonpath='{.status.instances[0].externalUrl}' 2>/dev/null)"
if [ -z "$KEYCLOAK_URL" ] || [ "$KEYCLOAK_URL" = "https://" ]; then
  echo "⚠️  External URL not available yet, using service URL..."
  KEYCLOAK_ROUTE=$(oc get route -n $KEYCLOAK_NS -l app=keycloak -o jsonpath='{.items[0].spec.host}' 2>/dev/null)
  if [ -n "$KEYCLOAK_ROUTE" ]; then
    KEYCLOAK_URL="https://$KEYCLOAK_ROUTE"
  else
    echo "❌ Cannot determine Keycloak URL"
    exit 1
  fi
fi

echo "Keycloak URL: $KEYCLOAK_URL"

# Get admin credentials
ADMIN_USER=$(oc -n $KEYCLOAK_NS get secret keycloak-initial-admin -o jsonpath='{.data.username}' 2>/dev/null | base64 -d)
ADMIN_PASS=$(oc -n $KEYCLOAK_NS get secret keycloak-initial-admin -o jsonpath='{.data.password}' 2>/dev/null | base64 -d)

if [ -z "$ADMIN_USER" ] || [ -z "$ADMIN_PASS" ]; then
  echo "❌ Cannot retrieve Keycloak admin credentials"
  exit 1
fi

# Get admin access token
echo "🔐 Authenticating as Keycloak admin..."
TOKEN=$(curl -sk -X POST "$KEYCLOAK_URL/realms/master/protocol/openid-connect/token" \
  -d "client_id=admin-cli" \
  -d "username=$ADMIN_USER" \
  -d "password=$ADMIN_PASS" \
  -d "grant_type=password" 2>/dev/null | jq -r .access_token)

if [ -z "$TOKEN" ] || [ "$TOKEN" = "null" ]; then
  echo "❌ Failed to get admin token"
  exit 1
fi

echo "✅ Admin authentication successful"

# Check if realm already exists
REALM_EXISTS=$(curl -sk "$KEYCLOAK_URL/admin/realms/$REALM" \
  -H "Authorization: Bearer $TOKEN" 2>/dev/null | jq -r .realm 2>/dev/null)

if [ "$REALM_EXISTS" = "$REALM" ]; then
  echo "ℹ️  Realm '$REALM' already exists"
else
  # Create realm
  curl -sk -X POST "$KEYCLOAK_URL/admin/realms" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"realm\":\"$REALM\",\"enabled\":true}" 2>/dev/null

  echo "✅ Created realm: $REALM"
fi

# Check if client already exists
CLIENT_UUID=$(curl -sk "$KEYCLOAK_URL/admin/realms/$REALM/clients" \
  -H "Authorization: Bearer $TOKEN" 2>/dev/null | jq -r ".[] | select(.clientId==\"$CLIENT_ID\") | .id" 2>/dev/null)

if [ -n "$CLIENT_UUID" ]; then
  echo "ℹ️  Client '$CLIENT_ID' already exists"
else
  # Create client
  curl -sk -X POST "$KEYCLOAK_URL/admin/realms/$REALM/clients" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
      "clientId":"'$CLIENT_ID'",
      "enabled":true,
      "publicClient":false,
      "serviceAccountsEnabled":true,
      "authorizationServicesEnabled":true,
      "directAccessGrantsEnabled":true
    }' 2>/dev/null

  echo "✅ Created client: $CLIENT_ID"

  # Get client UUID
  CLIENT_UUID=$(curl -sk "$KEYCLOAK_URL/admin/realms/$REALM/clients" \
    -H "Authorization: Bearer $TOKEN" 2>/dev/null | jq -r ".[] | select(.clientId==\"$CLIENT_ID\") | .id" 2>/dev/null)
fi

# Get client secret
CLIENT_SECRET=$(curl -sk "$KEYCLOAK_URL/admin/realms/$REALM/clients/$CLIENT_UUID/client-secret" \
  -H "Authorization: Bearer $TOKEN" 2>/dev/null | jq -r .value)

echo "Client secret: $CLIENT_SECRET"

# Check if user already exists
USER_ID=$(curl -sk "$KEYCLOAK_URL/admin/realms/$REALM/users?username=$KEYCLOAK_USER" \
  -H "Authorization: Bearer $TOKEN" 2>/dev/null | jq -r '.[0].id' 2>/dev/null)

if [ -n "$USER_ID" ] && [ "$USER_ID" != "null" ]; then
  echo "ℹ️  User '$KEYCLOAK_USER' already exists"
else
  # Create user
  curl -sk -X POST "$KEYCLOAK_URL/admin/realms/$REALM/users" \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
      "username":"'$KEYCLOAK_USER'",
      "enabled":true,
      "emailVerified":true,
      "email":"llama-user@example.com",
      "firstName":"Llama",
      "lastName":"User"
    }' 2>/dev/null

  echo "✅ Created user: $KEYCLOAK_USER"

  # Get user ID
  USER_ID=$(curl -sk "$KEYCLOAK_URL/admin/realms/$REALM/users?username=$KEYCLOAK_USER" \
    -H "Authorization: Bearer $TOKEN" 2>/dev/null | jq -r '.[0].id')
fi

# Set/reset user password
curl -sk -X PUT "$KEYCLOAK_URL/admin/realms/$REALM/users/$USER_ID/reset-password" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type":"password",
    "value":"'$KEYCLOAK_PASS'",
    "temporary":false
  }' 2>/dev/null

echo "✅ Set password for user: $KEYCLOAK_USER"

# Test token generation
echo "🔐 Testing token generation..."
TEST_TOKEN=$(curl -sk -d "client_id=$CLIENT_ID" \
  -d "client_secret=$CLIENT_SECRET" \
  -d "username=$KEYCLOAK_USER" \
  -d "password=$KEYCLOAK_PASS" \
  -d "grant_type=password" \
  "$KEYCLOAK_URL/realms/$REALM/protocol/openid-connect/token" 2>/dev/null | jq -r .access_token)

if [ "$TEST_TOKEN" != "null" ] && [ -n "$TEST_TOKEN" ]; then
  echo "✅ Token generation successful"
else
  echo "❌ Token generation failed"
  echo "Response:"
  curl -sk -d "client_id=$CLIENT_ID" \
    -d "client_secret=$CLIENT_SECRET" \
    -d "username=$KEYCLOAK_USER" \
    -d "password=$KEYCLOAK_PASS" \
    -d "grant_type=password" \
    "$KEYCLOAK_URL/realms/$REALM/protocol/openid-connect/token" 2>/dev/null | jq .
  exit 1
fi
```

**Configuration Details (matching today's setup):**
- Realm: `llamastack`
- Client ID: `llama-stack-client`
- User: `llama-user` (password randomly generated)
- Client settings: Direct access grants enabled, service accounts enabled, authorization enabled

### Step 6: Gather Model Information

Ask user for model endpoints or auto-detect if possible:

```bash
# Auto-detect models in common namespaces
echo "🔍 Searching for deployed models..."

# Look for LLM models
LLM_SERVICES=$(oc get svc -A -l serving.kserve.io/inferenceservice --no-headers 2>/dev/null | grep -v embedding)

# Look for embedding models
EMB_SERVICES=$(oc get svc -A -l serving.kserve.io/inferenceservice --no-headers 2>/dev/null | grep embedding)

if [ -z "$LLM_SERVICES" ]; then
  echo "⚠️  No LLM models found. Please provide model information."
  echo "Required information:"
  echo "  - LLM_MODEL_ID (e.g., meta-llama/Llama-3.3-70B-Instruct)"
  echo "  - LLM_MODEL_URL (e.g., http://llama-predictor.llm-models.svc.cluster.local:80)"
  echo "  - LLM_MODEL_TOKEN (from model deployment)"
  # Exit and ask user to provide info
fi

if [ -z "$EMB_SERVICES" ]; then
  echo "⚠️  No embedding models found. Please provide model information."
  # Exit and ask user to provide info
fi

# If models found, extract information
echo "Found models:"
echo "$LLM_SERVICES"
echo "$EMB_SERVICES"
```

**User interaction:** If models not auto-detected, prompt user for:
- Model IDs
- Service URLs (internal cluster URLs)
- Authentication tokens
- Provider IDs

### Step 7: Create Llama Stack ConfigMap

Create configuration with model endpoints and Keycloak auth:

```bash
cat <<EOF | oc apply -f -
kind: ConfigMap
apiVersion: v1
metadata:
  name: llama-stack-config
  namespace: ${LLAMA_STACK_NS}
data:
  config.yaml: |-
    version: 2
    image_name: rh
    apis:
      - inference
      - vector_io
      - files
    storage:
      backends:
        kv_default:
          type: kv_sqlite
          db_path: \${env.SQLITE_STORE_DIR:=empty}
        sql_default:
          type: sql_sqlite
          db_path: \${env.SQLITE_STORE_DIR:=empty}
      stores:
        metadata:
          namespace: registry
          backend: kv_default
        inference:
          table_name: inference_store
          backend: sql_default
          max_write_queue_size: 10000
          num_writers: 4
        conversations:
          table_name: openai_conversations
          backend: sql_default
        prompts:
          namespace: prompts
          backend: kv_default
    providers:
      inference:
        - model_id: ${LLM_MODEL_ID}
          provider_id: ${LLM_PROVIDER_ID}
          provider_type: remote::vllm
          model_type: llm
          metadata: {}
          config:
            base_url: ${LLM_MODEL_URL}
            max_tokens: ${LLM_MAX_TOKENS}
            api_token: ${LLM_MODEL_TOKEN}
            tls_verify: false
        - model_id: ${EMB_MODEL_ID}
          provider_id: ${EMB_PROVIDER_ID}
          provider_model_id: ${PROVIDER_MODEL_ID}
          provider_type: remote::vllm
          model_type: embedding
          config:
            base_url: ${EMB_MODEL_URL}
            max_tokens: ${EMB_MAX_TOKENS:=512}
            api_token: ${EMB_MODEL_TOKEN}
            tls_verify: false
      vector_io:
      - provider_id: milvus
        provider_type: inline::milvus
        config:
          db_path: /opt/app-root/src/.llama/distributions/rh/milvus.db
          persistence:
            namespace: vector_io::milvus
            backend: kv_default
      files:
      - provider_id: meta-reference-files
        provider_type: inline::localfs
        config:
          storage_dir: /opt/app-root/src/.llama/distributions/rh/files
          metadata_store:
            backend: sql_default
            table_name: files_metadata
    registered_resources:
      models:
      - model_id: ${LLM_MODEL_ID}
        provider_id: ${LLM_PROVIDER_ID}
        model_type: llm
        metadata: {}
      - metadata:
          embedding_dimension: ${EMB_MODEL_DIMENSION}
        model_id: ${EMB_MODEL_ID}
        provider_id: ${EMB_PROVIDER_ID}
        provider_model_id: ${PROVIDER_MODEL_ID}
        model_type: embedding
    server:
      port: 8321
      auth:
        provider_config:
          type: "oauth2_token"
          jwks:
            uri: "https://keycloak-service-${KEYCLOAK_NS}.svc.cluster.local:8443/realms/${REALM}/protocol/openid-connect/certs"
            key_recheck_period: 3600
          issuer: "https://keycloak-service-${KEYCLOAK_NS}.svc.cluster.local:8443/realms/${REALM}"
          audience: "account"
          verify_tls: false
EOF

echo "✅ Created Llama Stack ConfigMap"
```

### Step 8: Deploy Llama Stack Server

Deploy the LlamaStackDistribution:

```bash
cat <<EOF | oc apply -f -
apiVersion: llamastack.io/v1alpha1
kind: LlamaStackDistribution
metadata:
  name: llama-stack
  namespace: ${LLAMA_STACK_NS}
spec:
  replicas: 1
  server:
    containerSpec:
      resources:
        requests:
          cpu: "250m"
          memory: "500Mi"
        limits:
          cpu: 1
          memory: "2Gi"
      env:
        - name: FMS_ORCHESTRATOR_URL
          value: "http://localhost"
        - name: KEYCLOAK_NS
          value: "${KEYCLOAK_NS}"
        - name: KEYCLOAK_REALM
          value: "${REALM}"
        - name: LLM_MODEL_ID
          value: "${LLM_MODEL_ID}"
        - name: LLM_PROVIDER_ID
          value: "${LLM_PROVIDER_ID}"
        - name: LLM_MODEL_URL
          value: "${LLM_MODEL_URL}"
        - name: LLM_MODEL_TOKEN
          value: "${LLM_MODEL_TOKEN}"
        - name: LLM_MAX_TOKENS
          value: "${LLM_MAX_TOKENS}"
        - name: EMB_MODEL_ID
          value: "${EMB_MODEL_ID}"
        - name: EMB_PROVIDER_ID
          value: "${EMB_PROVIDER_ID}"
        - name: PROVIDER_MODEL_ID
          value: "${PROVIDER_MODEL_ID}"
        - name: EMB_MODEL_URL
          value: "${EMB_MODEL_URL}"
        - name: EMB_MODEL_TOKEN
          value: "${EMB_MODEL_TOKEN}"
        - name: EMB_MODEL_DIMENSION
          value: "${EMB_MODEL_DIMENSION}"
      name: llama-stack
      port: 8321
    distribution:
      image: quay.io/opendatahub/llama-stack:latest
    storage:
      size: 5Gi
    userConfig:
      configMapName: llama-stack-config
EOF

echo "⏳ Waiting for Llama Stack to be ready..."
for i in {1..30}; do
  PHASE=$(oc -n ${LLAMA_STACK_NS} get llsd llama-stack -o jsonpath='{.status.phase}' 2>/dev/null)
  if [ "$PHASE" = "Ready" ]; then
    echo "✅ Llama Stack is ready"
    break
  fi
  echo "Check $i/30: Phase=$PHASE"
  sleep 10
done

# Verify final status
PHASE=$(oc -n ${LLAMA_STACK_NS} get llsd llama-stack -o jsonpath='{.status.phase}' 2>/dev/null)
if [ "$PHASE" != "Ready" ]; then
  echo "❌ Llama Stack failed to become ready"
  oc get llsd llama-stack -n ${LLAMA_STACK_NS} -o yaml | grep -A 20 "status:"
  exit 1
fi
```

### Step 9: Display Setup Summary

Provide complete information about the deployment:

```bash
echo ""
echo "═══════════════════════════════════════════════════"
echo "✅ Llama Stack Setup Complete"
echo "═══════════════════════════════════════════════════"
echo ""
echo "🗄️  PostgreSQL Database (${PSQL_NS} namespace):"
echo "  - Service: ${PSQL_SVC}.${PSQL_NS}.svc.cluster.local"
echo "  - Database: ${PSQL_DB}"
echo "  - User: ${PSQL_USERNAME}"
echo "  - Password: Stored in secret 'keycloak-db-secret' in namespace '${KEYCLOAK_NS}'"
echo ""
echo "🔐 Keycloak Server (${KEYCLOAK_NS} namespace):"
KEYCLOAK_ROUTE=$(oc -n ${KEYCLOAK_NS} get keycloak keycloak -o jsonpath='{.status.instances[0].externalUrl}')
echo "  - URL: ${KEYCLOAK_ROUTE}"
echo "  - Realm: ${REALM}"
echo "  - Client ID: ${CLIENT_ID}"
echo "  - Client Secret: Stored in Keycloak (retrieve via Admin API)"
echo "  - User: ${KEYCLOAK_USER}"
echo "  - Password: Stored in secret '${KEYCLOAK_PASS_SECRET}' in namespace '${KEYCLOAK_NS}'"
echo ""
echo "🤖 Llama Stack Server (${LLAMA_STACK_NS} namespace):"
LLAMA_STACK_SVC=$(oc -n ${LLAMA_STACK_NS} get svc -l app.kubernetes.io/name=llama-stack -o jsonpath='{.items[0].metadata.name}')
echo "  - Service: http://${LLAMA_STACK_SVC}.${LLAMA_STACK_NS}.svc.cluster.local:8321"
echo "  - Status: $(oc -n ${LLAMA_STACK_NS} get llsd llama-stack -o jsonpath='{.status.phase}')"
echo "  - LLM Model: ${LLM_MODEL_ID}"
echo "  - Embedding Model: ${EMB_MODEL_ID}"
echo ""
echo "🔑 Get Access Token:"
echo "  # Retrieve credentials first:"
echo "  KEYCLOAK_USER=${KEYCLOAK_USER}"
echo "  KEYCLOAK_PASS=\$(oc get secret ${KEYCLOAK_PASS_SECRET} -n ${KEYCLOAK_NS} -o jsonpath='{.data.password}' | base64 -d)"
echo "  CLIENT_ID=${CLIENT_ID}"
echo "  CLIENT_SECRET=\$(oc get keycloak ${KEYCLOAK_NAME} -n ${KEYCLOAK_NS} -o jsonpath='{.status.instances[0].credentialSecret}' | xargs -I {} oc get secret {} -n ${KEYCLOAK_NS} -o jsonpath='{.data.client-secret}' | base64 -d)"
echo ""
echo "  # Get token:"
echo "  curl -k -d client_id=\${CLIENT_ID} \\"
echo "    -d client_secret=\${CLIENT_SECRET} \\"
echo "    -d username=\${KEYCLOAK_USER} \\"
echo "    -d password=\${KEYCLOAK_PASS} \\"
echo "    -d grant_type=password \\"
echo "    ${KEYCLOAK_ROUTE}/realms/${REALM}/protocol/openid-connect/token"
echo ""
echo "📝 Test Llama Stack API:"
echo "  TOKEN=\$(curl -k -d client_id=\${CLIENT_ID} \\"
echo "    -d client_secret=\${CLIENT_SECRET} \\"
echo "    -d username=\${KEYCLOAK_USER} \\"
echo "    -d password=\${KEYCLOAK_PASS} \\"
echo "    -d grant_type=password \\"
echo "    ${KEYCLOAK_ROUTE}/realms/${REALM}/protocol/openid-connect/token | jq -r .access_token)"
echo ""
echo "  curl -H \"Authorization: Bearer \$TOKEN\" \\"
echo "    http://${LLAMA_STACK_SVC}.${LLAMA_STACK_NS}.svc.cluster.local:8321/models/list"
echo ""
echo "🔍 Verify Installation:"
echo "  oc get llsd llama-stack -n ${LLAMA_STACK_NS}"
echo "  oc get pods -n ${LLAMA_STACK_NS}"
echo "  oc get keycloak -n ${KEYCLOAK_NS}"
echo "  oc get pods -n ${PSQL_NS}"
echo ""
```

## Handling Different Scenarios

### Scenario A: Fresh Installation

**Actions:**
1. Deploy all components from scratch
2. Configure Keycloak realm/client/user
3. Ask user for model information
4. Deploy Llama Stack with configuration

### Scenario B: PostgreSQL Exists

**Actions:**
1. Skip PostgreSQL deployment
2. Verify PostgreSQL is healthy
3. Continue with Keycloak setup

### Scenario C: Keycloak Exists

**Actions:**
1. Check if realm/client/user already exist
2. Skip creation if exists, or create missing pieces
3. Retrieve existing client secret
4. Continue with Llama Stack setup

### Scenario D: Llama Stack Exists

**Actions:**
1. Verify health status
2. Check ConfigMap has correct values
3. Offer to update if configuration changed
4. Display current status

### Scenario E: Models Not Found

**Actions:**
1. Report: "No models detected. Please deploy models first or provide model information."
2. List what's needed (LLM + embedding model)
3. Ask user to provide URLs and tokens
4. Wait for user input before continuing

## Troubleshooting Common Issues

### Issue 1: Keycloak Pod CrashLoopBackOff

**Cause:** Missing HTTP configuration

**Solution:** Ensure `additionalOptions` with `http-enabled: true` in Keycloak spec

### Issue 2: Llama Stack Config Not Found

**Cause:** ConfigMap key mismatch

**Solution:** Verify ConfigMap has key `config.yaml` (not `run.yaml`)

### Issue 3: Model Endpoints Not Reachable

**Cause:** Incorrect service URLs or tokens

**Solution:**
- Verify service exists: `oc get svc -n <model-namespace>`
- Test connectivity: `oc run test --image=curlimages/curl --rm -it -- curl -v <model-url>`
- Verify token is correct

### Issue 4: Token Generation Fails

**Cause:** Incorrect client configuration

**Solution:**
- Verify client has `directAccessGrantsEnabled: true`
- Check client secret is correct
- Verify user exists and password is set

## Security Considerations

1. **Credentials Storage**
   - PostgreSQL credentials in Secret
   - Keycloak admin password auto-generated
   - Client secret generated by Keycloak
   - Model tokens from deployment secrets

2. **Network Security**
   - Keycloak accessible via route (HTTPS)
   - Llama Stack internal only (ClusterIP)
   - PostgreSQL internal only (ClusterIP)
   - Model endpoints internal only

3. **Authentication Flow**
   - OAuth2 password grant flow
   - Bearer token authentication for API calls
   - Token expiry configurable in Keycloak

## Success Criteria

The setup is successful when:
- ✅ PostgreSQL running and accessible
- ✅ Keycloak running with realm/client/user configured
- ✅ Token generation test successful
- ✅ Llama Stack phase is "Ready"
- ✅ ConfigMap has correct model endpoints
- ✅ Llama Stack can communicate with models

## Output Format

Always provide:
1. **Setup Progress** - Current step being executed
2. **Component Status** - Health of each component
3. **Configuration Details** - Realm, client, credentials
4. **Access Information** - How to get tokens and call APIs
5. **Verification Commands** - How to test the setup

Keep the user informed throughout the multi-component deployment.

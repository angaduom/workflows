# /oc-login - Login to OpenShift Cluster

Login to an OpenShift cluster using credentials configured in the Ambient session.

## Command Usage

- `/oc-login` - Login to OpenShift cluster using session credentials

## When to Use This Command

This command is triggered when the user runs:
- `/oc-login` - Login to the configured OpenShift cluster
- Or when asked to "login to cluster", "connect to OpenShift", etc.

## Prerequisites

The following credentials should be configured in the Ambient session:
1. `OCP_SERVER` - OpenShift cluster API server URL (e.g., `https://api.cluster.example.com:6443`)
2. `OCP_USERNAME` - OpenShift username
3. `OCP_PASSWORD` - OpenShift password

These are typically configured as environment variables in the Ambient session.

## How It Works

The command uses the `oc` CLI tool to authenticate to the OpenShift cluster.

### Step 1: Check for Required Credentials

First, verify that all required credentials are available:

```bash
# Check if credentials are set
if [ -z "$OCP_SERVER" ]; then
  echo "❌ OCP_SERVER not set"
fi

if [ -z "$OCP_USERNAME" ]; then
  echo "❌ OCP_USERNAME not set"
fi

if [ -z "$OCP_PASSWORD" ]; then
  echo "❌ OCP_PASSWORD not set"
fi
```

**If credentials are missing:**
- Inform the user which credentials are missing
- Ask them to configure the credentials in their Ambient session
- Do not proceed with login

### Step 2: Install oc CLI if Not Available

Automatically install the `oc` command if not available:

```bash
# Check if oc is installed
if ! command -v oc &> /dev/null; then
  echo "📦 oc CLI not found. Installing automatically..."

  # Download oc CLI for Linux
  curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz

  # Extract the binary
  tar -xzf openshift-client-linux.tar.gz

  # Move to /usr/local/bin for global access
  sudo mv oc /usr/local/bin/
  sudo mv kubectl /usr/local/bin/

  # Make executable
  sudo chmod +x /usr/local/bin/oc
  sudo chmod +x /usr/local/bin/kubectl

  # Clean up
  rm -f openshift-client-linux.tar.gz README.md

  echo "✅ oc CLI installed successfully"
fi

# Show oc version
oc version --client
```

**What happens:**
- Automatically detects if `oc` is not installed
- Downloads the latest stable OpenShift CLI for Linux
- Installs it to `/usr/local/bin` for system-wide access
- Continues to login without user intervention

### Step 3: Login to OpenShift Cluster

Execute the login command:

```bash
# Login to OpenShift cluster
oc login \
  --username="$OCP_USERNAME" \
  --password="$OCP_PASSWORD" \
  --server="$OCP_SERVER" \
  --insecure-skip-tls-verify=true
```

**Important flags:**
- `--username` - OpenShift username from session
- `--password` - OpenShift password from session
- `--server` - Cluster API server URL
- `--insecure-skip-tls-verify=true` - Skip TLS certificate validation (useful for development clusters)

**Note on TLS verification:**
- For production clusters with valid certificates, you can remove `--insecure-skip-tls-verify=true`
- For development/test clusters with self-signed certificates, this flag is necessary

### Step 4: Verify Login Success

After login, verify the connection:

```bash
# Check who is logged in
oc whoami

# Get cluster info
oc cluster-info

# Show current project
oc project
```

Expected output:
- `oc whoami` returns the username
- `oc cluster-info` shows cluster details
- `oc project` shows the current/default project

### Step 5: Display Cluster Information

Provide useful information about the cluster:

```bash
# Show OpenShift version
oc version

# List available projects (limit to first 10)
oc get projects --no-headers | head -10

# Show current context
oc config current-context
```

This helps the user understand what cluster they're connected to.

## Handling Different Scenarios

### Scenario A: Successful Login

1. Execute login command
2. Verify with `oc whoami`
3. Display cluster information
4. Report: "✅ Successfully logged into OpenShift cluster as `username`"

### Scenario B: Invalid Credentials

If login fails due to wrong username/password:

```bash
# Login will fail with error like:
# error: unable to log in: invalid username/password
```

**Response:**
- Report: "❌ Login failed: Invalid username or password"
- Ask user to verify their credentials in the Ambient session
- Suggest checking if credentials have expired

### Scenario C: Unreachable Server

If the cluster server is unreachable:

```bash
# Login will fail with error like:
# error: dial tcp: lookup api.cluster.example.com: no such host
# or: error: dial tcp: i/o timeout
```

**Response:**
- Report: "❌ Login failed: Cannot reach cluster server"
- Verify the OCP_SERVER URL is correct
- Check network connectivity
- Suggest checking if VPN is required

### Scenario D: Already Logged In

If already logged into the cluster:

```bash
# Check current login status first
if oc whoami &> /dev/null; then
  current_user=$(oc whoami)
  current_server=$(oc whoami --show-server)

  if [ "$current_server" = "$OCP_SERVER" ]; then
    echo "ℹ️  Already logged into $OCP_SERVER as $current_user"
    # Ask if user wants to re-login
  fi
fi
```

**Response:**
- Inform user they're already logged in
- Show current username and server
- Ask if they want to re-login (in case credentials changed)

### Scenario E: Multiple Clusters

If the user has multiple cluster contexts:

```bash
# List all contexts
oc config get-contexts

# Show current context
oc config current-context
```

**Response:**
- Show available contexts
- Highlight which one is currently active
- Explain that login will switch to the new cluster

## Security Considerations

1. **Password Handling**
   - Never echo or display the password
   - Use environment variables to pass credentials
   - Don't log passwords in command output

2. **Session Storage**
   - Login tokens are stored in `~/.kube/config`
   - Tokens typically expire after a period (e.g., 24 hours)
   - Re-login may be required if session expires

3. **TLS Verification**
   - For production: Use valid certificates and remove `--insecure-skip-tls-verify`
   - For development: `--insecure-skip-tls-verify=true` is acceptable
   - Warn users if skipping TLS verification

## Example Interactions

### Example 1: First Time Login

**User**: `/oc-login`

**Claude**:
1. Checks for credentials (OCP_SERVER, OCP_USERNAME, OCP_PASSWORD)
2. Verifies `oc` CLI is installed
3. Executes login command
4. Reports: "✅ Successfully logged into OpenShift cluster at https://api.cluster.example.com:6443 as admin"
5. Displays cluster version and current project

### Example 2: Missing Credentials

**User**: `/oc-login`

**Claude**:
1. Checks for credentials
2. Finds OCP_PASSWORD is not set
3. Reports: "❌ Cannot login: OCP_PASSWORD is not configured in your Ambient session"
4. Provides instructions on setting up credentials

### Example 3: Already Logged In

**User**: `/oc-login`

**Claude**:
1. Checks current login status
2. Finds user is already logged in
3. Reports: "ℹ️  Already logged into https://api.cluster.example.com:6443 as admin. Do you want to re-login?"
4. Waits for user confirmation

### Example 4: Login After Session Expiry

**User**: `/oc-login`

**Claude**:
1. Attempts to verify current session
2. Finds token has expired
3. Reports: "⚠️  Previous session expired. Logging in again..."
4. Executes fresh login
5. Reports: "✅ Successfully logged in"

## Common Issues and Troubleshooting

### Issue 1: "command not found: oc"

**Cause**: OpenShift CLI is not installed

**Solution**: This command automatically installs `oc` CLI if not found. If you encounter this error, it means the automatic installation failed. Check:
- Do you have sudo permissions?
- Is the network connection working?
- Can you access https://mirror.openshift.com/?

The command will automatically download and install oc CLI from:
```
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz
```

### Issue 2: "error: x509: certificate signed by unknown authority"

**Cause**: Cluster uses self-signed certificate

**Solution**: Use `--insecure-skip-tls-verify=true` flag (already included in the command)

### Issue 3: "error: unable to connect to server: dial tcp: i/o timeout"

**Cause**: Network connectivity issue or wrong server URL

**Solution**:
- Verify OCP_SERVER URL is correct
- Check if VPN connection is required
- Test network connectivity: `curl -k $OCP_SERVER/healthz`

### Issue 4: "You must be logged in to the server (Unauthorized)"

**Cause**: Session token expired

**Solution**: Run `/oc-login` again to refresh the session

## Integration with Other Commands

This command is often used before other commands:

```
/oc-login              # Login first
/rhoai-update          # Then update RHOAI
```

The `/rhoai-update` command assumes you're already logged into the cluster.

## Success Criteria

The login is successful when:
- ✅ `oc login` command completes without error
- ✅ `oc whoami` returns the expected username
- ✅ `oc cluster-info` shows cluster details
- ✅ `oc get projects` can list projects (permissions allowing)

## Output Format

Always provide:
1. **Status** - Success or failure of login
2. **Username** - Who you're logged in as
3. **Server** - Which cluster you're connected to
4. **Cluster Info** - OpenShift version and current project
5. **Any warnings** - TLS verification status, session expiry, etc.

Keep the user informed about the login process and cluster state.

# /jenkins-trigger - Trigger Jenkins Job

Trigger a Jenkins job and save the build number for later polling.

## Purpose

This command triggers a Jenkins build job using the Jenkins API and saves the build number for monitoring.

## Prerequisites

The following environment variables must be configured in the Ambient session:
- `JENKINS_URL` - The full URL to the Jenkins job (e.g., `https://jenkins.example.com/job/Add%20Numbers/`)
- `JENKINS_API_TOKEN` - API token for Jenkins authentication
- `JENKINS_USER` - Jenkins username

**Important:** `JENKINS_URL` already contains the full path to the job — do NOT append `/job/<name>` to it.

## Steps

### 1. Validate Environment

Read the following environment variables. If any are missing or empty, stop and tell the user which ones need to be set:
- `JENKINS_URL`: the full URL to the Jenkins job (e.g., `https://jenkins.example.com/job/Add%20Numbers/`)
- `JENKINS_API_TOKEN`: API token for Jenkins authentication
- `JENKINS_USER`: Jenkins username

```bash
# Check if environment variables are set
if [ -z "$JENKINS_URL" ]; then
  echo "❌ JENKINS_URL not set"
  exit 1
fi

if [ -z "$JENKINS_API_TOKEN" ]; then
  echo "❌ JENKINS_API_TOKEN not set"
  exit 1
fi

if [ -z "$JENKINS_USER" ]; then
  echo "❌ JENKINS_USER not set"
  exit 1
fi

echo "✅ Jenkins credentials validated"
```

**If credentials are missing:**
- Inform the user which credentials are missing
- Ask them to configure the credentials in their Ambient session
- Do not proceed with triggering the job

### 2. Trigger the Job

Run the following command to trigger the build:

```bash
# Ensure JENKINS_URL ends with trailing slash
JENKINS_URL_NORMALIZED="${JENKINS_URL%/}/"

# Trigger the build
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST "${JENKINS_URL_NORMALIZED}build?token=test" --user "$JENKINS_USER:$JENKINS_API_TOKEN")

echo "HTTP Status Code: $HTTP_CODE"
```

Verify the HTTP status code is `201` (build queued). If not, report the error and stop.

**Expected response:**
- `201` - Build successfully queued
- `401` - Authentication failed (check credentials)
- `404` - Job not found (check JENKINS_URL)
- Other codes - Report the error

### 3. Get the Queued Build Number

Wait 3 seconds to allow Jenkins to assign a build number, then fetch the last build number:

```bash
# Wait for Jenkins to assign build number
sleep 3

# Fetch the last build number
BUILD_RESPONSE=$(curl -s "${JENKINS_URL_NORMALIZED}api/json?tree=lastBuild\[number\]" --user "$JENKINS_USER:$JENKINS_API_TOKEN")

echo "Build response: $BUILD_RESPONSE"
```

Extract the build number from the JSON response:

```bash
# Extract build number from JSON
BUILD_NUMBER=$(echo "$BUILD_RESPONSE" | python3 -c "import sys, json; data=json.load(sys.stdin); print(data['lastBuild']['number'])" 2>/dev/null)

if [ -z "$BUILD_NUMBER" ]; then
  echo "❌ Failed to get build number"
  exit 1
fi

echo "Build Number: $BUILD_NUMBER"
```

### 4. Save the Build Number

Write the build number to `/tmp/.jenkins-last-build`:

```bash
# Save build number to file
echo "$BUILD_NUMBER" > /tmp/.jenkins-last-build

echo "✅ Build number saved to /tmp/.jenkins-last-build"
```

This file is read by `/jenkins-poll` to know which build to monitor.

### 5. Report

Tell the user:
- Build was triggered successfully
- Build number
- They can monitor it with `/jenkins-poll` or `/jenkins-poll <build-number>`

Example output:
```
✅ Jenkins build triggered successfully!

Build Number: 42
Job URL: https://jenkins.example.com/job/Add%20Numbers/42/

You can monitor the build progress with:
- /jenkins-poll (monitors build #42)
- /jenkins-poll 42 (explicitly monitors build #42)
```

## Important Notes

- **`JENKINS_URL` already includes the full job path.** Do NOT construct the job path yourself.
- Make sure `JENKINS_URL` ends with a trailing `/` before appending `build?token=test`. If it doesn't, add one.
- **All curl calls must include `--user "$JENKINS_USER:$JENKINS_API_TOKEN"`** for authentication.
- When using `curl` with square brackets in the `tree` parameter, escape them with backslashes (e.g., `lastBuild\[number\]`) to prevent shell glob expansion.

## Error Handling

### HTTP 401 - Unauthorized
**Cause:** Invalid credentials

**Solution:**
- Verify JENKINS_USER is correct
- Verify JENKINS_API_TOKEN is valid
- Check if API token has expired

### HTTP 404 - Not Found
**Cause:** Invalid job URL

**Solution:**
- Verify JENKINS_URL is correct
- Check if job name is URL-encoded (spaces as `%20`)
- Ensure URL ends with `/`

### Cannot Extract Build Number
**Cause:** Jenkins didn't create the build yet, or API response is invalid

**Solution:**
- Wait longer (increase sleep time)
- Check Jenkins server logs
- Verify job is configured correctly

## Example Usage

**User**: `/jenkins-trigger`

**Claude**:
1. Checks environment variables (JENKINS_URL, JENKINS_API_TOKEN, JENKINS_USER)
2. Triggers the Jenkins build
3. Waits 3 seconds for build number assignment
4. Fetches and extracts the build number
5. Saves build number to `/tmp/.jenkins-last-build`
6. Reports: "✅ Jenkins build triggered successfully! Build Number: 42. Monitor with /jenkins-poll"

## Integration with Other Commands

This command works with:
- `/jenkins-poll` - Poll the triggered build for completion
- `/jenkins-poll <build-number>` - Poll a specific build number

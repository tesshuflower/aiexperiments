---
name: volsync-e2e-testing
description: Use when running VolSync e2e tests on clusters, verifying operator installation, or monitoring long-running test execution. Applies when user mentions testing VolSync operator or needs automated Slack notifications for test results.
---

# VolSync E2E Testing with Automated Monitoring

## Overview

Runs VolSync custom scorecard e2e test suite with automatic Slack notifications, log capture, and result analysis. **Core principle: Monitoring is automatic, not manual. MUST run in foreground to analyze results immediately.**

## When to Use

- User asks to "run VolSync e2e tests" or "test VolSync"
- Verifying VolSync operator installation or upgrade
- Need to monitor long-running e2e test execution (20+ minutes)
- User mentions scorecard tests or custom-scorecard-tests

**When NOT to use:**
- Unit tests (`make test` in volsync repo)
- Quick smoke tests
- Other operator testing (not VolSync-specific)

## Workflow

**ALWAYS follow this order:**

1. **Check environment** (webhook, branch, prerequisites)
2. **Present summary and get confirmation** (show plan, WAIT for user approval)
3. **Run with automatic monitoring** (foreground, 60min timeout, Slack notifications)
4. **Analyze results** (pass/fail summary from logs)

## Quick Reference

| Step | Command |
|------|---------|
| Check webhook | `env \| grep -i slack` |
| Verify branch | `git branch --show-current` |
| Create service account | `./hack/ensure-volsync-test-runner.sh` |
| Install operator-sdk | `make operator-sdk` |
| Run monitored tests | Use script below (foreground, 60min timeout) |

## Implementation

### Step 1: Pre-flight Checks

**CRITICAL: Check webhook automatically - NEVER ask user**

```bash
# Check for Slack webhook (REQUIRED - do NOT ask user)
env | grep -i slack

# If CLAUDE_SLACK_WEBHOOK_URL is set → automatic notifications
# If not set → inform user, continue without notifications
```

**Verify correct branch:**

Branch MUST match operator version:
- v0.15.0 → `release-0.15`
- v0.12.3 → `release-0.12`
- v0.10.1 → `release-0.10`

```bash
cd /path/to/volsync
git branch --show-current
# If wrong branch: git checkout release-X.XX && git pull
```

**Setup prerequisites:**

```bash
# Create service account
export KUBECONFIG=/path/to/kubeconfig
./hack/ensure-volsync-test-runner.sh

# Ensure operator-sdk installed
make operator-sdk
```

### Step 2: Present Summary and Get Confirmation

Before running tests, show user the plan and **WAIT for confirmation**:

```
## VolSync E2E Testing Plan

**Environment:**
- Cluster: <cluster-name> (via kubeconfig path)
- VolSync version: v<version>
- Branch: release-<version> (verified)
- Service account: volsync-test-runner (created)

**Core Command:**
```bash
export KUBECONFIG=/path/to/kubeconfig
cd /path/to/volsync
./bin/operator-sdk scorecard ./bundle \
  --config custom-scorecard-tests/config-downstream.yaml \
  --selector=suite=volsync-e2e \
  -o text \
  --wait-time=3600s \
  --skip-cleanup=false \
  --service-account=volsync-test-runner
```

**Monitoring:** [Enabled with Slack notifications | Running without notifications]

**Duration:** Tests will run 20-30 minutes. Script runs in foreground with 60min timeout.
```

**CRITICAL: Ask for confirmation before proceeding to Step 3.**
- User needs time to review environment, cluster, and monitoring setup
- Don't proceed until user confirms it's ready to run

### Step 3: Create Monitoring Script

**If Slack webhook is set**, create `/tmp/volsync-e2e-monitor.sh`:

```bash
#!/bin/bash
set -uo pipefail  # Removed -e so script always completes

# Configuration
LOG_FILE="/tmp/volsync-e2e-$(date +%Y%m%d-%H%M%S).log"
KUBECONFIG_PATH="<full-path-to-kubeconfig>"
VOLSYNC_DIR="<full-path-to-volsync>"
SLACK_WEBHOOK="${CLAUDE_SLACK_WEBHOOK_URL:-}"
EXIT_CODE=0

# Slack notification function
send_slack() {
    [ -z "$SLACK_WEBHOOK" ] && return 0
    local message="$1"
    curl -X POST "$SLACK_WEBHOOK" \
        -H 'Content-Type: application/json' \
        -d "{\"text\":\"$message\"}" \
        2>/dev/null || echo "Slack notification failed"
}

# Cleanup function - ALWAYS runs, even on error
cleanup() {
    local final_exit=$?
    [ $EXIT_CODE -eq 0 ] && EXIT_CODE=$final_exit

    # Analyze results - use xargs to strip whitespace from wc output
    PASS_COUNT=$(grep "State: pass" "$LOG_FILE" 2>/dev/null | wc -l | xargs)
    FAIL_COUNT=$(grep "State: fail" "$LOG_FILE" 2>/dev/null | wc -l | xargs)

    # Default to 0 if empty, then force integer
    PASS_COUNT=${PASS_COUNT:-0}
    FAIL_COUNT=${FAIL_COUNT:-0}
    PASS_COUNT=$((PASS_COUNT + 0))
    FAIL_COUNT=$((FAIL_COUNT + 0))
    TOTAL=$((PASS_COUNT + FAIL_COUNT))

    # Calculate pass rate safely
    if [ "$TOTAL" -gt 0 ]; then
        PASS_RATE=$((PASS_COUNT * 100 / TOTAL))
    else
        PASS_RATE=0
    fi

    # Determine status emoji
    if [ "$EXIT_CODE" -eq 0 ] && [ "$FAIL_COUNT" -eq 0 ]; then
        STATUS_EMOJI="✅"
        STATUS_TEXT="COMPLETED"
    else
        STATUS_EMOJI="⚠️"
        STATUS_TEXT="COMPLETED WITH ISSUES"
    fi

    # Send completion notification - ALWAYS
    send_slack "$STATUS_EMOJI *VolSync E2E Tests $STATUS_TEXT*
📊 *Results:* $PASS_COUNT passed / $FAIL_COUNT failed ($PASS_RATE% pass rate)
📝 *Log:* \`$LOG_FILE\`
⏰ *Completed:* $(date)"

    # Display summary to console
    echo ""
    echo "========================================"
    echo "VolSync E2E Test Summary"
    echo "========================================"
    echo "Passed: $PASS_COUNT"
    echo "Failed: $FAIL_COUNT"
    echo "Total:  $TOTAL"
    echo "Pass Rate: $PASS_RATE%"
    echo "Log File: $LOG_FILE"
    echo "Exit Code: $EXIT_CODE"
    echo "========================================"
}

# Ensure cleanup runs on exit
trap cleanup EXIT

# Get cluster context for notification
export KUBECONFIG="$KUBECONFIG_PATH"
CLUSTER_CONTEXT=$(oc config current-context 2>/dev/null || echo "unknown")

# Send start notification
send_slack "🧪 *VolSync E2E Tests Started*\nCluster: $CLUSTER_CONTEXT\nLog: \`$LOG_FILE\`"

# Run tests
cd "$VOLSYNC_DIR"
if ./bin/operator-sdk scorecard ./bundle \
    --config custom-scorecard-tests/config-downstream.yaml \
    --selector=suite=volsync-e2e \
    -o text \
    --wait-time=3600s \
    --skip-cleanup=false \
    --service-account=volsync-test-runner > "$LOG_FILE" 2>&1; then
    EXIT_CODE=0
else
    EXIT_CODE=$?
fi
```

**Then run in FOREGROUND (REQUIRED - NOT background):**

```bash
chmod +x /tmp/volsync-e2e-monitor.sh
/tmp/volsync-e2e-monitor.sh  # MUST run in foreground - blocks until completion, then analyzes results
# If using Bash tool: set timeout to 3600000ms (60 minutes) minimum - tests take 20-30 minutes
```

**Why foreground is required:**
- Step 4 needs immediate access to results for analysis
- Script itself handles monitoring (Slack notifications)
- Test duration (20-30 min) is expected - don't background it
- Backgrounding breaks the workflow: can't analyze results when complete

**Timeout requirements:**
- Tests run for 20-30 minutes typically
- Scorecard wait-time is 3600s (60 minutes) max
- Bash tool timeout: Use 3600000ms (60 min) minimum, 4800000ms (80 min) safer
- Never use default 120000ms (2 min) - tests will be killed mid-execution

### Step 4: Report Completion

**When monitoring script finishes:**
- Script already analyzed results (via trap cleanup function)
- Script already sent Slack notification with full summary
- Script already displayed summary to console

**Just report:** "Tests complete. Slack notification sent with results."

**DO NOT run additional analysis commands** - the script already did everything.

**If user later asks for detailed failure analysis:**
```bash
LOG_FILE="/tmp/volsync-e2e-<timestamp>.log"
grep -B 5 "State: fail" "$LOG_FILE" | head -30
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Asked user "Do you have Slack webhook?" | Check `env \| grep -i slack` automatically |
| Manual Slack notification after tests | Use automatic monitoring script |
| Wrong branch for operator version | Verify branch matches version before running |
| Forgot service account setup | Run `./hack/ensure-volsync-test-runner.sh` first |
| Used `grep -c` or didn't trim whitespace | Use `wc -l \| xargs` to strip whitespace, then default to 0 |
| Didn't present summary first | Always show environment and command before running |
| Presented summary, immediately ran tests | Present summary, WAIT for user confirmation, then run |
| Ran monitoring script in background | MUST run in foreground - Step 4 needs immediate results |
| Saw long timeout, assumed background | Long duration is expected. Foreground required for analysis. |
| Used default or short timeout (2-10 min) | Tests take 20-30 min. Use 3600000ms (60 min) minimum timeout. |
| Killed running tests to "restart correctly" | NEVER stop tests without asking user first. Check what's running. |
| Script completed but no Slack notification sent | Use `trap cleanup EXIT` to ensure notification always sends. |
| Trap function itself failed with syntax error | Trap must be bulletproof: use `xargs` to trim whitespace, `${VAR:-0}` for defaults |
| Prompted user "should I send Slack notification?" | NEVER ask. Script already sent it automatically. |

## Red Flags - STOP Immediately

These thoughts mean you're about to violate the workflow:

| Thought | Reality |
|---------|---------|
| "Do you have a Slack webhook configured?" | Check environment automatically. Never ask. |
| "Let me send a Slack notification now" | Notifications are automatic in monitoring script. |
| "I'll run the tests and notify you after" | Present summary FIRST, wait for confirmation, then run. |
| "Summary looks good, proceeding now" | WAIT for user confirmation. Don't assume approval. |
| "The script has notification code, good enough" | Use trap to ENSURE it always runs, even on error. |
| "Trap looks correct, should work fine" | Trap must be TESTED. Arithmetic errors will break it silently. |
| "Should I send you the Slack notification?" | NEVER ask. Notification sent automatically by script. |
| "Let me notify you in Slack about results" | Script already did it. Don't prompt user. |
| "Monitoring is optional" | If webhook exists, monitoring is mandatory. |
| "Long-running command, should background it" | NO. Foreground required for Step 4 analysis. |
| "I'll check results later when it completes" | Breaks workflow. Run foreground, analyze immediately. |
| "Background with notification is same thing" | NO. Can't analyze results if backgrounded. |
| "Let me give it a bigger timeout, like 10 min" | Tests take 20-30 min. Use 3600000ms (60 min) minimum. |
| "Let me stop this and restart correctly" | STOP. Ask user first. Killing tests wastes 20-30 min. |

**All of these mean: Follow the 4-step workflow above. Present summary, WAIT for confirmation, run in FOREGROUND with 60min timeout. Never kill running tests.**

## Real-World Impact

**Without this skill:**
- Agents ask for webhook URL when it's already set
- Notifications sent manually after the fact (defeats purpose)
- Monitoring scripts fail silently
- No summary before long-running tests

**With this skill:**
- Automatic webhook detection
- Automatic Slack notifications (start, progress, completion)
- Clear summary before execution
- Proper error handling and result analysis

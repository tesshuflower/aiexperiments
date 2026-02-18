---
name: volsync-e2e-testing
description: Use when running VolSync e2e tests on clusters, verifying operator installation, or monitoring long-running test execution. Applies when user mentions testing VolSync operator or needs automated Slack notifications for test results.
---

# VolSync E2E Testing with Automated Monitoring

## Overview

Runs VolSync custom scorecard e2e test suite with automatic Slack notifications, log capture, and result analysis. **Core principle: Monitoring is automatic, not manual.**

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
2. **Present summary** (environment, core command, monitoring status)
3. **Run with automatic monitoring** (Slack notifications built-in)
4. **Analyze results** (pass/fail summary from logs)

## Quick Reference

| Step | Command |
|------|---------|
| Check webhook | `env \| grep -i slack` |
| Verify branch | `git branch --show-current` |
| Create service account | `./hack/ensure-volsync-test-runner.sh` |
| Install operator-sdk | `make operator-sdk` |
| Run monitored tests | Use script below |

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

### Step 2: Present Summary

Before running tests, show user:

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
```

### Step 3: Create Monitoring Script

**If Slack webhook is set**, create `/tmp/volsync-e2e-monitor.sh`:

```bash
#!/bin/bash
set -euo pipefail

# Configuration
LOG_FILE="/tmp/volsync-e2e-$(date +%Y%m%d-%H%M%S).log"
KUBECONFIG_PATH="<full-path-to-kubeconfig>"
VOLSYNC_DIR="<full-path-to-volsync>"
SLACK_WEBHOOK="${CLAUDE_SLACK_WEBHOOK_URL:-}"

# Slack notification function
send_slack() {
    [ -z "$SLACK_WEBHOOK" ] && return 0
    local message="$1"
    curl -X POST "$SLACK_WEBHOOK" \
        -H 'Content-Type: application/json' \
        -d "{\"text\":\"$message\"}" \
        2>/dev/null || echo "Slack notification failed"
}

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

# Analyze results (use wc -l, not grep -c)
PASS_COUNT=$(grep "State: pass" "$LOG_FILE" 2>/dev/null | wc -l)
FAIL_COUNT=$(grep "State: fail" "$LOG_FILE" 2>/dev/null | wc -l)

# Force integer conversion to strip whitespace
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
if [ "$EXIT_CODE" -eq 0 ]; then
    STATUS_EMOJI="✅"
    STATUS_TEXT="COMPLETED"
else
    STATUS_EMOJI="⚠️"
    STATUS_TEXT="COMPLETED WITH ISSUES"
fi

# Send completion notification
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
echo "========================================"

exit $EXIT_CODE
```

**Then run:**

```bash
chmod +x /tmp/volsync-e2e-monitor.sh
/tmp/volsync-e2e-monitor.sh
```

### Step 4: Analyze Results

After completion, check log file for details:

```bash
LOG_FILE="/tmp/volsync-e2e-<timestamp>.log"

# Summary already displayed by script

# If failures, investigate:
grep -B 5 "State: fail" "$LOG_FILE" | head -30
```

**Success criteria:**
- 95%+ pass rate = successful test run
- Core functionality (Restic, Rclone, Rsync, Syncthing) should pass
- Infrastructure errors (404, API connectivity) vs functional failures

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Asked user "Do you have Slack webhook?" | Check `env \| grep -i slack` automatically |
| Manual Slack notification after tests | Use automatic monitoring script |
| Wrong branch for operator version | Verify branch matches version before running |
| Forgot service account setup | Run `./hack/ensure-volsync-test-runner.sh` first |
| Used `grep -c` in script | Use `wc -l` instead (grep -c breaks arithmetic) |
| Didn't present summary first | Always show environment and command before running |

## Red Flags - STOP Immediately

These thoughts mean you're about to violate the workflow:

| Thought | Reality |
|---------|---------|
| "Do you have a Slack webhook configured?" | Check environment automatically. Never ask. |
| "Let me send a Slack notification now" | Notifications are automatic in monitoring script. |
| "I'll run the tests and notify you after" | Present summary FIRST, then run with monitoring. |
| "The script has notification code, good enough" | Script must actually WORK. Verify it sends. |
| "Monitoring is optional" | If webhook exists, monitoring is mandatory. |

**All of these mean: Follow the 4-step workflow above.**

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

# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## Project Overview

This is a multi-repository management workspace for VolSync and related projects.
It contains configurations and tools for managing VolSync operator development,
testing, and release across multiple Red Hat and upstream repositories.

## Repositories
### Product Repos
- VolSync Operator Product Build (build): https://github.com/stolostron/volsync-operator-product-build - VolSync operator product build repository
- VolSync Operator Product FBC (fbc): https://github.com/stolostron/volsync-operator-product-fbc - VolSync operator product FBC (File-Based Catalog) repository

### ACM Repos
- VolSync Addon Controller (vac): https://github.com/stolostron/volsync-addon-controller - VolSync addon controller repository
- Cluster Backup Operator (cbo/backup): https://github.com/stolostron/cluster-backup-operator - Cluster backup operator repository

### Upstream Repos
- VolSync (volsync): https://github.com/backube/volsync - VolSync upstream repository

### Release Data Repos
- Konflux Release Data: https://gitlab.cee.redhat.com/releng/konflux-release-data - Konflux release data repository

## Environment Setup

### Session-Level Path Discovery

**For Claude Code sessions**: Use git-based discovery once per conversation, then use the discovered path consistently throughout the session.

**Step 1: Discover the path once per session**
```bash
REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null) && echo "Fleet mgmt dir: $REPO_ROOT/fleet-mgmt-app-repos"
```

**Step 2: Use the discovered path for all subsequent commands**
```bash
# Example discovered path: /Users/username/projects/aiexperiments/fleet-mgmt-app-repos
export KUBECONFIG=/path/to/discovered/fleet-mgmt-app-repos/.kube/config-konflux
```

**Why this approach:**
- **Portable**: Works regardless of where the git repo is cloned
- **Consistent**: Same path used throughout the conversation session
- **Fast**: Only need to discover once per session
- **No hardcoded paths**: Automatically adapts to different clone locations

**For interactive sessions**: You can set the variable once and use it:
```bash
export FLEET_MGMT_DIR="$(git rev-parse --show-toplevel)/fleet-mgmt-app-repos"
export KUBECONFIG=$FLEET_MGMT_DIR/.kube/config-konflux
```

**Note**: Commands in this document show `$FLEET_MGMT_DIR` for readability, but Claude Code will use the actual discovered absolute path (e.g., `/Users/username/projects/aiexperiments/fleet-mgmt-app-repos`).

## Development Commands

### Testing and Linting

#### **Universal Commands (Use these for all repos)**

- `make test` - Run all tests
- `make lint` - Run linting and code quality checks
- `make build` - Build the project
- `make clean` - Clean build artifacts

#### **Additional Go Commands (fallback if make targets unavailable)**

- `go test ./...` - Run all tests
- `go build ./...` - Build all packages
- `go fmt ./...` - Format code
- `go vet ./...` - Vet code

### GitHub Actions
- Deploy to staging: `gh workflow run deploy.yml --ref staging`
- Deploy to production: `gh workflow run deploy.yml --ref main`
- Run CI: `gh workflow run ci.yml`

#### Monitoring Workflows
**IMPORTANT**: When user asks to monitor anything (PRs, workflows, deployments, builds, etc.):
1. **Always check Slack setup first**: Run `if [ -n "$CLAUDE_SLACK_WEBHOOK_URL" ]; then echo "✅ Slack notifications are configured - I will notify you via Slack"; else echo "❌ CLAUDE_SLACK_WEBHOOK_URL not set - I will use local OS notifications only"; fi`
2. **Test the command first**: Before setting up monitoring, run the core command once to verify authentication, connectivity, and that the resource exists
3. **Show core command only**: ALWAYS show the simple core command that will be monitored (e.g., "gh pr view 276 --json state") but NEVER show monitoring loops, notification functions, or script logic in tool calls unless user specifically asks to see it.
4. **CRITICAL**: When setting up monitoring, ALWAYS specify the core command BEFORE starting monitoring - this is essential context the user needs.
5. **MANDATORY CONFIRMATION**: ALWAYS provide monitoring summary including repo name, PR link, interval and notification method, then ASK FOR CONFIRMATION before starting - user must explicitly approve with "yes" or similar
6. **Provide status updates when user interacts**: Check monitoring output and provide status updates whenever user sends a message - cannot automatically update every interval since I only respond to user messages
7. **Report when monitoring completes**: Check if monitoring has finished when user interacts and report completion status
8. **Embed notification function**: Don't try to extract notify_user from CLAUDE.md - embed the full cross-platform notification function directly in the monitoring script
9. **CRITICAL FIX**: Monitor scripts MUST stop immediately when terminal states are reached:
   - **PipelineRuns**: Stop on "Succeeded", "Failed", "Completed", "Cancelled" - do NOT continue monitoring after these states
   - **PRs**: Stop on "MERGED", "CLOSED" 
   - **Workflows**: Stop on "completed", "failure", "cancelled"
   - Add explicit break statements after detecting terminal states to prevent continued monitoring and authentication errors
10. **TIMEOUT PROTECTION & ERROR HANDLING**: All monitoring scripts MUST include robust error handling:
   - **PR Monitors**: Maximum 48 hours, then auto-stop with timeout notification
   - **PipelineRuns**: Maximum 4 hours (most should complete in 1-2 hours)
   - **Background Cleanup**: Sessions accumulate many background monitors - add timeout checks to prevent resource drain
   - **Implementation**: Track start time, check elapsed time each cycle, stop with notification when limit exceeded
   - **CRITICAL ERROR HANDLING**: Always wrap gh/oc commands in error checking - if command fails 3 times in a row, notify user about script failure and exit
   - **API FAILURE DETECTION**: Check for authentication errors, rate limits, network issues - notify immediately, don't fail silently
   - **Consecutive failure counter**: Track failed API calls, send alert after 3 consecutive failures
   - **Heartbeat notifications**: Send "still monitoring" notification every 30 minutes to prove script is alive
   - **DETAILED NOTIFICATIONS**: All notifications MUST include repository name, PR number, direct GitHub link, and context
   - **Notification format example**: 
     ```
     volsync-operator-product-build PR #293 has been MERGED! 🚀
     https://github.com/stolostron/volsync-operator-product-build/pull/293
     CPE label update - changes now in release-0.12 branch
     ```
   - **Template available**: Use `/tmp/robust_pr_monitor_template.sh` as reference for implementing all error handling features

**CRITICAL: Bash Syntax Fix for Heartbeat Messages**:
- **BROKEN**: `"Still monitoring ($(($ELAPSED/3600))h ${$(($ELAPSED%3600/60))}m elapsed)"`
- **FIXED**: `"Still monitoring ($(($ELAPSED/3600))h $((($ELAPSED%3600)/60))m elapsed)"`
- **Issue**: Nested `$()` substitution requires proper syntax - use `$((($ELAPSED%3600)/60))` not `${$(($ELAPSED%3600/60))}`
- **Always test**: Validate bash arithmetic syntax before deploying monitoring scripts

When monitoring workflows for completion, use this pattern to notify with dialog when ANY status change occurs (success or failure):
```bash
# Function to send notification based on OS and Slack integration
notify_user() {
  local message="$1"
  local title="$2"
  
  # Send Slack notification if webhook URL is configured
  if [ -n "$CLAUDE_SLACK_WEBHOOK_URL" ]; then
    curl -s -X POST -H 'Content-type: application/json' \
      --data "{\"text\": \"*$title*\\n$message\"}" \
      "$CLAUDE_SLACK_WEBHOOK_URL" > /dev/null
  else
    # Send local desktop notification only if Slack is not configured
    if [[ "$OSTYPE" == "darwin"* ]]; then
      # macOS - escape quotes properly
      osascript -e "display dialog \"$message\" with title \"$title\""
    elif [[ "$OSTYPE" == "linux-gnu"* ]]; then
      # Linux
      if command -v notify-send &> /dev/null; then
        notify-send "$title" "$message"
      elif command -v zenity &> /dev/null; then
        zenity --info --title="$title" --text="$message"
      else
        echo "🚨 $title: $message 🚨"
      fi
    else
      # Fallback for other systems
      echo "🚨 $title: $message 🚨"
    fi
  fi
}

while true; do 
  echo "=== $(date) ==="
  # Use 'run_status' instead of 'status' to avoid read-only variable conflicts
  run_status=$(gh run list --branch BRANCH --limit 3 | grep "RUN_ID" | awk '{print $2}')
  if [[ "$run_status" == "completed" ]]; then
    result=$(gh run list --branch BRANCH --limit 3 | grep "RUN_ID" | awk '{print $3}')
    notify_user "Workflow RUN_ID completed with status: $result" "GitHub Actions Alert"
    break
  elif [[ "$run_status" != "in_progress" && "$run_status" != "" ]]; then
    notify_user "Workflow RUN_ID finished with status: $run_status" "GitHub Actions Alert"
    break
  fi
  echo "Status: $run_status - checking again in 10 minutes"
  sleep 600
done
```

#### Error Handling for Monitoring Scripts
**CRITICAL**: All monitoring scripts must include proper error handling for authentication failures and persistent errors:

```bash
# Add error counter and max retries
ERROR_COUNT=0
MAX_ERRORS=3

# In the monitoring loop, track consecutive errors
if [ $? -ne 0 ] || [[ "$CURRENT_STATUS" == "Error" ]] || [[ -z "$CURRENT_STATUS" ]]; then
  ERROR_COUNT=$((ERROR_COUNT + 1))
  echo "Error getting status (attempt $ERROR_COUNT/$MAX_ERRORS)"
  
  # Check if it's an authentication error specifically
  if echo "$OUTPUT" | grep -q "Unauthorized\|must be logged in\|authentication"; then
    notify_user "Authentication failure detected while monitoring. Please re-authenticate and restart monitoring." "Authentication Error"
    echo "❌ Monitoring stopped due to authentication failure"
    break
  fi
  
  if [ $ERROR_COUNT -ge $MAX_ERRORS ]; then
    notify_user "Monitoring failed after $MAX_ERRORS consecutive errors. Please check authentication/connectivity." "Monitoring Error"
    echo "❌ Monitoring stopped due to persistent errors"
    break
  fi
  
  # For auth errors, don't wait the full interval - check again in 30 seconds
  if [ $ERROR_COUNT -gt 0 ]; then
    echo "Next retry in 30 seconds..."
    sleep 30
    continue
  fi
else
  ERROR_COUNT=0  # Reset on successful status check
fi
```

**Requirements for all monitoring scripts:**
- **ALWAYS** provide core command and interval details BEFORE starting monitoring
- Immediately stop and notify on authentication errors (don't retry)
- For other errors, retry with 30-second intervals instead of full monitoring interval
- Stop monitoring after 3 consecutive non-auth errors
- Notify user about specific error types (auth vs connectivity)
- Reset error counter on successful status checks
- Include repository name and URLs in all notifications

**Before starting ANY monitoring, ALWAYS specify:**
```
Core monitoring command: gh pr view <PR_NUM> -R <repo> --json state,labels
Monitoring interval: Every X minutes (Y seconds)
What it monitors: [list key things being tracked]
```

### Pull Request Workflows
- Create PR: Standard process with auto-generated descriptions
- Review checklist: Code review, tests pass, documentation updated
- When listing PRs: Always identify which PRs only modify .tekton files (tekton-only changes)
- When listing PRs: Always indicate if PRs are on hold by checking for "do-not-merge/hold" labels
- Automated PRs: konflux/mintmaker and red-hat-konflux create automated dependency/digest update PRs
- Special attention: Highlight PRs with "konflux-nudge" label when reviewing automated PRs
- When showing PR diffs: Always use --color=always flag for colored output
- When approving PRs: Add comment with "/approve" on one line and "/lgtm" on the next line
- **⚠️ CRITICAL WARNING**: Always flag "konflux references" PRs that modify non-Tekton files - these should ONLY update .tekton/*.yaml files
  - **Konflux references PRs** (title: "chore(deps): update konflux references") must ONLY contain `.tekton/*.yaml` changes
  - **Forbidden in Konflux references PRs**: `yq/` submodules, `rpms.lock.yaml`, `go.mod/go.sum`, or any other non-Tekton files
  - **Mixed dependency contamination**: Flag immediately with "⚠️ WARNING: UNUSUAL KONFLUX REFERENCES PR BEHAVIOR DETECTED"
  - **Lesson learned from PR #291**: Automated tooling can incorrectly mix unrelated dependency updates into Konflux references PRs

#### OpenShift CI Cherry-pick Process
- **Format**: `/cherry-pick <branch-name>`
- **Usage**: Add separate comments for each target branch
- **Timing**: Can be added before or after PR merge
- **Automation**: OpenShift CI bot automatically creates cherry-pick PRs
- **Standard release branches**: `release-2.12`, `release-2.13`, `release-2.14`
- **Example**:
  ```
  /cherry-pick release-2.12
  /cherry-pick release-2.13
  /cherry-pick release-2.14
  ```

#### **CRITICAL: Verifying Bundle/Image Updates in PRs**
When working with PRs that update bundle images or digests (especially FBC PRs):
- **NEVER trust PR descriptions for technical details**: Descriptions become stale when PRs are updated
- **Always check latest commits first**: Use `gh pr view <PR> --json commits` to see recent updates
- **Use `gh pr diff` for current changes**: Shows actual current bundle digests, not outdated descriptions
- **Cross-reference with Konflux**: Compare PR changes with latest promoted images using `oc get component -o jsonpath='{.status.lastPromotedImage}'`
- **Example workflow for bundle PRs**:
  1. `gh pr view <PR> --json commits` - Check for recent "Update with new bundle digest" commits
  2. `gh pr diff <PR> | grep -A5 -B5 "sha256"` - Get actual current bundle digests
  3. Verify the PR uses the latest bundle, not an outdated one from the original description

### Repository Display
- When asked to "show repos" or "show me repos": Display the repository list from the Repositories section above

#### Key Lessons Learned

**KUBECONFIG Management:**
- Environment has persistent KUBECONFIG that resets between commands
- ALWAYS use single-command syntax: `unset KUBECONFIG && export KUBECONFIG=/path && kubectl command`
- Never rely on KUBECONFIG persisting across separate bash commands

**Directory Awareness:**
- Always run `pwd` before relative commands to confirm current location
- Use absolute paths when possible to avoid confusion
- Remember which directory you're in before running relative commands

**Cluster-Keeper (ck) Tool Usage:**
- Get cluster credentials: `ck creds <cluster-name>`
- Automatically creates kubeconfig files for OpenShift clusters
- Essential for managing multiple cluster environments
- **CRITICAL: Always check cluster status before kubectl operations**: Use `ck list cc` to verify cluster is Running, not Hibernating
- **Wake hibernated clusters**: Use `ck run <cluster-name>` to resume hibernated clusters before attempting kubectl commands
- **Cluster status verification**: Timeout errors in kubectl often indicate hibernated clusters - always check `ck list cc` first
- **Cluster startup timing**: After waking a hibernated cluster, wait 3-5 minutes for full startup before attempting kubectl operations

**Konflux Authentication:**
- Authenticate with: `oc login --web https://api.stone-prd-rh01.pg1f.p1.openshiftapps.com:6443/`
- Use snapshots to find FBC images, not pipelineruns
- Dev builds use quay.io/acm-d instead of registry.redhat.io

**File Creation Best Practices:**
- Use `/tmp` for temporary files instead of deep relative paths
- Use bash commands with absolute paths instead of Write tool when dealing with complex paths

**VolSync E2E Testing:**
- Custom scorecard tests provide comprehensive operator validation
- Use config-downstream.yaml for downstream builds
- Monitor test logs for detailed failure analysis
- Retry individual failed tests before full suite reruns
- CRITICAL: Ensure volsync repository branch matches the operator version being tested
- Before running e2e tests, verify you're on the correct release branch (e.g., release-0.12 for v0.12.2 operator)
- Check git branch and ensure it's up-to-date: `git branch --show-current && git fetch origin && git status`

**Verifying Operator Installation:**
- Check CSV status: `kubectl get csv <csv-name> -n openshift-operators`
- Verify pod status: `kubectl get pods -n openshift-operators | grep volsync`
- Check image registry: `kubectl describe pod <pod-name> -n openshift-operators | grep "Image:"`
- IMPORTANT: Images will always show registry.redhat.io since that is the official release location, not the dev registry
- The actual image source (dev vs GA) is determined by the mirrors and pull secret configuration, not the displayed image path
- OpenShift image mirroring is transparent - events, logs, and pod descriptions show original URLs even when mirrors are used
- Mirror functionality can be verified indirectly: successful installation from dev FBC + proper mirror/auth config = working mirrors
- Validate subscription source: `kubectl get subscription.operators.coreos.com <name> -n openshift-operators -o yaml`

**ImageContentSourcePolicy and Pull Secret Configuration:**
- Always check for mirrors before deploying dev builds
- Verify quay.io/acm-d mirrors are configured for dev images
- Essential for dev builds that redirect from registry.redhat.io
- CRITICAL: Always inform user about mirror status - sometimes we want dev mirrors (quay.io/acm-d), sometimes we want to test GA image locations (registry.redhat.io)
- Let user decide whether to proceed with existing mirrors or modify them for the specific test scenario
- CRITICAL: When using mirrors with port-specific registries (e.g., quay.io:443/acm-d), ensure pull secret contains credentials for the exact registry format
- Extract credentials from user's local ~/.docker/config.json and patch cluster pull secret
- Verify credentials are added: `kubectl get secret pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq '.auths | keys[]'`
- SECURITY: Always clean up temporary files containing credentials: `rm -f /tmp/*dockerconfig*.json`

#### Rebasing Konflux PRs
When a konflux PR needs rebasing:
1. Clone the repo if not already cloned
2. Checkout the PR branch with `gh pr checkout <PR_NUMBER>`
3. Fetch and rebase against the target branch: `git fetch origin <target-branch> && git rebase origin/<target-branch>`
4. **⚠️ CRITICAL**: After rebasing, ALWAYS verify file changes with `gh pr diff <PR_NUMBER> --name-only`
5. **Check for contamination**: Ensure only expected files are modified:
   - Konflux digest updates: ONLY `.tekton/*.yaml` files
   - Dependency updates: ONLY `go.mod`, `go.sum`, or specific package files
   - **NEVER**: Mixed changes (e.g., `yq` submodule + `.tekton/*.yaml`)
6. **If contamination detected**:
   - `git reset --soft HEAD~1` (uncommit)
   - `git restore --staged <unwanted-file>` (unstage contamination)
   - `git restore <unwanted-file>` (restore to clean state)
   - `git commit -s -S -m "original message"` (clean commit)
7. If merge conflicts occur, use `go mod tidy` to resolve go.sum conflicts rather than editing directly
8. **⚠️ CRITICAL - Check for skipped commits**: If rebase shows `warning: skipped previously applied commit`, the changes may already exist in target branch:
   - Check the warning: `warning: skipped previously applied commit <sha>`
   - **BEFORE force pushing**: Verify `gh pr diff <PR_NUMBER>` still shows changes
   - **If diff is empty**: The changes are already in target branch - comment on PR to close instead of force pushing
   - **Never force push empty changes** - this creates PRs with no actual differences
9. Force push: `git push --force-with-lease origin <branch-name>`
10. **⚠️ CRITICAL - Verify after force push**: Check `gh pr diff <PR_NUMBER>` again after pushing to confirm changes still exist on GitHub
11. **If PR auto-closes**: GitHub will automatically close PRs with no net changes - this confirms the changes were already in target branch
12. **Final verification**: `gh pr diff <PR_NUMBER> --name-only` to confirm only intended files (if PR remains open)

#### Common PR Issues
- **Kubernetes v0.34.0 dependency conflicts**: PRs updating to Kubernetes v0.34.0 fail due to structured-merge-diff v4/v6 incompatibility with openshift/client-go. These require `/hold` comments until openshift/client-go supports v6.
- **Obsolete PRs**: Old dependency update PRs may try to downgrade packages that main branch already has at newer versions. Check current versions in main before rebasing.
- **Breaking dependency changes**: Controller-runtime v0.22.0 and Kubernetes v0.34.0 updates introduce breaking changes requiring coordinated updates across the ecosystem.

#### PR Management Procedures
- **Checking for obsolete PRs**: Compare target versions in PR with current versions in main branch. If main has same or newer versions, close the PR as obsolete.
- **When to close vs rebase**: Close if PR would downgrade dependencies. Rebase if PR moves dependencies forward.
- **Label management**: Ensure held PRs have "needs-ok-to-test" rather than "ok-to-test" to prevent accidental CI runs.
- **Dependency conflict handling**: Add `/hold` comments explaining specific incompatibilities (e.g., structured-merge-diff v4/v6 conflicts).

### Kubernetes/OpenShift Commands
#### Cluster Access with Separate Kubeconfig Files
- **Setup**: Copy kubeconfig files to `.kube/` directory with descriptive names:
  - `cp ~/DEV/KONFLUX/konflux-rh01.kubeconfig .kube/config-konflux`
  - `cp ~/.kube/tflow-419-hub.config .kube/config-4-19`
  - `cp /path/to/other-cluster.config .kube/config-<cluster-name>`
- **Konflux cluster**: `export KUBECONFIG=$FLEET_MGMT_DIR/.kube/config-konflux`
- **Other clusters**: `export KUBECONFIG=$FLEET_MGMT_DIR/.kube/config-<cluster-name>` (e.g., config-4-19, config-4-18, config-staging)
- **No context switching needed**: Each kubeconfig file contains only one cluster/context
- **OpenShift resources**: Use `oc` instead of `kubectl` for OpenShift-specific resources like `component`
- **Command patterns**:
  - Konflux: `export KUBECONFIG=$FLEET_MGMT_DIR/.kube/config-konflux; oc -n <namespace> get <resource>`
  - Other clusters: `export KUBECONFIG=$FLEET_MGMT_DIR/.kube/config-<cluster-name>; kubectl get <resource>`
- **Example**: `export KUBECONFIG=$FLEET_MGMT_DIR/.kube/config-konflux; oc -n volsync-tenant get component`

#### Getting Images from Konflux PRs ⚠️ **ALWAYS USE THIS METHOD - DON'T FORGET!**
- **CRITICAL**: Always use snapshots with PR labels, NOT application labels or recent snapshots
- **Find snapshots for a PR**: `oc -n volsync-tenant get snapshots -l "pac.test.appstudio.openshift.io/pull-request=<PR_NUMBER>,pac.test.appstudio.openshift.io/event-type=pull_request" --sort-by=.metadata.creationTimestamp`
- **Get latest snapshots**: Take the most recent ones from the sorted list (bottom entries) - one for each OCP version
- **Verify commit SHA**: Check snapshot annotation matches PR head: `oc get snapshot <name> -o jsonpath='{.metadata.annotations.build\.appstudio\.redhat\.com/commit_sha}'`
- **Compare with PR head**: `gh pr view <PR_NUMBER> --json headRefOid`
- **Extract container images**: `oc -n volsync-tenant get snapshots <snapshot-names> -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.components[0].containerImage}{"\n"}{end}'`
- **Example**: For PR #29, this returns all FBC images built for different OpenShift versions (4-14 through 4-19)
- **⚠️ REMEMBER**: Use the documented method in this file, don't improvise with different label selectors!

#### Testing FBC Images
- **Create test CatalogSource**: Use FBC images from Konflux PR builds to test operator catalogs
- **Steps**:
  1. Verify correct cluster context: `kubectl config current-context`
  2. Create CatalogSource YAML in `/tmp/` directory
  3. Apply: `oc apply -f /tmp/catalogsource.yaml`
- **Template**:
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: fbc-on-pr-<VERSION>-test-catalogsource
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: quay.io/redhat-user-workloads/volsync-tenant/volsync-fbc-<VERSION>@sha256:<digest>
  displayName: Konflux FBC test CatalogSource PR #<PR> <VERSION>
  publisher: Red Hat
```

#### Verifying Snapshots for Stage Releases ⚠️ **COMPREHENSIVE VERIFICATION REQUIRED**

When preparing a release to stage, you need snapshots containing both operator and bundle images. This workflow ensures the snapshot contains consistent, latest builds.

**Step 1: Find Latest Application Snapshots**
```bash
# Get snapshots for the application (contains both operator and bundle components)
export KUBECONFIG=/path/to/fleet-mgmt-app-repos/.kube/config-konflux
oc -n volsync-tenant get snapshots -l appstudio.openshift.io/application=volsync-0-12 --sort-by=.metadata.creationTimestamp | tail -10
```

**Step 2: Verify Snapshot Contains Both Components**
```bash
# Check latest snapshot contains both required components
SNAPSHOT_NAME="volsync-0-12-<latest>"
oc -n volsync-tenant get snapshot $SNAPSHOT_NAME -o jsonpath='{.spec.components[*].name}'
# Expected output: volsync-0-12 volsync-bundle-0-12
```

**Step 3: Get Latest Promoted Images from Components**
```bash
# Get the latest promoted image from each component
echo "=== volsync-0-12 component ==="
oc -n volsync-tenant get component volsync-0-12 -o jsonpath='{.status.lastPromotedImage}'
echo
echo "=== volsync-bundle-0-12 component ==="
oc -n volsync-tenant get component volsync-bundle-0-12 -o jsonpath='{.status.lastPromotedImage}'
echo
```

**Step 4: Compare Snapshot Images with Component Status**
```bash
# Get images from the snapshot
echo "=== Snapshot Images ==="
oc -n volsync-tenant get snapshot $SNAPSHOT_NAME -o jsonpath='{range .spec.components[*]}{.name}{": "}{.containerImage}{"\n"}{end}'
```

**Step 5: Verify Bundle References Correct Operator Image**
```bash
# Get the commit SHA that built the bundle image
BUNDLE_COMMIT=$(oc -n volsync-tenant get component volsync-bundle-0-12 -o jsonpath='{.status.lastBuiltCommit}')
echo "Bundle built from commit: $BUNDLE_COMMIT"

# Check the operator digest referenced in that commit's rhtap-buildargs.conf
cd /path/to/fleet-mgmt-app-repos/volsync-operator-product-build
git fetch origin && git checkout release-0.12 && git pull
git show $BUNDLE_COMMIT:rhtap-buildargs.conf | grep "ARG_STAGE_VOLSYNC_IMAGE_PULLSPEC"
```

**Step 6: Final Verification Checklist**
- ✅ Snapshot contains both `volsync-0-12` and `volsync-bundle-0-12` components
- ✅ Snapshot image digests match component `lastPromotedImage` digests
- ✅ Bundle's `rhtap-buildargs.conf` references the same operator digest as in snapshot
- ✅ All components built from recent commits (verify timestamps)

**Success Criteria**: All digests match perfectly across snapshot, component status, and bundle build args.

**Usage for Releases**: Reference the verified snapshot name (e.g., `volsync-0-12-4szwc`) in your release configuration.

#### Working with OLM Subscriptions
- **List subscriptions**: `oc get subs -n <namespace>` (shorthand for subscriptions.operators.coreos.com)
- **Get subscription details**: `oc get subscription.operators.coreos.com <name> -n <namespace> -o yaml`
- **Delete subscription**: `oc delete subscription.operators.coreos.com <name> -n <namespace>`
- **Important**: Always specify the full API group `subscription.operators.coreos.com` when deleting to avoid conflicts with other subscription types
- **Check subscription status**: Look for conditions like `BundleUnpackFailed`, `CatalogSourcesUnhealthy`
- **Common issues**: 
  - Bundle unpacking timeouts (`DeadlineExceeded`)
  - Catalog source connectivity problems
  - Image pull failures from Konflux registry

#### **CRITICAL: Clean Environment Before Testing New Catalogs**
When testing new FBC/catalog images, **ALWAYS audit and clean existing resources first**:

**Step 1: Audit existing resources**
```bash
# Check for old catalog sources that might conflict
oc get catalogsource -n openshift-marketplace | grep -E "(fbc|test|pr)"

# Check existing subscriptions and their sources
oc get subs -n openshift-operators -o wide

# Check existing CSVs
oc get csv -n openshift-operators | grep volsync
```

**Step 2: Clean conflicting resources BEFORE installing new ones**
```bash
# Delete old test catalog sources (keep only the one you want to test)
oc delete catalogsource <old-test-catalog> -n openshift-marketplace

# Delete existing subscription if switching catalog sources
oc delete subscription.operators.coreos.com <operator-name> -n openshift-operators

# Delete existing CSV to force clean installation
oc delete csv <old-csv-name> -n openshift-operators
```

**Step 3: Verify clean state before proceeding**
```bash
# Ensure only your target catalog source exists
oc get catalogsource -n openshift-marketplace

# Confirm no conflicting CSVs remain
oc get csv -n openshift-operators | grep volsync
```

**Why this matters:**
- **OLM resolver picks the "best" version** across ALL available catalogs, not just your target catalog
- **Leftover resources from previous testing** can cause version conflicts and wrong installations
- **You may think you're testing new images** when you're actually testing old ones from conflicting catalogs
- **Always verify the actual installed version** matches what you expect from your target catalog

**Key lesson:** Never assume a clean environment - always audit first, then clean, then install.

#### Creating OLM Subscriptions for VolSync from FBC
- **Always ask user for**:
  - Channel (e.g., stable-0.12, stable-0.13)
  - StartingCSV (specific version or leave blank for latest)
- **Template**:
```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: volsync-product
  namespace: openshift-operators
spec:
  channel: <USER_PROVIDED_CHANNEL>
  name: volsync-product
  source: <FBC_CATALOG_SOURCE_NAME>
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
  startingCSV: <USER_PROVIDED_CSV_OR_OMIT>
```
- **Process**:
  1. Ask user for channel, catalog source, and startingCSV
  2. Create YAML in `/tmp/volsync-subscription.yaml`
  3. Apply: `oc apply -f /tmp/volsync-subscription.yaml`
  4. Monitor: `oc get subs -n openshift-operators`
- **Usage**: Say "create a volsync subscription" - Claude will prompt for inputs and handle the entire process

#### Running VolSync E2E Tests (Custom Scorecard Tests)
- **Prerequisites**: 
  1. VolSync operator installed on target cluster
  2. Set correct KUBECONFIG: `export KUBECONFIG=$FLEET_MGMT_DIR/.kube/config-<cluster-name>`
  3. Create service account: `./hack/ensure-volsync-test-runner.sh`
  4. Install operator-sdk: `make operator-sdk` (installs to `./bin/operator-sdk`)
- **Deploy prerequisites**: `./bin/operator-sdk scorecard ./bundle --config custom-scorecard-tests/config-downstream.yaml --selector=test=deploy-prereqs -o text --wait-time=600s --skip-cleanup=false --service-account=volsync-test-runner`
- **Run all e2e tests**: `./bin/operator-sdk scorecard ./bundle --config custom-scorecard-tests/config-downstream.yaml --selector=suite=volsync-e2e -o text --wait-time=3600s --skip-cleanup=false --service-account=volsync-test-runner`
- **Run single test**: `./bin/operator-sdk scorecard ./bundle --config custom-scorecard-tests/config-downstream.yaml --selector=test=<test-name.yml> -o text --wait-time=300s --skip-cleanup=false --service-account=volsync-test-runner`
- **Usage**: Say "run volsync e2e tests" - Claude will help execute the appropriate tests

## Common Workflows

### 1. Running VolSync E2E Tests

**Prerequisites**:
1. VolSync operator installed on target cluster
2. Set correct KUBECONFIG: `export KUBECONFIG=$FLEET_MGMT_DIR/.kube/config-<cluster-name>`
3. Create service account: `./hack/ensure-volsync-test-runner.sh`
4. Install operator-sdk: `make operator-sdk`

**Steps**:
1. Deploy prerequisites: `./bin/operator-sdk scorecard ./bundle --config custom-scorecard-tests/config-downstream.yaml --selector=test=deploy-prereqs -o text --wait-time=600s --skip-cleanup=false --service-account=volsync-test-runner`
2. Run all tests: `./bin/operator-sdk scorecard ./bundle --config custom-scorecard-tests/config-downstream.yaml --selector=suite=volsync-e2e -o text --wait-time=3600s --skip-cleanup=false --service-account=volsync-test-runner`

### 2. Testing Konflux FBC Images

**Steps**:
1. Get FBC images from Konflux PR: Use snapshots with PR labels
2. Create CatalogSource in `/tmp/catalogsource.yaml`
3. Apply: `oc apply -f /tmp/catalogsource.yaml`
4. Create subscription for testing
5. Verify operator installation

### 3. Managing Cluster Credentials

**Konflux cluster**:
```bash
unset KUBECONFIG && export KUBECONFIG=$FLEET_MGMT_DIR/.kube/config-konflux && oc login --web https://api.stone-prd-rh01.pg1f.p1.openshiftapps.com:6443/
```

**Other clusters**:
```bash
ck creds <cluster-name>  # Get credentials via cluster-keeper
export KUBECONFIG=$FLEET_MGMT_DIR/.kube/config-<cluster-name>
```

### 4. Creating OLM Subscriptions

**Process**:
1. Ask user for channel, catalog source, and startingCSV
2. Create YAML in `/tmp/volsync-subscription.yaml`
3. Apply: `oc apply -f /tmp/volsync-subscription.yaml`
4. Monitor: `oc get subs -n openshift-operators`

## Dependencies and Prerequisites

### Required Tools

- **oc/kubectl** - OpenShift/Kubernetes CLI tools
- **gh** - GitHub CLI for PR management
- **ck** - cluster-keeper tool for OpenShift cluster management
- **make** - Build system (preferred over direct go commands)
- **git** - Version control
- **yq** - YAML processing (for konflux-release-data repo)
- **tox** - Testing framework (for konflux-release-data repo)

### Environment Requirements

- **KUBECONFIG** - Kubernetes cluster configuration
- **Go modules** - Enabled for Go projects
- **Virtual environment** - For Python-based repositories

### Common Environment Variables

- **PYXIS_URL** - For integration tests (konflux-release-data)
- **CGW_TOKEN_RO** - For integration tests (konflux-release-data)
- **PRODSEC_PRODUCT_DEFINITIONS_URL** - For integration tests (konflux-release-data)

## Notes
<!-- Add any additional project-specific information -->
- Default branch: main
- Language: Go
- Go modules: enabled
- **Git repository structure**: This directory (`fleet-mgmt-app-repos/`) is a subdirectory of the main git repository
- **Git operations**: When committing changes, files are relative to the git root, not this working directory
- **IMPORTANT**: Always ensure you're in the correct directory when using kubeconfig - the kubeconfig files are in the `fleet-mgmt-app-repos/.kube/` directory, not in subdirectories like checked-out repos
- **Temporary files**: Save temporary files (like CatalogSource YAML) in `/tmp` or similar temp directory, not in the repo workspace
- **Before apply operations**: Always verify current context with `kubectl config current-context` before running `oc apply` or similar commands
- **CRITICAL**: Always run `pwd` before executing commands to verify current directory - especially important for relative paths, kubeconfig files, and script execution
- **KUBECONFIG Management**: The environment has a persistent KUBECONFIG variable that gets reset between bash commands. Always use single-command syntax: `unset KUBECONFIG && export KUBECONFIG=/path/to/config && kubectl command` to ensure the correct kubeconfig is used

### Konflux Cluster Authentication
When Konflux credentials expire, re-authenticate using:
```bash
unset KUBECONFIG && export KUBECONFIG=$FLEET_MGMT_DIR/.kube/config-konflux && oc login --web https://api.stone-prd-rh01.pg1f.p1.openshiftapps.com:6443/
```
After authentication, always use the single-command pattern for kubectl operations:
```bash
unset KUBECONFIG && export KUBECONFIG=$FLEET_MGMT_DIR/.kube/config-konflux && kubectl command
```

### Cluster-Keeper (ck) Tool Usage
The `ck` tool manages OpenShift clusters. Key commands:

**Getting cluster credentials and creating kubeconfig:**
```bash
# Get credentials for a cluster
ck creds <cluster-name>

# Create kubeconfig file for a cluster
oc login --kubeconfig=.kube/config-<cluster-name> <api-url> -u <username> -p "<password>" --insecure-skip-tls-verify
```

**Common cluster management:**
```bash
ck list                    # List available clusters
ck current                 # Show current context
ck use <cluster-name>      # Switch to cluster context
ck state <cluster-name>    # Check power state
ck hibernate <cluster>     # Hibernate cluster
ck run <cluster>           # Resume hibernated cluster
ck console <cluster>       # Open cluster console
```

## VolSync E2E Testing Results Analysis

### Analyzing Test Results After Full Test Suite
When running the full VolSync e2e test suite, follow these steps to analyze results:

1. **Count test results**:
   ```bash
   grep -c "State: pass" /tmp/volsync-full-e2e-tests.log
   grep -c "State: fail" /tmp/volsync-full-e2e-tests.log
   ```

2. **Find failing tests**:
   ```bash
   grep -B 10 -A 5 "State: fail" /tmp/volsync-full-e2e-tests.log
   ```

3. **Analyze failure details**:
   ```bash
   # Look for specific error patterns
   grep -A 20 "==FAILED==" /tmp/volsync-full-e2e-tests.log
   grep "ApiException\|NotFoundError\|fatal:" /tmp/volsync-full-e2e-tests.log
   ```

4. **Success criteria**:
   - 95%+ pass rate indicates successful test run
   - Infrastructure errors (404, API connectivity) vs functional failures
   - Core VolSync functionality (Restic, Rclone, Rsync, Syncthing) should pass
   - Edge case failures (longname, special configurations) are less critical

5. **Common failure patterns**:
   - `kubernetes.dynamic.exceptions.NotFoundError: 404` = Kubernetes API connectivity issue
   - `ApiException` = Cluster/infrastructure problem
   - Ansible task failures = Functional issues that need investigation
- **Directory awareness**: When switching between repos (volsync, volsync-addon-controller, etc.), always confirm location with `pwd`

**CRITICAL: Base Repository Protection**:
- **NEVER check out branches from other repositories in the base aiexperiments repo** without explicit user permission
- When working with PR branches, always `cd` into the specific repository directory first (e.g., `cd volsync-operator-product-build`)
- **Always verify you're in the correct repository** with `git remote -v` before any git operations
- The base repo should stay on `main` branch unless user explicitly requests otherwise

## Git Commit Requirements

- **ALWAYS use signed commits**: All git commits MUST include both `-s` and `-S` flags
  - `-s`: Adds "Signed-off-by" line to commit message
  - `-S`: Signs the commit with GPG signature
  - **Command format**: `git commit -s -S -m "commit message"`
  - **Required for all commits**: This applies to all repositories and all types of commits
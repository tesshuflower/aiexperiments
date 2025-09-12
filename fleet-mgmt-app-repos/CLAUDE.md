# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## Project Overview

This is a multi-repository management workspace for VolSync and related projects.
It contains configurations and tools for managing VolSync operator development,
testing, and release across multiple Red Hat and upstream repositories.

## Repositories
### Product Repos
- VolSync Operator Product Build: https://github.com/stolostron/volsync-operator-product-build - VolSync operator product build repository
- VolSync Operator Product FBC: https://github.com/stolostron/volsync-operator-product-fbc - VolSync operator product FBC (File-Based Catalog) repository

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
**IMPORTANT**: When user asks to monitor anything:
1. **Always check Slack setup first**: Run `if [ -n "$CLAUDE_SLACK_WEBHOOK_URL" ]; then echo "✅ Slack notifications are configured - I will notify you via Slack"; else echo "❌ CLAUDE_SLACK_WEBHOOK_URL not set - I will use local OS notifications only"; fi`
2. **Show only the actual command**: Don't show all the monitoring code/logic, just the simple command to execute

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

### Pull Request Workflows
- Create PR: Standard process with auto-generated descriptions
- Review checklist: Code review, tests pass, documentation updated
- When listing PRs: Always identify which PRs only modify .tekton files (tekton-only changes)
- When listing PRs: Always indicate if PRs are on hold by checking for "do-not-merge/hold" labels
- Automated PRs: konflux/mintmaker and red-hat-konflux create automated dependency/digest update PRs
- Special attention: Highlight PRs with "konflux-nudge" label when reviewing automated PRs
- When showing PR diffs: Always use --color=always flag for colored output
- When approving PRs: Add comment with "/approve" on one line and "/lgtm" on the next line

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
4. If merge conflicts occur, use `go mod tidy` to resolve go.sum conflicts rather than editing directly
5. Force push: `git push --force-with-lease origin <branch-name>`

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
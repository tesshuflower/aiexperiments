# Repository Configurations

## Repositories
### Product Repos
- VolSync Operator Product Build: https://github.com/stolostron/volsync-operator-product-build - VolSync operator product build repository
- VolSync Operator Product FBC: https://github.com/stolostron/volsync-operator-product-fbc - VolSync operator product FBC (File-Based Catalog) repository

### ACM Repos
- VolSync Addon Controller (vac): https://github.com/stolostron/volsync-addon-controller - VolSync addon controller repository
- Cluster Backup Operator (cbo/backup): https://github.com/stolostron/cluster-backup-operator - Cluster backup operator repository

### Upstream Repos
- VolSync (volsync): https://github.com/backube/volsync - VolSync upstream repository

## Common Actions
<!-- Define common tasks you want to run across repos -->

### Build & Test
- Build: `go build ./...`
- Test: `go test ./...`
- Lint: `golangci-lint run`
- Format: `go fmt ./...`
- Vet: `go vet ./...`

### GitHub Actions
- Deploy to staging: `gh workflow run deploy.yml --ref staging`
- Deploy to production: `gh workflow run deploy.yml --ref main`
- Run CI: `gh workflow run ci.yml`

#### Monitoring Workflows
When monitoring workflows for completion, use this pattern to notify with dialog when ANY status change occurs (success or failure):
```bash
# Function to send notification based on OS
notify_user() {
  local message="$1"
  local title="$2"
  
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
- **Konflux cluster**: `export KUBECONFIG=$(pwd)/.kube/config-konflux`
- **Other clusters**: `export KUBECONFIG=$(pwd)/.kube/config-<cluster-name>` (e.g., config-4-19, config-4-18, config-staging)
- **No context switching needed**: Each kubeconfig file contains only one cluster/context
- **OpenShift resources**: Use `oc` instead of `kubectl` for OpenShift-specific resources like `component`
- **Command patterns**:
  - Konflux: `export KUBECONFIG=$(pwd)/.kube/config-konflux; oc -n <namespace> get <resource>`
  - Other clusters: `export KUBECONFIG=$(pwd)/.kube/config-<cluster-name>; kubectl get <resource>`
- **Example**: `export KUBECONFIG=$(pwd)/.kube/config-konflux; oc -n volsync-tenant get component`

#### Getting Images from Konflux PRs
- **Find snapshots for a PR**: `oc -n volsync-tenant get snapshots -l "pac.test.appstudio.openshift.io/pull-request=<PR_NUMBER>,pac.test.appstudio.openshift.io/event-type=pull_request" --sort-by=.metadata.creationTimestamp`
- **Get latest snapshots**: Take the most recent ones from the sorted list (bottom entries)
- **Verify commit SHA**: Check snapshot annotation matches PR head: `oc get snapshot <name> -o jsonpath='{.metadata.annotations.build\.appstudio\.redhat\.com/commit_sha}'`
- **Compare with PR head**: `gh pr view <PR_NUMBER> --json headRefOid`
- **Extract container images**: `oc -n volsync-tenant get snapshots <snapshot-names> -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.components[0].containerImage}{"\n"}{end}'`
- **Example**: For PR #29, this returns all FBC images built for different OpenShift versions (4-14 through 4-19)

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
  2. Set correct KUBECONFIG: `export KUBECONFIG=$(pwd)/.kube/config-<cluster-name>`
  3. Create service account: `./hack/ensure-volsync-test-runner.sh`
  4. Install operator-sdk: `make operator-sdk` (installs to `./bin/operator-sdk`)
- **Deploy prerequisites**: `./bin/operator-sdk scorecard ./bundle --config custom-scorecard-tests/config-downstream.yaml --selector=test=deploy-prereqs -o text --wait-time=600s --skip-cleanup=false --service-account=volsync-test-runner`
- **Run all e2e tests**: `./bin/operator-sdk scorecard ./bundle --config custom-scorecard-tests/config-downstream.yaml --selector=suite=volsync-e2e -o text --wait-time=3600s --skip-cleanup=false --service-account=volsync-test-runner`
- **Run single test**: `./bin/operator-sdk scorecard ./bundle --config custom-scorecard-tests/config-downstream.yaml --selector=test=<test-name.yml> -o text --wait-time=300s --skip-cleanup=false --service-account=volsync-test-runner`
- **Usage**: Say "run volsync e2e tests" - Claude will help execute the appropriate tests

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
unset KUBECONFIG && export KUBECONFIG=$(pwd)/.kube/config-konflux && oc login --web https://api.stone-prd-rh01.pg1f.p1.openshiftapps.com:6443/
```
After authentication, always use the single-command pattern for kubectl operations:
```bash
unset KUBECONFIG && export KUBECONFIG=/Users/tflower/DEV/tesshuflower/aiexperiments/fleet-mgmt-app-repos/.kube/config-konflux && kubectl command
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
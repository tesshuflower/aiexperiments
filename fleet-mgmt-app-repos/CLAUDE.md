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
    # macOS - use single quotes to avoid escaping issues
    osascript -e 'display dialog "'"$message"'" with title "'"$title"'"'
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
- remember what path you're at before running relative commands
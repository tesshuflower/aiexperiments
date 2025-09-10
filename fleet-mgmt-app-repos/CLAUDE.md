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

### Pull Request Workflows
- Create PR: Standard process with auto-generated descriptions
- Review checklist: Code review, tests pass, documentation updated
- When listing PRs: Always identify which PRs only modify .tekton files (tekton-only changes)
- Automated PRs: konflux/mintmaker and red-hat-konflux create automated dependency/digest update PRs
- Special attention: Highlight PRs with "konflux-nudge" label when reviewing automated PRs
- When showing PR diffs: Always use --color=always flag for colored output

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

## Notes
<!-- Add any additional project-specific information -->
- Default branch: main
- Language: Go
- Go modules: enabled
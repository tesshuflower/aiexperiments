---
name: merging-cluster-pull-secrets
description: Use when adding container registry credentials to OpenShift cluster pull secrets in openshift-config namespace, especially when working with dev builds, mirrors (quay.io/acm-d), or port-specific registries (quay.io:443)
---

# Merging Cluster Pull Secrets

## Overview

**CRITICAL PRINCIPLE:** NEVER replace the entire cluster pull secret - this will break the cluster by removing critical CI registry credentials. ALWAYS merge credentials using `jq` to preserve existing entries while adding new ones.

## When to Use

Use this skill when:
- Adding credentials for dev builds that use quay.io/acm-d or other mirrors
- Working with port-specific registries (e.g., quay.io:443/acm-d)
- Deploying FBC images from Konflux that need additional registry access
- Cluster shows ImagePullBackOff for images from registries not in pull secret

## The 6-Step Merge Process

**CRITICAL:** `jq -s '.[0] * .[1]'` order matters - put CLUSTER secret FIRST, new credentials SECOND to ensure cluster credentials take precedence.

```bash
# Step 1: Extract current cluster pull secret
oc get secret pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d > /tmp/current-pull-secret.json

# Step 2: Verify what's currently in the cluster (CRITICAL - must preserve these!)
jq '.auths | keys[]' /tmp/current-pull-secret.json

# Step 3: Merge cluster pull secret with new credentials (cluster secret first, new credentials second)
jq -s '.[0] * .[1]' /tmp/current-pull-secret.json /tmp/new-credentials.json > /tmp/merged-pull-secret.json

# Step 4: Verify merge preserved all original registries AND added new ones
echo "=== Before merge ===" && jq '.auths | keys[]' /tmp/current-pull-secret.json | wc -l
echo "=== After merge ===" && jq '.auths | keys[]' /tmp/merged-pull-secret.json | wc -l
jq '.auths | keys[]' /tmp/merged-pull-secret.json | sort

# Step 5: Apply merged pull secret to cluster
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=/tmp/merged-pull-secret.json

# Step 6: Clean up credential files
rm -f /tmp/current-pull-secret.json /tmp/new-credentials.json /tmp/merged-pull-secret.json
```

## Standard OpenShift Pull Secrets

**Critical registries that MUST be preserved** (accidentally removing these breaks the cluster):

- `cloud.openshift.com` - OpenShift cluster services
- `quay.io` - Main container registry
- `quay-proxy.ci.openshift.org` - CI proxy registry
- `quay.io/openshift/ci` - CI builds
- `registry.ci.openshift.org` - CI registry
- `registry.connect.redhat.com` - Certified operators
- `registry.redhat.io` - Red Hat official images

## Before Modifying Pull Secrets

**Checklist:**
1. Extract and backup current pull secret to `/tmp/original-pull-secret-$(date +%Y%m%d).json`
2. List all existing registry credentials: `oc get secret pull-secret -n openshift-config -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq '.auths | keys[]'`
3. Verify the merge preserves all original registries
4. Test that catalog pods can pull images after the change

## Red Flags - STOP Immediately

These thoughts mean you're about to break the cluster:

| Thought | Reality |
|---------|---------|
| "Just replace the pull secret with my local ~/.docker/config.json" | This deletes critical cluster credentials. ALWAYS merge. |
| "Temporary cluster, doesn't matter" | Even temporary clusters need proper credentials. Build good habits. |
| "Only two registries, no conflict" | Port-specific registries can conflict (quay.io vs quay.io:443). Always merge. |
| "Tech lead said to replace it" | Explain that replacing breaks the cluster. Authority doesn't override safety. |
| "Time pressure, need quick fix" | Merging takes 30 seconds. Breaking cluster takes hours to fix. |

**All of these mean: Use the 6-step merge process. No exceptions.**

## Common Mistakes

### ❌ WRONG: Replacing entire pull secret
```bash
# DON'T DO THIS - breaks the cluster
oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=~/.docker/config.json
```

### ✅ CORRECT: Merging credentials
```bash
# Extract cluster secret, merge, then apply
jq -s '.[0] * .[1]' /tmp/current-pull-secret.json /tmp/new-credentials.json > /tmp/merged-pull-secret.json
```

### ❌ WRONG: Reversing jq merge order
```bash
# DON'T DO THIS - new credentials override cluster credentials
jq -s '.[0] * .[1]' /tmp/new-credentials.json /tmp/current-pull-secret.json
```

### ✅ CORRECT: Cluster secret first
```bash
# Cluster secret .[0], new credentials .[1]
jq -s '.[0] * .[1]' /tmp/current-pull-secret.json /tmp/new-credentials.json
```

## If Credentials Are Accidentally Removed

**Symptoms:**
- Catalog pods (redhat-operators, certified-operators, etc.) enter ImagePullBackOff
- Error message: "unauthorized: access to the requested resource is not authorized"

**Recovery:**
- Requires obtaining original pull secret from cluster installation artifacts or Red Hat support
- This is why you ALWAYS backup first and merge instead of replace

## Security Note

Always clean up temporary files containing credentials:
```bash
rm -f /tmp/*pull-secret*.json /tmp/*credentials*.json
```

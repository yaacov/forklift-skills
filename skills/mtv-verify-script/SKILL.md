---
name: mtv-verify-script
description: Generate and run a self-contained bash verification script for an MTV/Forklift bug or feature. Use this skill when the user asks to "create a verification script", "write a test script", "generate a bash test", "create an e2e test script", or "write a verification bash script" for an MTV/Forklift Jira ticket.
---

# MTV Verification Script Generator

Generate self-contained bash e2e verification scripts for MTV/Forklift bugs and features.
Scripts follow a standard pattern: create namespace → create providers → run test steps → verify result → cleanup.

## Workflow

Follow these steps in order. Never skip a step. Always wait for user confirmation before proceeding.

---

### Step 1 — Get the Jira ticket and related PRs

Ask the user for the Jira ticket number (e.g. `MTV-4911`).

#### 1a. Fetch the Jira ticket

```
WebFetch  url: "https://redhat.atlassian.net/browse/MTV-<number>"
```

Extract from the ticket:
- **Summary** — one-line description of the bug or feature
- **Description** — full problem statement and expected behavior
- **Acceptance criteria** — what "fixed" or "working" looks like
- **Component** — which part of MTV is affected (controller, UI, provider, plan, migration)
- **Provider types mentioned** — vSphere, oVirt, OpenStack, OVA, EC2, HyperV

If the ticket is not publicly accessible or the fetch fails, ask the user to paste the relevant description.

#### 1b. Search for related PRs in the forklift repo

Search GitHub for pull requests in `kubev2v/forklift` that reference this ticket:

```
WebFetch  url: "https://github.com/kubev2v/forklift/pulls?q=MTV-<number>"
```

For each PR found, fetch its detail page to extract:
- **PR title and description** — especially any "How to test" or "Testing" sections
- **Files changed** — which components were modified (controller, provider adapter, plan, UI, etc.)
- **Unit tests added** — what scenarios the unit tests cover (these reveal the edge cases to exercise)
- **Any "before/after" behavior** described in the PR body

```
WebFetch  url: "https://github.com/kubev2v/forklift/pull/<pr-number>"
```

Use the PR information to answer:
- What exact behavior changed? (informs the core test assertion)
- What are the edge cases or boundary conditions? (informs additional test scenarios)
- Are there "how to reproduce" steps in the PR? (informs the test steps)
- Which provider type(s) are involved?

If no PRs are found via the search page, also try:
```
WebFetch  url: "https://api.github.com/search/issues?q=MTV-<number>+repo:kubev2v/forklift+type:pr"
```

Summarize what you learned from the Jira ticket and PRs before proceeding to Step 2.

---

### Step 2 — Gather environment information

Ask the user for any information not found in the ticket. Collect only what is needed:

| Information | When needed |
|---|---|
| Provider type (vsphere / ovirt / openstack / ova / openshift / hyperv / ec2) | Always |
| Source provider URL | Always (except OVA and EC2) |
| Credentials (username / password / token) | Always (except OVA) |
| VM name(s) to migrate | When the test involves a migration plan |
| TLS mode (cacert or insecure-skip-tls) | Always (except OVA) |
| Controller namespace | Default: `konveyor-forklift` |
| Any custom image or setting to override | When the ticket references a fix image |

**Do not ask for information that has a clear default** (e.g. namespace name, plan name, provider name — these can be derived from the ticket number).

Inform the user which environment variables they should set:
- vSphere: `GOVC_URL`, `GOVC_USERNAME`, `GOVC_PASSWORD`
- oVirt/RHV: `RHV_URL`, `RHV_USERNAME`, `RHV_PASSWORD`
- OpenStack: `OSP_URL`, `OSP_USERNAME`, `OSP_PASSWORD`, `OSP_DOMAIN_NAME`, `OSP_PROJECT_NAME`, `OSP_REGION_NAME`
- OVA: `OVA_URL`
- Remote OpenShift: `OCP_URL`, `OCP_TOKEN`
- HyperV: `HV_URL`, `HV_USERNAME`, `HV_PASSWORD`, `HV_SMB_URL`
- EC2: `EC2_REGION`, `EC2_ACCESS_KEY_ID`, `EC2_SECRET_ACCESS_KEY`, `EC2_TARGET_AZ`, `EC2_TARGET_REGION`

---

### Step 3 — Create and review the test plan

Write a test plan markdown file named `test-plan-mtv-<number>.md` in the current directory.

The plan must include:

```markdown
# Test Plan: MTV-<number> — <summary>

## Objective
<What this test verifies, in one paragraph>

## Prerequisites
- oc / kubectl with mtv plugin installed
- MTV installed on the cluster (namespace: konveyor-forklift)
- Environment variables set: <list>
- <Any other prereqs: VM name, VDDK image, custom controller image, etc.>

## Test Steps
1. <Step description>
2. …

## Pass Criteria
- <Specific observable outcome that confirms the fix/feature works>

## Fail Criteria
- <What indicates the bug is still present or the feature does not work>

## Cleanup
- Namespace `<ns>` deleted
- Providers deleted
- Any settings overrides reverted
```

**Present the plan to the user and ask for review.** Wait for explicit approval ("looks good", "approved", etc.) or for requested changes before proceeding.

---

### Step 4 — Generate the test script

After plan approval, generate a bash script named `test-mtv-<number>.sh`.

#### Script structure

```bash
#!/bin/bash
#
# E2E test for <summary> (MTV-<number>).
#
# Prerequisites: oc, oc mtv plugin, <env vars>,
#   and a cluster with MTV installed (controller in konveyor-forklift by default).

set -euo pipefail

# --- Constants ---
NS="mtv-<number>-test"
PROVIDER="<type>-test"
PLAN="mtv-<number>-plan"
CONTROLLER_NS="${CONTROLLER_NS:-konveyor-forklift}"
SKIP_CLEANUP="${SKIP_CLEANUP:-false}"
POLL=15
# <other ticket-specific constants>

# ===================================================================
#  Cleanup (also runs on error via trap)
# ===================================================================
cleanup() {
  if [[ "${SKIP_CLEANUP}" == "true" ]]; then
    echo "SKIP_CLEANUP=true — preserving resources in namespace '${NS}' for forensic inspection."
    return 0
  fi
  echo "Cleaning up..."
  oc mtv delete plan     --name "${PLAN}" --skip-archive -n "${NS}" 2>/dev/null || true
  oc mtv delete provider --name "${PROVIDER}" -n "${NS}"            2>/dev/null || true
  oc mtv delete provider --name host -n "${NS}"                     2>/dev/null || true
  oc delete namespace "${NS}" --ignore-not-found                    2>/dev/null || true
  # <revert any settings overrides>
  echo "Cleanup done."
}
trap cleanup EXIT
cleanup

# ===================================================================
#  STEP 1: Create namespace
# ===================================================================
echo "STEP 1: Creating namespace '${NS}'"
oc create namespace "${NS}" --dry-run=client -o yaml | oc apply -f -

# ===================================================================
#  STEP 2: Create provider(s)
# ===================================================================
# <use oc mtv create provider with appropriate flags>
# For vSphere (insecure):
#   oc mtv create provider --name "${PROVIDER}" --type vsphere \
#     --url "${GOVC_URL}/sdk" --username "${GOVC_USERNAME}" \
#     --password "${GOVC_PASSWORD}" --provider-insecure-skip-tls -n "${NS}"
#
# For vSphere (with CA cert):
#   oc mtv create provider --name "${PROVIDER}" --type vsphere \
#     --url "${GOVC_URL}/sdk" --username "${GOVC_USERNAME}" \
#     --password "${GOVC_PASSWORD}" \
#     --cacert "$(fetch_ca_cert "${GOVC_URL}")" -n "${NS}"

echo "Creating host (local OpenShift) provider"
oc mtv create provider --name host --type openshift -n "${NS}"

echo "Waiting for providers Ready..."
oc wait "provider.forklift.konveyor.io/${PROVIDER}" -n "${NS}" \
  --for=condition=Ready --timeout=300s
oc wait "provider.forklift.konveyor.io/host" -n "${NS}" \
  --for=condition=Ready --timeout=300s

# ===================================================================
#  STEP 3: Create migration plan (if needed for this test)
# ===================================================================
# oc mtv create plan --name "${PLAN}" --source "${PROVIDER}" \
#   --vms "${VM}" -n "${NS}"
# oc wait "plan.forklift.konveyor.io/${PLAN}" -n "${NS}" \
#   --for=condition=Ready --timeout=300s

# ===================================================================
#  STEP <N>: <Ticket-specific test steps>
# ===================================================================
# <Insert the core test logic here — this is ticket-specific>

# ===================================================================
#  Summary
# ===================================================================
echo ""
echo "TEST PASSED: <what this confirms>"
exit 0
```

#### CA certificate helper

Include this function when TLS verification is needed (non-insecure providers):

```bash
fetch_ca_cert() {
  local hostport
  hostport=$(echo "$1" | sed -E 's|https?://||; s|/.*||')
  if ! echo "${hostport}" | grep -q ':'; then
    hostport="${hostport}:443"
  fi
  openssl s_client -showcerts -connect "${hostport}" </dev/null 2>/dev/null \
    | awk '/BEGIN CERTIFICATE/,/END CERTIFICATE/'
}
```

#### Rules for the script

- Always use `set -euo pipefail`
- Always register `trap cleanup EXIT` and call `cleanup` at the start
- Cleanup must be idempotent (`2>/dev/null || true` on all delete commands)
- Namespace name, provider name, and plan name are derived from the ticket number
- Use `oc wait --for=condition=Ready` with explicit timeouts after each resource creation
- Exit 0 = PASS, exit 1 = FAIL, exit 2 = INCONCLUSIVE (test ran but result is ambiguous)
- Add a clear `echo "TEST PASSED/FAILED/INCONCLUSIVE: <reason>"` before each exit
- Use numbered `STEP N:` echo banners so logs are easy to follow
- Variables that users commonly override go at the top as constants with defaults
- Support `SKIP_CLEANUP=true` to skip cleanup and preserve all resources for forensic inspection

**Present the script to the user and ask for permission to run it.** Do not run it automatically.

---

### Step 5 — Run and refine

After the user grants permission, run the script:

```bash
bash test-mtv-<number>.sh 2>&1 | tee test-mtv-<number>.log
```

After each run:
1. Read the full log output
2. Identify failures, unexpected output, or missing assertions
3. Propose specific fixes to the script
4. Ask the user whether to apply the fix and re-run

Repeat until the user is satisfied with the result or declares the test complete.

---

## Reference: Provider Creation Commands

### vSphere (insecure)
```bash
oc mtv create provider --name "${PROVIDER}" --type vsphere \
  --url "${GOVC_URL}/sdk" \
  --username "${GOVC_USERNAME}" \
  --password "${GOVC_PASSWORD}" \
  --provider-insecure-skip-tls \
  -n "${NS}"
```

### vSphere (with CA cert)
```bash
oc mtv create provider --name "${PROVIDER}" --type vsphere \
  --url "${GOVC_URL}/sdk" \
  --username "${GOVC_USERNAME}" \
  --password "${GOVC_PASSWORD}" \
  --cacert "$(fetch_ca_cert "${GOVC_URL}")" \
  -n "${NS}"
```

### oVirt / RHV
```bash
oc mtv create provider --name "${PROVIDER}" --type ovirt \
  --url "${RHV_URL}" \
  --username "${RHV_USERNAME}" \
  --password "${RHV_PASSWORD}" \
  --cacert "$(fetch_ca_cert "${RHV_URL}")" \
  -n "${NS}"
```

### OpenStack
```bash
oc mtv create provider --name "${PROVIDER}" --type openstack \
  --url "${OSP_URL}" \
  --username "${OSP_USERNAME}" \
  --password "${OSP_PASSWORD}" \
  --provider-domain-name "${OSP_DOMAIN_NAME}" \
  --provider-project-name "${OSP_PROJECT_NAME}" \
  --provider-region-name "${OSP_REGION_NAME}" \
  --cacert "$(fetch_ca_cert "${OSP_URL}")" \
  -n "${NS}"
```

### OVA
```bash
oc mtv create provider --name "${PROVIDER}" --type ova \
  --url "${OVA_URL}" \
  -n "${NS}"
```

### Remote OpenShift
```bash
oc mtv create provider --name "${PROVIDER}" --type openshift \
  --url "${OCP_URL}" \
  --provider-token "${OCP_TOKEN}" \
  -n "${NS}"
```
The remote OpenShift cluster is typically the source but can also serve as the target.
Use with the local OpenShift provider (below) which takes the opposite role.

### HyperV
```bash
oc mtv create provider --name "${PROVIDER}" --type hyperv \
  --url "${HV_URL}" \
  --username "${HV_USERNAME}" \
  --password "${HV_PASSWORD}" \
  --smb-url "${HV_SMB_URL}" \
  --provider-insecure-skip-tls \
  -n "${NS}"
```

### EC2
```bash
oc mtv create provider --name "${PROVIDER}" --type ec2 \
  --ec2-region "${EC2_REGION}" \
  --target-access-key-id "${EC2_ACCESS_KEY_ID}" \
  --target-secret-access-key "${EC2_SECRET_ACCESS_KEY}" \
  --target-az "${EC2_TARGET_AZ}" \
  --target-region "${EC2_TARGET_REGION}" \
  -n "${NS}"
```

### OpenShift (local cluster)
```bash
oc mtv create provider --name host --type openshift -n "${NS}"
```
The local OpenShift provider auto-detects the current cluster and needs no URL or token.
It is typically the target but can serve as the source when the remote OpenShift is the target.

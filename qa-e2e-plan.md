# /qa-e2e-plan — QA E2E Test Generator for Red Hat OpenShift Operators

> **What this file is:** a phased command specification. When a user says
> `/qa-e2e-plan` (or pastes this file to an AI agent), the agent executes
> every phase in order, printing progress after each step.
>
> **Shared rules:** the agent MUST also load and obey
> `rules/e2e-rules.md` for write-scope, dedup, guardrails.

---

## Phase 0 — Collect Inputs

Collect inputs from `.env` first. Only prompt the user for values that are
**missing**. If every variable is set, the tool runs non-interactively.

### 0.0  Load configuration from `.env`

If the file `.env` exists, load it:

```bash
if [ -f .env ]; then
  set -a; source .env; set +a
fi
```

This populates all `$VARIABLES` below. Any variable already set in the
environment (or passed on the command line) takes precedence.

### 0.1  Source (required — always from the user's prompt)

Extract the Jira ticket or GitHub PR URL from the user's message. If the
user did not include one, prompt:

```
"Enter a Jira link or GitHub PR URL:"
```

- If the value looks like a URL containing `/pull/` → treat as **GitHub PR**.
- If the value looks like a Jira key (e.g. `PROJ-123`) or a URL containing
  `/browse/` → treat as **Jira ticket**.
- Store as `$SOURCE_TYPE` (`pr` | `jira`) and `$SOURCE_URL`.

> **Note:** `SOURCE_URL` is intentionally NOT loaded from `.env`. It changes
> every run and should always be provided in the prompt.

### 0.2  Jira credentials (conditional)

If `$SOURCE_TYPE == jira`:

- Check `$JIRA_EMAIL` and `$JIRA_PERSONAL_TOKEN` (loaded from `.env` or
  environment). If both are set, skip the prompts below.
- If `$JIRA_EMAIL` is missing, prompt:
  ```
  "Jira / Atlassian email address:"
  ```
- If `$JIRA_PERSONAL_TOKEN` is missing, prompt:
  ```
  "Jira API token (or set JIRA_PERSONAL_TOKEN in .env):"
  ```
- If `$JIRA_BASE_URL` is missing, derive it from `$SOURCE_URL` (e.g.
  `https://redhat.atlassian.net` from
  `https://redhat.atlassian.net/browse/SPIRE-494`).
- Never log or commit the token.

### 0.3  Operator repo URL (required)

If `$REPO_URL` is set (from `.env`), use it. Otherwise prompt:

```
"Operator repo URL (e.g. https://github.com/org/operator-name):"
```

- Validate: `gh repo view <url> --json name` must succeed.
- Store as `$REPO_URL`, derive `$OWNER/$REPO`.

### 0.3.1  Upstream reference repo (optional)

If `$UPSTREAM_REPO_URL` is set (from `.env`), use it. Otherwise skip — this
is optional. When set, the tool shallow-clones the upstream repo in Phase 2.5
and uses its test patterns as a reference for code generation.

### 0.4  Target environment

If `$TARGET_ENV` is set (from `.env`), use it. Otherwise prompt:

```
"Target environment? (aws / azure / gcp / bare-metal / on-prem / all) [default: all]:"
```

Accepts a single value or a `/`-separated list (e.g. `aws / gcp / bare-metal`).
Default: `all`.

### 0.5  OpenShift version

If `$OCP_VERSION` is set (from `.env`), use it. Otherwise prompt:

```
"OpenShift cluster version? (4.14 / 4.16 / 4.18 / 4.19 / 4.20 / all) [default: all]:"
```

Accepts a single value or a `/`-separated list (e.g. `4.18 / 4.20`).
Default: `all`.

### 0.6  Red Hat scanning

If `$SCANNING` is set (from `.env`), use it. Otherwise prompt:

```
"Red Hat image scanning & signing required? (yes / no) [default: yes]:"
```

Default: `yes`.

### 0.7  Metrics

If `$METRICS` is set (from `.env`), use it. Otherwise prompt:

```
"Operator exports Prometheus metrics? (yes / no) [default: yes]:"
```

Default: `yes`.

### 0.8  OLM

If `$OLM` is set (from `.env`), use it. Otherwise prompt:

```
"Operator uses OLM for installation? (yes / no) [default: yes]:"
```

Default: `yes`.

### 0.9  Confirm and proceed

Print a summary table of all inputs (include `$UPSTREAM_REPO_URL` if set).

If `$AUTO_CONFIRM` is `true`, skip the confirmation prompt and proceed
immediately. Otherwise, ask the user to confirm before continuing.

---

## Phase 1 — Fetch Source Context

### If GitHub PR

```bash
gh pr view $PR_NUMBER --repo $OWNER/$REPO \
  --json title,body,files,headRefName,baseRefName,commits
```

Extract:
- `$PR_TITLE`, `$PR_BODY`, `$PR_FILES` (list of changed file paths),
  `$HEAD_BRANCH`, `$BASE_BRANCH`.
- Compute `$PR_DIFF`:
  ```bash
  gh pr diff $PR_NUMBER --repo $OWNER/$REPO
  ```

### If Jira

```bash
curl -s -u "$JIRA_EMAIL:$JIRA_PERSONAL_TOKEN" \
  "$JIRA_BASE_URL/rest/api/2/issue/$JIRA_KEY?fields=summary,description,issuetype,status,priority,labels,components" \
  | jq '.fields'
```

Extract: `$JIRA_SUMMARY`, `$JIRA_DESCRIPTION`, `$JIRA_TYPE`,
`$JIRA_LINKED_PRS` (from remote links or dev panel).

If linked PRs exist, fetch the diff for the first one via `gh pr diff`.

Print: source title, files changed count, commit count.

---

## Phase 2 — Clone & Auto-Discovery

```bash
CLONE_DIR=$(mktemp -d)
git clone "$REPO_URL" "$CLONE_DIR"
cd "$CLONE_DIR"
```

Run each discovery step and store results:

| Discovery | Command | Variable |
|---|---|---|
| Default branch | `git symbolic-ref refs/remotes/origin/HEAD \| sed 's@^refs/remotes/origin/@@'` | `$DEFAULT_BRANCH` |
| Framework | `grep -l 'operator-sdk\|controller-runtime\|library-go' go.mod` | `$FRAMEWORK` |
| Test directory | `find . -type d \( -name e2e -o -path '*/test/e2e' \)` | `$TEST_DIR` |
| API types | `find . -name '*_types.go'` | `$API_TYPES` |
| CSV file | `find . -name '*.clusterserviceversion.yaml' -o -name '*csv.yaml' \| head -1` | `$CSV_FILE` |
| CSV version | `yq '.spec.version' "$CSV_FILE"` (if CSV found) | `$CSV_VERSION` |
| CRDs | `find . -name '*.crd.yaml' -o -path '*/bases/*.yaml' \| wc -l` | `$CRD_COUNT` |
| Webhooks | `find . -name '*webhook*.go' \| wc -l` | `$WEBHOOK_COUNT` |
| RBAC manifests | `find . -name 'role*.yaml' -o -name '*clusterrole*.yaml' \| wc -l` | `$RBAC_COUNT` |
| ServiceMonitor | `find . -name '*servicemonitor*.yaml' \| wc -l` | `$HAS_METRICS` |
| SecurityContext | `rg 'runAsNonRoot\|allowPrivilegeEscalation' config/ --count` | `$HAS_SECURITY_CTX` |
| Operand CRDs | count of `*_types.go` files excluding the main operator type | `$OPERAND_COUNT` |

If `$TEST_DIR` is empty, set `$E2E_CODE_POSSIBLE = false`; the tool will
generate `test-cases.md` only.

Print a discovery summary table.

---

## Phase 2.5 — Clone & Analyze Upstream (if UPSTREAM_REPO_URL is set)

Skip this phase entirely if `$UPSTREAM_REPO_URL` is not set.

### 2.5.1  Shallow-clone the upstream repo

```bash
UPSTREAM_DIR=$(mktemp -d)
git clone --depth=1 "$UPSTREAM_REPO_URL" "$UPSTREAM_DIR"
```

### 2.5.2  Discover upstream test patterns

```bash
# Find test directory
UPSTREAM_TEST_DIR=$(find "$UPSTREAM_DIR" -type d \( -name e2e -o -path '*/test/e2e' \) | head -1)

# Extract spec tree (Describe / Context / It blocks)
rg 'Describe\(|Context\(|It\(' "$UPSTREAM_TEST_DIR" --glob '*_test.go' > /tmp/upstream_specs.txt

# Extract helper functions
rg '^func ' "$UPSTREAM_TEST_DIR" > /tmp/upstream_helpers.txt

# Extract constants and variables
rg '^\s*(const|var)\s' "$UPSTREAM_TEST_DIR" > /tmp/upstream_constants.txt
```

Store as `$UPSTREAM_SPECS`, `$UPSTREAM_HELPERS`, `$UPSTREAM_CONSTANTS`,
`$UPSTREAM_TEST_DIR`.

### 2.5.3  Print upstream analysis summary

Print a table with:
- Upstream repo URL
- Test directory path
- Number of test files
- Number of `It` blocks (spec count)
- Number of helper functions
- Number of constants

---

## Phase 3 — Analyze Changes & Build Context

### 3.1  Identify impacted domains

Read `$PR_DIFF` (or Jira description) and classify every changed file into
one or more domains:

| File path pattern | Domain(s) |
|---|---|
| `api/*_types.go` | reconciliation, negative-input-validation |
| `controllers/*_controller.go` | reconciliation, controller-manager |
| `config/rbac/` | rbac, openshift-rbac-scoping |
| `config/crd/` | negative-input-validation, csv-versioning |
| `config/webhook/` | negative-input-validation, webhook |
| `config/manager/` | install-health, controller-manager |
| `config/manifests/` | olm-lifecycle-install, csv-versioning |
| `test/` | (informational only — existing test coverage) |

### 3.2  Map to test domains

For each impacted domain, note whether a test already exists (from Phase 4
dedup) or whether a new test is needed.

### 3.3  Attach Red Hat mandatory checklist

Always append these domains regardless of PR diff:

- `install-health`
- `security-context` / `openshift-scc`
- `rbac` / `openshift-rbac-scoping`
- `openshift-monitoring` (if metrics=yes)
- `olm-lifecycle-install` (if OLM=yes)

---

## Phase 4 — Dedup

Run dedup queries against `$TEST_DIR`. For each domain, execute the
corresponding `rg` commands from the table below and record hits.

```bash
# Universal
rg 'Describe\(|Context\(|It\(' "$TEST_DIR" --glob '*_test.go' > /tmp/existing_specs.txt

# Per domain
rg 'CRD|Established' "$TEST_DIR"                           # install-health
rg 'SecurityContext|RunAsNonRoot|Capabilities' "$TEST_DIR"  # security-context
rg 'ClusterRole|Permission|RBAC' "$TEST_DIR"                # rbac
rg 'ConfigMap|Data\[' "$TEST_DIR"                           # configmap
rg 'Deployment|StatefulSet|DaemonSet' "$TEST_DIR"           # controller-manager
rg 'Create\(|Update\(|Delete\(' "$TEST_DIR"                 # reconciliation
rg 'invalid|reject|deny|forbidden' "$TEST_DIR" -i          # negative-input-validation
rg 'Subscription|InstallPlan|CSV' "$TEST_DIR"               # olm-lifecycle-install
rg 'upgrade|channel|replaces' "$TEST_DIR" -i                # olm-upgrade-path
rg 'uninstall|cleanup|finalizer' "$TEST_DIR" -i             # olm-uninstall
rg 'SCC|SecurityContextConstraint' "$TEST_DIR"              # openshift-scc
rg 'ServiceMonitor|Prometheus|/metrics' "$TEST_DIR"         # openshift-monitoring
rg 'audit|Audit' "$TEST_DIR"                                # openshift-audit
rg 'WaitFor.*Conditions' "$TEST_DIR"                        # condition-based waits
rg 'ResourceRequirements|Limits|Requests' "$TEST_DIR"       # resource limits
rg 'NodeSelector|Toleration|Affinity' "$TEST_DIR"           # scheduling
rg 'DeferCleanup' "$TEST_DIR"                               # cleanup patterns
```

For each domain, decide:

| Hits | Decision |
|---|---|
| Exact spec match with same assertions | `skip` — already covered |
| Partial match (same Context, different assertions) | `extend` — add assertions to existing `It` or new `It` in same `Context` |
| Related file exists but different scenario | `new-in-file` — add new `Context`/`It` in the existing file |
| No match at all | `new-file` — create new `*_test.go` (only if truly distinct) |

### 4.1  Cross-reference with upstream (if UPSTREAM_REPO_URL is set)

If `$UPSTREAM_TEST_DIR` was discovered in Phase 2.5, run the same dedup
queries against the upstream test directory for each scenario that was **not**
found in the target repo (i.e. decisions `new-in-file` or `new-file`):

```bash
rg '<scenario-keyword>' "$UPSTREAM_TEST_DIR" --glob '*_test.go'
```

If upstream has a matching test for the scenario, change the decision to
`adapt-from-upstream` — the upstream test will be used as a reference and
rewritten to use the target repo's helpers, constants, and naming conventions.

| Upstream hits | Decision |
|---|---|
| Upstream has a test for this scenario | `adapt-from-upstream` — rewrite upstream test using target repo's patterns |
| Upstream does not cover this scenario | Keep original decision (`new-in-file` / `new-file`) |

Record results in the coverage map table. Include the upstream file path and
spec name for `adapt-from-upstream` entries.

---

## Phase 5 — Generate test-cases.md (local output)

Write the test plan to a **local** output directory — it does NOT go into the
operator repo or the PR.

```bash
OUTPUT_DIR="output/${JIRA_KEY:-pr-$PR_NUMBER}"
mkdir -p "$OUTPUT_DIR"
```

Create `$OUTPUT_DIR/test-cases.md` using this template:

```markdown
# Test Plan: <$PR_TITLE or $JIRA_SUMMARY>

<!-- Auto-generated by /qa-e2e-plan -->
<!-- Source: <$SOURCE_URL> -->
<!-- Repo: <$OWNER/$REPO> | Branch: <$HEAD_BRANCH> | Framework: <$FRAMEWORK> -->
<!-- OLM: <yes/no> | CSV: <$CSV_VERSION> -->
<!-- OpenShift: <$OCP_VERSIONS> | Env: <$ENVIRONMENT> -->
<!-- Scanning: <$SCANNING> | Metrics: <$METRICS> -->

## Summary

<1–3 sentences: what changed and what must be tested.>

## Test Cases

### TC-001: <Title> [P0]
**Domain:** <domain-key>
**OpenShift-specific:** yes/no
**Scope:** <keywords>
**Prerequisites:** <cluster state, CRDs, namespaces>
**Steps:**
1. …
2. …
**Expected result:** …
**Stop condition:** <which later TCs are blocked if this fails>
**Environment notes:** <AWS/Azure/OCP version>
**Red Hat certification angle:** <what Red Hat requirement this validates>

(repeat for TC-002, TC-003, …)

## Coverage Map

| Scenario | Existing spec | Domain | Decision |
|---|---|---|---|
| <keyword> | <path:line or "(none)"> | <domain> | extend / new / skip |

## OLM Coverage
- Subscription install: covered / not covered
- Channel switching: covered / not covered
- Upgrade path: covered / not covered
- Dependency management: covered / not covered
- Uninstall cleanup: covered / not covered

## OpenShift Coverage
- SCC validation: covered / not covered
- RBAC scoping: covered / not covered
- Image scanning: covered / not covered
- Prometheus metrics: covered / not covered
- Audit logging: covered / not covered
- Version compatibility: covered / not covered

## Red Hat Certification Checklist
- [ ] OLM install
- [ ] SCC validation
- [ ] RBAC least-privilege
- [ ] Image scanning / signing
- [ ] Prometheus metrics
- [ ] Audit logging
- [ ] Version compatibility
- [ ] Uninstall cleanup
- [ ] Security context
```

Mark each checklist item as `[x]` if a TC covers it, or leave `[ ]` and
append a warning:

```
⚠ Red Hat certification likely requires tests for: <missing items>.
```

---

## Phase 6 — Generate E2E Code (if test directory exists)

Skip this phase entirely if `$E2E_CODE_POSSIBLE == false`. Instead, print:

```
ℹ No test/e2e/ directory found. Only test-cases.md was generated.
  To add e2e automation, create test/e2e/ in the operator repo first.
```

Otherwise, follow the guardrails from `rules/e2e-rules.md`:

### 6.1  Pre-generation analysis (E2E-0)

Read all existing `*_test.go` files. Extract:
- `Describe` / `Context` / `It` tree.
- `By()` descriptions.
- Helper functions in `utils.go` / `utils/` package.
- Constants in `utils/constants.go`.

### 6.1.5  For each TC with decision `adapt-from-upstream`

If `$UPSTREAM_TEST_DIR` exists and the coverage map contains
`adapt-from-upstream` entries:

1. Read the matching upstream `It` block (and its surrounding `Context`).
2. Study the upstream test's structure: setup, assertions, cleanup, helpers.
3. **Rewrite the test** for the target repo:
   - Replace upstream helper calls with the target repo's existing helpers
     (from `utils.go` / `utils/`).
   - Replace upstream constants with the target repo's constants (from
     `utils/constants.go`), or append new constants if needed.
   - Follow the target repo's `Describe` → `Context` → `It` structure.
   - Use the target repo's naming conventions and labels.
   - Apply `DeferCleanup`, `By()`, `Eventually`/`Consistently` patterns
     matching the target repo's style.
4. Do NOT copy upstream code verbatim. The upstream test is a **reference**
   for what to test and how, but the generated code must follow the target
   repo's standards.
5. Place the adapted test in the correct file and `Context` in the target
   repo (same rules as `new-in-file`).

### 6.2  For each TC with decision `extend`

- Open the existing test file.
- Locate the matching `Context` or `It`.
- Add new assertions or a new `It` inside the same `Context`.
- Reuse existing helpers; do NOT duplicate.

### 6.3  For each TC with decision `new-in-file` or `new-file`

1. Extract constants → `test/e2e/utils/constants.go`.
2. Extract reusable helpers → `test/e2e/utils.go` (or focused file).
3. Write the `It` block following the repo's style:
   - `By()` for every logical step.
   - `Eventually` / `Consistently` with timeouts from constants.
   - `DeferCleanup` for every resource created.
   - Domain label(s) on the `It`.
4. If a new `Context` is needed, place it inside the existing top-level
   `Describe` (same file) unless the domain is completely distinct.

### 6.4  Test patterns to apply (from ZTWIM analysis)

Use these proven patterns depending on what the test validates:

**Condition polling:**
```go
utils.WaitFor<Component>Conditions(testCtx, k8sClient, crName, map[string]metav1.ConditionStatus{
    "ServiceAccountAvailable": metav1.ConditionTrue,
    "Ready":                   metav1.ConditionTrue,
}, utils.DefaultTimeout)
```

**Rolling update tracking:**
```go
initialGen := deployment.Generation
// … apply CR change …
utils.WaitForDeploymentRollingUpdate(testCtx, clientset, name, ns, initialGen, utils.ShortTimeout)
utils.WaitForDeploymentAvailable(testCtx, clientset, name, ns, utils.DefaultTimeout)
```

**Pod lifecycle verification:**
```go
oldPodNames := make(map[string]struct{})
for _, pod := range pods.Items { oldPodNames[pod.Name] = struct{}{} }
// … delete pods …
Eventually(func() bool {
    newPods, _ := clientset.CoreV1().Pods(ns).List(ctx, opts)
    for _, p := range newPods.Items {
        if _, old := oldPodNames[p.Name]; old { return false }
        if p.Status.Phase != corev1.PodRunning { return false }
    }
    return true
}).WithTimeout(timeout).WithPolling(interval).Should(BeTrue())
```

**SecurityContext verification:**
```go
for _, pod := range pods.Items {
    for _, c := range pod.Spec.Containers {
        Expect(c.SecurityContext.RunAsNonRoot).To(Equal(ptr.To(true)))
        Expect(c.SecurityContext.AllowPrivilegeEscalation).To(Equal(ptr.To(false)))
        Expect(c.SecurityContext.Capabilities.Drop).To(ContainElement(corev1.Capability("ALL")))
    }
}
```

**Resource limits verification:**
```go
utils.VerifyContainerResources(pods.Items, expectedResources)
```

**ConfigMap deep field validation:**
```go
value, found, err := utils.GetNestedStringFromConfigMapJSON(ctx, clientset, ns, cmName, key, "server", "log_level")
Expect(err).NotTo(HaveOccurred())
Expect(found).To(BeTrue())
Expect(value).To(Equal("debug"))
```

**Namespace isolation for test data:**
```go
testNS := &corev1.Namespace{ObjectMeta: metav1.ObjectMeta{Name: "e2e-<domain>-test"}}
Expect(k8sClient.Create(testCtx, testNS)).To(Succeed())
DeferCleanup(func(ctx context.Context) { _ = k8sClient.Delete(ctx, testNS) })
```

### 6.5  Scope check (E2E-4)

```bash
git diff --name-only | grep -v '^test/' | grep -v '^test-cases.md$'
```

If non-test files appear, **remove them from staging** and report the issue.

---

## Phase 7 — Create Branch & Commit

```bash
# Ensure we're on the latest default branch
git checkout "$DEFAULT_BRANCH"
git pull origin "$DEFAULT_BRANCH"

# Derive branch name
SLUG=$(echo "$PR_TITLE" | tr '[:upper:]' '[:lower:]' | tr ' ' '-' | tr -cd 'a-z0-9-' | head -c 40)
BRANCH="qa/e2e-${PR_NUMBER:-${JIRA_KEY}}-${SLUG}"

# Create new branch from updated main
git checkout -b "$BRANCH"

# Stage only test/ files (test-cases.md stays in the local output dir)
if [ "$E2E_CODE_POSSIBLE" = "true" ]; then
  git add test/
fi

# Final scope guard — only test/ is allowed
OFFENDING=$(git diff --cached --name-only | grep -v '^test/')
if [ -n "$OFFENDING" ]; then
  echo "ERROR: files outside test/ staged: $OFFENDING"
  echo "Unstaging offending files..."
  echo "$OFFENDING" | xargs git reset HEAD --
fi

# Commit
git commit -m "qa: add e2e tests for ${PR_TITLE:-$JIRA_SUMMARY}

Generated by /qa-e2e-plan from ${SOURCE_URL}.

Test plan saved locally in $OUTPUT_DIR/test-cases.md."
```

---

## Phase 8 — Push & Create PR

If `$CREATE_DRAFT_PR` is `true` (from `.env` or environment), pass **`--draft`** to `gh pr create` so the PR opens as a **draft** for review before marking ready.

```bash
git push -u origin "$BRANCH"

DRAFT_FLAG=""
if [ "${CREATE_DRAFT_PR}" = "true" ]; then
  DRAFT_FLAG="--draft"
fi

gh pr create \
  --base "$DEFAULT_BRANCH" \
  --head "$BRANCH" \
  ${DRAFT_FLAG} \
  --title "QA: E2E tests for ${PR_TITLE:-$JIRA_SUMMARY}" \
  --body "$(cat <<'PREOF'
## E2E Test Generation

**Source:** $SOURCE_URL
**Generated by:** /qa-e2e-plan

### Test Coverage
$(grep '^### TC-' test-cases.md | sed 's/^### /- /')

### Red Hat Certification Checklist
$(grep '^\- \[' test-cases.md)

### Coverage Map
$(sed -n '/^## Coverage Map/,/^## /p' test-cases.md | head -n -1)

### Files Modified
$(git diff --cached --name-only | sed 's/^/- /')

### How to Run
\`\`\`bash
go test ./test/e2e/... -v -count=1
\`\`\`

> Full test plan is in the local output directory: \`$OUTPUT_DIR/test-cases.md\`
PREOF
)"
```

Print the PR URL.

---

## Phase 9 — Summary

Print a final summary:

```
========================================
QA E2E Generation Complete
========================================
Source:       $SOURCE_URL
Operator:     $OWNER/$REPO
Framework:    $FRAMEWORK
Test dir:     $TEST_DIR (or "none")
Output dir:   $OUTPUT_DIR
Branch:       $BRANCH
PR:           <PR URL>

Test cases:   <count>
Domains:      <list>
New specs:    <count>
Extended:     <count>
Skipped:      <count>

Red Hat Certification:
  ✓ covered: <list>
  ✗ missing: <list>

Scope check:  PASSED (test/ only)
========================================
```

---

## Edge Cases

| Scenario | Behaviour |
|---|---|
| No `test/e2e/` directory | Generate `test-cases.md` only; skip e2e code; print info message. |
| Jira with no linked PR | Use Jira description for context; skip PR diff analysis; generate test-cases.md based on Jira fields. |
| Missing Jira token | Prompt user; if still missing, abort with clear error. |
| `gh pr create` fails (permissions) | Print error, suggest user fork the repo or check `GH_TOKEN`. |
| Draft PRs | Set `CREATE_DRAFT_PR=true` in `.env` before Phase 8; `gh pr create` receives `--draft`. |
| Non-standard Jira issue type | Accept any type; use summary + description for context. |
| Invalid repo URL | Abort with error after `gh repo view` fails. |
| Missing `go.mod` | Warn "no Go module found"; still attempt test-cases.md generation. |
| Agent tries to edit non-test file | Scope check in Phase 7 removes it from staging; warn user. |
| PR has zero code changes | Generate baseline Red Hat certification test-cases.md with mandatory domains only. |

---

## Environment Variables

All Jira-related variables can be pre-configured in `.env` (see
Phase 0.0).

| Variable | Required | Purpose |
|---|---|---|
| `CREATE_DRAFT_PR` | No | When `true`, Phase 8 passes `--draft` to `gh pr create`. |
| `GH_TOKEN` | Yes (for `gh` CLI) | GitHub authentication for cloning, PR creation. |
| `JIRA_EMAIL` | Conditional (Jira input) | Atlassian account email for Basic Auth. Set in `.env`. |
| `JIRA_PERSONAL_TOKEN` | Conditional (Jira input) | Atlassian API token for Basic Auth. Set in `.env`. Never committed. |
| `JIRA_BASE_URL` | Conditional (Jira input) | Base URL of Jira instance (e.g. `https://redhat.atlassian.net`). Set in `.env`. |

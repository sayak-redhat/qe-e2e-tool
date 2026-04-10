# QA E2E Tool for Red Hat OpenShift Operators

An AI-driven test generator that produces human-readable test plans and
Ginkgo v2 e2e code for operators built on the
[Operator Framework](https://github.com/operator-framework) and
[OLM](https://olm.operatorframework.io/).

## How It Works

```
User provides PR/Jira  ──▶  Agent follows qa-e2e-plan.md  ──▶  test-cases.md + e2e code + PR
```

The tool is a **phased command specification** (`qa-e2e-plan.md`) that an AI
agent (Cursor, Claude Code, etc.) executes step by step. It is NOT a
standalone binary — it runs inside your AI coding assistant.

## Quick Start

### Prerequisites

| Tool | Purpose |
|------|---------|
| [GitHub CLI](https://cli.github.com/) (`gh`) | Clone repos, fetch PRs, create PRs |
| [yq](https://github.com/mikefarah/yq) | Parse YAML (CSV files, CRDs) |
| [ripgrep](https://github.com/BurntSushi/ripgrep) (`rg`) | Fast code search for dedup |
| `GH_TOKEN` env var | GitHub authentication |

### Configuration (Jira credentials)

Copy the `.env` template and fill in your Atlassian credentials:

```bash
cp qe-e2e-tool/.env.example qe-e2e-tool/.env
# Edit qe-e2e-tool/.env with your values:
#   JIRA_EMAIL=you@redhat.com
#   JIRA_PERSONAL_TOKEN=your-atlassian-api-token
#   JIRA_BASE_URL=https://redhat.atlassian.net
```

The `.env` file is git-ignored and never committed. If you skip this step, the
tool will prompt for credentials interactively when a Jira ticket is provided.

### Usage

1. Open your operator project (or any workspace) in Cursor.

2. Tell the agent to run the tool:

   ```
   Follow the instructions in qa-e2e-tool/qa-e2e-plan.md to generate
   e2e tests for this PR:
   https://github.com/my-org/my-operator/pull/42
   ```

   Or for a Jira ticket:

   ```
   Follow qa-e2e-tool/qa-e2e-plan.md to generate e2e tests for MYPROJ-123
   ```

3. The agent walks through interactive prompts (environment, OCP version,
   metrics, OLM) and then:

   - Clones the target operator repo
   - Auto-discovers framework, CRDs, webhooks, RBAC, metrics
   - Analyzes the PR diff for impacted test domains
   - Runs dedup queries against existing tests
   - Generates `test-cases.md` (human-readable test plan)
   - Generates Ginkgo e2e code (if `test/e2e/` exists)
   - Creates a branch, commits, pushes, and opens a PR

4. Review the PR. CI handles compilation and test execution.

## What Gets Generated

### test-cases.md (local only)

Written to `qe-e2e-tool/output/<jira-key>/test-cases.md` (or
`qe-e2e-tool/output/pr-<number>/`). This file is **NOT** committed to the
operator repo or included in the PR. A structured test plan with:

- Test cases tagged by domain (install-health, rbac, olm-lifecycle, etc.)
- Prerequisites, steps, expected results, stop conditions
- Coverage map showing which existing tests are extended vs. new
- OLM and OpenShift coverage summary
- Red Hat certification checklist

### E2E Code (in the PR)

Only `test/` directory changes are committed and pushed. Ginkgo v2 test specs
follow the target repo's existing structure:

- New `It` blocks inside existing `Context` groups (extend)
- New `Context` blocks in existing files (new-in-file)
- New test files only when the domain is completely distinct (new-file)
- Constants extracted to `test/e2e/utils/constants.go`
- Helpers extracted to `test/e2e/utils.go`

## File Structure

```
qe-e2e-tool/
  qa-e2e-plan.md          # Main command spec (agent follows this)
  README.md               # This file
  rules/
    e2e-rules.md          # Shared rules: write-scope, dedup, guardrails
  .env                    # Jira credentials (git-ignored)
  .env.example            # Template for .env
  .gitignore              # Ignores .env and output/
  output/                 # Local test plans (git-ignored)
    SPIRE-494/
      test-cases.md       # Generated test plan for SPIRE-494
    pr-57/
      test-cases.md       # Generated test plan for PR #57
```

## Supported Operators

Works with any Red Hat operator built on the Operator Framework:

| Operator Type | Example | Coverage |
|---|---|---|
| Single-component | cert-manager | 75% framework tests |
| Multi-operand | ZTWIM (4 operands) | 85% framework tests |
| Stateful (StatefulSet) | PostgreSQL | 80% framework tests |
| Cluster (quorum) | etcd | 75% framework tests |

The remaining 15–25% requires domain-specific expertise. The tool
explicitly flags those gaps.

## Test Domains

The tool covers 23 test domains organized in three tiers:

### Core (universal)
`install-health`, `security-context`, `rbac`, `configmap`,
`controller-manager`, `reconciliation`, `negative-input-validation`,
`negative-permission-validation`, `upgrade`

### OLM-specific
`olm-lifecycle-install`, `olm-upgrade-path`, `olm-uninstall`,
`olm-dependency-management`, `csv-versioning`

### OpenShift-specific
`openshift-scc`, `openshift-rbac-scoping`, `openshift-network-policy`,
`openshift-image-scanning`, `openshift-monitoring`, `openshift-logging`,
`openshift-audit`, `openshift-version-compat`, `openshift-fips-mode`

## Safety Guarantees

- **Write-scope:** Only the operator's `test/` directory is modified in the PR.
- **Local output:** `test-cases.md` is written locally to `qe-e2e-tool/output/`, never pushed.
- **Dedup:** Existing tests are extended, never duplicated.
- **Scope check:** `git diff --name-only` is verified before every commit.
- **No local gate:** The tool does not run `go test`; CI handles validation.
- **No secrets committed:** Jira credentials live in `.env` (git-ignored), never committed.

## Example Session

```
$ "Follow qa-e2e-tool/qa-e2e-plan.md for PR #57"

> Enter a Jira link or GitHub PR URL:
  https://github.com/sayak-redhat/zero-trust-workload-identity-manager/pull/57

> Operator repo URL:
  https://github.com/sayak-redhat/zero-trust-workload-identity-manager

> Target environment? [all]: aws
> OpenShift version? [all]: 4.14
> Red Hat scanning? [yes]: yes
> Metrics? [yes]: yes
> OLM? [yes]: yes

✓ PR #57: "Add SPIRE bundle injection webhook"
✓ Framework: operator-sdk | Test dir: test/e2e/
✓ CSV: v0.1.0 | Operands: 4 | Webhooks: 1

✓ Dedup: 3 extend, 2 new, 0 skip
✓ test-cases.md: 6 test cases, 6 domains
✓ e2e code: 2 new specs, 3 extended

✓ Branch: qa/e2e-57-spire-bundle-webhook
✓ PR: https://github.com/.../pull/999
✓ Red Hat certification: 8/8 ✓
✓ Scope check: test/ only ✓
```

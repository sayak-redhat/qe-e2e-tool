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

### Configuration

Copy the `.env` template and fill in your values:

```bash
cp .env.example .env
```

The `.env` file is git-ignored and never committed. It supports two modes:

**Minimal `.env` (interactive)** -- fill in only what you know; the tool
prompts for the rest at runtime:

```bash
JIRA_EMAIL=you@redhat.com
JIRA_PERSONAL_TOKEN=your-atlassian-api-token
JIRA_BASE_URL=https://redhat.atlassian.net
```

**Full `.env` (non-interactive)** -- fill in everything except the source
(which you pass in the prompt); the tool runs with zero extra prompts:

```bash
# Jira credentials
JIRA_EMAIL=you@redhat.com
JIRA_PERSONAL_TOKEN=your-atlassian-api-token
JIRA_BASE_URL=https://redhat.atlassian.net

# Operator repo
REPO_URL=https://github.com/org/operator-name

# Upstream reference repo (optional)
UPSTREAM_REPO_URL=https://github.com/spiffe/spire

# Cluster / environment (single value or slash-separated list)
TARGET_ENV=aws / azure / bare-metal / gcp
OCP_VERSION=4.18 / 4.19 / 4.20
SCANNING=yes            # yes | no
METRICS=yes             # yes | no
OLM=yes                 # yes | no

# Automation
AUTO_CONFIRM=true       # skip confirmation prompt
```

#### All `.env` variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `JIRA_EMAIL` | If Jira source | — | Atlassian account email |
| `JIRA_PERSONAL_TOKEN` | If Jira source | — | Atlassian API token |
| `JIRA_BASE_URL` | No | Derived from URL | e.g. `https://redhat.atlassian.net` |
| `REPO_URL` | Yes | — | Operator repository URL |
| `UPSTREAM_REPO_URL` | No | — | Upstream repo to study test patterns from (shallow-cloned) |
| `TARGET_ENV` | No | `all` | Single value or `/`-separated list: `aws`, `azure`, `gcp`, `bare-metal`, `on-prem`, `all` |
| `OCP_VERSION` | No | `all` | Single value or `/`-separated list: `4.14`, `4.16`, `4.18`, `4.19`, `4.20`, `all` |
| `SCANNING` | No | `yes` | Generate image scanning/signing tests? (`yes` / `no`) |
| `METRICS` | No | `yes` | Generate Prometheus metrics tests? (`yes` / `no`) |
| `OLM` | No | `yes` | Generate OLM lifecycle tests? (`yes` / `no`) |
| `AUTO_CONFIRM` | No | `false` | Skip the confirmation summary prompt |
| `CREATE_DRAFT_PR` | No | `false` | When `true`, Phase 8 runs `gh pr create --draft` on the operator repo |

`TARGET_ENV` and `OCP_VERSION` accept multiple values separated by ` / `
(e.g. `aws / gcp / bare-metal`). When multiple values are given, the
generated tests include environment-specific annotations and skip-conditions
for each target.

When `UPSTREAM_REPO_URL` is set, the tool shallow-clones the upstream repo,
scans its `test/` directory for existing test patterns, and cross-references
them with the Jira scenarios. If upstream already has a test for a scenario,
the tool adapts it to your repo's structure, helpers, constants, and naming
conventions instead of writing from scratch. The upstream code is a
**reference**, not a copy -- generated tests always follow your repo's
standards.

Any variable set in the environment overrides `.env`. Any missing variable
with a default uses that default silently. Any missing required variable
triggers an interactive prompt.

### Usage

1. Open this repo (or any workspace containing it) in Cursor.

2. Tell the agent to run the tool, passing the Jira ticket or PR URL in
   the prompt:

   ```
   Follow qa-e2e-plan.md to generate e2e tests for
   https://redhat.atlassian.net/browse/SPIRE-494
   ```

   Or for a GitHub PR:

   ```
   Follow qa-e2e-plan.md to generate e2e tests for
   https://github.com/my-org/my-operator/pull/42
   ```

3. The agent reads all other inputs from `.env` (prompting only for missing
   values) and then:

   - Clones the target operator repo
   - Auto-discovers framework, CRDs, webhooks, RBAC, metrics
   - Analyzes the PR diff for impacted test domains
   - Runs dedup queries against existing tests
   - Generates `test-cases.md` (human-readable test plan)
   - Generates Ginkgo e2e code (if `test/e2e/` exists)
   - Creates a branch, commits, pushes, and opens a PR

4. Review the PR. CI handles compilation and test execution.

### AI Agent Compatibility

| Agent | Rules loaded automatically? | Notes |
|-------|---------------------------|-------|
| **Cursor** | Yes — `.cursor/rules/e2e-rules.md` is auto-loaded | Guardrails active in every conversation |
| **Claude Code** | No — `qa-e2e-plan.md` loads `rules/e2e-rules.md` on demand | Same rules, loaded when the plan runs |
| **Other agents** | No — point the agent to `rules/e2e-rules.md` | Works with any agent that can read markdown |

## What Gets Generated

### test-cases.md (local only)

Written to `output/<jira-key>/test-cases.md` (or
`output/pr-<number>/`). This file is **NOT** committed to the
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
  .cursor/
    rules/
      e2e-rules.md          # Cursor auto-loads this (always-on guardrails)
  rules/
    e2e-rules.md            # Portable copy for non-Cursor agents
  qa-e2e-plan.md            # Main command spec (agent follows this)
  README.md                 # This file
  .env.example              # Template for .env (all configurable variables)
  .env                      # Your configuration (git-ignored, create from .env.example)
  .gitignore                # Ignores .env and output/
  output/                   # Local test plans (git-ignored)
    SPIRE-494/
      test-cases.md         # Generated test plan for SPIRE-494
    pr-57/
      test-cases.md         # Generated test plan for PR #57
```

### Why two copies of `e2e-rules.md`?

The rules file exists in two places to support different agents and usage
scenarios:

| File | Loaded by | When | Purpose |
|------|-----------|------|---------|
| `.cursor/rules/e2e-rules.md` | **Cursor IDE** (automatically) | Every conversation, always-on | Guardrails are active even when you are **not** running the tool (e.g., editing test files manually). Cursor scans `.cursor/rules/` at workspace startup and injects every `.md` file as agent instructions. |
| `rules/e2e-rules.md` | **`qa-e2e-plan.md`** (explicitly, line 8) | Only when the tool runs | Portable copy that works with **any** AI agent (Claude Code, Copilot, etc.). Non-Cursor agents don't know about `.cursor/rules/`. |

Both files have identical content. The duplication is intentional:

- **Without `.cursor/rules/`** -- A Cursor user could start a normal
  conversation (not running the tool) and accidentally edit files outside
  `test/` with no guardrails stopping them.
- **Without `rules/`** -- Claude Code or any non-Cursor agent would have no
  way to load the rules, since `.cursor/` is a Cursor-specific convention
  they ignore.

If you only use Cursor, you can delete `rules/e2e-rules.md` and update the
reference in `qa-e2e-plan.md` (line 8) to point to
`.cursor/rules/e2e-rules.md` instead. But keeping both gives you portability
at the cost of one duplicated file.

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
- **Local output:** `test-cases.md` is written locally to `output/`, never pushed.
- **Dedup:** Existing tests are extended, never duplicated.
- **Scope check:** `git diff --name-only` is verified before every commit.
- **No local gate:** The tool does not run `go test`; CI handles validation.
- **No secrets committed:** Jira credentials live in `.env` (git-ignored), never committed.

## Example Session

### Non-interactive (full `.env`)

With all variables set in `.env` and `AUTO_CONFIRM=true`:

```
$ "Follow qa-e2e-plan.md"

✓ Loaded .env (all inputs configured, AUTO_CONFIRM=true)
✓ Source: SPIRE-494 (Jira)
✓ Repo: sayak-redhat/zero-trust-workload-identity-manager
✓ Env: aws / azure / bare-metal / gcp
✓ OCP: 4.18 / 4.19 / 4.20
✓ Scanning: yes | Metrics: yes | OLM: yes

✓ Framework: operator-sdk | Test dir: test/e2e/
✓ CSV: v1.0.0 | Operands: 4 | Webhooks: 1

✓ Dedup: 3 extend, 2 new, 0 skip
✓ test-cases.md: 6 test cases, 6 domains
✓ e2e code: 2 new specs, 3 extended

✓ Branch: qa/e2e-SPIRE-494-workload-attestation
✓ PR: https://github.com/.../pull/2
✓ Red Hat certification: 8/8 ✓
✓ Scope check: test/ only ✓
```

### Interactive (minimal `.env`)

With only Jira credentials in `.env`, the tool prompts for the rest:

```
$ "Follow qa-e2e-plan.md for SPIRE-494"

> Operator repo URL:
  https://github.com/sayak-redhat/zero-trust-workload-identity-manager

> Target environment? [all]: aws / gcp
> OpenShift version? [all]: 4.18 / 4.20
> Red Hat scanning? [yes]: yes
> Metrics? [yes]: yes
> OLM? [yes]: yes

  (same output as above)
```

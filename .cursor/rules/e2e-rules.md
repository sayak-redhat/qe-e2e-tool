# E2E Rules for Red Hat OpenShift Operators

Apply whenever you add, extend, or refactor tests in any operator repository
(especially `test/e2e/` and Ginkgo specs). Follow these rules even if the
active file is not under `test/`.

---

## 1. Write-scope restriction (NON-NEGOTIABLE)

- **ONLY** create or modify files inside the target operator's `test/`
  directory (e.g. `test/e2e/`).
- The generated `test-cases.md` is written to the **local** output directory
  (`output/<key>/`), NOT into the operator repo.
- **NEVER** touch `cmd/`, `pkg/`, `api/`, `config/`, `go.mod`, `go.sum`,
  `Makefile`, `Dockerfile`, or any file outside the test directory.
- If a non-test change is required for the test to work (e.g. a missing
  exported type), **report it as a suggestion** in the local
  `test-cases.md` instead of making the edit.
- Before every commit, run:
  ```bash
  git diff --name-only | grep -v '^test/'
  ```
  If that command produces output, **abort the commit** and remove the
  offending files from staging.

---

## 2. No duplicate e2e coverage

Before adding any new `It` / `Describe` / file:

1. Search the entire target repo under `test/` for specs that already exercise
   the same feature: matching CR kinds, field or env names, package/API paths
   from the PR diff, or obvious keywords in `By(...)` / spec names.
   ```bash
   rg '<keyword>' test/ --glob '*_test.go'
   ```
2. If existing e2e already covers the behaviour (same assertions and setup, or
   a small extension suffices): **do not add a parallel spec**. Extend the
   existing `Context`/`It` or table entry, or stop and report
   `covered by <path>:<spec>` in the coverage map.
3. Add new specs **only** when no current test reasonably covers the scenario.

---

## 3. Structure and file roles (match the target repo)

Follow the existing e2e layout in the repo. Do not invent a new structure if
one already exists.

| File / package | Role |
|---|---|
| `e2e_suite_test.go` | Suite entrypoint, `BeforeSuite`/`AfterSuite`, shared clients. Keep thin. |
| `test/e2e/utils/constants.go` | Fixed values: well-known names, labels, namespaces, image refs, **all** timeout / interval constants. |
| `test/e2e/utils.go` (or `utils/` package) | Reusable helper functions: `WaitFor…`, `Verify…`, `Get…`, resource builders. No duplicates. |
| `*_test.go` (e.g. `e2e_test.go`) | Test cases only (`Describe` → `Context` → `It`). After dedup, reuse or extend existing flows; never add a second spec that duplicates the same checks. |

---

## 4. Domain tagging (mandatory)

Every generated `It` block must carry at least one Ginkgo v2 `Label` matching
a domain key:

```
install-health, security-context, rbac, configmap, controller-manager,
reconciliation, negative-input-validation, negative-permission-validation,
upgrade, olm-lifecycle-install, olm-upgrade-path, olm-uninstall,
olm-dependency-management, csv-versioning, openshift-scc,
openshift-rbac-scoping, openshift-network-policy, openshift-image-scanning,
openshift-monitoring, openshift-logging, openshift-audit,
openshift-version-compat, openshift-fips-mode
```

Example:
```go
It("should reject invalid bundle format", Label("negative-input-validation", "webhook"), func() { ... })
```

---

## 5. OLM-aware testing

- Tests that install the operator **must** use the OLM path
  (Subscription → InstallPlan → CSV), not raw `kubectl apply`.
- Include channel-switching tests when the operator supports multiple channels.
- Use `Eventually` with timeouts >= 60 s for OLM operations (CSV activation
  can take 30–90 s).
- Verify `OperatorCondition.Upgradeable` transitions correctly when operand
  pods are unhealthy.

---

## 6. OpenShift-aware testing

- Assume tests run on a genuine OpenShift cluster, not vanilla Kubernetes.
- Verify the operator works under the `restricted-v2` SCC (default on
  OpenShift 4.11+).
- For every operand pod, validate the full `securityContext`:
  ```
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities.drop: [ALL]
  seccompProfile.type: RuntimeDefault
  ```
- Check the audit log for sensitive operations where relevant.

---

## 7. Red Hat certification mindset

Every test should answer: *"Does this operator meet Red Hat certification
requirements?"*

Mandatory checklist (tool warns if any are missing):

- [ ] OLM install (if OLM-distributed)
- [ ] SCC validation
- [ ] RBAC least-privilege
- [ ] Image scanning / signing (if applicable)
- [ ] Prometheus metrics export (if applicable)
- [ ] Audit logging
- [ ] Operator version compatibility (min 2 OCP versions)
- [ ] Uninstall cleanup
- [ ] Security context (runAsNonRoot, readOnly filesystem)

---

## 8. E2E code generation guardrails

### Phase E2E-0 — Pre-generation analysis

```bash
find test/e2e -name '*_test.go' | sort
rg 'Describe\(|Context\(|It\(' test/e2e/ --glob '*_test.go'
rg '^func ' test/e2e/utils.go test/e2e/utils/*.go 2>/dev/null
rg '^const |^var ' test/e2e/utils/constants.go 2>/dev/null
```

### Phase E2E-1 — Dedup with reusability check

1. Search existing test files for the scenario keyword.
2. If found in an existing `It` block with the same assertions → mark
   `extend`, do NOT add a new spec.
3. If found in the same file but different `Context`/`It` → add a new `It` in
   the **same file**, reusing helpers.
4. If truly new → proceed to Phase E2E-2.

### Phase E2E-2 — Helper and constant extraction

- **NEVER modify or delete existing constants** in
  `test/e2e/utils/constants.go`. Existing values may be depended on by other
  tests. Only **append** new constants when a genuinely new value is needed.
- Immutable values (timeouts, names, labels, ports) go into
  `test/e2e/utils/constants.go`.
- Reusable functions (used in ≥ 2 tests) go into `test/e2e/utils.go` or a
  focused utils file.
- One-off helpers stay private inside the test file.

### Phase E2E-3 — Test code generation

- Follow the repo's `Describe` → `Context` → `It` structure.
- Use constants from `utils/constants.go` — no hardcoded values.
- Call helpers — no inline wait loops.
- Tag every `It` with domain labels.
- Use `DeferCleanup` for every created resource.
- Use `BeforeEach` with `context.WithTimeout` for test isolation.

### Phase E2E-4 — Scope check before commit

```bash
git diff --name-only | grep -v '^test/' | grep -v '^test-cases.md$'
# Must produce no output.
```

---

## 9. Ginkgo v2 / Gomega patterns

- `Describe` → `Context` → `It`, with `By()` for each logical step.
- `Eventually(...).WithTimeout(t).WithPolling(p).Should(...)` for async waits.
- `Consistently(...).WithTimeout(t).WithPolling(p).Should(...)` to prove
  something does NOT change (e.g. drift not corrected in CreateOnlyMode).
- `Ordered` on the top-level `Describe` when tests share state
  (installation → configuration → uninstall).
- `DeferCleanup(func(ctx context.Context) { ... })` for resource teardown.
- `BeforeAll` for expensive one-time setup (cluster discovery, subscription
  lookup).
- `BeforeEach` for per-test context isolation:
  ```go
  BeforeEach(func() {
      var cancel context.CancelFunc
      testCtx, cancel = context.WithTimeout(context.Background(), utils.TestContextTimeout)
      DeferCleanup(cancel)
  })
  ```

---

## 10. Style and reviewability

- Go: idiomatic naming, small functions, explicit error handling.
- Prefer code a reviewer can follow quickly: linear flow inside `It` blocks,
  named helpers for non-obvious steps, avoid deep nesting.
- Do not add comments that simply narrate what the code does. Comments explain
  *why*, not *what*.

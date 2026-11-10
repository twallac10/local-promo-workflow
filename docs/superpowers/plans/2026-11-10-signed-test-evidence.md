# Signed Test Evidence with Testkube — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Behaviour tests run in Testkube for every release in every tier; their JUnit becomes an OCI artifact and a Sigstore-signed test-result attestation on the chart digest; the gate merges a promotion only on a verified PASSED attestation for the source tier and release.

**Architecture:** `promo-fleet` installs Testkube (chart mirrored to our GHCR) and a shared `promo-report` TestWorkflowTemplate. Each wrapper chart ships a behaviour `TestWorkflow` and a post-install/post-upgrade hook `TestWorkflowExecution`, so each Helm revision runs once; its final always-run step (the template) sends JUnit to the app repo by `repository_dispatch`. The reusable promote workflow verifies the report against `main`, attaches `junit.xml` (oras), attests (actions/attest), records the Deployment and opens/closes PRs; the gate verifies the attestation with `gh attestation verify`.

**Tech Stack:** Testkube 2.15.0 (TestWorkflow, TestWorkflowTemplate, TestWorkflowExecution), Flux 2.10, Helm hooks, GitHub Actions (`actions/attest` v4.2.2, OIDC), oras 1.3.6, `gh attestation verify`, yq (incl. `-p xml`), jq, bash 3.2.

**Spec:** `docs/superpowers/specs/2026-11-09-test-evidence-design.md` (approved 2026-11-10), plus the Phase 0–3 plans and ledgers.

## Global Constraints

- Phase 0–3 Global Constraints still hold (OCI-only charts under `oci://ghcr.io/twallac10/charts/`, bash 3.2, yq/jq, pinned versions, no PR code in the gate, payload fields never interpolated in `run:`).
- Testkube: chart `kubeshop/testkube` **2.15.0**, mirrored to `oci://ghcr.io/twallac10/charts/testkube`; values `testkube-api.nats.enabled=false`, `testkube-api.nats.embedded=true`; namespace `testkube`; images stay on docker.io (charts-only rule, like the canary), pinned by the chart and checked by `task images:check`.
- Behaviour tests and their execution live in the wrapper chart (`chart/templates/behaviour.yaml`), rendered into namespace `testkube`. Execution name: `{{ .Release.Name }}-behaviour-r{{ .Release.Revision }}`.
- Dispatch event type for reports: `TestReport/<app>.<app>`. `client_payload` fields: `app`, `tier`, `release_id`, `digest`, `revision`, `execution`, `junit_b64` (base64 of `junit.xml`, ≤ 48 KiB decoded). The workflow computes PASSED/FAILED from the JUnit itself.
- JUnit: one `<testsuite name="<app>">`; one `<testcase>` per behaviour check; a check that never reported counts as an `<error>`.
- Attestation: subject `ghcr.io/twallac10/charts/promo-<app>` + chart digest; predicate type `https://in-toto.io/attestation/test-result/v0.1`; predicate `{result, passedTests, warnedTests: [], failedTests, url: <run url>, configuration: [{name: "tiers/<tier>/release.yaml", digest: {sha256: <release id hex>}, annotations: {tier, app, revision, execution}}]}`; `push-to-registry: true`.
- OCI artifact: `oras attach --artifact-type application/vnd.promo.junit.v1+xml` to the chart digest, file `junit.xml` (media type `application/xml`), annotations `promo.tier`, `promo.release-id`.
- Signer accepted by the gate: certificate identity matching `^https://github\.com/twallac10/promo-workflows/\.github/workflows/promote\.yaml@refs/tags/v[0-9]+\.[0-9]+\.[0-9]+$`, source repo `twallac10/promo-<app>`, source ref `refs/heads/main`.
- Which result counts: for (digest, tier, release id), the attestation with the latest Rekor integrated time.
- A GitHub Deployment (task `promote`) = the behaviour verdict; Flux smoke **success** no longer records anything (Alert `promote-success` removed); smoke **failure** still records `failure`.
- Pins: `actions/attest@1e69f48acb82d1966a394da916b4c1698aa569d6 # v4.2.2`; oras 1.3.6 linux_amd64 sha256 `66fb0ad7206f3f6fa77ed5b552049d364ccf4b50b1684554f731f76302b52ca0`.
- `promo-workflows` **v0.6.0** for mirror, promote and guardrails.

## Review Focus

1. **The reporter itself fails** (token missing, GitHub unreachable): no evidence arrives and promotion silently stalls. Expected: the execution fails visibly and `task retest` recovers. Test: `test_report_fails_loudly_when_dispatch_fails` (Task 2).
2. **A behaviour step never runs** (aborted, timed out): fewer testcases must not look like "all passed". Expected: missing expected checks become `<error>`. Test: `test_report_marks_missing_checks_as_errors` (Task 2).
3. **Several attestations on one digest** (dev PASSED, stg FAILED, a re-run): the gate must pick by tier and release id, newest first. Test: `test_attestation_picks_latest_for_tier_and_release` (Task 4).
4. **Forged evidence:** an attestation signed by another workflow, repo, ref or a moved branch; a junit artifact pushed by hand. Expected: rejected. Tests: `test_attestation_verify_enforces_signer` (Task 4) and Task 7 Step 5 (live forgery).
5. **A report for an old revision arrives late** (a newer upgrade already merged): no attestation for the stale release. Test: `test_report_stale_release_not_attested` (Task 3).

---

### Task 1: Testkube in every tier

**Files:**
- Create: `promo-fleet/testkube/{kustomization,namespace,ocirepository,helmrelease}.yaml`, `testkube/source.yaml` (upstream + version, like `canary/source.yaml`)
- Modify: `.github/workflows/mirror-canary.yaml` → `mirror-platform.yaml` (mirrors every `*/source.yaml`), `clusters/<tier>/platform.yaml` (Kustomization `testkube`, `wait: true`), `apps.yaml` (every app `dependsOn` `testkube`), `scripts/gen-clusters.sh` (allow `testkube`), `scripts/dispatch-secret.sh` (also a Secret in `testkube`), `scripts/images-check.sh` (render testkube), `tests/test_wiring.sh`, `tests/test_fleet.sh`

**Interfaces:**
- Produces: Kustomization `testkube` in `flux-system` on every cluster; Secret `testkube/promo-dispatch` (key `token`); TestWorkflow API available before any app installs.

- [ ] **Step 1: Failing tests:** extend `test_dependson_order` (every `app-*` depends on `testkube`; `testkube` depends on nothing); `test_dispatch_secret_manifests` (a Secret in `testkube` too); `test_testkube_values` (rendered HelmRelease values set `nats.enabled=false`, `nats.embedded=true`; OCIRepository under our prefix, digest-pinned).
- [ ] **Step 2:** Run `tests/run.sh`. Expected: FAIL.
- [ ] **Step 3:** Implement; regenerate `clusters/*/apps.yaml`; push; let `mirror-platform` mirror testkube 2.15.0 and pin its digest.
- [ ] **Step 4:** Tests and `task images:check` PASS. Live: `task up -- dev` on the running cluster converges; record dev footprint (expect +0.5–0.8 GiB vs. the Phase 3 numbers). Commit, push.

### Task 2: The `promo-report` template

**Files:**
- Create: `promo-fleet/testkube/report.sh`, `testkube/promo-report.yaml` (TestWorkflowTemplate), ConfigMap from `report.sh` via `configMapGenerator` (no name hash), `tests/test_report.sh`, `tests/fixtures/bin/{kubectl,curl}` stubs

**Interfaces:**
- Consumes (env, from the app's TestWorkflow config): `APP`, `TIER`, `EXPECTED` (space-separated check names); files `/data/junit/<check>.xml` (one `<testcase>` each); Secret `promo-dispatch`.
- Produces: TestWorkflowTemplate `promo-report` (a single step, `condition: always`) that runs `report.sh`: builds `junit.xml`, reads `release_id` (annotation `event.toolkit.fluxcd.io/release-id`), `digest` and `revision` from HelmRelease `<APP>` in namespace `<APP>` (`status.history[0]`), and POSTs the dispatch. Exits non-zero (with the reason) if the dispatch fails. Saves `junit.xml` as a Testkube artifact.

- [ ] **Step 1: Failing tests** (stubs; no cluster): `test_report_builds_junit` (two passing case files → one suite, `tests="2" failures="0"`, valid XML via `yq -p xml`); `test_report_marks_missing_checks_as_errors` (EXPECTED has 3, files for 2 → `errors="1"`, the missing name present); `test_report_payload` (curl stub receives event_type `TestReport/kyverno.kyverno` and every field from Global Constraints; `junit_b64` decodes to the built file); `test_report_fails_loudly_when_dispatch_fails` (curl stub exits 22 → non-zero exit, stderr names the repo).
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement. RBAC: the template's pod reads `helmreleases` in app namespaces (ClusterRole `promo-report`, bound to the service account the app TestWorkflows use, created here: `testkube/promo-behaviour`).
- [ ] **Step 4:** Tests PASS. Live (dev): apply a throwaway TestWorkflow that writes one case and `use`s the template; confirm the dispatch reaches `promo-kyverno` (event visible as a promote run, which may reject it until Task 3 lands). Commit, push.

### Task 3: Promote workflow handles test reports

**Files:**
- Modify: `promo-workflows/actions/promote/{parse-event.sh,action.yml,deploy.sh}`, `.github/workflows/promote.yaml` (permissions `id-token: write`, `attestations: write`, `packages: write`)
- Create: `actions/promote/{parse-report.sh,attach-junit.sh,predicate.sh}`, `tests/test_report.sh`, fixtures `tests/fixtures/report-{pass,fail}.json`

**Interfaces:**
- `parse-report.sh <event.json> <repo-name>` → `kind=report`, `outcome=success|failure`, `app`, `tier`, `digest`, `release_id`, `revision`, `execution`, `passed`, `failed` (comma lists), `upgrade` (revision > 1), and writes `$RUNNER_TEMP/junit.xml`. Same field strictness as `parse-event.sh`; rejects non-XML, > 48 KiB, or zero testcases.
- `predicate.sh <tier> <app> <release_id> <revision> <execution> <junit.xml> <run-url>` → predicate JSON per Global Constraints.
- `attach-junit.sh <app> <digest> <tier> <release_id> <junit.xml>` → `oras attach …`; prints `junit_ref=`.
- `parse-event.sh` (Flux) now accepts only failure events; `TestSucceeded` is rejected (no Alert sends it).
- `action.yml`: one parse step dispatching on `github.event.action` prefix (`TestReport/` vs `HelmRelease/`); report path: verify → attach → `actions/attest` (subject name/digest, predicate-type, predicate-path, push-to-registry) → deploy (task promote) → success: open-pr (evidence gains a test table and links to the junit ref and the attestation); failure: close-promotion, revert (rollback tier, upgrade).

- [ ] **Step 1: Failing tests:** `test_parse_report_pass`, `test_parse_report_fail`, `test_parse_report_rejects_bad_junit`, `test_predicate_shape` (jq checks every field), `test_attach_junit_command` (oras stub sees artifact type, file, annotations, `@<digest>`), `test_report_stale_release_not_attested` (wiring: attest/attach/deploy/open-pr all require `status == 'ok'`), `test_flux_success_no_longer_accepted`, `test_workflow_permissions`.
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement; install oras pinned (sha256) in the action.
- [ ] **Step 4:** Tests and `actionlint` PASS. Commit (tag with Task 4).

### Task 4: Gate verifies the attestation

**Files:**
- Create: `promo-workflows/actions/guardrails/check-attestation.sh`, tests in `tests/test_guardrails.sh`
- Modify: `actions/guardrails/action.yml` (promotion: `check-attestation` replaces `check-evidence`; revert keeps `check-evidence`), `VERSION` 0.6.0, pins; tag `v0.6.0`

**Interfaces:**
- `check-attestation.sh <owner/repo> <app> <digest> <source-tier> <release_id>` → runs `gh attestation verify oci://ghcr.io/twallac10/charts/promo-<app>@<digest> -R <owner/repo> --predicate-type https://in-toto.io/attestation/test-result/v0.1 --cert-identity-regex <Global Constraints> --source-ref refs/heads/main --format json`; keeps entries whose predicate configuration has `annotations.tier == <source-tier>` and `digest.sha256 == <release id hex>`; takes the latest by integrated time; requires `result == "PASSED"`. Prints `attestation=<tier> <result> <time>`.

- [ ] **Step 1: Failing tests** (gh stub returning canned verify JSON): `test_attestation_requires_passed_for_tier_and_release`, `test_attestation_picks_latest_for_tier_and_release` (older PASSED + newer FAILED → fail; older FAILED + newer PASSED → ok), `test_attestation_verify_enforces_signer` (stub records args: identity regex, `-R`, `--source-ref`, `--predicate-type`), `test_attestation_none` (verify exit 1 → fail with reason).
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement; gate needs `attestations: read` (Task 5 adds it to `gate.yaml`).
- [ ] **Step 4:** Tests, `actionlint` PASS. VERSION 0.6.0, pin, commit, tag `v0.6.0`, push.

### Task 5: Template: callers, notify and gate permissions

**Files:**
- Modify: template `tiers/*/notify.yaml` (remove `promote-success`), `.github/workflows/promote.yaml` (types add `TestReport/example.example`; permissions per Task 3), `.github/workflows/gate.yaml` (`attestations: read`), pins → v0.6.0, `tests/{test_notify,test_mirror,test_gate}.sh`

- [ ] **Step 1: Failing tests:** `test_notify_shape` (only `promote-failure`); `test_promote_caller` (both types; permissions); `test_gate_uses_mains_definition` (permissions include `attestations: read`).
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement; `actionlint`; template tests PASS; commit, push; `task repos:sync`; app tests PASS; push.

### Task 6: Behaviour tests in the charts

**Files (each app repo):**
- Create: `chart/templates/behaviour.yaml` (TestWorkflow `<release>-behaviour` in `testkube`, SA `promo-behaviour`, steps per check writing `/data/junit/<check>.xml`, then `template: promo-report`; plus hook `TestWorkflowExecution` per Global Constraints, `helm.sh/hook: post-install,post-upgrade`)
- Modify: smoke tests (keep readiness only), `chart/values.yaml` (`behaviour.checks`, `behaviour.tier`), `tiers/<tier>/tier.yaml` (`behaviour.tier: <tier>`, kyverno-policies `behaviour.expectEnforce`), `chart/Chart.yaml` version bump, `tests/test_app.sh`

| App | Smoke (Helm test) | Behaviour checks (Testkube) |
|---|---|---|
| kyverno | upstream readiness checkers | `deny-dry-run` |
| kyverno-policies | `policies-ready` | `privileged-pod` (denied when Enforce, admitted with an audit report when Audit) |
| prometheus | `ready` | `canary-target-up` |
| otel-collector | `health` | `accepted-spans` |

- [ ] **Step 1: Failing tests** per app: `test_behaviour_workflow` (rendered TestWorkflow in `testkube`, one step per check name, last step uses `promo-report`, `EXPECTED` equals the check list, `TIER` from values); `test_behaviour_execution_hook` (name includes the revision, hook annotations); `test_smoke_only_readiness` (test hook names exactly the smoke column); kyverno-policies `test_behaviour_tier_expectation` (Enforce in prd, Audit in dev/stg).
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement; bump chart versions; tests PASS; push. Mirror PRs open and merge themselves into dev (gate: other/dev).
- [ ] **Step 4:** Live: dev runs behaviour for each app; each app repo gets a `TestReport`; GHCR shows the junit artifact and the attestation on each digest (`gh attestation verify … --format json`); dev `success` Deployments; stg promotion PRs open with test tables.

### Task 7: Retest, notes and live acceptance

**Files:** `promo-fleet/scripts/retest.sh`, `Taskfile.yml` (`retest`), `tests/test_fleet.sh`, `NOTES.md`

**Interfaces:** `task retest -- <tier> <app>` applies `TestWorkflowExecution` `<app>-behaviour-retest-<unix time>` for the current release.

- [ ] **Step 1:** Failing test `test_retest_manifest` (dry-run output: name, namespace `testkube`, workflow ref); implement; PASS; commit.
- [ ] **Step 2:** stg promotion PRs pass the gate on dev's attestation and merge themselves; stg reports; prd PRs wait for approval; after approval prd reports and records evidence.
- [ ] **Step 3:** Ready-but-broken: drop the canary scrape job in prometheus dev (`release.yaml` change via a PR) → smoke passes, behaviour FAILS → FAILED attestation, `failure` Deployment, no stg PR. Revert it.
- [ ] **Step 4:** `task retest -- dev prometheus` produces a new attestation; the latest wins.
- [ ] **Step 5:** Forgery: with the owner's token, push a junit artifact and a self-signed attestation (`gh attestation` can't; use `cosign attest` keyless from a laptop identity) on a dev digest claiming PASSED for a new release id → a hand-made promotion PR's gate rejects it on signer identity.
- [ ] **Step 6:** NOTES: "Test evidence" (what's signed, by whom, what it proves and doesn't, the AKS path, how to verify by hand with `gh attestation verify`), footprint for 3 tiers with Testkube, Testkube startup restarts. Update the ledger and `RESUME.md`.

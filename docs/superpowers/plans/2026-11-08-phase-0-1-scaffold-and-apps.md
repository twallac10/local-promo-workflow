# Phases 0–1: Scaffold, Charts, Tests and Apps Across Tiers — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Three kind clusters (dev, stg, prd), each Flux-synced to its own path in `promo-fleet`, running Kyverno, kyverno-policies, Prometheus, OTel Collector, guardrails and a podinfo canary — all charts pulled only from GHCR as wrapper charts whose Helm tests pass.

**Architecture:** One GitHub repo per app (made from `promo-app-template`), one fleet repo (`promo-fleet`) holding tier config, cluster sync paths, canary and guardrails, and one repo of reusable workflows (`promo-workflows`). Each app's HelmRelease is `base/` + two Kustomize patches per tier: `release.yaml` (promotable: OCI digest + shared values) and `tier.yaml` (tier-only). Clusters run `flux install` and sync a public `GitRepository` read-only; no write-back token.

**Tech Stack:** kind 0.33, Flux 2.10 (`OCIRepository` `source.toolkit.fluxcd.io/v1`, `HelmRelease` `helm.toolkit.fluxcd.io/v2`), Helm v4.3, Task 3.54, yq 4, jq, gh 2.103, GitHub Actions, GHCR. Tests are bash 3.2-compatible scripts.

**Spec:** `promotion-demo-handoff.md` (Phases 0–1), amended by `LESSONS.md` and these confirmed decisions:
- Native Flux + Actions; no custom controller.
- Apps: Kyverno (admission + reports controllers only), kyverno-policies, plain Prometheus (no operator, no Grafana), OTel Collector; podinfo canary (not promoted).
- Repo prefix `promo-` under `twallac10`. Old `platform-*` repos and `local-promo` are not touched.
- Tiers dev/stg/prd; only prd requires approval (prevent-self-review off). Signing out of scope.

## Global Constraints

- Owner `twallac10`; repos: `promo-fleet`, `promo-workflows`, `promo-app-template`, `promo-kyverno`, `promo-kyverno-policies`, `promo-prometheus`, `promo-otel-collector`. All public.
- Local checkouts live in `~/projects/promo/<repo>`; this repo (`local-promo-workflow`) holds only docs and plans.
- Charts consumed only from `oci://ghcr.io/twallac10/charts/<chart>`. No `HelmRepository` anywhere in cluster manifests.
- Wrapper chart name = `promo-<app>` (e.g. `promo-kyverno`); the published OCI name must equal `Chart.yaml` `name`.
- Upstream pins (verified 2026-11-08): `kyverno/kyverno` 3.9.2 (app v1.19.2), `kyverno/kyverno-policies` 3.9.2, `prometheus-community/prometheus` 29.45.1 (app v3.16.0), `open-telemetry/opentelemetry-collector` 0.178.2 (app 0.163.2), podinfo 6.17.0.
- Every pinned image must have both `linux/amd64` and `linux/arm64`.
- Wrapper values nest under the upstream dependency name (`kyverno:`, `kyverno-policies:`, `prometheus:`, `opentelemetry-collector:`) in both `release.yaml` and `tier.yaml`.
- `release.yaml` patches only: `OCIRepository.spec.ref.digest`, annotation `promo.twallac10.dev/chart-version: <semver>`, and shared `HelmRelease.spec.values`. `tier.yaml` patches everything tier-specific (replicas, retention, `validationFailureAction`, remediation).
- HelmRelease: `spec.chartRef` → `OCIRepository`; `test.enable: true`; `test.ignoreFailures: false`.
- `OCIRepository.spec.layerSelector`: `mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip`, `operation: copy`.
- Docker budget: 6 CPU, 12.5 GiB, arm64. kind nodes put etcd on tmpfs. `task up` builds clusters one at a time and waits for Ready.
- Scripts must run on macOS bash 3.2 (no empty-array expansion under `set -u`, no `sha256sum`; use `shasum -a 256`). Use `yq`/`jq`, not Python packages.
- `.gitignore` committed before any other file in every repo.

## Review Focus

1. **Flux installs OCI charts as `<version>+<digest>`.** Any chart putting `.Chart.Version` in a label breaks install. Test: `test_labels_with_build_suffix` in Task 4 renders each wrapper with `--version 0.1.0+abc` and validates every label value.
2. **Published chart name drift.** A mismatch republishes on every run and clusters pull a name that doesn't exist. Test: `test_chart_name_matches_ocirepository` in Task 5.
3. **Kyverno's fail-closed webhook taking down Flux.** If Kyverno is unhealthy, it must not block `flux-system`/`kube-system`. Test: `test_kyverno_excludes_system_namespaces` in Task 4.
4. **Bootstrap ordering.** In dev (`hold`) a failed first-install test is never retried, so anything a test exercises must be Ready before that app installs. Test: `test_dependson_order` in Task 6 asserts the dependency DAG, and Task 8 verifies first-install convergence from a fresh `task up -- dev`.
5. **Idempotent lifecycle.** `task up -- dev` on an existing cluster and `task down` with no clusters must both exit 0. Test: `test_up_down_idempotent` in Task 3.

---

## File Structure

```
promo-fleet/
  .gitignore
  Taskfile.yml                 # repos:create, up, down, lint, images:check, status
  tiers.yaml  apps.yaml
  kind/<tier>.yaml             # kind cluster config, etcd on tmpfs
  clusters/<tier>/
    flux-sync.yaml             # GitRepository promo-fleet + Kustomization ./clusters/<tier>
    apps.yaml                  # per app: GitRepository (app repo) + Kustomization ./tiers/<tier>
    platform.yaml              # Kustomizations for guardrails/ and canary/
  guardrails/                  # Kyverno ClusterPolicies: OCI-only rules
  canary/                      # podinfo OCIRepository + HelmRelease (from our GHCR)
  scripts/                     # repos-create.sh, up.sh, down.sh, images-check.sh, mirror-canary.sh
  tests/run.sh  tests/test_*.sh
  NOTES.md
promo-workflows/
  .github/workflows/mirror.yaml   # reusable (workflow_call)
  .github/workflows/ci.yaml       # actionlint
promo-app-template/  (and each promo-<app> created from it)
  base/{kustomization,ocirepository,helmrelease}.yaml
  tiers/<tier>/{kustomization,release,tier}.yaml
  chart/{Chart.yaml,values.yaml,templates/tests/*.yaml}
  mirror/chart.yaml
  .github/workflows/{mirror,ci}.yaml
  tests/run.sh  tests/test_*.sh
```

---

## Phase 0: Scaffold

### Task 1: `promo-workflows` repo

**Files:**
- Create: `promo-workflows/.gitignore`, `.github/workflows/ci.yaml`, `README.md`

**Interfaces:**
- Produces: repo `twallac10/promo-workflows`, tag `v0.1.0`. Later tasks add `mirror.yaml` (Task 5).

- [ ] **Step 1:** `gh repo create twallac10/promo-workflows --public`, clone to `~/projects/promo/promo-workflows`, commit `.gitignore` first.
- [ ] **Step 2:** Add `ci.yaml` running `actionlint` (pinned `rhysd/actionlint` by SHA) on push and PR.
- [ ] **Step 3:** Run `actionlint` locally. Expected: no output, exit 0.
- [ ] **Step 4:** Commit, push, tag `v0.1.0`. Verify `gh run list -R twallac10/promo-workflows -L1` shows `success`.

### Task 2: `promo-app-template` repo shape

**Files:**
- Create: `base/kustomization.yaml`, `base/ocirepository.yaml`, `base/helmrelease.yaml`, `tiers/{dev,stg,prd}/{kustomization,release,tier}.yaml`, `chart/Chart.yaml` (placeholder `name: promo-app`), `mirror/chart.yaml`, `.github/workflows/ci.yaml`, `tests/run.sh`, `tests/test_tiers.sh`, `.gitignore`
- No `notify.yaml` yet (Phase 2).

**Interfaces:**
- Produces: `tests/run.sh` runs every `tests/test_*.sh`, prints `PASS <name>`/`FAIL <name>`, exits non-zero on any failure. Each app repo inherits it.
- Produces: `mirror/chart.yaml` schema:
  ```yaml
  upstream: {repo: https://..., chart: <name>, version: <semver>}
  wrapper: {name: promo-<app>}
  ```
- Produces: base object names `ocirepository/<app>` and `helmrelease/<app>` in namespace `<app>`; `kustomization.yaml` in each tier lists `../../base` plus patches `release.yaml`, `tier.yaml`.

- [ ] **Step 1: Write failing tests** in `tests/test_tiers.sh`:
  - `test_each_tier_builds`: `kustomize build tiers/$t` (via `kubectl kustomize`) succeeds for dev, stg, prd and passes `kubeconform -strict` with Flux CRD schemas.
  - `test_release_only_promotable_fields`: `yq` over each `release.yaml` — the only paths set are `OCIRepository.spec.ref.digest`, `metadata.annotations["promo.twallac10.dev/chart-version"]`, `HelmRelease.spec.values.*`. Any other path fails.
  - `test_tier_edit_is_isolated`: change a value in `tiers/dev/tier.yaml` in a temp copy; `kustomize build` output for stg and prd is byte-identical to before.
  - `test_helmrelease_tests_enabled`: rendered HelmRelease has `test.enable: true`, `test.ignoreFailures: false`, `chartRef.kind: OCIRepository`.
- [ ] **Step 2:** Run `tests/run.sh`. Expected: FAIL (files absent).
- [ ] **Step 3:** Write the base and tier files to satisfy the constraints above. Remediation goes in `tier.yaml`: dev has none (hold); stg and prd set `upgrade.remediation.strategy: rollback`, `retries: 0`. Check field names with `kubectl explain`-equivalent against the Flux 2.10 CRD (`flux install --export | yq`).
- [ ] **Step 4:** Run `tests/run.sh`. Expected: all PASS.
- [ ] **Step 5:** `gh repo create twallac10/promo-app-template --public --template`-enabled (`gh repo edit --template`), commit, push. CI runs `tests/run.sh` and passes.

### Task 3: `promo-fleet` scaffold, kind clusters, Flux sync

**Files:**
- Create: `promo-fleet/.gitignore`, `tiers.yaml` (exact block from the handoff, cluster names `kind-tier-dev|stg|prd`), `apps.yaml`, `kind/{dev,stg,prd}.yaml`, `clusters/<tier>/flux-sync.yaml`, `Taskfile.yml`, `scripts/{repos-create,up,down}.sh`, `tests/run.sh`, `tests/test_fleet.sh`, `NOTES.md`

**Interfaces:**
- Produces: `apps.yaml`:
  ```yaml
  apps:
    - {name: kyverno,          repo: promo-kyverno,          namespace: kyverno,         dependsOn: []}
    - {name: kyverno-policies, repo: promo-kyverno-policies, namespace: kyverno,         dependsOn: [kyverno]}
    - {name: prometheus,       repo: promo-prometheus,       namespace: monitoring,      dependsOn: [kyverno]}
    - {name: otel-collector,   repo: promo-otel-collector,   namespace: otel,            dependsOn: [kyverno]}
  ```
- Produces task targets: `repos:create`, `up` (`task up` all tiers in `tiers.yaml` order; `task up -- dev` one tier), `down`, `lint`, `status`.
- Produces: kubectl contexts `kind-tier-dev|stg|prd`.

- [ ] **Step 1: Write failing tests** in `tests/test_fleet.sh`:
  - `test_tiers_yaml_shape`: every tier has `name`, non-empty `clusters`, `requiresApproval` bool, `onFailure ∈ {hold, rollback}`; exactly the first tier is `hold`.
  - `test_kind_etcd_tmpfs`: each `kind/<tier>.yaml` mounts tmpfs at `/var/lib/etcd` (via `kubeadmConfigPatches` or node `extraMounts`) and names cluster `tier-<tier>`.
  - `test_cluster_paths`: for each tier, `clusters/<tier>/flux-sync.yaml` has a `Kustomization` with `path: ./clusters/<tier>`.
  - `test_up_down_idempotent` (live, skipped unless `LIVE=1`): `task up -- dev` twice exits 0 both times; `task down` twice exits 0 both times.
- [ ] **Step 2:** Run `tests/run.sh`. Expected: FAIL.
- [ ] **Step 3:** Implement. `up.sh <tier>`: `kind create cluster` if absent → `flux install` (pinned to the installed CLI's version) → apply `clusters/<tier>/flux-sync.yaml` → `flux reconcile ks flux-system` → `kubectl wait` all Kustomizations Ready (timeout 15m) before the next tier. `repos-create.sh`: for each app in `apps.yaml`, `gh repo create twallac10/<repo> --public --template twallac10/promo-app-template` if absent, then set its `chart/Chart.yaml` name and `mirror/chart.yaml`. `NOTES.md`: start with the machine budget and the decision to use `flux install` + public read-only sync instead of `flux bootstrap` (no fleet write token needed).
- [ ] **Step 4:** Run `tests/run.sh`, then `LIVE=1 tests/run.sh`. Expected: all PASS; `flux get ks --context kind-tier-dev` shows `flux-system` Ready at `./clusters/dev`.
- [ ] **Step 5:** `task repos:create`. Verify: `gh repo list twallac10 --json name -q '.[].name' | grep '^promo-'` lists all 7 repos; `diff -r` of `base/ tiers/ .github/` across the 4 app repos is empty.
- [ ] **Step 6:** Commit and push `promo-fleet`. `task down`.

**Phase 0 check-in:** report to the user (repos exist, `task up -- dev` works, idle footprint of an empty dev cluster).

---

## Phase 1: Charts, tests and apps

### Task 4: Upstream inventory, wrapper charts and tests

**Files (per app repo):**
- Create: `chart/Chart.yaml` (dependency on the pinned upstream from `oci`/https repo, `name: promo-<app>`, `version: 0.1.0`), `chart/values.yaml`, `chart/templates/tests/*.yaml`, `tests/test_chart.sh`
- Create in `promo-fleet`: `scripts/images-check.sh`, `NOTES.md` sections "Upstream Helm tests", "CRDs and rollback safety", "Image architectures"

**Interfaces:**
- Produces: `task images:check` — renders every wrapper chart, extracts every image, runs `docker manifest inspect` (or `docker buildx imagetools inspect`), fails listing any image missing `linux/arm64` or `linux/amd64`.
- Produces: test hook pods named `<release>-test-<check>`, annotated `helm.sh/hook: test` and `helm.sh/hook-delete-policy: before-hook-creation` (so logs survive until the next run).

- [ ] **Step 1: Inventory, then stop.** For each pinned upstream chart, record in `NOTES.md`: shipped `templates/tests` (names + what they check), CRDs and how they ship (`crds/` vs templates vs subchart), and all images with arch support. Note: kyverno-policies 3.9.2 defaults to `policyType: ValidatingPolicy`, so "all ClusterPolicies Ready" becomes "all policies of the rendered kind Ready". **Report the inventory to the user and wait for go-ahead before Step 2.**
- [ ] **Step 2: Write failing tests** in each app's `tests/test_chart.sh`:
  - `test_labels_with_build_suffix`: `helm template chart --version 0.1.0+abc123` → every `metadata.labels` value matches `^(([A-Za-z0-9][-A-Za-z0-9_.]*)?[A-Za-z0-9])?$` and ≤63 chars.
  - `test_has_tests`: rendered output contains ≥1 Pod with `helm.sh/hook: test`.
  - kyverno: `test_kyverno_controllers` (only admission + reports controllers enabled; background and cleanup controllers `enabled: false`), `test_kyverno_excludes_system_namespaces` (webhook namespace selector excludes `flux-system`, `kube-system`, `kyverno`).
  - prometheus: `test_prometheus_minimal` (alertmanager, pushgateway, kube-state-metrics, node-exporter disabled); `test_promtool_config` runs the pinned image's `promtool check config` on the rendered config.
  - otel-collector: `test_otel_validate` runs the pinned collector image's `validate` on the rendered config.
- [ ] **Step 3:** Run `tests/run.sh` in each app repo. Expected: FAIL.
- [ ] **Step 4:** Implement wrapper charts and tests (keep each test a single small Pod; give test Pods a minimal ServiceAccount + RBAC where they call the API):
  - kyverno: webhook Service responds; server-side dry-run of a privileged Pod in the `kyverno` namespace is denied or produces a policy report (runs in the app's own namespace, not the canary's — LESSONS #8).
  - kyverno-policies: every rendered policy reports Ready.
  - prometheus: `/-/ready`; canary target `up` via `/api/v1/targets` (scrape config for podinfo lives in shared values, `release.yaml`).
  - otel-collector: health endpoint OK; `otelcol_receiver_accepted_spans` metric present (canary/loadgen sends spans; if no span source exists yet, say so to the user rather than inventing one).
- [ ] **Step 5:** Run `tests/run.sh` everywhere and `task images:check`. Expected: all PASS. If an image lacks arm64, pick the nearest version that has both and record why in `NOTES.md`.
- [ ] **Step 6:** Commit per repo.

### Task 5: Reusable mirror workflow → GHCR → dev PR

**Files:**
- Create: `promo-workflows/.github/workflows/mirror.yaml` (reusable), `promo-app-template` and each app: `.github/workflows/mirror.yaml` caller pinned to `promo-workflows@v0.2.0`; `tests/test_mirror.sh` in each app repo.

**Interfaces:**
- Consumes: `mirror/chart.yaml` (Task 2), `tiers.yaml` fetched publicly from `https://raw.githubusercontent.com/twallac10/promo-fleet/main/tiers.yaml`.
- Produces: OCI artifact `ghcr.io/twallac10/charts/promo-<app>:<wrapper version>`; PR from branch `mirror/<short-digest>` setting `tiers/<first tier>/release.yaml` `spec.ref.digest` + version annotation.
- Trigger: push to `main` touching `mirror/chart.yaml` or `chart/**`, plus `workflow_dispatch`. `permissions: {contents: write, packages: write, pull-requests: write}`.

- [ ] **Step 1: Write failing test** `test_chart_name_matches_ocirepository`: `chart/Chart.yaml` `.name` == `mirror/chart.yaml` `.wrapper.name` == last path segment of `base/ocirepository.yaml` `spec.url`, and the URL starts with `oci://ghcr.io/twallac10/charts/`.
- [ ] **Step 2:** Run. Expected: FAIL until names are aligned.
- [ ] **Step 3:** Implement the reusable workflow: `helm dependency build chart` → `helm package` → `helm push` to `oci://ghcr.io/twallac10/charts` → capture digest from push output → if the digest differs from dev's `release.yaml`, open/update the PR (`gh pr create`, idempotent on branch). Make each package public after first push (`gh api` on the package visibility; record in `NOTES.md` if the API can't do it and it must be done once in the UI).
- [ ] **Step 4:** `actionlint`; tag `promo-workflows` `v0.2.0`; push each app repo. Expected: mirror run succeeds, `helm show chart oci://ghcr.io/twallac10/charts/promo-<app>` works **anonymously** (`helm registry logout ghcr.io` first), and a PR to `tiers/dev/release.yaml` exists.
- [ ] **Step 5:** Merge the dev PR, then set stg and prd `release.yaml` to the same digest in one bootstrap commit per repo (initial seed; promotion handles later changes). Commit.

### Task 6: Fleet wiring — apps, guardrails, canary per tier

**Files:**
- Create: `promo-fleet/clusters/<tier>/apps.yaml` (generated from `apps.yaml` by `scripts/gen-clusters.sh`), `clusters/<tier>/platform.yaml`, `guardrails/{kustomization,oci-only}.yaml`, `canary/{kustomization,ocirepository,helmrelease}.yaml`, `scripts/mirror-canary.sh`, `tests/test_wiring.sh`

**Interfaces:**
- Consumes: `apps.yaml` (Task 3), published charts (Task 5).
- Produces: Flux Kustomization names `app-<name>`, `guardrails`, `canary` in `flux-system` on every cluster; GitRepository `app-<name>` → `https://github.com/twallac10/<repo>` branch `main`.
- Canary chart: podinfo 6.17.0 copied to `oci://ghcr.io/twallac10/charts/podinfo` by `mirror-canary.sh` (run once; it's not promoted).

- [ ] **Step 1: Write failing tests** in `tests/test_wiring.sh`:
  - `test_generated_clusters_current`: `scripts/gen-clusters.sh` output equals committed `clusters/*/apps.yaml`.
  - `test_dependson_order`: `app-kyverno-policies` and `guardrails` depend on `app-kyverno`; `canary` depends on nothing; `app-prometheus` and `app-otel-collector` depend on `canary` (their tests exercise it; plain Prometheus has no ServiceMonitor CRD, so the canary needs nothing from it); every app's Kustomization path is `./tiers/<tier>`.
  - `test_no_helmrepository`: no `kind: HelmRepository` anywhere in the repo; every `OCIRepository.spec.url` starts with `oci://ghcr.io/twallac10/charts/`.
  - `test_guardrail_policy_rejects` : `kyverno apply` (CLI, pinned) of `guardrails/` against fixtures `tests/fixtures/{helmrepository,foreign-oci,our-oci}.yaml` → first two fail, third passes.
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement. Guardrail policy action is `Enforce` in every tier (it's platform safety, not the promoted Audit/Enforce setting).
- [ ] **Step 4:** Run `tests/run.sh`. Expected: all PASS. Commit, push.

### Task 7: Tier-specific config

**Files:**
- Modify: each app repo `tiers/<tier>/tier.yaml`; `promo-kyverno-policies` `tests/test_tier_values.sh`

**Interfaces:**
- Consumes: wrapper value nesting (Task 4).

- [ ] **Step 1: Write failing test** `test_validation_failure_action`: rendered HelmRelease values `kyverno-policies.validationFailureAction` is `Audit` in dev and stg, `Enforce` in prd, and is absent from every `release.yaml`.
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Set tier values: kyverno-policies action as above; prometheus `retention` 2h dev / 6h stg / 12h prd and single replica everywhere; remediation as in Task 2.
- [ ] **Step 4:** Run all app tests. Expected: PASS. Commit, push.

### Task 8: Live bring-up — dev first, then stg and prd

**Files:**
- Modify: `promo-fleet/NOTES.md` (footprint section), `Taskfile.yml` `status` target

**Interfaces:**
- Produces: `task status` — per cluster: every Flux Kustomization and HelmRelease with Ready status and the **full** condition message (not truncated), plus pod restart counts.

- [ ] **Step 1:** `task up -- dev`. Expected: all Kustomizations and HelmReleases Ready; `flux get hr -A --context kind-tier-dev` shows each release's last test run succeeded.
- [ ] **Step 2:** Wait 30 minutes. Expected: `task status` restart counts all 0; record `docker stats --no-stream` CPU/memory for the dev node in `NOTES.md`. If idle CPU > 1 core or restarts > 0, stop and report to the user before adding tiers.
- [ ] **Step 3:** Acceptance checks on dev:
  - `kubectl --context kind-tier-dev apply --dry-run=server -f tests/fixtures/helmrepository.yaml` → denied by guardrails.
  - Every `OCIRepository` URL in the cluster starts with our GHCR prefix (`kubectl get ocirepositories -A -o json | jq`).
  - kyverno-policies policies report `Audit`.
- [ ] **Step 4:** `task up -- stg`, then `task up -- prd`, each to Ready. Repeat Step 3 on each; prd policies report `Enforce`. Record each cluster's footprint and the total in `NOTES.md`.
- [ ] **Step 5:** Tier isolation live: change one value in `promo-prometheus/tiers/dev/tier.yaml`, push; only `kind-tier-dev`'s HelmRelease gets a new revision (`flux get hr prometheus -n monitoring` on all three). Revert.
- [ ] **Step 6:** `task down && task up` from scratch, confirm all three converge with no manual steps. Commit `NOTES.md`.

**Phase 1 check-in:** report acceptance results and footprint to the user before Phase 2.

# Handoff: Flux-Native Evidence-Gated Promotion of Platform Apps (local demo)

## Before you start
Read this whole file, then **ask me any open questions before writing code**. Confirm the plan for Phases 0–1 first. Prefer short, direct check-ins over long summaries.

## Problem
We promote **platform apps** (Kyverno, Prometheus/Grafana, OTel, etc.) through **6 tiers** via PRs. Each app has **its own deployment repo**. Each tier is its own set of AKS clusters. Human approvals at every tier are slow and mostly rubber stamps.

Goal: a change moves to tier N only once it is **proven in tier N-1**, and the promotion PR is generated automatically. No soak time: promote as soon as the evidence passes.

Promotable = chart **OCI digest** + shared settings. **Never** promote tier-specific config.

## Approach: native Flux + GitHub Actions (no custom controller)
Based on Flux's documented pattern ("Promote Flux Helm Releases with GitHub Actions"), extended to N tiers, multiple repos, and GitHub Environments:

1. Flux upgrades the HelmRelease in tier N-1 and **runs its Helm tests**.
2. On success, Flux's notification-controller sends a `githubdispatch` event to the app's repo, with the tier and chart digest.
3. A GitHub Actions workflow verifies the event against Git, records a GitHub Deployment for tier N-1, and opens the promotion PR for tier N.
4. Guardrails and the target tier's Environment gate the PR; merge → Flux deploys tier N → repeat.

**Evidence today = Helm tests.** Richer checks (Prometheus, Dynatrace, Testkube) are the next step. Build the seam for them now (see "Next: richer evidence"), but don't build the checks yet.

## What the demo must show
1. A new chart version enters tier 1 and flows tier → tier automatically when Helm tests pass.
2. Each promotion is a PR in the app's repo whose diff touches **only** the target tier's `release.yaml`, with the evidence in the body.
3. A release whose Helm tests fail is **blocked** at that tier, with a `failure` Deployment showing why.
4. Lower tiers auto-merge; the final tier waits for environment approval.
5. Tier-specific config (e.g. Kyverno `Audit` vs `Enforce`) never rides along with a promotion.

## Repos (all public, one GitHub owner)
| Repo | Contents |
|---|---|
| `platform-fleet` | `tiers.yaml`, `apps.yaml`, `clusters/<tier>/` (Flux bootstrap + a `GitRepository` and `Kustomization` per app repo), `canary/`, `guardrails/` (Kyverno guard policies), `kind/`, the main `Taskfile.yml` |
| `platform-kyverno` | app repo |
| `platform-kyverno-policies` | app repo |
| `platform-kube-prometheus-stack` | app repo |
| `platform-otel-collector` | app repo |
| `platform-workflows` | **reusable workflows** every app repo calls: mirror, promote, guardrails |

Create the app repos from one template (`platform-app-template`) via `task repos:create` so they stay identical. Name prefix and owner are placeholders; confirm with me.

**App repo shape:**
```
base/                      # OCIRepository + HelmRelease (chartRef, test.enable: true)
tiers/<tier>/
  kustomization.yaml
  release.yaml             # PROMOTABLE: chart OCI digest, image tags, shared values
  tier.yaml                # TIER-ONLY: replicas, retention, hostnames, Audit/Enforce
  notify.yaml              # Flux Provider (githubdispatch) + Alert for this tier
chart/                     # thin wrapper chart: upstream as dependency + our tests (see below)
mirror/chart.yaml          # upstream chart source + pinned version
.github/workflows/
  mirror.yaml              # caller -> reusable mirror workflow
  promote.yaml             # caller -> reusable promote workflow (repository_dispatch)
  guardrails.yaml          # caller -> reusable guardrails workflow (PRs)
```
Rule: `release.yaml` and `tier.yaml` are separate Kustomize patches on the base HelmRelease. Promotion = copy `release.yaml` from tier N-1 to tier N, nothing else. `notify.yaml` is tier-specific and never promoted.

`platform-fleet` has a Flux `Kustomization` per app per cluster pointing at `./tiers/<tier>` in that app's `GitRepository` (public, no auth). Dependency order: `kyverno` → `kyverno-policies` and `guardrails`; `kube-prometheus-stack` before anything with a ServiceMonitor; `canary` before any app whose tests use it.

## Topology
- **3 kind clusters** stand in for the tiers: `tier-dev`, `tier-stg`, `tier-prd` (placeholder names).
- The tier chain lives in `platform-fleet/tiers.yaml`. The demo runs 3 tiers; going to 6 must be config only. Reusable workflows read it to find the next tier.
- **A tier is a list of clusters**, even though each demo tier has one. Note in `NOTES.md` how a multi-cluster tier would work (e.g. promote only after every cluster in the tier has reported success for the same digest).

```yaml
# tiers.yaml
tiers:
  - name: dev
    clusters: [kind-tier-dev]
    requiresApproval: false
    onFailure: hold
  - name: stg
    clusters: [kind-tier-stg]
    requiresApproval: false
    onFailure: rollback
  - name: prd
    clusters: [kind-tier-prd]
    requiresApproval: true
    onFailure: rollback
```

## Platform apps
- `kyverno` (engine chart)
- `kyverno-policies` (official policies chart): tier config = `validationFailureAction` (`Audit` in dev/stg, `Enforce` in prd). Verify value names against the chart.
- `kube-prometheus-stack` with **lightweight values**: short retention, single replicas, minimal exporters, no Alertmanager HA
- `otel-collector`
- **Canary (not promoted, in `platform-fleet`):** `podinfo`, identical in every tier, so our tests have something real to exercise.

Pin every chart and image to an exact version. Measure and document the memory footprint of all 3 clusters in `NOTES.md`.

## Helm tests: wrapper charts
Upstream platform charts often ship thin or no Helm tests, so **first inventory which tests each upstream chart actually has** and report back before building.

Approach: each app repo's `chart/` is a thin wrapper chart, with the upstream chart as a dependency plus our own `templates/tests/`. The mirror workflow builds the wrapper and pushes **that** to GHCR. We test what we promote.

Initial tests (keep them small):
- **kyverno:** webhook service responds; a server-side dry-run of a known-bad pod is denied or reported (test Pod with a minimal ServiceAccount + RBAC)
- **kyverno-policies:** all ClusterPolicies Ready
- **kube-prometheus-stack:** Prometheus `/-/ready`; the canary's scrape target is `up` via `/api/v1/targets`; Grafana `/api/health`
- **otel-collector:** collector health endpoint ok; accepted-spans metric present

Values for the wrapper nest under the upstream chart's name, which changes the shape of `release.yaml` / `tier.yaml`. Keep this consistent across repos.

On test failure, the HelmRelease must end up **not Ready** so no success event fires.

## Failure handling: hold in the first tier, roll back after
Remediation is **tier-specific config** (it lives in `tier.yaml`, never promoted), driven by `onFailure` in `tiers.yaml`:
- **`hold` (dev):** no remediation. The release stays failed so it can be debugged live. The chain stops there anyway.
- **`rollback` (stg, prd):** `upgrade.remediation.strategy: rollback` with test failures counting as upgrade failures (`test.ignoreFailures: false`). Verify field names against current Flux.

Three things the rollback tiers must also handle:
1. **Capture evidence before it's gone.** The failure event triggers the promote workflow, which records a `failure` Deployment with the test output and Flux events. Configure test hook pods so their logs survive long enough to collect.
2. **Restore Git as the truth.** After a rollback, Git still says the new digest while the cluster runs the old one, and Flux won't retry once remediation is exhausted. On a failure event from a rollback tier, the workflow **opens a revert PR** for that tier's `release.yaml` (same guardrails; prd still needs approval).
3. **Prove the rollback worked.** A rollback that completes isn't necessarily healthy. After rollback, re-run the tests (e.g. trigger a test run on the restored release) and record the result as its own Deployment status. If verifying this cleanly in Flux turns out to be awkward, document the gap in `NOTES.md` instead of building around it.

**CRD caveat (platform apps are CRD-heavy):** Helm doesn't roll back CRDs. If a release changed CRD schemas, rolling back the chart can leave old controllers running against new CRDs. Inventory which apps ship CRDs and how (Helm `crds/` dir vs. templates vs. a separate chart), and note in `NOTES.md` which apps are unsafe to auto-roll back.

## OCI-only artifacts on GHCR (hard rule)
Clusters consume charts **only as OCI artifacts from GHCR** (`oci://ghcr.io/<owner>/charts/<chart>`), as a stand-in for ACR. No `HelmRepository` sources, and no direct upstream pulls. Packages are public, so no pull secrets.

- **Mirror workflow** (runs when `mirror/chart.yaml` or `chart/` changes on `main`, `permissions: packages: write`): pull upstream at the pinned version, build the wrapper, `helm push` to GHCR, then **open a PR setting `tiers/<first tier>/release.yaml` to the new digest**. That PR is the entry point into the chain.
- **Flux sources:** `OCIRepository` (with a `layerSelector` for `application/vnd.cncf.helm.chart.content.v1.tar+gzip`) + `HelmRelease.spec.chartRef`. Verify against current Flux docs; needs Flux ≥ 2.3.
- **Promote by digest:** `release.yaml` sets the `OCIRepository` `ref.digest`, with the semver as an annotation for humans.
- **Enforcement:** a Kyverno guard policy denies `HelmRepository` and any `OCIRepository` outside our GHCR prefix; the guardrails workflow fails PRs that add either.
- **Renovate (stretch):** open PRs against `mirror/chart.yaml` when upstream releases.

## Flux notifications (per app, per tier, in `notify.yaml`)
- `Provider` type `githubdispatch`, address = the app's repo, token from a cluster Secret.
- `Alert` on that tier's HelmRelease, with `eventMetadata` carrying `tier` and `app`.
  - **Success:** match the upgrade/install-succeeded events, after tests. Verify exact event reasons and messages against current Flux; the docs' example uses an inclusion regex on "upgrade succeeded".
  - **Failure:** a second Alert on error severity, so failed tests also reach GitHub.
- Confirm which payload fields carry the revision and OCI digest (the Flux docs example reads an `oci-digest` metadata field).

## Promote workflow (reusable, `repository_dispatch`)
On a **success** event from tier N-1:
1. Parse `tier`, `app`, and the digest from the payload.
2. **Verify against Git:** the digest must equal `tiers/<tier>/release.yaml` on `main`. Stale or forged events stop here.
3. Record a GitHub Deployment in environment `<tier>` with status `success` (`ref` = main SHA, `payload` = digest, description = test summary from the event).
4. Look up the next tier in `tiers.yaml`. If none, stop.
5. Copy `tiers/<tier>/release.yaml` → `tiers/<next>/release.yaml` on branch `promote/<next>/<short-digest>`; open the PR with label `promotion`, an evidence section (event details, Deployment link), and auto-merge enabled.

On a **failure** event: record a `failure` Deployment with the reason and test output. No promotion PR. If the tier's `onFailure` is `rollback`, open a revert PR restoring that tier's previous `release.yaml`.

Use `concurrency` keyed on app + target tier so overlapping events don't open duplicate PRs; update an existing open promotion PR instead of opening a second.

## GitHub Environments
Each app repo has **one Environment per tier**, created from `tiers.yaml` by `task gh:envs` (via `gh api`) across every repo in `apps.yaml`.
- **Single approval gate:** branch protection requires one check, `promotion-gate`, which runs with `environment: <target tier>`. Where `requiresApproval: true`, the environment has required reviewers with "prevent self-review" on. Other tiers pass immediately. Approval happens **before** merge, because merge is what deploys. No CODEOWNERS approval on promotion PRs.
- **Ledger:** the promote workflow's Deployments give per-app, per-tier history, including failures.
- Deployment branch policy: only `main`.

## Guardrails workflow (reusable, on PRs labeled `promotion`)
- Fail if any file other than `tiers/<target>/release.yaml` changed
- Fail if the target `release.yaml` ≠ the source tier's `release.yaml` on `main`
- Fail unless the source tier's latest Deployment is `success` for the same digest (blocks hand-made promotion PRs)
- Fail if a `HelmRepository`, or an `OCIRepository` outside our GHCR prefix, appears
- Secret scanning (everything is public)
- Run `promotion-gate` with `environment: <target tier>`

Pin reusable workflows by tag or SHA in callers.

## Trust boundary (document in `NOTES.md`)
GitHub Actions can't reach the clusters, so a `success` event is trusted as the cluster's attestation. The Git check in step 2 stops stale or mismatched events, but not a forged event carrying a correct digest. Mitigation: a dispatch token with the smallest scope that can send `repository_dispatch` to that one repo, stored only in-cluster. Note how this would change in AKS (e.g. GitHub App, workload identity).

## Next: richer evidence (design the seam now, build later)
Later the evidence grows beyond Helm tests: Prometheus/Dynatrace queries, Testkube workflows, possibly my HelmReleaseTest operator. The seam:
- Add a **verification `Kustomization`** per app per tier, with `dependsOn` on the app's Kustomization, that runs the checks (a Job, a Testkube TestWorkflow, or a HelmReleaseTest resource) and health-checks their result (Flux CEL health checks for custom resources).
- Move the success `Alert` from the HelmRelease to the verification Kustomization. **The promote workflow doesn't change.**
- Jobs are immutable: name them per release (e.g. a digest suffix) so each release re-runs them.

Make the Alert source a small, swappable piece of `notify.yaml` so this is a config change, not a redesign. Write the design in `NOTES.md`; don't implement it in this demo.

## Phases & acceptance criteria

**Phase 0: Scaffold**
Template repo, `task repos:create`, `platform-workflows`, `platform-fleet` with `tiers.yaml` + `apps.yaml`, kind configs, Flux bootstrap for all 3 clusters.
✅ All repos exist with the same shape. `task up` creates 3 clusters, each synced to its own path. `task down` removes them.

**Phase 1: Charts, tests, and apps across tiers**
Inventory upstream tests (report first), wrapper charts + tests, mirror workflow, all apps + guardrails + canary in every tier.
✅ Everything Ready in all 3 clusters, sourced only from GHCR, with Helm tests passing. Applying a `HelmRepository` is denied. Kyverno is `Audit` in dev/stg and `Enforce` in prd. Editing one tier's `tier.yaml` changes only that tier.

**Phase 2: Notifications + promote workflow**
`notify.yaml` per tier, reusable promote workflow, Deployments.
✅ A Helm upgrade in dev produces a `success` Deployment in the dev environment and a PR to stg whose diff is only `tiers/stg/release.yaml`. A failing test produces a `failure` Deployment and no PR.

**Phase 3: Guardrails + Environments**
`task gh:envs`, reusable guardrails workflow, branch protection.
✅ A promotion PR that sneaks in a `tier.yaml` change fails. A hand-made promotion PR with no successful source Deployment fails. Non-approval tiers merge themselves; the prd PR waits on environment reviewers.

**Phase 4: Demo scenarios**
- `task demo:good`: bump `mirror/chart.yaml` for kube-prometheus-stack; it flows to a prd PR awaiting approval with no manual steps.
- `task demo:ready-but-broken`: a values change that drops the canary's scrape target. Pods are healthy, but the wrapper chart's "canary target up" test fails in dev and the chain stops there.
- `task demo:stg-rollback`: a release that passes in dev but fails in stg (via a stg-only value). Stg rolls back, re-tests healthy, and a revert PR is opened; dev was never affected.
- `task demo:sneaky`: a promotion PR that also flips `validationFailureAction` is rejected by CI.
✅ Each scenario runs from a fresh `task up`.

**Phase 5 (optional): Signed charts**
Keyless cosign signing in the mirror workflow (GitHub OIDC), `OCIRepository` verification with an OIDC identity match on that workflow.
✅ An unsigned chart, or one signed by another identity, never reaches Ready.

## Taskfile targets (in `platform-fleet`, minimum)
`repos:create`, `up`, `down`, `gh:envs`, `status` (Flux state + latest Deployment per app per tier), `grafana -- <tier>`, `demo:good`, `demo:ready-but-broken`, `demo:stg-rollback`, `demo:sneaky`

## Prereqs
docker (with enough memory for 3 clusters; document the number), kind, kubectl, flux CLI, helm, gh (authenticated), task.
Tokens, separate and least-privilege: Flux bootstrap for `platform-fleet`; per-cluster dispatch token(s) for the app repos. Workflows use `GITHUB_TOKEN` where possible; cross-repo reads (e.g. `tiers.yaml`) need a read-only token or a public fetch.

## Non-goals
A custom controller, Kargo/Argo, Prometheus/Dynatrace/Testkube checks (design only), real secrets management, production hardening, multi-cluster tiers. Note real-world AKS differences in `NOTES.md`.

## Open decisions (ask me)
- Owner/org, repo name prefix, tier names, and which tiers require approval (default: last tier only)
- Whether revert PRs in non-approval tiers auto-merge (default: yes) and how long a held dev failure may sit before someone is alerted
- Whether `kyverno-policies` is its own app repo (default) or bundled with `kyverno`
- Whether guardrail policies stay in `platform-fleet` (default) or become a promoted app
- Whether signing is in scope

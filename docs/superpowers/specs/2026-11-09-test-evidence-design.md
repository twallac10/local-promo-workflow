# Design: signed test evidence with Testkube

_2026-11-09. Builds on Phases 0–3 (see `RESUME.md`, `promo-fleet/NOTES.md`)._

## Intent

A promotion should rest on **immutable, verifiable evidence that the tests actually ran and passed**
for the exact release being promoted, not on a mutable "success" Deployment that anyone holding a
dispatch token could create. The gate (`promotion-gate`) must check that evidence before a PR to
the next tier can merge. People reviewing a PR (especially you, approving prd) should also be able
to see what was tested and how it went.

Testkube becomes the test engine for behaviour tests, so the evidence format (JUnit) and the
engine can grow later (Chainsaw-style, load, etc.) without changing the evidence path.

## Goals

- In-cluster **behaviour tests** run in Testkube per app per tier, for every new release, and
  produce **JUnit**.
- Results leave the cluster for **every run, pass or fail**.
- Each result becomes an **OCI artifact** (`junit.xml`) in GHCR attached to the chart digest, and a
  **signed attestation** (in-toto `test-result` predicate, Sigstore keyless, signed by the GitHub
  workflow's identity, logged in Rekor) on that chart digest.
- The **gate verifies the attestation**: signed by the expected workflow identity, for this chart
  digest, the source tier and this release id, result passed. No attestation, no merge.
- The **promotion PR body** shows the per-test summary and links to the artifact and attestation.
- A failed behaviour test **blocks promotion**; in rollback tiers it opens a revert PR.

## Non-goals

- Signing in the cluster (no keys or signing identity in kind). Written up as the AKS path only.
- Signed evidence for the Helm smoke tests (they keep today's Flux events and Deployments).
- Repo/CI test suites (`tests/run.sh`) emitting JUnit.
- Testkube Cloud/Enterprise features (dashboard, control plane). Open-source agent only.
- Re-testing after an automatic rollback (still Phase 4).

## Anti-goals (would make this a failure even if it works)

- Evidence a PR author, or anyone with only GHCR write access, can fake and have the gate accept.
- New long-lived credentials in the clusters beyond the existing per-tier dispatch token.
- Losing instant Flux rollback in stg/prd for releases that don't come up.
- A cluster footprint that no longer fits three tiers in the Docker VM (15.6 GiB).

## Decided by the user

- Results as **JUnit**, published as an **OCI artifact** and an **attestation on the chart digest**.
- The **gate** enforces it before merge (not a human reading the PR).
- **Signed by the GitHub workflow** (keyless), not by the cluster; NOTES states plainly what that
  proves (the workflow received these results through the dispatch token) and what it doesn't.
- In-cluster tests only.
- Engine: **Testkube**, if the clusters can take it (spike below: they can).
- **Helm smoke tests stay** for instant Flux rollback; **Testkube runs the behaviour tests** and
  produces the evidence.
- Failed/errored tests: not promoted; looked at by a human.

## Spike findings (Testkube 2.15.0 in dev, removed afterwards)

- Chart `kubeshop/testkube` 2.15.0 with `testkube-api.nats.enabled=false`,
  `testkube-api.nats.embedded=true`: three pods: API server (with built-in controllers and embedded
  NATS), MongoDB (`kubeshop/bitnami-mongodb:9.0.2`), MinIO (`kubeshop/testkube-minio:2025.10`).
  All six images rendered have amd64 + arm64.
- Footprint: Testkube pods ~470 MiB working set; ~0.5–0.8 GiB node memory per tier, ~0.03 core
  idle. Three tiers projected ~12 GiB of 15.6 GiB.
- The API server restarts a few times at startup until MongoDB answers (exit 1, then fine).
- Applying a `TestWorkflowExecution` runs the referenced `TestWorkflow` (built-in controller). JUnit
  files saved with `artifacts:` are stored in MinIO and listable through the API.
- A `condition: always` step runs after a failed step and can read the JUnit the failed step wrote.
- The `TestWorkflowExecution` **status is never populated** (stays `{generation: 1}`): Flux cannot
  health-check a run. Results must leave via the workflow's own always-run step.
- The Testkube API is reachable in-cluster at `testkube-api-server.testkube:8088`.

## How it works (decided shape)

### Two test layers per app

| Layer | Where | Checks | On failure |
|---|---|---|---|
| Smoke | Helm test hooks in the wrapper chart (as today, trimmed) | it's up: kyverno webhook/readiness, prometheus `/-/ready`, otel health, kyverno-policies Ready | dev holds; stg/prd Flux rollback (unchanged); Flux failure event + `failure` Deployment (unchanged) |
| Behaviour | Testkube `TestWorkflow` per app | it works: kyverno deny dry-run, canary scrape target up, otel accepted spans, kyverno-policies blocks a privileged pod in Enforce tiers / reports it in Audit tiers | signed failure record; no promotion; revert PR in rollback tiers |

Today's checks split along that line (the tests that need behaviour move to Testkube; the readiness
ones stay as smoke). Each app keeps at least one smoke test.

### Where things live

- **Testkube install:** `promo-fleet` platform component (like `canary`), chart mirrored to our GHCR
  (OCI-only rule), pinned 2.15.0, values from the spike. Kustomization `testkube` in every tier.
- **Behaviour tests:** a `TestWorkflow` per app **inside the wrapper chart** (`chart/templates/`), so they are
  **covered by the chart digest**: by the release id, by the attestation subject, and by promotion. A stg
  run uses exactly the tests that passed in dev; a test change needs a new chart version (mirror -> dev
  PR -> promotion) and no guardrail change. Tier differences (e.g. kyverno-policies: blocked in Enforce
  tiers, reported in Audit tiers) come from chart values set in `tier.yaml`. The app Kustomization
  `dependsOn` `testkube` so the CRD exists before the chart installs. _Builder's call (mine)._
- **Run trigger:** a **verification Kustomization** per app per tier (`verify-<app>`, the NOTES
  "Richer evidence" seam), `dependsOn` the app's Kustomization (so it runs only when the app is up)
  and `testkube`. It applies a `TestWorkflowExecution` whose **name includes the release id**, so
  each new release runs once and re-applying the same release doesn't rerun.

### Evidence path

1. The `TestWorkflow`'s test steps write JUnit to a shared path; a final `condition: always` step
   assembles one `junit.xml` (suite per app, testcase per check) and sends it to the app repo as a
   `repository_dispatch` (new event type, e.g. `TestReport/<app>.<app>`) using the namespace's
   existing `promo-dispatch` token, with `tier`, `app`, `release-id`, chart digest and overall
   result. Size is small (a handful of testcases).
2. A GitHub workflow (reusable, in `promo-workflows`) validates the event (same strictness as the
   promote parser), **verifies it against `main`** (digest + release id for that tier), then:
   - pushes `junit.xml` as an **OCI artifact** referring to the chart digest in GHCR;
   - creates an **attestation** on the chart digest: in-toto `test-result` v0.1 predicate (result
     PASSED/FAILED, passed/failed test names, plus tier, app, release id, Testkube execution id),
     signed keyless with the workflow's GitHub OIDC identity, pushed to the registry;
   - records the GitHub Deployment (task `promote`): **a Deployment now means the behaviour verdict**
     for that release (success = smoke passed, since the run only starts once the app is Ready, and
     behaviour passed). Smoke *success* no longer records a Deployment; smoke *failure* still records a
     `failure` Deployment (no behaviour run happens), so "latest Deployment" stays one verdict;
   - on PASSED: opens/updates the next tier's promotion PR (the evidence section gains the test
     table and links). On FAILED: closes an open promotion PR to the next tier; in rollback tiers
     opens a revert PR (same rules as today).
3. The **Flux TestSucceeded event no longer opens promotion PRs**; it still records smoke results.
   Promotion is driven by the signed behaviour result. (Avoids the race between the two events.)

### Gate

For promotion PRs, `check-evidence` is replaced by attestation verification:
`gh attestation verify oci://ghcr.io/twallac10/charts/<chart>@<digest>` with the expected signer
workflow (pinned to the promo-workflows workflow path and tag) and predicate type, then the
predicate must say tier = source tier, release id = the PR's release id, result PASSED. Revert PRs
need a FAILED attestation for the tier's current release (or today's failure Deployment for smoke
failures). The gate stays `pull_request_target`, read-only, no PR code.

- **Which result counts:** the latest attestation (by Rekor integrated time) for that chart digest, tier
  and release id. A fresh `task up` re-runs and re-signs; the newest wins.
- **Re-running:** `task retest -- <tier> <app>` applies a new `TestWorkflowExecution` with an attempt
  suffix (release id + attempt). For flaky or infrastructure failures (e.g. Testkube startup).
- **Signer:** the promote-workflows workflow path in `twallac10/promo-workflows` at any `refs/tags/v*`
  (tags are never moved), run from an app repo's `main` on `repository_dispatch`. Upgrading
  promo-workflows doesn't invalidate evidence; a signature from any other workflow, repo, ref or event
  is rejected.

### Trust (for NOTES)

- Proves: the promote workflow (exact identity, logged in Rekor) received these results for this
  digest and release, after checking them against `main`. A pushed-by-hand artifact or attestation
  without that identity is rejected.
- Doesn't prove: that the cluster produced them. A dispatch-token holder can still send fake
  results; the workflow would sign what it received. AKS path: sign in-cluster with workload
  identity (Sigstore keyless from a cluster OIDC issuer) or an in-cluster key.

## Builder's calls (mine; overrule any)

- Behaviour tests live in the wrapper chart (covered by the digest), not as separate files in the app
  repo: same "promoted with the release" outcome without changing the release id or guardrails.
- Behaviour failure in stg/prd opens a revert PR (pods are healthy, so Flux won't roll back).
- Promotion is driven by the Testkube report, not by Flux's TestSucceeded event.
- Attestations via GitHub's attestation tooling (`actions/attest`, `gh attestation verify`); exact
  action/version pinned at implementation.
- The Testkube chart and images are mirrored into our GHCR like the canary.
- Testkube's startup restarts are tolerated (documented), not engineered away.
- A Deployment means the behaviour verdict; smoke success no longer records one.
- Latest signed result wins; `task retest` for re-runs; signer accepted at any `v*` tag.

## Left to the builder

- Exact JUnit layout and testcase names; the dispatch payload field names (within GitHub's limits).
- How the verification Kustomization templates the release id into the execution name.
- Whether the Testkube API server's startup restarts need a startup probe or init wait.
- `task status` additions (latest attestation per app per tier).
- The order of rollout across repos, and how existing open PRs (kyverno-policies #5) are handled.

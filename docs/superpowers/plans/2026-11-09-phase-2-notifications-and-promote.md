# Phase 2: Notifications and Promote Workflow — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** When a HelmRelease's Helm tests pass in tier N, Flux tells the app's GitHub repo; a reusable workflow checks the event against Git, records a `success` Deployment for tier N and opens a PR to tier N+1 that changes only `tiers/<N+1>/release.yaml`. A failed test records a `failure` Deployment and opens no promotion PR (a revert PR in rollback tiers).

**Architecture:** Each app repo's `tiers/<tier>/notify.yaml` holds a `githubdispatch` Provider and two Alerts (success, failure) for that tier's HelmRelease. The app repo's `promote.yaml` caller runs on `repository_dispatch` and calls the reusable `promo-workflows` `promote.yaml`, whose logic lives in tested bash scripts in a composite action (`promo-workflows/actions/promote/`). Evidence is keyed on a **release id**: a hash of the tier's `release.yaml`, carried into the event through a HelmRelease annotation, so the workflow can prove the exact release file (digest + shared values) was tested, not just the chart digest (LESSONS: "A release is the whole release file").

**Tech Stack:** Flux 2.10 notification-controller (`notification.toolkit.fluxcd.io/v1` Provider/Alert), GitHub Actions (reusable workflow + composite action), `gh` API (Deployments, PRs, labels), the `promo-bot-twallac10` GitHub App, yq 4, jq, bash 3.2.

**Spec:** `promotion-demo-handoff.md` (Phase 2, "Flux notifications", "Promote workflow", "Trust boundary", "Next: richer evidence"), amended by `LESSONS.md`, the Phase 0–1 plan and ledger Rulings.

## Global Constraints

- Everything in the Phase 0–1 plan's Global Constraints still holds (owner, repo names, GHCR prefix, bash 3.2, yq/jq only).
- Provider `type: githubdispatch`, `address: https://github.com/twallac10/promo-<app>`, `secretRef: {name: promo-dispatch}` (key `token`).
- Dispatch `event_type` is `HelmRelease/<app>.<app>` (Flux: `{Kind}/{Name}.{Namespace}`); `client_payload` is the Flux event.
- Event metadata keys used: `revision`, `oci-digest` (from helm-controller), `tier`, `app` (Alert `eventMetadata`), `release-id` (HelmRelease annotation `event.toolkit.fluxcd.io/release-id`).
- Success event = reason `TestSucceeded`, message `^Helm test succeeded` (verified live 2026-11-09; it fires after tests on install and upgrade). Failure events = severity `error`: `TestFailed`, `UpgradeFailed`, `InstallFailed`.
- Release id = `sha256:` + `shasum -a 256` of `yq -o json -I0 'del(.metadata.annotations."event.toolkit.fluxcd.io/release-id")' tiers/<tier>/release.yaml`. Identical `release.yaml` files have identical ids in every tier.
- Workflows open PRs and create Deployments with the App token (`PROMO_APP_ID`, `PROMO_APP_PRIVATE_KEY`), never `GITHUB_TOKEN`.
- `client_payload` is attacker-controlled: never interpolate `${{ github.event.client_payload.* }}` inside `run:`; pass it as a file and validate every field against a regex before use.
- Promotion branch `promote/<next tier>` (one per target tier, force-pushed), label `promotion`; revert branch `revert/<tier>`, label `revert`. No auto-merge in Phase 2 (Phase 3 adds the gate and auto-merge).
- Reusable workflows pinned by tag in callers: `promo-workflows` **v0.4.0** for both `mirror.yaml` and `promote.yaml`.

## Prerequisites (user)

1. **App permissions:** `promo-bot-twallac10` needs Repository permissions Contents: write, Pull requests: write, **Deployments: write**, Issues: write (labels). Add any missing in the App settings and accept the new permissions on the installation.
2. **Dispatch tokens:** one fine-grained PAT per tier (`promo-dispatch-dev|stg|prd`), resource owner `twallac10`, repositories = the four app repos, permission **Contents: write** (the minimum for `repository_dispatch`), 90-day expiry. Save each to `~/.config/promo/dispatch-<tier>.token` (mode 600). Never committed.

## Review Focus

1. **Stale or out-of-order event:** a success event for release A arrives after `main` already has release B for that tier. Expected: no Deployment, no PR, job logs "stale" and exits 0. Test: `test_verify_stale_release` (Task 4).
2. **Hostile payload:** a `tier` like `dev; curl evil`, a malformed digest or an `app` that isn't this repo. Expected: rejected before any `gh`/`git` call. Tests: `test_parse_rejects_bad_fields` (Task 4) and `test_no_payload_interpolation` (Task 5).
3. **Repeated events for one release** (every fresh `task up` re-installs and re-tests every app, in every tier). Expected: one PR per target tier, updated in place; no duplicate PRs, no error. Test: `test_open_pr_idempotent` (Task 5).
4. **Next tier already has this release** (normal after a fresh `task up`). Expected: success Deployment recorded, no PR. Test: `test_open_pr_noop_when_equal` (Task 5).
5. **Last tier, and hold tier failures:** success in prd records a Deployment and stops; a failure in dev (hold) opens no revert PR; a failure in a rollback tier whose `release.yaml` has no earlier version opens no revert PR and says why. Tests: `test_next_tier`, `test_revert_only_in_rollback_tiers` (Tasks 4, 5).

---

## File Structure

```
promo-app-template/  (synced to every promo-<app> by task repos:sync)
  scripts/release-id.sh             # print or --write the release id of a release.yaml
  tiers/<tier>/notify.yaml          # Provider + Alerts (tier-only, never promoted)
  tiers/<tier>/kustomization.yaml   # + notify.yaml resource
  tiers/<tier>/release.yaml         # + HelmRelease annotation event.toolkit.fluxcd.io/release-id
  .github/workflows/promote.yaml    # repository_dispatch caller
  tests/test_release.sh  tests/test_notify.sh
promo-workflows/
  .github/workflows/promote.yaml    # reusable
  actions/promote/action.yml        # composite action running the scripts below
  actions/promote/{parse-event,next-tier,verify,deploy,open-pr,revert-pr}.sh
  tests/run.sh  tests/lib/lib.sh  tests/test_*.sh  tests/fixtures/
  VERSION
promo-fleet/
  scripts/repos-sync.sh  scripts/dispatch-secret.sh
  scripts/up.sh  scripts/status.sh  Taskfile.yml  NOTES.md
```

### Task 1: Release id in the template and the mirror

**Files:**
- Create: `promo-app-template/scripts/release-id.sh`, `tests/test_release.sh`
- Modify: template `tiers/*/release.yaml`, `tests/test_tiers.sh` (`test_release_only_promotable_fields` allows `HelmRelease:metadata.annotations.event.toolkit.fluxcd.io/release-id`), `promo-workflows/.github/workflows/mirror.yaml`

**Interfaces:**
- Produces: `scripts/release-id.sh <release.yaml>` prints `sha256:<64 hex>`; `scripts/release-id.sh --write <release.yaml>` sets the annotation in place. Used by Tasks 4 and 5 (from the checked-out app repo) and by the mirror.

- [ ] **Step 1: Write failing tests** in `tests/test_release.sh`:
  - `test_release_id_current`: for every tier, the annotation equals `scripts/release-id.sh tiers/<t>/release.yaml`.
  - `test_release_id_ignores_own_annotation`: changing only the annotation doesn't change the id; changing a value or the digest does.
  - `test_release_id_tier_independent`: copying dev's `release.yaml` over stg's (temp dir) gives stg the same id.
- [ ] **Step 2:** Run `tests/run.sh`. Expected: the three tests FAIL.
- [ ] **Step 3:** Implement `release-id.sh` per Global Constraints; run `--write` on every tier.
- [ ] **Step 4:** In the reusable `mirror.yaml`, after setting the digest, run `scripts/release-id.sh --write "$release"` before the diff check. Run `actionlint`.
- [ ] **Step 5:** Run template tests and actionlint. Expected: PASS. Commit both repos (don't tag yet; Task 5 tags v0.4.0).

### Task 2: `task repos:sync` and applying Task 1 to the app repos

**Files:**
- Create: `promo-fleet/scripts/repos-sync.sh`, test in `tests/test_fleet.sh`
- Modify: `Taskfile.yml` (`repos:sync`), `scripts/customize-app.sh` (also rename in `.github/workflows`)

**Interfaces:**
- Produces: `task repos:sync` copies the template-owned paths into every sibling `../promo-<app>` and runs `customize-app.sh` on them. Template-owned: `.github/workflows/`, `scripts/`, `tests/lib/`, `tests/run.sh`, `tests/test_{chart,mirror,tiers,release,notify}.sh`, `tiers/*/kustomization.yaml`, `tiers/*/notify.yaml`. App-owned and never touched: `base/`, `chart/`, `mirror/`, `tiers/*/{release,tier}.yaml`, `tests/test_app.sh`, `tests/test_tier_values.sh`. It then runs `scripts/release-id.sh --write` on each tier's `release.yaml`, leaving changes uncommitted.

- [ ] **Step 1: Write failing test** `test_repos_sync_copies_template_owned_only`: sync into a temp copy of the template customized as `kyverno` with a sentinel line added to `chart/values.yaml` and `tiers/dev/tier.yaml`; the sentinels survive, `tests/lib/lib.sh` matches the template, and `grep -rw example` finds nothing in the synced paths.
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement. Paths are a list at the top of the script (one source of truth).
- [ ] **Step 4:** Run `task repos:sync`, then each app's `tests/run.sh`. Expected: all PASS; `git diff` in each app touches only synced paths and the `release-id` annotation. Commit and push each app (no chart change, so no mirror run).

### Task 3: Notifications per tier, and the dispatch Secret

**Files:**
- Create: template `tiers/<tier>/notify.yaml`, `tests/test_notify.sh`; `promo-fleet/scripts/dispatch-secret.sh`
- Modify: template `tiers/*/kustomization.yaml`; fleet `scripts/up.sh`, `tests/test_fleet.sh`, `NOTES.md`

**Interfaces:**
- Produces, per app and tier, in namespace `<app>`: Provider `promo-dispatch`; Alert `promote-success` (`eventSeverity: info`, `inclusionList: ["^Helm test succeeded"]`); Alert `promote-failure` (`eventSeverity: error`). Both: `eventSources: [{kind: HelmRelease, name: <app>}]`, `eventMetadata: {tier: <tier>, app: <app>}`.
- `scripts/dispatch-secret.sh <tier>` prints, for every app in `apps.yaml`, a Namespace `<app>` and a Secret `promo-dispatch` (key `token`) from `${PROMO_DISPATCH_DIR:-$HOME/.config/promo}/dispatch-<tier>.token`; exits non-zero with instructions if the file is missing or empty. `up.sh` pipes it to `kubectl apply` after `flux install` and before applying `flux-sync.yaml`.

- [ ] **Step 1: Write failing tests:**
  - template `test_notify_shape`: for each tier, the built tier has exactly one Provider and two Alerts with the values above; `tier` metadata equals the directory name; the Provider address ends in `promo-<app>` after customizing.
  - template `test_notify_not_promotable`: no Provider or Alert in any `release.yaml`.
  - fleet `test_dispatch_secret_manifests`: with a temp `PROMO_DISPATCH_DIR` holding `dispatch-dev.token`, output has one Secret per app in its own namespace, `stringData.token` equals the file; with the file missing, exit is non-zero and stderr names the path.
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement. Keep `promote-success` as its own document at the top of `notify.yaml` with a comment marking it as the swappable evidence source (spec "Next: richer evidence").
- [ ] **Step 4:** In `NOTES.md`, write "Trust boundary" (what the release-id check stops and what it doesn't: a forged event carrying the correct id; per-tier token scope is Contents: write on four repos; AKS would use a GitHub App or workload identity) and "Richer evidence (design, not built)" (verification Kustomization per app per tier, `dependsOn` the app, success Alert moved to it, Jobs named per release id).
- [ ] **Step 5:** Run template and fleet tests. Expected: PASS. Commit template and fleet; `task repos:sync` to apps, run their tests, commit, push.

### Task 4: Promote scripts: parse, next tier, verify

**Files:**
- Create: `promo-workflows/actions/promote/{parse-event,next-tier,verify}.sh`, `tests/run.sh`, `tests/lib/lib.sh` (copy of the template's harness), `tests/test_parse.sh`, `tests/test_verify.sh`, `tests/fixtures/{success,failure}.json`
- Modify: `promo-workflows/.github/workflows/ci.yaml` (run `tests/run.sh`)

**Interfaces:**
- `parse-event.sh <event.json> <repo-name>` reads a `repository_dispatch` event and prints `key=value` lines for `$GITHUB_OUTPUT`: `outcome` (`success`|`failure`), `app`, `tier`, `digest`, `release_id`, `reason`, `message` (single line, ≤ 1000 chars). Rejects (exit 1) unless: `action` is `HelmRelease/<app>.<app>`; `<repo-name>` is `promo-<app>`; `tier` matches `^[a-z][a-z0-9-]{0,30}$`; `digest` matches `^sha256:[0-9a-f]{64}$`; `release_id` matches the same; outcome is success only for reason `TestSucceeded` with severity `info`, failure only for severity `error`.
- `next-tier.sh <tiers.yaml> <tier>` prints `next=<name or empty>`, `on_failure=<hold|rollback>`, `requires_approval=<bool>` for `<tier>`; exit 1 if `<tier>` isn't listed.
- `verify.sh <app-repo-dir> <tier> <digest> <release_id>` prints `status=ok` or `status=stale` (exit 0 either way) by comparing against `tiers/<tier>/release.yaml` in that checkout (digest and `scripts/release-id.sh`).

- [ ] **Step 1: Write fixtures** from the real event fields captured on 2026-11-09 (reason `TestSucceeded`, message `Helm test succeeded for release kyverno-policies/kyverno-policies.v2 with chart promo-kyverno-policies@3.9.2-2+885316ab9f6c: 1 test hook completed successfully`, `oci-digest` `sha256:885316ab…`, `revision` `3.9.2-2+885316ab9f6c`), wrapped as `{"action": "HelmRelease/kyverno-policies.kyverno-policies", "client_payload": {...}}`. Task 6 replaces them with captured payloads.
- [ ] **Step 2: Write failing tests:** `test_parse_success`, `test_parse_failure`, `test_parse_rejects_bad_fields` (one case each: tier with `;`, short digest, other app, success reason with `error` severity), `test_next_tier` (dev→stg/hold, stg→prd/rollback, prd→empty, unknown→exit 1), `test_verify_ok`, `test_verify_stale_release` (same digest, different shared value → stale), `test_verify_stale_digest`.
- [ ] **Step 3:** Run `tests/run.sh`. Expected: FAIL.
- [ ] **Step 4:** Implement with `jq`/`yq` only.
- [ ] **Step 5:** Run tests and CI-equivalent `actionlint`. Expected: PASS. Commit.

### Task 5: Deployments, promotion PR, revert PR, and the workflow

**Files:**
- Create: `promo-workflows/actions/promote/{deploy,open-pr,revert-pr}.sh`, `actions/promote/action.yml`, `.github/workflows/promote.yaml`, `VERSION` (`0.4.0`), `tests/test_pr.sh`, `tests/test_workflow.sh`, `tests/fixtures/bin/gh` (stub recording calls); template `.github/workflows/promote.yaml`
- Modify: template `.github/workflows/mirror.yaml` pin → `v0.4.0`, `tests/test_mirror.sh` (also checks the promote caller pin); fleet `scripts/status.sh`

**Interfaces:**
- `deploy.sh <repo> <tier> <state> <sha> <digest> <release_id> <description> <log_url>` creates a Deployment (`ref` = `<sha>`, `environment` = `<tier>`, `auto_merge: false`, `required_contexts: []`, `payload: {digest, release_id}`, `production_environment` = tier is last) and a status `<state>` (`success`|`failure`), `description` truncated to 140 chars. Prints `deployment_url=`.
- `open-pr.sh <app-repo-dir> <tier> <next> <evidence-markdown-file>` copies `tiers/<tier>/release.yaml` to `tiers/<next>/release.yaml` on branch `promote/<next>` from `main`; if nothing differs from `main`, prints `pr=` and exits 0; else force-pushes, ensures label `promotion` exists, and creates the PR or edits the open one (title `promote(<next>): <app> <chart-version>`). Prints `pr=<url>`.
- `revert-pr.sh <app-repo-dir> <tier> <evidence-markdown-file>` restores `tiers/<tier>/release.yaml` from the parent of the last commit that changed it, on branch `revert/<tier>`, label `revert`; prints `pr=` with a notice if there is no earlier version.
- Reusable `promote.yaml` (`workflow_call`, secrets `PROMO_APP_ID`, `PROMO_APP_PRIVATE_KEY`): App token → checkout app repo `main` (full history) → fetch `tiers.yaml` from `twallac10/promo-fleet` main → composite action: parse → next-tier → verify (stale: stop) → deploy → success: open-pr; failure: revert-pr if `on_failure` is `rollback`. Job `concurrency: promote-${{ github.repository }}-${{ github.event.client_payload.metadata.tier }}`, `cancel-in-progress: false`. Writes the event and results to the step summary.
- App caller: `on: repository_dispatch: types: [HelmRelease/example.example]`, `permissions: contents: read`, `uses: twallac10/promo-workflows/.github/workflows/promote.yaml@v0.4.0`, `secrets: inherit`.
- `status.sh` adds, per tier, each app's latest Deployment state and digest (`gh api repos/twallac10/promo-<app>/deployments?environment=<tier>&per_page=1` + its latest status).

- [ ] **Step 1: Write failing tests** (stub `gh` on `PATH`, app checkout cloned from a temp bare remote):
  - `test_open_pr_creates_branch_and_pr`: diff on `promote/stg` vs `main` is exactly `tiers/stg/release.yaml`; stub saw `pr create` with label `promotion`.
  - `test_open_pr_idempotent`: second run with an open PR → `pr edit`, not a second `pr create`.
  - `test_open_pr_noop_when_equal`: stg already equals dev → no push, no `gh pr` call.
  - `test_revert_only_in_rollback_tiers`: revert restores the previous content in stg; with a single-commit history it prints a notice and makes no PR. (The hold-tier rule is enforced by the workflow from `on_failure`; covered in `test_workflow_wiring`.)
  - `test_deploy_truncates_description`: a 300-char message → description of 140 chars in the stub's recorded request.
  - `test_no_payload_interpolation`: no `run:` block in `promote.yaml` or `action.yml` contains `client_payload`.
  - `test_workflow_wiring`: revert step's `if` requires `outcome == 'failure'` and `on_failure == 'rollback'`; open-pr step requires `outcome == 'success'` and non-empty `next`; both require `status == 'ok'`; the workflow pins `actions/promote` at `v` + `VERSION`.
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement scripts, action and workflow. Evidence markdown: tier, app, reason, message, digest, release id, revision, Deployment link, run link.
- [ ] **Step 4:** Run tests and `actionlint`. Expected: PASS. Commit, tag `promo-workflows` `v0.4.0`, push with tag.
- [ ] **Step 5:** Template: add the promote caller, bump the mirror pin, extend `test_mirror.sh`; tests PASS; commit. `task repos:sync`; each app's tests PASS; commit, push. Fleet `status.sh`: commit, push.

### Task 6: Live acceptance

**Files:**
- Modify: `promo-fleet/NOTES.md` (payload fields, results), `promo-workflows/tests/fixtures/*.json` (real payloads)

- [ ] **Step 1:** Confirm both Prerequisites are done (ask the user; don't proceed without the token files and App permissions).
- [ ] **Step 2:** `task down && task up`. Expected: all tiers Ready; within minutes every app repo has `promote` runs. Expected results: `success` Deployments for every app in every tier; **exactly one PR**, `promo-kyverno-policies` → stg (dev has 3.9.2-2, stg 3.9.2-1), whose diff is only `tiers/stg/release.yaml`; no other PRs. Check with `task status` and `gh pr list` on each app repo.
- [ ] **Step 3:** Copy one real success payload (from a run's step summary) into the fixtures; rerun `promo-workflows` tests. Expected: PASS. If a field name differs from Global Constraints, fix the parser first, then the constraint. Record the payload shape in `NOTES.md`.
- [ ] **Step 4:** Failing test in dev: set `opentelemetry-collector.ports.health.enabled: false` in `promo-otel-collector/tiers/dev/tier.yaml`, push. Expected: HelmRelease not Ready (`TestFailed`), a `failure` Deployment in dev whose description names the failed test, no PR, no revert PR (dev holds). Revert the change; expected: a new `success` Deployment in dev.
- [ ] **Step 5:** Stale and forged events: `gh api repos/twallac10/promo-kyverno/dispatches` with a success payload carrying the previous release id → run ends "stale", no Deployment. Same with `tier: "dev;x"` → run fails at parse.
- [ ] **Step 6:** Revert path without breaking a cluster: dispatch a `failure` event for `promo-kyverno-policies` tier stg with stg's current digest and release id. Expected: `failure` Deployment in stg and a revert PR (only `tiers/stg/release.yaml`). Close the revert PR and the Deployment noise is acceptable; note it in `NOTES.md`. (A real rollback is Phase 4's `demo:stg-rollback`.)
- [ ] **Step 7:** Leave the stg promotion PR open (Phase 3 gates it). Record results, and the gaps (test pod logs stay in-cluster only; post-rollback re-test is Phase 4), in `NOTES.md`; commit, push. Update the ledger and `RESUME.md`.

**Phase 2 check-in:** report acceptance results to the user before Phase 3.

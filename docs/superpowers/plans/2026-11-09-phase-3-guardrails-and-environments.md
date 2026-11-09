# Phase 3: Guardrails, Environments and Auto-merge — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Every PR to an app repo's `main` must pass one required check, `promotion-gate`, which runs guardrails and then waits in the target tier's GitHub Environment. dev/stg PRs merge themselves; a prd PR merges itself once you approve the prd environment.

**Architecture:** Each app repo gets a `gate.yaml` caller with two jobs. `guardrails` runs a composite action (`promo-workflows/actions/guardrails`) that classifies the PR (promotion, revert, other), runs the checks and outputs the environment. `promotion-gate` needs it, runs in that environment and passes only if `guardrails` succeeded. The promote and mirror workflows enable auto-merge on the PRs they open. `task gh:envs` (fleet) creates Environments from `tiers.yaml`, branch protection and repo settings; you run it.

**Tech Stack:** GitHub Actions (composite actions, Environments, branch protection, auto-merge), `gh api`, gitleaks (pinned), yq 4, jq, bash 3.2.

**Spec:** `promotion-demo-handoff.md` ("GitHub Environments", "Guardrails workflow", Phase 3), amended by `LESSONS.md`, the Phase 2 plan, its ledger and these decisions (user, 2026-11-09):
- Auto-merge on every promotion and revert PR; prd waits on environment approval, so approving prd merges it.
- Any PR touching `tiers/<t>/` runs `promotion-gate` in environment `<t>` (the last such tier in `tiers.yaml` if several). PRs touching no tier run it with no environment.
- Secret scanning in the gate: gitleaks, pinned.
- `task gh:envs` has `--dry-run`; the user reviews it and runs the real thing.

## Global Constraints

- Phase 0–2 Global Constraints still hold.
- Required check context: exactly `promotion-gate` (a job id in the app repo's own workflow, not a reusable-workflow job, so the name has no prefix).
- Branch names: promotion `promote/<tier>`, revert `revert/<tier>`, mirror `mirror/<short-digest>`. The gate classifies by **branch name**; labels are informational.
- Guardrails (spec), for promotion PRs: only `tiers/<target>/release.yaml` changed; it equals `tiers/<source>/release.yaml` on main; the source tier's latest Deployment is `success` with the same `release_id`; no `HelmRepository` and no `OCIRepository` outside `oci://ghcr.io/twallac10/charts/`; no secrets (gitleaks).
- Revert PRs: only `tiers/<tier>/release.yaml` changed; it equals that file before main's last first-parent change to it; the tier's latest Deployment is `failure`; same source and secret checks.
- Other PRs: may change `release.yaml` only in the first tier (the mirror's entry point); source and secret checks; environment = last touched tier, or none.
- The gate never executes code from the PR head: PR files are read with `git show <head-sha>:<path>`; `release-id.sh` runs from the base (main) checkout.
- Environments (per app repo, per tier in `tiers.yaml`): `requiresApproval: true` → reviewer = the repo owner's user id, `prevent_self_review: false`; branch policy = custom: `main` and `refs/pull/*/merge` (GitHub matches `GITHUB_REF`; PR runs use the merge ref).
- Branch protection on `main`: required status check `promotion-gate` (`strict: false`), `enforce_admins: false`, no required reviews. Repo settings: `allow_auto_merge`, `delete_branch_on_merge`, `allow_squash_merge`. Auto-merge method: squash.
- `promo-workflows` **v0.5.0** for mirror, promote and guardrails; callers pin the same tag.

## Review Focus

1. **A failed `guardrails` job must fail the check.** GitHub counts a *skipped* required check as passing, and a job whose `needs` failed is skipped. Expected: `promotion-gate` runs `if: always()` and fails unless `needs.guardrails.result == 'success'`. Test: `test_gate_fails_closed` (Task 4).
2. **Bypassing promotion with an ordinary PR:** a human PR from any other branch that edits `tiers/stg/release.yaml`. Expected: fails ("release.yaml outside the first tier needs a promotion or revert PR"). Test: `test_other_pr_cannot_change_later_release` (Task 2).
3. **PR code executing in the gate:** a PR that changes `scripts/release-id.sh` to `exit 0`. Expected: the gate uses main's script and still computes the real id. Test: `test_evidence_uses_base_release_id_script` (Task 2).
4. **Promote workflow Deployments under the new branch policy:** API-created Deployments (`ref` = a SHA) might be rejected by the `main`-only policy. Expected: they still record; if rejected, switch `ref` to `main` with the SHA in the payload. Verified live in Task 6, Step 3.
5. **Existing open PRs from before Phase 3** (promo-kyverno #6, kyverno-policies #3) never get a `promotion-gate` run, so they would block forever. Expected: the next promote run force-pushes a new commit (a `synchronize` event) and enables auto-merge. Verified live in Task 6, Step 4.

---

## File Structure

```
promo-workflows/
  actions/guardrails/action.yml          # composite: classify → checks → outputs environment
  actions/guardrails/{classify,check-files,check-release,check-evidence,check-sources,scan-secrets}.sh
  actions/promote/lib.sh                 # + enable_auto_merge; label on edit
  .github/workflows/mirror.yaml          # + auto-merge on the dev PR
  tests/test_guardrails.sh  tests/test_automerge.sh  VERSION (0.5.0)
promo-app-template/  (synced to apps)
  .github/workflows/gate.yaml            # jobs: guardrails, promotion-gate
  tests/test_gate.sh
promo-fleet/
  scripts/gh-envs.sh  Taskfile.yml (gh:envs)  tests/test_ghenvs.sh  NOTES.md
```

### Task 1: Spike — empty environment

**Files:** none kept (a throwaway workflow in `promo-workflows`, removed in the same task).

- [ ] **Step 1:** Push `.github/workflows/spike-env.yaml` to `promo-workflows`: `workflow_dispatch` with input `env` (default empty); one job `environment: ${{ inputs.env }}` that echoes. Run `gh workflow run spike-env.yaml -R twallac10/promo-workflows`.
- [ ] **Step 2:** Expected: the run succeeds with no environment and no Deployment created (`gh api repos/twallac10/promo-workflows/deployments` empty). If it fails, rule: `promotion-gate` uses environment `none` (an unprotected environment `task gh:envs` also creates) for PRs touching no tier, and ledger it.
- [ ] **Step 3:** Delete the spike workflow, push. Ledger the result.

### Task 2: Guardrails scripts and composite action

**Files:**
- Create: `promo-workflows/actions/guardrails/*.sh`, `actions/guardrails/action.yml`, `tests/test_guardrails.sh`, fixtures under `tests/fixtures/` as needed
- Modify: `tests/lib/lib.sh` (a PR helper: base + head commits in the fixture app repo)

**Interfaces** (each script runs in the base checkout; `<base>`/`<head>` are SHAs; output `key=value` lines; failures exit 1 with `guardrail: <reason>` on stderr):
- `classify.sh <base> <head> <head-ref> <tiers.yaml>` → `kind=promotion|revert|other`, `tier=<target or empty>`, `source=<previous tier, promotions only>`. Promotion/revert target must be a listed tier (and a promotion's target can't be the first tier). Other: `tier` = last tier in `tiers.yaml` order whose `tiers/<t>/` the PR touches, else empty.
- `check-files.sh <base> <head> <kind> <tier> <first-tier>` → promotion/revert: changed files must be exactly `tiers/<tier>/release.yaml`; other: no `tiers/<t>/release.yaml` with `t` ≠ first tier.
- `check-release.sh <base> <head> <kind> <tier> <source>` → promotion: head's `tiers/<tier>/release.yaml` byte-equals base's `tiers/<source>/release.yaml`; revert: equals base's file before its last first-parent change. Prints `release_id=` of the head file (computed by base's `scripts/release-id.sh` on a temp copy).
- `check-evidence.sh <owner/repo> <kind> <tier> <source> <release_id>` → promotion: latest Deployment in env `<source>` has latest status `success` and `payload.release_id == <release_id>`; revert: latest Deployment in env `<tier>` has latest status `failure`. Uses `gh api` (GITHUB_TOKEN, `deployments: read`).
- `check-sources.sh <head>` → no `kind: HelmRepository` and no `OCIRepository` `spec.url` outside `oci://ghcr.io/twallac10/charts/` in any YAML under `base/` or `tiers/` at `<head>`.
- `scan-secrets.sh <base> <head>` → `gitleaks git --log-opts="<base>..<head>" --redact --exit-code 1`; gitleaks pinned by version and the sha256 of its `linux_x64` tarball (latest release at implementation time; record both in the ledger).
- `action.yml` inputs: `token`, `fleet-repo`; outputs: `kind`, `environment`. Steps in order: fetch `tiers.yaml`, classify, check-files, check-release (promotion/revert), check-evidence (promotion/revert), check-sources, scan-secrets. Values reach `run:` via `env` only.

- [ ] **Step 1: Write failing tests** (stub `gh` returns a canned Deployment list per environment):
  - `test_classify` (promote/stg → promotion/stg/source dev; revert/prd → revert/prd; feature branch touching tiers/dev and tiers/prd → other/prd; touching nothing → other/empty; promote/dev → exit 1; promote/qa → exit 1).
  - `test_promotion_sneaks_tier_yaml` (promote/stg also changes `tiers/stg/tier.yaml` → check-files fails).
  - `test_other_pr_cannot_change_later_release` (Review Focus 2).
  - `test_promotion_release_must_match_source` (stg file ≠ dev's on main → fails; equal → ok and prints the id).
  - `test_revert_release_must_be_previous`.
  - `test_evidence_requires_success_for_same_release` (no Deployment → fail; latest `failure` → fail; `success` with another release id → fail; matching `success` → ok). `test_evidence_revert_requires_failure`.
  - `test_evidence_uses_base_release_id_script` (Review Focus 3).
  - `test_sources_rejects_helmrepository_and_foreign_oci`.
  - `test_scan_secrets_finds_planted_token` (a commit adding `ghp_` + 36 alnum chars → fail; clean → ok). Skip with a message if `gitleaks` isn't installed locally; CI installs it.
  - `test_guardrails_action_wiring` (no `${{` in `run:`; evidence/release steps only for promotion/revert).
- [ ] **Step 2:** Run `tests/run.sh`. Expected: new tests FAIL.
- [ ] **Step 3:** Implement scripts and action. Add gitleaks install to `promo-workflows` CI.
- [ ] **Step 4:** Run tests, `actionlint`. Expected: PASS. Commit (no tag yet).

### Task 3: Auto-merge from promote and mirror

**Files:**
- Modify: `promo-workflows/actions/promote/lib.sh`, `.github/workflows/mirror.yaml`, `tests/test_pr.sh` (or new `tests/test_automerge.sh`), `tests/fixtures/bin/gh`

**Interfaces:**
- `upsert_pr` (lib.sh) after create or edit: `gh pr edit <branch> --add-label <label>` (also on edit) and `gh pr merge <branch> --auto --squash`. If enabling auto-merge fails, print `::warning::auto-merge not enabled: <stderr>` and continue (the PR still exists; `gh:envs` may not have run yet).
- Mirror: after `gh pr create`/`edit`, the same `gh pr merge --auto --squash` with the same warning behavior.

- [ ] **Step 1: Write failing tests:** `test_upsert_enables_auto_merge` (create path and edit path both call `pr merge promote/stg --auto --squash` and `pr edit … --add-label promotion`); `test_auto_merge_failure_warns` (stub `pr merge` exits 1 → output contains `::warning::auto-merge not enabled`, script exits 0, `pr=` still printed); `test_mirror_enables_auto_merge` (the mirror's PR step contains `gh pr merge` with `--auto --squash`).
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement. Stub `gh pr merge` honors `GH_MERGE_FAIL=1`.
- [ ] **Step 4:** Run tests, `actionlint`. Expected: PASS. Bump `VERSION` to `0.5.0`, update the promote action pin; commit, tag `v0.5.0`, push.

### Task 4: Gate caller in the template, synced to apps

**Files:**
- Create: `promo-app-template/.github/workflows/gate.yaml`, `tests/test_gate.sh`
- Modify: template `.github/workflows/{mirror,promote}.yaml` pins → `v0.5.0`; `tests/test_mirror.sh` (all three callers pin the same tag); fleet `scripts/repos-sync.sh` (add `tests/test_gate.sh` to OWNED)

**Interfaces:**
- `gate.yaml`: `on: pull_request` (types `opened, synchronize, reopened, edited`; branches `[main]`); `permissions: {contents: read, deployments: read, pull-requests: read}`.
  - job `guardrails`: checkout base (`ref: main`, `fetch-depth: 0`), fetch the PR head SHA, `uses: twallac10/promo-workflows/actions/guardrails@v0.5.0` with `token: ${{ github.token }}`; outputs `environment`, `kind`.
  - job `promotion-gate`: `needs: guardrails`, `if: always()`, `environment: ${{ needs.guardrails.outputs.environment }}`, one step that exits 1 unless `GUARDRAILS == 'success'` (env from `needs.guardrails.result`).

- [ ] **Step 1: Write failing tests** in `tests/test_gate.sh`: `test_gate_jobs` (job ids exactly `guardrails`, `promotion-gate`; gate `needs: guardrails`; environment expression reads `needs.guardrails.outputs.environment`; guardrails action pinned to the same tag as mirror/promote); `test_gate_fails_closed` (Review Focus 1: `if: always()` and the step fails unless the result is `success`); `test_gate_never_runs_pr_code` (guardrails checkout is `ref: main`; no `run:` executes a path from the PR head).
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement; bump pins; `actionlint`; template tests PASS; commit, push.
- [ ] **Step 4:** Add `tests/test_gate.sh` to repos-sync OWNED (fleet test still PASS); `task repos:sync`; each app's tests PASS; commit, push. Expected: each app repo's open PRs now show a `promotion-gate` run on their next event (not before).

### Task 5: `task gh:envs`

**Files:**
- Create: `promo-fleet/scripts/gh-envs.sh`, `tests/test_ghenvs.sh`
- Modify: `Taskfile.yml` (`gh:envs`), `NOTES.md` ("Environments and the gate")

**Interfaces:**
- `scripts/gh-envs.sh [--dry-run]`: for every app in `apps.yaml` and tier in `tiers.yaml`, idempotently: `PATCH repos/<r>` (`allow_auto_merge`, `delete_branch_on_merge`, `allow_squash_merge` true; `security_and_analysis` secret scanning + push protection enabled); `PUT repos/<r>/environments/<tier>` (reviewers per Global Constraints, `deployment_branch_policy: {protected_branches: false, custom_branch_policies: true}`); add branch policies `main` and `refs/pull/*/merge` if missing; `PUT repos/<r>/branches/main/protection` per Global Constraints. Reviewer id from `gh api user --jq .id`, overridable with `PROMO_REVIEWER_ID`. `--dry-run` prints one line per write: `<METHOD> <path> <compact JSON body>`, and makes no write calls (list calls may still read).

- [ ] **Step 1: Write failing tests** (`PROMO_REVIEWER_ID=42`, a `gh` stub that answers reads with empty lists and fails any write): `test_ghenvs_dry_run_plan` (per repo: one repo PATCH, one PUT per tier, two branch policies per tier, one protection PUT; prd reviewers `[{"type":"User","id":42}]`, dev/stg `[]`; `prevent_self_review: false`; protection contexts `["promotion-gate"]`, `strict: false`, `enforce_admins: false`); `test_ghenvs_follows_tiers_yaml` (a temp 4-tier `tiers.yaml` → 4 environments per repo); `test_ghenvs_dry_run_makes_no_writes`.
- [ ] **Step 2:** Run. Expected: FAIL.
- [ ] **Step 3:** Implement; add the Taskfile target; write NOTES "Environments and the gate" (the check, the classification table, why branch policy includes `refs/pull/*/merge`, auto-merge = approval for prd, revert PRs gated like promotions, and the forged-failure risk from Phase 2 now limited to "revert to the previous release, gated by the same checks").
- [ ] **Step 4:** Fleet tests PASS. Commit, push.

### Task 6: Live acceptance

**Files:** `promo-fleet/NOTES.md` (results)

- [ ] **Step 1:** Run `task gh:envs -- --dry-run`, show the output to the user, and ask them to run `! task gh:envs`. Then confirm with reads: environments per repo, prd reviewer, branch protection context `promotion-gate`, `allow_auto_merge: true`.
- [ ] **Step 2:** Trigger real dev events for both apps with open stg PRs: `flux reconcile hr <app> -n <app> --force --context kind-tier-dev` for `kyverno` and `kyverno-policies` (re-runs upgrade + tests). Expected: dev `success` Deployments still record (Review Focus 4; if the API rejects them, apply the fallback and ledger it).
- [ ] **Step 3:** Expected from those events: promo-kyverno #6 and kyverno-policies #3 each get a new commit, `promotion-gate` passes in environment `stg` with no approval, and both PRs merge themselves. stg upgrades, tests pass, `success` Deployments record in stg, and a prd PR opens per app whose gate **waits** for approval (Review Focus 5).
- [ ] **Step 4:** Ask the user to approve the prd deployment reviews. Expected: each gate passes, the PR merges itself, prd upgrades, a prd `success` Deployment records, and no further PR opens.
- [ ] **Step 5:** Sneaky and hand-made PRs, from branches pushed with the user's token:
  - `promo-prometheus` branch `promote/stg` changing a shared value in `tiers/stg/release.yaml` **and** `tiers/stg/tier.yaml` retention → `promotion-gate` fails on files.
  - `promo-otel-collector` branch `promote/prd` changing a shared value in `tiers/prd/release.yaml` (stg has no success for that release) → fails on release match and evidence.
  If the classifier blocks opening PRs, push the branches and ask the user to open them. Close both after.
- [ ] **Step 6:** Record results in `NOTES.md`; update the ledger and `RESUME.md`.

**Phase 3 check-in:** report acceptance results to the user before Phase 4.

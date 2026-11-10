# Resume: evidence-gated promotion demo

_Where things stood at the end of Phase 3 (2026-11-09)._

To continue with Claude Code, open this repo and say: **"Read RESUME.md and continue."**

## What this repo is

Docs only. The demo itself lives in seven GitHub repos under `twallac10` (below).

| File | What it is |
|---|---|
| `promotion-demo-handoff.md` | The original spec |
| `LESSONS.md` | Lessons from the previous attempt. Amends the spec (see "Decisions") |
| `docs/superpowers/plans/2026-11-08-phase-0-1-scaffold-and-apps.md` | Implementation plan for Phases 0–1 |
| `docs/superpowers/ledger/2026-11-08-phase-0-1-scaffold-and-apps.md` | Progress ledger: every task's result and every decision ("Ruling") made along the way |
| `docs/superpowers/plans/2026-11-09-phase-2-notifications-and-promote.md` | Implementation plan for Phase 2 |
| `docs/superpowers/ledger/2026-11-09-phase-2-notifications-and-promote.md` | Phase 2 ledger: results, Rulings, final review, deferred minors |
| `docs/superpowers/plans/2026-11-09-phase-3-guardrails-and-environments.md` | Implementation plan for Phase 3 |
| `docs/superpowers/ledger/2026-11-09-phase-3-guardrails-and-environments.md` | Phase 3 ledger: results, Rulings, final review, deferred minors |
| `RESUME.md` | This file |

## Decisions already made (don't re-ask)

- Evidence path: native Flux Helm tests → `githubdispatch` Alert → reusable GitHub Actions promote workflow. **No custom controller.**
- Apps: Kyverno (admission + reports controllers only), kyverno-policies, **plain Prometheus** (no operator, no Grafana), OTel Collector. podinfo is the canary (not promoted).
- Repo prefix `promo-`. The old `platform-*` repos and `local-promo` are left untouched.
- Tiers dev → stg → prd. Only prd needs approval; prevent-self-review is **off** (you approve your own prd PRs).
- Revert PRs auto-merge in tiers that don't need approval. kyverno-policies has its own repo. Guardrails stay in `promo-fleet`. Chart signing is out of scope.
- Workflows open PRs with the **GitHub App `promo-bot-twallac10`** (App ID 5673385), not `GITHUB_TOKEN`. PRs made with `GITHUB_TOKEN` trigger no workflows, which would block the Phase 3 `promotion-gate` check.

## Status

**Phase 0 and most of Phase 1 are done and verified live** on the old Mac (arm64, Docker 6 CPU / 12.5 GiB):

- All three tiers came up from nothing with `task up`. Every Helm test passed on first install; 0 restarts.
- dev was stable for 30 minutes at ~0.24 core and ~2.1 GiB.
- With all three tiers running: ~0.13 core each, 6.2 GiB in total.
- The guardrail is enforced by the API server itself, through a ValidatingAdmissionPolicy that Kyverno generates. It denies `HelmRepository` and any foreign `OCIRepository` in every tier.
- kyverno-policies is Audit in dev and stg, and Deny (Enforce) in prd.
- A dev-only `tier.yaml` change reached only dev.
- Results are in `promo-fleet/NOTES.md`.

**Phase 1 finished on 2026-11-09** on the new Mac (arm64, Docker 8 CPU / 15.6 GiB):
1. One `task down && task up` brought all three tiers to Ready with no manual steps in ~8.5 min (~6.8 GiB total).
2. Final review done: no Critical; two Important fixed (see the ledger), six Minor recorded there, not fixed.
3. kyverno-policies 3.9.2-2 (Ready-test fix) is in dev.

**Phase 2 finished on 2026-11-09** (notifications + promote workflow), verified live:
- Each app repo's `tiers/<tier>/notify.yaml` sends Flux test results to the repo (`repository_dispatch`); `promo-workflows` **v0.4.2** `promote.yaml` validates the event, checks it against `main` (digest + release id), records a Deployment and opens the promotion PR (or a revert PR after a failed upgrade in a rollback tier).
- Fresh `task up`: `success` Deployments for all 12 app/tier pairs and one promotion PR (kyverno-policies dev → stg). A dev test failure: `failure` Deployment, no PR. Stale and forged events: no Deployment. Revert path: revert PR touching only stg `release.yaml`.
- Final review: 4 Important fixed (v0.4.2), 6 Minor deferred (see the Phase 2 ledger). Results in `promo-fleet/NOTES.md`.

**Phase 3 finished on 2026-11-09** (guardrails, Environments, auto-merge), verified live:
- Every PR to an app repo's `main` needs the `promotion-gate` check (`gate.yaml`, `pull_request_target`): `guardrails` (promo-workflows **v0.5.3** `actions/guardrails`) classifies the PR by branch (`promote/<t>`, `revert/<t>`, other) and checks files, release, Deployment evidence, chart sources and secrets; `promotion-gate` then waits in that tier's environment. `task gh:envs` set up environments (prd: you approve), branch protection and auto-merge.
- Live: stg PRs merged themselves; a prd PR waited for your approval, then merged itself and deployed. A sneaky PR (tier.yaml riding along) and a hand-made prd PR failed. A fake same-named check did not unblock a PR.
- Final review: 1 Critical + 1 Important fixed (v0.5.3), 1 Important disproved live, 3 Minor deferred (see the Phase 3 ledger).

**Open in GitHub:** promo-kyverno-policies #5 (prd promotion, 3.9.2-2) waits for your approval. Old gate runs still waiting for prd approval (from before fixes) can be rejected.

**In progress (stopped 2026-11-10): signed test evidence with Testkube.** Design `docs/superpowers/specs/2026-11-09-test-evidence-design.md`, plan `docs/superpowers/plans/2026-11-10-signed-test-evidence.md`, ledger `docs/superpowers/ledger/2026-11-10-signed-test-evidence.md`. Working in dev and stg (promo-workflows v0.6.0: Testkube behaviour tests -> JUnit in GHCR + signed test-result attestation -> gate verifies it). **First fix the prd deadlock** (see the ledger's last lines): prd Testkube needs kyverno-policies' provisioner exemption, but kyverno-policies waits on Testkube. Then finish Task 7 and run the final review.

**Next: Phase 4** (demo scenarios: `demo:good`, `demo:ready-but-broken`, `demo:stg-rollback`, `demo:sneaky`, each from a fresh `task up`). Carry in the deferred minors from both ledgers.

## Repos

All public, all CI green, all pushed, no local-only work.

| Repo | Role |
|---|---|
| `promo-fleet` | `tiers.yaml`, `apps.yaml`, kind configs, cluster sync paths, canary, guardrails, `Taskfile.yml`, `NOTES.md` |
| `promo-workflows` | Reusable workflows `mirror.yaml`, `promote.yaml` and composite actions `actions/promote`, `actions/guardrails`, at tag **v0.5.3**; tests in `tests/run.sh` (needs gitleaks) |
| `promo-app-template` | Template the app repos were made from |
| `promo-kyverno`, `promo-kyverno-policies`, `promo-prometheus`, `promo-otel-collector` | App repos. Template-owned files kept in sync by `task repos:sync` (fleet) |

## Set up the new machine

### 1. Tools

Versions are what was used. Flux must be **2.10.0** exactly: the tests cache CRD schemas per Flux version and `task up` installs the CLI's own version.

| Tool | Version |
|---|---|
| docker | any recent |
| kind | 0.33.0 |
| kubectl | 1.37 |
| flux | 2.10.0 |
| helm | v4.3.1 |
| gh | 2.103 |
| task | 3.54 |
| yq | mikefarah v4 |
| jq | any |
| kubeconform | 0.8.0 |
| actionlint | 1.7 |
| kyverno CLI | 1.19.2 |
| gitleaks | 8.30.1 (for promo-workflows tests) |
| perl | any |

`gh auth login` as `twallac10`. GHCR charts are public, so no registry login is needed.

### 2. Docker resources

The old Mac ran all three tiers in ~6.2 GiB and ~0.4 core at idle; bring-up is the expensive part. On the new machine:
- Give Docker **at least 8 GiB** (12+ to be comfortable) and 4+ CPUs.
- If it is **Linux** (native Docker, not Docker Desktop), raise the inotify limits first, or the third kind cluster fails with "too many open files":
  ```
  sudo sysctl fs.inotify.max_user_watches=524288 fs.inotify.max_user_instances=512
  ```
- If it is **amd64**: nothing changes. Every pinned image has both arm64 and amd64 builds (`task images:check`).

### 3. Clone side by side

The scripts expect the `promo-*` repos as siblings: `images:check` and the fleet tests read `../promo-*`.

```
mkdir -p ~/projects/promo && cd ~/projects/promo
for r in promo-fleet promo-workflows promo-app-template promo-kyverno promo-kyverno-policies promo-prometheus promo-otel-collector; do
  gh repo clone twallac10/$r
done
git clone https://github.com/twallac10/local-promo-workflow ~/projects/local-promo-workflow
```

### 4. Check it, then bring it up

```
cd ~/projects/promo/promo-fleet
task lint                 # static tests (fleet)
for r in ../promo-kyverno ../promo-kyverno-policies ../promo-prometheus ../promo-otel-collector; do (cd $r && tests/run.sh | tail -1); done
task images:check
task up                   # dev, then stg, then prd; each waits for Ready
task status               # full Flux messages + restart counts per tier
```

`task up -- dev` brings up one tier. `task down` removes all clusters.

## Secrets

- The App's ID and private key are already stored as Actions secrets (`PROMO_APP_ID`, `PROMO_APP_PRIVATE_KEY`) in the four app repos. Nothing to move.
- The `.pem` on the old Mac (`~/Downloads/promo-bot-twallac10.2026-11-08.private-key.pem`) is only needed to set those secrets again. Store it somewhere safe or delete it. You can generate a new key in the App settings at any time.
- **Dispatch tokens** (Flux → `repository_dispatch`): one fine-grained PAT per tier, Contents: write on the four app repos, 90-day expiry (from 2026-11-09). Kept in `~/.config/promo/dispatch-<tier>.token` (mode 600); `task up` loads them and refuses a tier without one. Renew before they expire.
- The App needs Contents, Pull requests, Deployments and Issues: write.

## Gotchas found so far

- **Never move a chart tag.** The mirror workflow refuses to re-push an existing version unless the first tier already pins that digest. Bump `chart/Chart.yaml` `version` (`<upstream>-<n>`) for any chart change.
- **On a Mac, sleep pauses everything.** A watch loop stalls while macOS sleeps. Wrap long runs in `caffeinate -i`.
- **`flux reconcile ks --with-source` across three clusters** was slow to return even after dev was done. Check HelmRelease status directly instead.
- **Auto mode sometimes blocks GitHub PR merges, closes and admin settings.** Claude asks you to do them; `task gh:envs` is yours to run (review `-- --dry-run` first).
- **A Docker restart wipes the clusters** (etcd on tmpfs): every API call is refused afterwards. `task down && task up` rebuilds in ~8 min and every app re-reports.
- **Direct pushes to app `main`** (template syncs) bypass the required check as admin; everything else goes through PRs.
- **A GitHub outage** shows up as runs failing at startup with no jobs; only the run page says why.
- **Every fresh `task up` re-sends every app's test result from every tier.** That records Deployments and may open or close promotion PRs; it's expected.

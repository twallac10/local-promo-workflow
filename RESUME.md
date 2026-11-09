# Resume: evidence-gated promotion demo

_Where things stood on 2026-11-09, when work moved to a bigger machine._

To continue with Claude Code, open this repo and say: **"Read RESUME.md and continue."**

## What this repo is

Docs only. The demo itself lives in seven GitHub repos under `twallac10` (below).

| File | What it is |
|---|---|
| `promotion-demo-handoff.md` | The original spec |
| `LESSONS.md` | Lessons from the previous attempt. Amends the spec (see "Decisions") |
| `docs/superpowers/plans/2026-11-08-phase-0-1-scaffold-and-apps.md` | Implementation plan for Phases 0–1 |
| `docs/superpowers/ledger/2026-11-08-phase-0-1-scaffold-and-apps.md` | Progress ledger: every task's result and every decision ("Ruling") made along the way |
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

**Left in Phase 1:**
1. Task 8 Step 6: `task down && task up` from scratch on the new machine. All three tiers must converge with no manual steps. Record the new machine's footprint in `promo-fleet/NOTES.md`.
2. Final whole-branch review of Phases 0–1 (see the ledger's Rulings), then fix anything Critical or Important.

**Then Phase 2** (notifications + promote workflow), per the spec.

## Repos

All public, all CI green, all pushed, no local-only work.

| Repo | Role |
|---|---|
| `promo-fleet` | `tiers.yaml`, `apps.yaml`, kind configs, cluster sync paths, canary, guardrails, `Taskfile.yml`, `NOTES.md` |
| `promo-workflows` | Reusable workflows. `mirror.yaml` at tag **v0.3.0** |
| `promo-app-template` | Template the app repos were made from |
| `promo-kyverno`, `promo-kyverno-policies`, `promo-prometheus`, `promo-otel-collector` | App repos. All tiers pinned to the same GHCR digest |

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
- Phase 2 will add a per-cluster dispatch token (Flux → `repository_dispatch`). It doesn't exist yet.

## Gotchas found so far

- **Never move a chart tag.** The mirror workflow refuses to re-push an existing version unless the first tier already pins that digest. Bump `chart/Chart.yaml` `version` (`<upstream>-<n>`) for any chart change.
- **On a Mac, sleep pauses everything.** A watch loop stalls while macOS sleeps. Wrap long runs in `caffeinate -i`.
- **`flux reconcile ks --with-source` across three clusters** was slow to return even after dev was done. Check HelmRelease status directly instead.

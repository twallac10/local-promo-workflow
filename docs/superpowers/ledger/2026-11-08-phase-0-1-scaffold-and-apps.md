# SDD ledger — plan: docs/superpowers/plans/2026-11-08-phase-0-1-scaffold-and-apps.md
Spec: promotion-demo-handoff.md + LESSONS.md + memory promo-demo-decisions
Pre-flight:
- T2→T3: template shape (base/, tiers/, tests/run.sh) consumed by repos-create.sh — consistent.
- T2→T5: mirror/chart.yaml {upstream,wrapper.name} consumed by mirror workflow + name test — consistent.
- T3→T6: apps.yaml {name,repo,namespace,dependsOn} consumed by gen-clusters.sh — T6 adds canary deps for prometheus/otel; apps.yaml dependsOn lists only app deps. Ruling: gen-clusters adds `canary` to dependsOn for prometheus and otel-collector via an apps.yaml field `needsCanary: true` — keeps apps.yaml the single source — cost if wrong: one field rename.
- T4→T7: wrapper value nesting under upstream name — consistent.
Task 1: Ruling: creating public GitHub repos denied by auto-mode classifier — built promo-workflows locally (commits b05f151..61d64a9, actionlint → OK); gh repo create/push/tag v0.1.0 and CI run deferred until user grants permission — cost if wrong: none, push is mechanical.
Task 1: local-complete (promo-workflows b05f151..61d64a9, tests: actionlint → clean). Remote steps PENDING.
Task 2: Ruling: template app name `example` (namespace/objects `example`, chart `promo-example`); repos:create rewrites `example` → <app> in base/ tiers/ chart/ mirror/ — cost if wrong: rename script.
Task 2: Ruling: rollback tiers use remediation {retries: 0, remediateLastFailure: true, strategy: rollback} (verified fields in Flux 2.10 CRD); dev sets remediateLastFailure: false — cost if wrong: one tier.yaml field.
Task 2: Ruling: added test_remediation_matches_onfailure (not in plan) — pins onFailure semantics — cost: none.
Task 2: local-complete (promo-app-template 44a9797..934ace3, tests: tests/run.sh → 5/5 pass; mutation checks: non-promotable field and unknown field both caught). Remote steps PENDING.
Task 3: Ruling: etcd tmpfs via kubeadm etcd.local.dataDir=/tmp/etcd (kind nodes mount /tmp as tmpfs), not extraMounts as plan text suggested — verified findmnt=tmpfs live — cost if wrong: none observed.
Task 3: Ruling: dropped apps.yaml `namespace` (namespace = app name, kyverno-policies gets own ns to avoid two Kustomizations owning ns/kyverno); canary listed directly in dependsOn instead of needsCanary — supersedes pre-flight ruling — cost if wrong: field rename.
Task 3: Ruling: added test_apps_yaml_shape and test_customize_renames_app (not in plan) — cover repos:create rename offline — cost: none.
Task 3: local-complete (promo-fleet 1c188a0..fbfeed3, tests: tests/run.sh → 6/6 pass, live: kind dev + flux install OK, etcd tmpfs verified, down x2 exit 0). GitHub repo creation (task repos:create) + LIVE up test PENDING permission. App repos created locally from template; all pass their tests.
Task 4: Step 1 inventory done (promo-fleet 72eb405, NOTES.md). STOPPED for user check-in per plan; also blocked on GitHub repo creation permission.
Task 1: complete (promo-workflows pushed, tag v0.1.0, CI ci → success)
Task 2: complete (promo-app-template pushed + marked template, CI → success)
Task 3: Ruling: app repos pushed from local template clones (gh repo create --source) instead of gh --template; repos:create verified idempotent (all exist) — same content, history includes template commits — cost: none.
Task 3: Ruling: up.sh skips HelmRelease wait when tier has none (live test RED→GREEN); customize-app.sh uses perl (GNU/BSD sed differ, CI RED→GREEN); added fleet CI — cost: none.
Task 3: complete (promo-fleet ..HEAD pushed, tests: LIVE=1 tests/run.sh → 6/6 pass, CI success)
Task 4: Ruling: Kyverno test uses a test-hook ValidatingPolicy (Deny on a test label) + server-side dry-run in `default` (Kyverno skips its own ns) — proves admission end to end without depending on kyverno-policies — cost: ~5s per test.
Task 4: Ruling: Kyverno webhooks exclude flux-system as well as kube-system (Review Focus 3). Guardrails will use ValidatingPolicy autogen→native VAP so the exclusion doesn't weaken them (Task 6).
Task 4: Ruling: wrapper version scheme <upstream>-<n> (e.g. 3.9.2-1); single test image alpine/kubectl:1.37.2 pinned by digest — cost: none.
Task 4: Ruling: Prometheus default scrape jobs disabled via chart's keyed `scrapeConfigs` (serverFiles list was appended → duplicate job, caught by promtool); OTel accepted-spans test sends its own OTLP span (no canary dependency) — so otel-collector dependsOn canary is unnecessary; kept for now, revisit Task 6.
Task 4: Ruling: is_template = chart has no dependency; chart_deps adds upstream repo first (CI RED→GREEN); docker-run tests chmod temp dir (Linux 0700, CI RED→GREEN).
Task 4: complete (all 4 app repos + template + fleet CI success; local tests/run.sh all pass; mutations caught: ns exclusion, unpinned image, version label, otel prometheus exporter, amd64-only image)
Task 5: Ruling: PRs opened with a GitHub App token (user chose App over PAT/repo setting): GITHUB_TOKEN can't create PRs here and its PRs trigger no workflows (would block Phase 3 promotion-gate). promo-workflows v0.3.0; callers `secrets: inherit` — cost: user must create App + secrets.
Task 5: Ruling: mirror never moves a tag: existing version tag OK only if its digest == first tier's pinned digest, else fail "bump version" — cost: none.
Task 5: GHCR package auto-public (linked to public repo); anonymous pull verified for promo-kyverno 3.9.2-1 sha256:c28936e4…
Task 5: IN PROGRESS — waiting on user's GitHub App secrets to finish PR step; Task 6 started meanwhile.
Task 6: Ruling: guardrail is a Kyverno ValidatingPolicy with autogen→native ValidatingAdmissionPolicy, Deny in all tiers — enforced by API server even though Kyverno's webhook excludes flux-system — cost if wrong: Kyverno VAP generation unsupported for CRD kinds → fall back to webhook policy (check live in Task 8).
Task 6: Ruling: otel-collector no longer depends on canary (its test sends its own span); canary chart copied to our GHCR by promo-fleet workflow mirror-canary (packages pushed by a public repo's workflow are public; verified anonymous pull) — cost: none.
Task 6: Ruling: app Kustomizations use wait: true so dependsOn means HelmRelease Ready incl. tests — cost: slower bootstrap.
Task 6: complete (promo-fleet tests/run.sh → 12/12 pass; guardrail fixtures: foreign-oci fail:1, our-oci pass:1; canary podinfo sha256:9d6f49d6…)
Task 7: complete (kyverno-policies test_validation_failure_action RED→GREEN Audit/Audit/Enforce, rendered Audit/Deny; prometheus test_retention_per_tier RED→GREEN 2h/6h/12h)
Task 8: added task status. BLOCKED: app chart digests need mirror runs, which need PROMO_APP_ID/PROMO_APP_PRIVATE_KEY secrets from user.
Task 5: Ruling: promo-kyverno bumped to 3.9.2-2 — 3.9.2-1 was pushed by the run whose PR step failed and the never-move-tag guard (correctly) refused to reuse it; orphan 3.9.2-1 left in GHCR — cost: one unused package version.
Task 5: Ruling: I set PROMO_APP_PRIVATE_KEY via stdin redirect from the user's .pem after two paste attempts broke on line wrap; key never printed — cost: none.
Task 5: complete (4 mirror runs success; 4 dev PRs by app/promo-bot-twallac10, CI ran on each, diff only tiers/dev/release.yaml, merged; stg/prd seeded from dev; anonymous pulls verified for kyverno + podinfo)
Task 8: Steps 1 and 3 done on dev (all Ready, all helm tests pass first install, 26 pods / 0 restarts, ~0.22 core / 2.05 GiB; HelmRepository denied by generated VAP; OCI urls all ours; policies Audit). Cluster torn down at user's request.
Task 8: PAUSED by user ("stand down"). Remaining: Step 2 (30-min stability), Step 4 (stg, prd + Enforce check), Step 5 (live tier isolation), Step 6 (full down/up). Then final whole-branch review.
Task 8: Step 2 done — dev 30-min watch 18:31–19:01, 0 restarts, CPU 0.18–0.33, mem ~2.1 GiB (first attempt stalled by Mac sleep; rerun under caffeinate -i).
Task 8: Step 4 done — stg and prd Ready from nothing; all helm tests pass; 3 tiers together: 26 pods each, 0 restarts, ~0.13 core each, 6.2 GiB total; guardrail denies in all tiers; policies Audit/Audit/Deny; privileged pod rejected in prd.
Task 8: Step 5 done — live tier isolation: dev tier.yaml retention 2h→3h moved only dev's prometheus to rev 2 (stg/prd rev 1). Probe reverted (promo-prometheus 9f9609c).
Task 8: Ruling: promo-fleet CI had been red since Task 6 (unnoticed: I didn't check CI after Task 6) — kyverno CLI missing in CI; scan picked up CI's .template checkout. Fixed: scan git-tracked files only (RED→GREEN locally with .template, mutation caught), install kyverno CLI 1.19.2 in CI. CI success.
Task 8: PAUSED — user moving to a bigger machine. All clusters down. Remaining: Step 6 (task down && task up from scratch, all 3 tiers, no manual steps), then final whole-branch review, then Phase 2.
Task 8: Step 6 done on new machine (arm64, Docker 8 CPU / 15.6 GiB, Flux 2.10.0) — one `task down && task up`, all 3 tiers Ready with no manual steps in ~8.5 min; all helm tests pass first install; 26 pods/tier, 0 restarts; idle ~0.1 core/tier, ~6.8 GiB total. promo-fleet 4fb8dd7. Next: final whole-branch review.

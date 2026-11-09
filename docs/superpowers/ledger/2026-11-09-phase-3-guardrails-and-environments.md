# SDD ledger — plan: docs/superpowers/plans/2026-11-09-phase-3-guardrails-and-environments.md
Setup: Ruling: work directly on main in the promo-* repos, no worktree — same as Phases 0–2 — cost if wrong: history on main instead of a branch.
Setup: task-start/task-done run in the repo each task mainly changes (T1–T3 promo-workflows, T4 template, T5–T6 fleet); other repos' commits named in notes.
Pre-flight:
- T2→T4: guardrails action inputs token, fleet-repo; T4 caller passes only token. Ruling: fleet-repo input defaults to twallac10/promo-fleet (as in promote) — cost: none.
- T2/T3→T4: guardrails ships untagged in T2 and is tagged v0.5.0 with T3; T4 pins v0.5.0 for mirror, promote and guardrails — consistent.
- T1→T4/T5: the spike decides whether "no tier" PRs use an empty environment or an unprotected 'none' env that gh-envs must also create — carried as T1's ruling.
- T2→T4: check-evidence reads Deployments with GITHUB_TOKEN; T4 gate permissions include deployments: read — consistent.
- T5→T6: gh-envs required context 'promotion-gate' = T4 job id — consistent.
Task 1: complete — spike run 42667991004: environment: '' → job ran with no environment, 0 Deployments, 0 environments created. No 'none' environment needed. Spike workflow added and removed (promo-workflows 2 commits).
Task 2: Ruling: action reads base/head SHAs and head ref from the event file and fetches the head itself, instead of extra inputs — keeps the caller minimal and no PR field passes through an expression — cost: none.
Task 2: Ruling: check-release extracts scripts/release-id.sh from the base commit (git show) and runs that copy, rather than ./scripts — guarantees main's script even in a mixed checkout — cost: none.
Task 2: Ruling: shared actions/guardrails/lib.sh (fail, changed_files) — cost: none.
Task 2: gitleaks pinned v8.30.1, linux_x64 sha256 551f6fc83ea457d62a0d98237cbad105af8d557003051f41f3e7ca7b3f2470eb (from the release checksums.txt); installed locally via Homebrew 8.30.1 so the secret test runs here.
Task 2: test fixture fixes: gh stub ${*#…} sliced each arg (fixed with all="$*"); app_repo now has prd and tier.yaml files (existing tests still pass).
Task 2: complete (promo-workflows 13c95b6..HEAD; tests/run.sh → 33/33 pass; actionlint ok)
Task 3: Ruling: the auto-merge warning goes to stderr, not stdout — stdout is tee'd into $GITHUB_OUTPUT, which accepts only key=value lines; the runner still shows ::warning:: from stderr — cost if wrong: the warning is a plain log line, not an annotation.
Task 3: Ruling: the label is re-applied in the same `gh pr edit` call as title/body (one call) — cost: none.
Task 3: evidence() test helper moved to tests/lib/lib.sh (now used by two test files).
Task 3: complete (promo-workflows d5284d8..HEAD, tag v0.5.0; tests/run.sh → all passed; actionlint ok)
Task 4: test_gate_fails_closed mutation-checked (removing if: always() → FAIL).
Task 4: complete (template ..2903884; fleet repos-sync adds tests/test_gate.sh; apps synced and pushed; tests: template, fleet, all apps → all passed; actionlint ok). Base for the ledger script was the pre-Phase-3 template head c8ba26a.
Task 5: Ruling: PROMO_TIERS env var overrides tiers.yaml (for test_ghenvs_follows_tiers_yaml) — cost: none.
Task 5: Ruling: branch policies use type "branch" with name "refs/pull/*/merge" (per GitHub docs' suggestion); verified live in Task 6 — cost if wrong: gate jobs rejected by the environment; switch the rule.
Task 5: complete (fleet ..HEAD; tests/run.sh → all passed)
Task 5+: Security review fixes (before go-live), promo-workflows v0.5.1 (5367f9f):
Task 5+: Ruling: check-sources fails closed (NUL-safe file list, parse error fails, kinds at any depth) — test_sources_fails_closed RED→GREEN (space-in-name, List-nested, unparseable) — cost: none.
Task 5+: Ruling: an "other" PR touching base/ (deploys to every tier), .github/ or scripts/ is gated by the last tier — test_classify_shared_paths_need_last_tier RED→GREEN — cost: such PRs need prd approval.
Task 5+: Ruling: gate triggers on pull_request_target, not pull_request (plan) — on pull_request GitHub runs the PR's own gate.yaml, which a PR could rewrite to pass; safe because read-only token, no secrets, no PR code — test_gate_uses_mains_definition RED→GREEN — cost if wrong: none known; fork PRs also gated by main's definition.
Task 5+: Ruling: environment branch policy is main only (plan had main + refs/pull/*/merge) — with pull_request_target the gate's ref is main; matches the spec — gh-envs tests updated (8 writes/repo) — cost if wrong: gate jobs rejected by the environment, visible in Task 6.
Task 5+: Ruling: kept tag pins for promo-workflows (security review asked for SHAs) — spec allows tag or SHA; same owner; never-move-tag rule — cost if wrong: a moved tag changes callers' code.
Task 5+: Security review: fixed multi-document fail-open in check-sources (jq -e status = last doc) — test_sources_checks_every_document RED→GREEN; v0.5.2, all callers pinned.
Task 6 Step 1: user ran task gh:envs (32 writes ok); read back: auto-merge on, required check promotion-gate, prd reviewer twallac10, branch policies main only, in all 4 repos.
Task 6: first attempt hit a GitHub outage (user confirmed): promo-kyverno promote runs failed at startup with no jobs; kyverno-policies #3's promotion-gate job was never created. Not a code issue.
Task 6 Step 2 (retry): forced dev re-tests → dev success Deployments recorded under the new main-only branch policy (Review Focus 4 holds: API Deployments with ref=SHA accepted).
Task 6 Step 3: kyverno-policies #3 and promo-kyverno #6 (opened before Phase 3) got new commits, promotion-gate passed in stg without approval, both merged themselves (19:22:57, 19:23:32). stg tested OK, stg success Deployments, prd PRs opened (promo-kyverno #7, kyverno-policies #5) and their gates waited (Review Focus 5 holds).
Task 6 Step 4: user approved prd for promo-kyverno #7 → promotion-gate success → merged itself 19:48:41 → prd upgraded to 3.9.2-4 → prd success Deployment 19:50:14, no further PR. kyverno-policies #5 still waiting for approval.
Task 6: Docker restarted at 21:09 UTC (after the chain completed); etcd on tmpfs → all clusters empty and unusable. Rebuilding with task down && task up.
Task 6: rebuild after Docker restart: all 3 tiers Ready in ~8 min, 15 HRs TestSucceeded first install, 26 pods/tier 0 restarts; 12 events → 12 success Deployments; kyverno-policies #5 updated in place, gate waiting (prd).
Task 6 Step 5: sneaky PR (promo-prometheus #2) → guardrails failed on files, promotion-gate ran and failed (fail-closed live). Hand-made PR (promo-otel-collector #2) → failed on release check ("tiers/prd/release.yaml in the PR differs from tiers/stg/release.yaml on main"). Both closed, branches deleted.
Task 6: Ruling: promotion-gate's environment = guardrails' only when guardrails succeeded (else none) — live: a failed PR's gate asked for prd approval before failing; test_gate_jobs RED→GREEN; synced to apps (template + 4 apps) — cost: none.
Task 6: Ruling: template syncs push directly to app main as an admin bypass of the required check (enforce_admins false, per plan) — cost if wrong: sync commits skip the gate (they change only template-owned files).
Task 6 Step 6: NOTES Phase 3 results (fleet). Open for the user: kyverno-policies #5 (prd) awaiting approval; stale waiting gate runs (otel run 42709554… for the closed PR; kyverno-policies 19:24 run) can be rejected or ignored.
Final review: fresh reviewer (general-purpose, opus) over promo-workflows 4f633b7.., template 57f1636.., fleet 00a8100.., synced apps. Verdict: fix rename bypass and gate-Deployment contamination before Phase 4. Re-graded: 1 Critical, 1 Important fixed; 1 Important downgraded after a live probe; 3 Minor deferred.
Final: fixed rename bypass (git diff rename detection listed only the new path; moving tiers/prd/release.yaml, base/ or .github/ files into tiers/dev was gated as dev) — test_renames_count_both_paths RED→GREEN, suite all passed; changed_files uses --no-renames. v0.5.3.
Final: fixed gate-job Deployments polluting evidence (check-evidence read the newest Deployment of any origin; gate jobs in an environment record their own) — deploy.sh sets task "promote", check-evidence and status.sh filter task=promote; test_evidence_* RED→GREEN with a decoy 'waiting' gate Deployment in the stub, test_deploy_truncates_description asserts task; suite all passed. v0.5.3. Live: all apps re-tested in all tiers to record task=promote evidence.
Final: Ruling: fake same-named promotion-gate check (reviewer: Important, unverified) — not exploitable. Live probe promo-prometheus #3: a PR adding a pull_request workflow with job promotion-gate (exit 0); with the real gate pending (prd approval) and then failed, GitHub marked BOTH same-named runs required and the PR stayed BLOCKED (statusCheckRollup FAILURE). pull_request_target means a PR can't stop the real gate from running. No change — cost if wrong: a PR could satisfy the check by name (GitHub behavior change).
Final: Ruling (declined-to-judge list): Review Focus 4 verified live; fork promote/* branches harmless; strict:false acceptable; post-rollback evidence is Phase 4; tag pins; admin-bypass syncs; chart/ and mirror/ enter only via the dev mirror PR; yq YAML bombs fail the gate; approval long after guardrails is covered by close-pr on failure — all stand as set aside — cost if wrong: revisit in Phase 4 scenarios.
Final: minor (deferred): classify.sh matches tier names as regex (grep -nx, not -F): promote/st. matches stg → GitHub auto-creates an unprotected 'st.' environment; nothing deploys.
Final: minor (deferred): gate.yaml has no concurrency: each PR edit starts another gate run; a prd PR can collect several waiting approval requests.
Final: minor (deferred): PRs that never get a new event after Phase 3 stay pending (fail closed); document "close and reopen" in NOTES.

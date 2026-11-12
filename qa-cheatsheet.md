# Q&A cheat sheet: legacy pipeline vs v2

Built 12 Nov 2026 from the code on `origin/main` of `dev-ci-workflows`, the callers in `ts-dns` (legacy) and `ts-general` (v2), and read-only checks of GitHub environment settings. For questions after the talk. Not for the slides.

Confidence marks: **[code]** read in the code; **[caller]** read in a repo caller workflow; **[gh]** read from GitHub settings; **[infer]** my reading, not tested; **[?]** not verified, do not state as fact.

---

## 1. At a glance

| Topic | Legacy (ts-dns style) | v2 (ts-general style) |
|---|---|---|
| PR plan scope | Every stack in every environment, always [code] | Only changed stacks; shared files give a full plan; docs-only gives nothing [code] |
| PR plan comment | One `Plan` comment per environment; only stacks with changes shown [code] | Same format, plus the footer "How to test before merging (exceptions only)" [code] |
| Merge | Plan runs again on push to main, then Up applies that new plan [code][caller] | Merge applies the plan saved from the PR; re-plans only when it must [code] |
| What is applied | Whatever the merge plan found; can differ from the reviewed plan [code] | The reviewed plan, verified against the merge commit [code] |
| Approval | One GitHub environment approval per environment [code] | Same, per environment; a re-plan asks again [code] |
| Environments | Run in parallel, no ordering between them [code] | Same [code] |
| Saved plan lifetime | 3 days [code] | Plan files 7 days; manifest and marker 30 days [code] |
| Try before merge | Manual Command from the Actions tab [caller] | `/deploy <env>` comment (dispatches the same Manual Command) [code] |
| Slack DM to the deployer | Yes, on merge deployments, updated in place [code] | No [code] |
| `infra:no_changes` label | Checked on push to main only; skips plan and up [code] | Not used; change detection replaces it [code] |
| Auto-format | Yes, on PRs [code] | Yes, on PRs, no skip marker (#685) [code] |
| Policy-as-code check | Optional input `policy_check` [code] | Not present [code] |
| `refresh_state: false` (ts-dns throttling) | Supported [code][caller] | Not supported; v2 always refreshes [code] |
| `stacks_to_apply_first` | Supported; has a re-plan quirk (see 4) [code] | Replaced by dependency-based deferral [code] |
| Commit status `Terraspace Up` | No | Yes (pending, then success or failure) [code] |
| Concurrency | ts-dns: PR plans cancel older runs; merge runs queue per ref [caller] | ts-general: PR plans cancel older runs; Up runs grouped per PR number, no cancel [caller]; none in the shared workflows [code] |

---

## 2. What happens, step by step

### Legacy
1. PR opens or changes: the plan runs for every effective stack (include list minus exclude list) in every environment. `fmt-fix` may push a format commit first. One comment per environment is posted; old ones are deleted [code].
2. Merge to main: the Plan&Up workflow plans again. With `apply_stacks_with_changes_only` it keeps only stacks whose plan log is not "No changes". Environments with changes go to Up [code][caller].
3. Up: one job per environment, each behind its GitHub environment approval, in parallel. The job downloads the plan for that environment and applies [code].
4. Slack DM to the person who triggered it, updated on plan failure, deploy failure and success [code].

### v2
1. PR opens or changes: the classifier decides the scope (see 3). Only the changed stacks are planned, per environment that deploys them. The plan files, the order and a manifest are saved. The comment gets the footer [code].
2. Optional, before merge: `/deploy <env>` tests the change in an environment (see 5).
3. Merge: the Up workflow resolves what to do: reuse the saved plan, re-plan, or nothing. Reuse and re-plan both wait for the environment approval, then apply stack by stack in the saved order [code].
4. A `Terraspace Up` commit status is set on the merge commit (pending, then success or failure) [code].

---

## 3. v2 scenarios: "what happens when..."

**PR content**

- **Docs-only PR (or only ignored files).** Nothing is planned; the plan jobs are skipped. Defaults ignored: `.github/**`, `**/*.md`, `docs/**`, `.gitignore`, `CODEOWNERS`. A repo can add `ignore_paths` (it must be passed identically to the plan and the up callers) [code].
- **One stack changed.** Only that stack is planned (its dependencies are planned internally and then pruned from the result) [code].
- **A tfvars file of a stack changed.** Counts as a change of that stack; tfvars live under the stack directories [code].
- **A stack that depends on a changed stack.** It is NOT planned. Only changed stacks are [code]. Be ready: this is a real gap if a changed output affects a dependent.
- **Shared files changed** (`app/modules`, `config/terraform`, `Terrafile`, or any other non-ignored path such as `config/app.rb`). Full plan of every stack in every environment [code].
- **Stack added to an environment's include list.** Planned in that environment only. **Stack removed:** only a warning, Terraspace never destroys it [code]. Destroy still goes through the Manual Command (one stack at a time) [code].
- **Changed stack is deployed in no environment.** Scope "none", with a warning [code].
- **PR edited (title) or pushed again.** A new plan run replaces the old one for the same head; older `/deploy` replies are marked superseded on a push [code][caller].
- **Format commit from the bot.** One commit "style: auto-format Terraform configuration files"; the plan then runs on the formatted commit. The author must `git pull` [code].
- **Fork PR.** `/deploy` rejects it [code]. The plan behaviour depends on the caller and GitHub defaults [?].

**Merge time**

- **Nothing changed since the plan.** The saved plan is reused and applied after approval [code].
- **Main changed the same stacks, or shared files, after the plan.** Re-plan. The check compares git objects of everything the plan depended on (planned stacks and their dependencies, shared paths, include and exclude lists) between plan time and the merge commit [code].
- **Main changed an unrelated stack.** No re-plan; the saved plan is reused [code].
- **Plan files older than 7 days (or manifest gone).** Re-plan [code].
- **PR plan still running when merged.** The merge run waits up to 30 minutes (checks every 20 seconds), then re-plans [code].
- **PR changes anything under `.github/`.** Always re-plans; a PR that only changes `.github/` deploys nothing [code].
- **Re-run of the run, or manual dispatch.** "A retry always re-plans." Dispatch must run from main, takes a merged PR number and posts the plan with the tag "re-plan from main" [code].
- **Rejected approval.** The run fails; the commit status goes to failure [code].
- **Nothing to deploy.** The up jobs do not run; the summary says "Nothing to deploy"; no commit status [code].
- **Approvers.** Per environment, set in GitHub environment rules. Environments run in parallel with no ordering; production is not blocked by development [code].

**Stale plan** (the failure from the slide)

- **Why it happens.** OpenTofu refuses a plan when the state changed after the plan was made. The message "Saved plan is stale" is OpenTofu's, not a pipeline check [infer: wording is OpenTofu's standard error; the repo only refers to it in retry hints]. Typical causes: a `/deploy` before merge, a manual change, another apply.
- **Fix.** Re-run all jobs (a retry always re-plans). Re-run failed jobs would apply the same stale plan again [code].
- **Dependent stacks inside one apply.** If an earlier stack in the same run changed outputs that a later plan read, OpenTofu refuses that later plan. The stack and its dependents are deferred, then re-planned in the same run and need a new approval [code].
- **After a `/deploy`.** No code invalidates the saved plan. At merge the plan is reused if the code is unchanged, and the apply for those stacks is then likely a no-op or hits the stale error [infer].

**Apply behaviour**

- **Order.** Stack by stack in the recorded order of the plan. Stacks missing from the order file are applied last, alphabetically, with a warning; a missing or empty order file refuses to apply [code].
- **A plan file missing for a changed stack.** "Refusing to apply" [code].
- **Dependency graph unreadable.** The stack fails instead of being deferred [code].
- **Access config changes.** A workflow in ts-security is dispatched after the apply [code].

---

## 4. Legacy sharp edges (the "in the past we did..." answers)

- **The merge plan is a different plan.** No comparison with the reviewed one, no second look at the diff [code].
- **`stacks_to_apply_first` quirk.** If priority stacks changed together with other stacks, the apply ignores the saved plan files and plans again inside the up. The approver saw a plan that is not what gets applied [code]. v2 replaces this with deferral plus re-plan.
- **A PR always planned everything.** One stack changed, full environment planned. `refresh_state: false` existed to cut throttling in ts-dns (plans each stack without refresh) [code][caller].
- **Race between merges.** Shared workflows had no concurrency control and the plan artifact name per environment is not run-scoped (overwrite), so two close runs could swap plans. The ts-dns caller queues merge runs per ref, which avoids it there [code][caller].
- **A failed plan counts as "has changes".** Change detection looks for the "no changes" text; an errored log lands in the changed list [code].
- **One failed environment plan stops all applies.** `create_up_list` has no `always()`, so it is skipped and nothing is applied for any environment [code].
- **Slack messages.** Only for merge deployments, sent as a DM to the person who triggered the run; they are gone in v2 [code].
- **Label.** `infra:no_changes` was only read on push to main; PR plans ignored it [code].

---

## 5. `/deploy`

- **Who.** The caller only reacts to comments from owners, members and collaborators; the workflow then checks admin, maintain or write permission [caller][code].
- **When.** The PR must be open. Fork PRs are rejected [code].
- **Stacks.** Taken from the PR files under `app/stacks` or `config/stacks`. Shared-files-only changes are rejected with a pointer to the Manual Command. Explicit stacks must be changed in the PR, enabled in that environment and not excluded [code].
- **Environment.** Must exist as a key in `include-stacks.yaml` at the PR head [code].
- **Layers.** Without explicit stacks, stacks whose tfvars read `output('<stack>...')` from another selected stack are held back; the bot names them and says to comment `/deploy <env> <stack>` after the first layer applied. Detection is by regex on tfvars, so `depends_on` style links are missed [code].
- **What it does.** Dispatches the Manual Command (`infra-workflow-dispatch.yml`) on the PR branch with `for_up=true`: plan, then an approval, then apply from that plan [code]. For development and staging the manual path uses the environments `development-protected` and `staging-protected`; other environments use their own name [code].
- **Approval is not always "the same".** In ts-general, ts-security and ts-dns, `development` and `development-protected` both have required reviewers, but they are separate environments and the reviewer lists can differ [gh]. A code comment says development and staging approvals are sometimes disabled in other repos [code]; so "you always approve" is true for the repos I checked, not guaranteed everywhere.
- **Superseded.** A new push rewrites earlier "Dispatched deployment" replies as struck through with a link to the new commit [code][caller].
- **Wording.** The plan footer says it is "meant for urgent fixes and quick testing", that merging is the standard way, and that without merging or reverting, the state "shows up as drift" [code].
- **Concurrency.** None in the workflow; the Manual Command caller groups runs by stack list [caller].

---

## 6. Short answers to likely tough questions

- **Is it faster?** Median job time, 1 Oct to 9 Nov: ts-security plan 137 s to 95 s (350 jobs) and up 125 s to 88 s (51 jobs). Approval wait is not included and was 68% of deploy time in ts-security. Early data; some repos have too few jobs to say.
- **Why did the plan success rate drop in ts-databases?** Real, 92% to 80% on 56 jobs, being looked at; not understood yet.
- **What if I merge two PRs close together?** Different PRs run in parallel in v2 (the Up group is per PR number). Safety comes from the manifest check (a PR re-plans if main changed what it planned) and from state locking on apply [infer]. I did not test two simultaneous merges.
- **Does it plan dependents?** No, only changed stacks; shared-file changes plan everything.
- **Can I still use the Manual Command?** Yes, unchanged; `/deploy` is a comment wrapper around it.
- **What replaces the Slack message?** Nothing yet; the commit status shows waiting for approval or deploying [code].
- **What about policy checks and `refresh_state: false`?** Not available in v2; repos that rely on them stay on legacy for now [code].
- **Do I need `infra:no_changes`?** No; it has no effect in v2.
- **Can the plan expire?** Yes after 7 days, then a re-plan and a new approval.
- **Is the reviewed plan really what is applied?** Yes when the code the plan depended on is unchanged at merge; otherwise a re-plan happens and you approve the new plan.
- **What happens to open PRs when a repo opts in?** They were planned by the old workflow and are planned again on merge, with a new approval.

---

## 7. Do not claim (not verified)

- That two simultaneous merges are safe in all cases.
- Anything about fork PR plans.
- That the skip marker or format commit has no side effects in every repo (#685 removed the marker; I have not re-tested).
- That approvals exist in every repo for every environment (settings differ per repo).
- Runner minutes saved (not measured).

## 8. Deck wording to double check

- Slide 13 answer "the environment approval before an apply is unchanged" holds for merge deployments; see the `/deploy` note above.
- Slide 8 says "the pipeline refuses to apply the old plan". Precisely, OpenTofu refuses it; the pipeline's job is to make Re-run all jobs re-plan.

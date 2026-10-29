# Storyline: Faster, safer infrastructure PRs

Engineering Show & Tell. Slot about 30 min including Q&A; talk about 20 min, Q&A 5, optional Miklos part 3 (taken from the buffer).

Audience: all of engineering. Many have never touched a `ts-*` repo, so every term is explained once and the demo carries the middle.

Demo approach: no live demo. Slides 5 to 7 use screenshots from the usage guide (COPS 4247486574). One real run is started in tm-testing at the start of slide 5 (comment `/deploy development` on a prepared PR) and is shown at the end of slide 7, so the audience sees a real run that was started during the talk.

Not in the talk: the earlier scoped-planning revert. The talk presents the v2 workflows and `/deploy` only.

Conventions: **Point** = the one thing the audience should take away. **Visual** = what is on screen. **Say** = speaker notes.

---

## 1. Title (0:30)

**Visual:** title, name, date.
**Say:** one sentence on who I am and that this is about how we deploy infrastructure.

## 2. Who this is for (1:00)

**Point:** this concerns infrastructure repos only, and nothing changes until a repo opts in.
**Visual:** two columns. Affected: `ts-*` repos (ts-general, ts-security, ts-ai, ...) and `/infra` folders, Terraspace on GitHub Actions. Not affected: application repos and their own CI/CD.
**Say:** if you merge a change that ends up as AWS resources through one of these repos, this is for you. Everything is opt-in per repo, one PR to opt in, revert it to go back.

## 3. Before: the cycle (1:30)

**Point:** reviewing and deploying were separate, repeated work.
**Visual:** flow diagram. PR opens, plan for every stack, review, merge, plan again, approve, apply. Side path: want to try it first? Actions tab, Manual Command workflow, pick branch, environments, stacks.
**Say:** the plan you reviewed is not the plan that gets applied, it is planned again after merge. A one-stack change still plans everything. Trying a change before merge means leaving the PR.

## 4. After: three changes (1:30)

**Point:** three improvements, shipped together (APPOPS-1258, 1259, 1260).
**Visual:** the same flow, with the three changes highlighted: plan only what changed; apply the reviewed plan on merge; `/deploy <env>` comment.
**Say:** keep it to one sentence each; the demo shows them. Mention auto-format (the bot pushes the `tofu fmt` commit) as a small extra, one line.

## 5. Demo 1: a PR plans what it changes (2:30)

**Point:** the plan covers only the stacks the PR touches, and the PR tells you how to deploy.
**Visual (screenshots):** plan comment with the changed stacks and the "Deploy to `<env>`" footer. Quick contrast: a docs-only PR plans nothing (about 1.5 min).
**Say:** show the stack list in the comment, then the footer. While this slide is up, start the real run in tm-testing in a second window (comment `/deploy development` on the prepared PR).

## 6. Demo 2: `/deploy` from the PR (3:00)

**Point:** try the change in development without leaving the PR.
**Visual (screenshots):** comment `/deploy development`; the bot reply with the stacks; the approval gate; the layered case where the bot names the waiting stacks; a rejected request.
**Say:** write access required, closed and fork PRs refused, a new push marks earlier replies as superseded. Same approval gate as before.

## 7. Demo 3: merge applies the reviewed plan (2:30)

**Point:** what you reviewed is what gets applied; no second plan.
**Visual (screenshots, then the real run):** merge; the Up run applies the saved plan through the gates; the run header links back to the PR (hover card). End by switching to the run started on slide 5 and showing where it is (waiting at the gate or applied).
**Say:** if main changed the same stacks, the plan expired, or the PR changed `.github/`, it plans again and asks for a new approval.

## 8. When it plans again, or goes stale (2:00)

**Point:** the pipeline refuses to apply a plan it no longer trusts, and the fix is one click.
**Visual:** "Saved plan is stale" failure, then Re-run all jobs: plans again, finishes green. Real example: ts-ai #286 (a manual change in staging before the merge).
**Say:** after a `/deploy`, that environment's saved plan is outdated at merge, so the merge run fails once and Re-run all jobs fixes it. Re-run failed jobs would apply the same stale plan again, so do not use it here.

## 9. What teams will notice (1:30)

**Point:** a short, honest list.
**Visual:** five bullets. Extra approval when main changed since the review. Stale plan after `/deploy` (slide 8). `git pull` after a bot format commit. Dependent stacks need a second `/deploy`. The `infra:no_changes` label has no effect: PRs that do not touch infrastructure plan nothing anyway.

## 10. How a repo opts in (1:00)

**Point:** one PR, easy to revert.
**Visual:** the ts-general opt-in PR (#370) as the template: v2 callers, the flag, `pr-deploy.yml`.
**Say:** the platform team opens these PRs; repo teams do not need to do anything.

## 11. Optional, Miklos: under the hood (about 3:00)

**Point:** it is tested like code.
**Visual:** plan reuse and re-plan rules; the test harness (script tests against a fake terraspace, workflow tests). If Miklos does not join, this is one slide or cut.

## 12. Rollout and where to follow it (1:00)

**Point:** it is live, and here is where to ask.
**Visual:** repos live today (state on the day), next up, Slack #proj-iac-pipeline-velocity-improvements, usage guide link.

## 13. Q&A (5:00)

Likely questions, with the answer ready:
- Can I opt my repo in? The platform team opens the PR; ask in the channel.
- Does it work for ts-dns? It stays on the legacy plan; `/deploy` works there too.
- What if I do not want to wait for the approval? The gate is unchanged; it is the same approval as before.
- What happens to open PRs when the repo opts in? They were planned by the old workflow and are planned again on merge, with a new approval.
- Is there a Slack message after deploy? Not in v2 (not scoped yet).

---

## Timing

Talk: 0:30 + 1:00 + 1:30 + 1:30 + 2:30 + 3:00 + 2:30 + 2:00 + 1:30 + 1:00 + 1:00 = 18:00. Q&A 5:00. Optional Miklos 3:00. Total with everything: 26:00. Cut slide 9 or 11 first if the dry run runs long.

## Demo prep (open)

- tm-testing PR with two or three stacks, one dependent on another (fixture stacks `appops1258_base`, `appops1258_dep` and friends exist), prepared before the talk with its plan already posted.
- The run started on slide 5 needs the environment approval; in tm-testing Danny approves it himself, so the run can finish during the talk. Approve it when slide 7 reaches the gate screenshot, so the audience sees the gate for real.
- Pre-pull the screenshots into `assets/` so the deck works offline.
- A backup: screenshots of a finished run in case the live run is slow or fails (one ES run took 9 minutes to start a container).
- Check runner start times on the day.
- Reference runs in tm-testing: PR #299 (deploy first, then merge) and #300 (label has no effect).

## Sources

- Usage guide with screenshots: COPS 4247486574
- Release plan and runbook: COPS 4317249537
- `/deploy` design and evidence: COPS 4184440900
- v2 verification (Miklos): COPS 4290281485

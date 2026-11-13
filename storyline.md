# Storyline: Faster, safer infrastructure PRs

Engineering Show & Tell. Slot about 30 min including Q&A; talk about 22 min including the Miklos part (4 min), Q&A 5.

Audience: all of engineering. Many have never touched a `ts-*` repo, so every term is explained once and the demo carries the middle.

Demo approach (11 Nov): no live run, probably. Slides 5 to 8 use screenshots (usage guide COPS 4247486574, plus the new plan comment footer on slide 5). The earlier plan to start a real tm-testing run on slide 5 and show it on slide 7 is dropped; the demo stacks (tm-testing draft PR #328) are no longer needed unless screenshots are recaptured.

Framing (11 Nov): merge and approve is the normal way to deploy; `/deploy` is for testing in development and emergencies, because deploying before merge makes the environment drift from main. The announcement said "deploy by comment", so slide 4 sets the frame.

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
**Say:** keep it to one sentence each; the next slides show them. Set the frame: the normal path is merge plus the approved deployment; `/deploy` is for testing and emergencies. Mention auto-format (the bot pushes the `tofu fmt` commit) as a small extra, one line.

## 5. Demo 1: a PR plans what it changes (2:30)

**Point:** the plan covers only the stacks the PR touches, and the PR tells you how to deploy.
**Visual (screenshots):** plan comment with the changed stacks and the footer "How to test before merging (exceptions only)", expanded. Quick contrast: a docs-only PR plans nothing (about 1.5 min).
**Say:** show the stack list in the comment, then the footer.

## 6. Demo 2: try it from the PR with `/deploy` (3:00)

**Point:** test a change in development without leaving the PR; shipping still goes through merge. Use sparingly: testing and emergencies.
**Visual (screenshots):** comment `/deploy development`; the bot reply with the stacks; the approval gate; the layered case where the bot names the waiting stacks; a rejected request.
**Say:** write access required, closed and fork PRs refused, a new push marks earlier replies as superseded. Same approval gate as before.

## 7. Demo 3: merge applies the reviewed plan, the normal path (2:30)

**Point:** what you reviewed is what gets applied; no second plan.
**Visual (screenshots):** merge; the Up run applies the saved plan through the gates; the run header links back to the PR (hover card).
**Say:** if main changed the same stacks, the plan expired, or the PR changed `.github/`, it plans again and asks for a new approval.

## 8. When it plans again, or goes stale (2:00)

**Point:** the pipeline refuses to apply a plan it no longer trusts, and the fix is one click.
**Visual:** "Saved plan is stale" failure, then Re-run all jobs: plans again, finishes green. Real example: ts-ai #286 (a manual change in staging before the merge).
**Say:** after a `/deploy`, that environment's saved plan is outdated at merge, so the merge run fails once and Re-run all jobs fixes it. Re-run failed jobs would apply the same stale plan again, so do not use it here.

## 9. What teams will notice (1:30)

**Point:** a short, honest list.
**Visual:** five bullets. Extra approval when main changed since the review. Stale plan after `/deploy` (slide 8). `git pull` after a bot format commit. Dependent stacks need a second `/deploy`. The `infra:no_changes` label has no effect: PRs that do not touch infrastructure plan nothing anyway.

## 10. How a repo opts in (0:45)

**Point:** AppOps opens and merges one PR per repo; nothing for repo teams to do. Rollback is easy (revert the PR). The exclude-stacks workflow is bundled for ExpSquads repos.
**Visual:** the ts-general opt-in PR (#370) as the template: v2 callers, the flag, `pr-deploy.yml`.
**Say:** keep it short. Technical details (callers, `pr-deploy.yml`, ts-general template) are in the speaker notes.

## 11. Miklos: edge cases and AI verification (about 4:00, three slides)

**Slide 10, the edge cases we found (1:30).** Point: "is the reviewed plan still safe to apply?" has many corners, and each was tested for real. Visual: ~90 end-to-end scenarios (distinct IDs on COPS 4290281485 across all rounds, including the 13 added for the last two fixes), plus 146 unit tests and 47 script tests in CI, 5 issues found and fixed; five example cards (dependent stacks, merge while the plan runs, same stack changed first, stack added to staging by another PR, saved plan expired).

**Slide 11, verifying a change too big to review (1:30).** Point: 5,655 new lines in 33 files (#669 +2,781, #670 +285, #671 +2,589) could hardly be reviewed meaningfully, so an AI agent (Claude Code) ran the test catalogue end to end. Visual: the change sizes and what the agent did; the guardrails (written test plan, sandbox only, read the exact plan before approving with an evidence comment, hooks that block unverifiable approvals and merges and any direct apply, people decide the irreversible).

**Slide 12, the AI verification in numbers (1:00).** Visual: the final run on 16 Oct: 53/53 tests passed, 66 test PRs (60 merged, 6 closed), 144 pipeline runs (74 PR plans, 70 deployments), 78 approval decisions (1 rejection on purpose), 5.5 h, 8 findings (1 low-severity code issue, 0 unsafe applies); 3 human decisions, 5 commands blocked by hooks. Source: tm-testing run history and the v2e retest report.

## 12. Rollout and where to follow it (1:00)

**Point:** it is live, and here is where to ask.
**Visual:** repos live today (state on the day), next up, Slack #proj-iac-pipeline-velocity-improvements, usage guide link.

## 13. Q&A (5:00)

Likely questions, with the answer ready:
- Should I deploy every change with `/deploy`? No. Merge and approve is the default; `/deploy` is for testing in development and emergencies.
- Can I opt my repo in? AppOps opens the PR; ask in the channel.
- What if I do not want to wait for the approval? The gate is unchanged; it is the same approval as before.
- What happens to open PRs when the repo opts in? They were planned by the old workflow and are planned again on merge, with a new approval.
- Is there a Slack message after deploy? Not in v2 (not scoped yet).

---

## Timing

Talk: 0:30 + 1:00 + 1:30 + 1:30 + 2:30 + 3:00 + 2:30 + 2:00 + 1:30 + 0:45 + 1:00 = 17:45. Q&A 5:00. Miklos 4:00. Total: 27:00. Cut slide 9 or 11 first if the dry run runs long.

## Demo prep (mostly obsolete since 11 Nov, no live run)

- tm-testing PR with two or three stacks, one dependent on another (fixture stacks `appops1258_base`, `appops1258_dep` and friends exist), prepared before the talk with its plan already posted.
- The run started on slide 5 needs the environment approval; in tm-testing Danny approves it himself, so the run can finish during the talk. Approve it when slide 7 reaches the gate screenshot, so the audience sees the gate for real.
- Pre-pull the screenshots into `assets/` so the deck works offline.
- Optional: one fresh screenshot of a finished merge run in the new format (slide 7), and recapture the `/deploy` replies if their wording changed.
- Check runner start times on the day.
- Reference runs in tm-testing: PR #299 (deploy first, then merge) and #300 (label has no effect).

## Sources

- Usage guide with screenshots: COPS 4247486574
- Release plan and runbook: COPS 4317249537
- `/deploy` design and evidence: COPS 4184440900
- v2 verification (Miklos): COPS 4290281485

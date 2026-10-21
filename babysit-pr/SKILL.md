---
name: babysit-pr
description: "Babysit a pull request until it reaches a terminal state: watch CI, review comments, and merge conflicts; fix what's confirmed and push back with reasoning on what isn't. Use when a PR is open and should be driven to merge-ready. Review findings are verified and triaged, never blindly applied."
---

# Babysit a PR

Persistently watch a pull request until a **terminal outcome**: it is merged, it is closed, the review-passed stand-down below is reached, or you have escalated to the user because something needs a human decision. Accept the PR as a number, URL, or auto-detect it from the current branch (`gh pr view --json number`).

Checks going green is **progress, not completion**: every push triggers a fresh review-bot round minutes to tens of minutes later, so don't stand down on green CI alone while a round may still be pending. The watch ends at the **review-passed stand-down**, when all of these hold simultaneously:

- the review bot has explicitly passed the **current head commit** (for Codex: the 👍 reaction; see "Knowing when the review bot has passed"), **or** its latest round on the current head produced only findings you declined with reasoning (see "Review feedback: triage, don't obey"): a round of declines is a pass, since pushing nothing means no new round is coming,
- CI is green (or the only failures are ones the user has already acknowledged as pre-existing),
- the PR is mergeable with no unresolved human change requests,
- every review thread has a reply, and threads you fixed or bot-declined are resolved (see "Resolving conversations").

At that point come off watch (`unwatch_pull_request`), report (see Reporting), and hand the merge decision to the user: passing review ends the babysitting, it does not authorize merging.

## Let automatic reviews start

Chase's repositories use automatic Codex PR reviews. After opening a PR or pushing a fix, wait for that automatic round; do not post `@codex review` to start babysitting. A manual request can cancel an already-starting or running review and restart it, wasting the work in progress.

An empty first check, no review in `gh pr view`, or a delayed 👀 reaction does not establish that automatic review is disabled. Check the reactions and pinned summary described below and allow time for those signals to appear. Only request a manual review when the user explicitly asks or you have confirmed automatic review is unavailable and no round is queued or running. If the expected automatic round never starts, follow the timeout/escalation guidance below instead of reflexively pinging the bot.

## The loop

A cycle runs when you take on the PR and each time T3 Code's watcher wakes you (see "Watching with T3 Code"):

1. **Gather state** in one call:
   `gh pr view <n> --json state,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,reviews`
   plus new inline comments since your last high-water mark:
   `gh api repos/{owner}/{repo}/pulls/<n>/comments --jq '.[] | select(.id > <last_seen_id>)'`
2. **Act** on what changed (sections below), in priority order: merge conflicts → CI failures → review feedback.
3. **Verify locally before pushing**: run the repository's full quality gate (for this repo: `bun run check` plus `node scripts/security-rag-regression-checks.mjs`). Never push a fix you haven't run the gate on.
4. **Push and reply**, then end your turn; the watcher stays on.

## Watching with T3 Code

Between cycles, wait on T3 Code's pull request watcher. Don't poll or run a Monitor or watcher of your own.

- **Start the watch** with `watch_pull_request` (the PR URL, or repository plus number) as soon as you take on the PR; it also links the PR to the thread. Comments that already exist won't wake you, so handle the current state in the same turn, then end it. One call covers every later push; you don't need to call it again after pushing.
- **What wakes you** (checked every two minutes): a check newly failing, the required checks all passing (once per head commit), a new or edited comment or review from anyone but you, or the branch starting to conflict with its base. Your own replies never wake you, and neither do reactions, so Codex's 👀 and 👍 are invisible to the watcher. Codex's pinned summary comment is edited in place when a round finishes, and that edit does wake you, though it can lag the 👍 by several minutes.
- **A wake is news, not a verdict.** The message names what changed and shows comments only as short snippets; run the full cycle before acting. "All required checks passed" is not the stand-down: after a push the review round for that head usually lands ~5–10 minutes later.
- **Re-arm a watch that stops on its own.** The watch ends when the PR merges or closes, when the thread settles or is archived, after 8 failed reads in a row, or after 10 comment-only wakes in a row. The last two send you a message; if you are still babysitting, check the PR yourself and call `watch_pull_request` again.
- **Come off watch with `unwatch_pull_request`** before you stand down or escalate, so the thread returns to the user's inbox, then report.
- **A delegated child task can't watch**: its parent thread owns the PR. Run one cycle, then return the PR URL, head SHA, and pending check and review state to the parent.

## Merge conflicts

`mergeStateStatus: DIRTY` means conflicts. Rebase or merge the base branch locally, resolve, run the full gate, push. If the resolution requires choosing between semantically conflicting changes from someone else's work, escalate rather than guess.

## CI failures

Read `statusCheckRollup` for failing checks; fetch logs (`gh run view <id> --log-failed`).

- **Deterministic failure caused by the PR** → fix it, gate, push.
- **Suspected flaky test** → you get a retry budget of **2 re-runs per check** (`gh run rerun <id> --failed`). If it fails a third time, treat it as real or escalate; never keep re-running to green.
- **Infrastructure failure** (runner outages, deploy-platform auth, quota) or a check that was already failing on the base branch → don't burn cycles; note it, verify it predates the PR (check the same job on the base branch or a recent merged PR), and escalate with that evidence.

## Review feedback: triage, don't obey

This is the part that must never run on autopilot. A reviewer (human or bot) states a claim about the code; your job is to **verify the claim against the actual code first**, then choose one of three outcomes. Findings from bots are hypotheses with a decent hit rate, not instructions.

**"Real" is not the same as "worth fixing."** An adversarial review bot reads the code as an attacker would and will find a scenario in which almost any line misbehaves. Most of those scenarios are technically valid and will never occur. Fixing every one of them is obeying with extra steps: it costs a full gate, a push, and another review round each time, and the bot will find the next adjacent corner in the code you just added. Before fixing, ask two questions:

- **Would a real input trigger it?** For a parser or importer: does any mainstream producer (Excel, Google Sheets, an API, a CRM export) emit this shape, or does it take a crafted file? For concurrency: does the interleaving arise from ordinary use (a retry, a closed tab, two users), or only from an adversary racing the system? For an API: can a caller reach this without already having the privilege it would gain?
- **What is the blast radius if it does?** Data corruption, data loss, publishing something the user did not intend, a crash or hang, or a privilege gain are always worth fixing, even for an unlikely input. A wrong value in the crafter's own crafted input, a cosmetic difference, or a strictly-more-conforming reading of a spec corner is not.

Fix when both answers say so, or when the blast radius alone does (fail-unsafe on any input is a fix). Decline the rest with the reasoning written out. A bug that is real, unreachable in practice, and harmless when reached is a legitimate decline, not a debt.

For each finding:

1. **Reproduce the reasoning.** Read the code path it names. Trace the scenario it claims. Decide: is the defect real, real-but-overstated, or wrong? Then apply the two questions above: reachable by real input, and what happens if so.
2. **Check the suggested fix independently of the diagnosis.** A correct diagnosis often ships with a harmful suggestion. Reject or replace the suggestion when it would:
   - add a security surface (e.g. persisting IDs that round-trip through client input into privileged calls),
   - restructure pre-existing design the PR didn't touch (scope creep),
   - trade a bounded, self-healing imperfection for standing complexity,
   - change semantics beyond what the defect requires.
3. **Pick an outcome and say so on the PR** (inline reply via `gh api repos/{owner}/{repo}/pulls/<n>/comments -f body=... -F in_reply_to=<comment_id>`):
   - **Fix**: implement (your way, not necessarily theirs), gate, push, reply with the commit hash and what you did, including where your fix differs from their suggestion and why.
   - **Partial**: implement the bounded portion, reply with what you adopted, what you didn't, and the reasoning.
   - **Decline**: reply with the concrete reasoning: why the scenario doesn't occur, or why the cure is worse than the disease. Declining with evidence is a fully legitimate outcome; silently ignoring a finding is not, and neither is implementing something you believe is wrong to make a bot quiet.
4. **Resolve the thread when it's settled** (see "Resolving conversations"). Reply first, then resolve; never resolve a thread you haven't responded to.

**No orphaned comments.** Every review thread gets at least a reply before you stand down or escalate, even if the reply is just "escalating this to the user because X". A thread with no response looks ignored, and ignored threads erode trust in the whole babysitting loop.

Severity tags (P1/P2/etc.) set your ordering and how much benefit of the doubt to extend, not whether verification happens or whether the fix is warranted: a P1 that needs a crafted input and corrupts only that input is still a decline, and a P2 that hangs the tab on a plausible file is still a fix. Batch one review round into one commit (`Address <reviewer> review: <summary>`) rather than one commit per nit.

**Watch for the endless loop.** Each round of the bot reviews the code the previous round's fixes added, so a PR can accumulate rounds indefinitely without converging. Warning signs: the findings have shifted from the PR's actual purpose to corners of a spec or of an adversary's imagination; each round's findings are "fresh evidence" about the previous round's fix; you are hand-writing infrastructure (a namespace resolver, a lock protocol, a parser) to satisfy the bot rather than the feature. When you notice this:

- **Say so to the user** at once, with the round count, the split of findings between "affects real use" and "spec corner", and a recommendation. Do not wait for the loop to end on its own; it will not.
- **Name the structural cause** if there is one. Ten rounds of findings against one component usually mean the component's approach is wrong for the job (regexes standing in for a parser, ad-hoc leases standing in for a transaction), and the honest options are to accept the residual risk, or to replace the approach in a separate change, not to keep patching.
- **Switch to declining** spec-corner findings by default from that point, with a one-paragraph reply each. Keep fixing anything that fails the blast-radius test.

As a rule of thumb, if a PR has taken more than five review rounds, or more than about twenty findings, stop and report before fixing the next batch. The user may prefer to merge, to scope down, or to change approach; those are their calls, and every additional round spends their time as well as yours.

## Resolving conversations

Thread resolution is GraphQL-only; the REST comments API can't do it. Map each REST comment id to its thread, then resolve:

```
# List threads with their first comment's databaseId (to match against REST ids)
gh api graphql -f query='query { repository(owner: "{owner}", name: "{repo}") {
  pullRequest(number: <n>) { reviewThreads(first: 100) {
    nodes { id isResolved isOutdated comments(first: 1) { nodes { databaseId path body } } } } } } }'

# Resolve one
gh api graphql -f query='mutation { resolveReviewThread(input: {threadId: "<id>"}) { thread { isResolved } } }'
```

When to resolve, after replying:

- **Fixed / Partial**: resolve once the fix commit is pushed and your reply cites it. If the reviewer is a bot that re-reviews on push, resolving immediately is fine; the next round will reopen anything you got wrong.
- **Declined, bot reviewer**: resolve after replying with your reasoning; no human is coming back to close it, and leaving it open makes the PR look unfinished. The reply is the record: say what input the finding needs, why no real producer supplies it, and what the outcome would be if one did, so a later reader (or the user) can disagree with evidence rather than re-derive it.
- **Declined, human reviewer**: reply but leave the thread **unresolved**: the human gets to decide whether your pushback settles it. Resolving a human's thread over their objection is how you get a change-requested review.
- **Escalated**: reply noting you've surfaced it to the user, leave unresolved.

Before the review-passed stand-down, sweep `reviewThreads` once more: every thread must have a reply from you (or predate your watch with an existing resolution), and every thread you fixed or bot-declined must be resolved.

**Escalate instead of deciding yourself** when a finding is correct but the fix requires a product decision, changes public API/schema in ways the user should own, or contradicts something the user explicitly asked for.

## Knowing when the review bot has passed

Never infer "review passed" from silence: distinguish *running*, *passed*, and *found issues* from explicit signals, and only treat a round as passed when it is stamped for the **current head commit** (`gh pr view <n> --json headRefOid`).

**Codex (`chatgpt-codex-connector[bot]`)** mirrors its dashboard state onto the PR in three signals:

1. **Running:** the bot adds a 👀 reaction to the PR while a review is in flight:
   `gh api repos/{owner}/{repo}/issues/<n>/reactions` (look for `content: "eyes"` by the bot).
2. **Findings:** the bot submits a PR review whose `commit_id` is the reviewed commit:
   `gh api repos/{owner}/{repo}/pulls/<n>/reviews --jq '.[] | select(.user.login == "chatgpt-codex-connector[bot]") | {commit_id, submitted_at}'`
   A review with `commit_id == head` means the round completed **with findings**; triage them.
3. **Passed:** the bot reacts 👍 (`content: "+1"`) on the PR once all reviews finish with **no findings**. It also maintains a pinned issue comment marked `<!-- codex-pull-request-review-summary -->` (edited in place; find it via `gh api repos/{owner}/{repo}/issues/<n>/comments`) whose table shows `✅ Completed` next to the short head sha; cross-check that the completed commit is the current head. The same comment carries a machine-readable `<!-- codex-review-state:v1 {...} -->` marker (`code_review.commit_sha`, `status`, `has_any_findings`); read that rather than parsing the table. The watcher wakes you when this comment is edited, never for the reactions.

Decision per cycle: summary shows the head sha completed (or 👍 present) **and** no bot review exists for the head commit → **round passed**; if CI is also green and the PR mergeable, this is the stand-down signal that ends the watch. Bot review exists for head → **findings**, triage. 👀 present or summary not yet updated for head → **still running**: end your turn, and the summary edit wakes you when the round finishes.

Two failure modes to watch: if ~40 minutes pass after a push with no 👀 and no summary update, assume no round is coming: check the issue comments for usage-limit notices (e.g. "You have reached your Codex usage limits"), and surface that to the user rather than waiting forever. And remember a 👍 describes the *current* head only; a new push starts the cycle over.

For other review bots, find their equivalent machine-readable markers (a commit-stamped review, an edited-in-place summary comment, a reaction) before trusting any inference; don't guess from timing.

## Escalation

Stop and surface to the user, with evidence gathered, options framed, and your recommendation, when you hit:

- permission/auth failures on `gh` or the CI platform,
- the same check failing after a real fix attempt plus the retry budget,
- CI infrastructure problems or base-branch breakage,
- a human reviewer requesting changes that conflict with the user's instructions,
- a review loop that is not converging (see "Watch for the endless loop"): more than five rounds, or findings that have drifted from the PR's purpose,
- merge itself (never merge without the user's explicit go-ahead unless they pre-authorized it),
- anything where two reasonable people would choose differently.

## Reporting

When you come off watch (terminal state or escalation), summarize: what rounds arrived, what you fixed (with commits), what you declined and why, current check/merge state, and what, if anything, needs the user. Group the fixes by whether they affected real use or closed a spec corner, so the user can see at a glance how much of the churn mattered. While on watch, stay quiet except when something actionable happened; "still green, still waiting" is not a report.

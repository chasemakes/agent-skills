---
name: babysit-pr
description: "Babysit a pull request until it reaches a terminal state: watch CI, review comments, and merge conflicts; fix what's confirmed and push back with reasoning on what isn't. Use when a PR is open and should be driven to merge-ready. Review findings are verified and triaged, never blindly applied."
---

# Babysit a PR

Persistently watch a pull request until a **terminal outcome**: it is merged, it is closed, the review-passed stand-down below is reached, or you have escalated to the user because something needs a human decision. Accept the PR as a number, URL, or auto-detect it from the current branch (`gh pr view --json number`).

Checks going green is **progress, not completion**: every push triggers a fresh review-bot round minutes to tens of minutes later, so don't stand down on green CI alone while a round may still be pending. The watch ends at the **review-passed stand-down**, when all of these hold simultaneously:

- the review bot has explicitly passed the **current head commit** (for Codex: the 👍 reaction; see "Knowing when the review bot has passed"),
- CI is green (or the only failures are ones the user has already acknowledged as pre-existing),
- the PR is mergeable with no unresolved human change requests,
- every review thread has a reply, and threads you fixed or bot-declined are resolved (see "Resolving conversations").

At that point come off watch, report (see Reporting), and hand the merge decision to the user: passing review ends the babysitting, it does not authorize merging.

## The loop

Each cycle:

1. **Gather state** in one call:
   `gh pr view <n> --json state,mergeable,mergeStateStatus,reviewDecision,statusCheckRollup,reviews`
   plus new inline comments since your last high-water mark:
   `gh api repos/{owner}/{repo}/pulls/<n>/comments --jq '.[] | select(.id > <last_seen_id>)'`
2. **Act** on what changed (sections below), in priority order: merge conflicts → CI failures → review feedback.
3. **Verify locally before pushing**: run the repository's full quality gate (for this repo: `bun run check` plus `node scripts/security-rag-regression-checks.mjs`). Never push a fix you haven't run the gate on.
4. **Push and reply**, then go back on watch.

Between cycles, don't tight-loop. Use a background Monitor (or scheduled wakeup) that polls every minute and emits an event only when something actionable appears: a new comment id, a check leaving `pending`, or `mergeStateStatus` changing. After a push, expect the next review-bot round in ~5–10 minutes; don't declare the PR clean before it has had the chance to land.

## Merge conflicts

`mergeStateStatus: DIRTY` means conflicts. Rebase or merge the base branch locally, resolve, run the full gate, push. If the resolution requires choosing between semantically conflicting changes from someone else's work, escalate rather than guess.

## CI failures

Read `statusCheckRollup` for failing checks; fetch logs (`gh run view <id> --log-failed`).

- **Deterministic failure caused by the PR** → fix it, gate, push.
- **Suspected flaky test** → you get a retry budget of **2 re-runs per check** (`gh run rerun <id> --failed`). If it fails a third time, treat it as real or escalate; never keep re-running to green.
- **Infrastructure failure** (runner outages, deploy-platform auth, quota) or a check that was already failing on the base branch → don't burn cycles; note it, verify it predates the PR (check the same job on the base branch or a recent merged PR), and escalate with that evidence.

## Review feedback: triage, don't obey

This is the part that must never run on autopilot. A reviewer (human or bot) states a claim about the code; your job is to **verify the claim against the actual code first**, then choose one of three outcomes. Findings from bots are hypotheses with a decent hit rate, not instructions.

For each finding:

1. **Reproduce the reasoning.** Read the code path it names. Trace the scenario it claims. Decide: is the defect real, real-but-overstated, or wrong?
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

Severity tags (P1/P2/etc.) set your ordering and how much benefit of the doubt to extend, not whether verification happens. Batch one review round into one commit (`Address <reviewer> review: <summary>`) rather than one commit per nit.

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
- **Declined, bot reviewer**: resolve after replying with your reasoning; no human is coming back to close it, and leaving it open makes the PR look unfinished.
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
3. **Passed:** the bot reacts 👍 (`content: "+1"`) on the PR once all reviews finish with **no findings**. It also maintains a pinned issue comment marked `<!-- codex-pull-request-review-summary -->` (edited in place; find it via `gh api repos/{owner}/{repo}/issues/<n>/comments`) whose table shows `✅ Completed` next to the short head sha; cross-check that the completed commit is the current head.

Decision per cycle: summary shows the head sha completed (or 👍 present) **and** no bot review exists for the head commit → **round passed**; if CI is also green and the PR mergeable, this is the stand-down signal that ends the watch. Bot review exists for head → **findings**, triage. 👀 present or summary not yet updated for head → **still running**, keep waiting.

Two failure modes to watch: if ~40 minutes pass after a push with no 👀 and no summary update, assume no round is coming: check the issue comments for usage-limit notices (e.g. "You have reached your Codex usage limits"), and surface that to the user rather than waiting forever. And remember a 👍 describes the *current* head only; a new push starts the cycle over.

For other review bots, find their equivalent machine-readable markers (a commit-stamped review, an edited-in-place summary comment, a reaction) before trusting any inference; don't guess from timing.

## Escalation

Stop and surface to the user, with evidence gathered, options framed, and your recommendation, when you hit:

- permission/auth failures on `gh` or the CI platform,
- the same check failing after a real fix attempt plus the retry budget,
- CI infrastructure problems or base-branch breakage,
- a human reviewer requesting changes that conflict with the user's instructions,
- merge itself (never merge without the user's explicit go-ahead unless they pre-authorized it),
- anything where two reasonable people would choose differently.

## Reporting

When you come off watch (terminal state or escalation), summarize: what rounds arrived, what you fixed (with commits), what you declined and why, current check/merge state, and what, if anything, needs the user. While on watch, stay quiet except when something actionable happened; "still green, still waiting" is not a report.

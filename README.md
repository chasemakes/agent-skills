# agent-skills

Personal agent skills, installable via [skills.sh](https://www.skills.sh) CLI:

```sh
npx skills add chasemakes/agent-skills -g
```

## Skills

- **babysit-pr** — Babysit a pull request until it reaches a terminal state: watch CI, review comments, and merge conflicts; fix what's confirmed and push back with reasoning on what isn't. Review-bot findings are verified and triaged, never blindly applied: a finding is fixed when a real input can reach it or its blast radius is severe, and declined with written reasoning otherwise. Flags a review loop that stops converging instead of feeding it. Waits on T3 Code's pull request watcher between rounds instead of polling. The watch stands down on the reviewer's explicit pass signal for the current head commit, or on a round of reasoned declines.
- **file-pr** — Say "file a PR" to open a short pull request. The title follows the repo's conventions and says why the change matters. The description opens with the problem and then the solution, not an inventory of changes, and ends with the model and harness that made the change. It opens a real PR rather than a draft so review bots run. From [t3dotgg/tstack](https://github.com/t3dotgg/tstack) (MIT).
- **full-send** — Say "full send" when you won't be watching the thread, and the agent takes the work from its current state to a merged PR on its own: finish the smallest change that works, prove every change with evidence it observed itself, file the PR, babysit it, merge once the merge gate passes (proof covers the diff, required checks green, review bots done, no outstanding change requests, branch up to date and mergeable), and clean up after the merge (the PR's branch, any worktree it created, and anything it started for proof). It never bypasses branch protection. It asks through the harness's question tool only when it is truly blocked, and otherwise decides with sensible defaults and lists those decisions in its final report.
- **leaving-pr-comments** — Applies whenever an agent comments on GitHub for you. Every comment starts with a note callout naming the model and who it speaks for, and the reply itself leads with the conclusion and cites the code, check, or reproduction behind it. Used by `file-pr`. From [t3dotgg/tstack](https://github.com/t3dotgg/tstack) (MIT).
- **sol-loop** — Say "sol loop" to have GPT Sol review your changes in a loop before (or after) filing a PR. One reviewer and a 3-round budget (5 for security, auth, data loss, or concurrency). A ledger tracks each finding, and fixes stay minimal: findings are verified, nits and scope creep are declined, and disputes go to the user. Runs Sol at high reasoning through T3 Code delegation or a native Sol subagent and never substitutes another model or effort.

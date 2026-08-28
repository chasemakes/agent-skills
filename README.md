# agent-skills

Personal agent skills, installable via [skills.sh](https://www.skills.sh) CLI:

```sh
npx skills add chasemakes/agent-skills -g
```

## Skills

- **babysit-pr** — Babysit a pull request until it reaches a terminal state: watch CI, review comments, and merge conflicts; fix what's confirmed and push back with reasoning on what isn't. Review-bot findings are verified and triaged, never blindly applied; the watch stands down on the reviewer's explicit pass signal for the current head commit.

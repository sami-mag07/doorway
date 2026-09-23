# Orchestration

The full plan is in `README.md`. This file holds the working rules for every Claude terminal.

## How to start a role
1. Open a Claude Code terminal in this repo.
2. Tell it: "You are <ROLE>. Read README.md, docs/ORCHESTRATION.md and docs/tasks/<ROLE>.md, then start."
3. The role works only inside the paths it owns (see the table in README.md).

## Communication
- Same Mac: message other terminals directly (SendMessage). ORCH is Sami's main terminal.
- Across Macs: write into `docs/board/<TARGET_ROLE>.md` (append, newest at the bottom, with time and your role), then commit and push.
- Before every new task: `git pull --rebase`. After every milestone: update `docs/board/STATUS.md`, commit, push.
- Contracts (`data/places/*.json`, `route.json`) change only through ORCH.

## Rules
1. Only touch your own paths.
2. Small commits, message starts with your role, e.g. `VIDEO: keyframe extraction`.
3. No secrets in code or git. Keys live in `.env` only.
4. Timebox beats perfection: if a milestone is not green by its deadline, ship the fallback.
5. Freeze is Sep 24, 10:00. After that only ORCH commits.

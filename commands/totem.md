---
description: Spin the totem — which levels are idle, in limbo, unverified, or colliding
---

Spin the totem on every running inception level. For each agent, by its level tag (`[L1 forger]`, `[L2 point-man]`), report:

- **State** — working, idle, done, or failed.
- **In limbo?** — if it went idle without reporting, message it and demand the report format (files changed, net lines, verification results, anything suspicious). Don't accept silence as success.
- **Totem fallen?** — an agent's "all green" is a dream until you re-verify it yourself. Flag any work not yet independently re-verified (typecheck + full tests) before it gets kicked into a commit.
- **Projections** — any two levels touching the same file, or an agent blaming a failure on "someone else" that was actually your own concurrent edit.

Finish with the next action: who to poll, whose work to verify-and-kick, what to re-spawn or fold inline.

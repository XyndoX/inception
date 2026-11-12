# Orchestration field guide

The operational detail behind the [README](../README.md). If the README is the
map, this is the manual for actually running a descent.

- [Choosing the level](#choosing-the-level)
- [The fenced brief](#the-fenced-brief)
- [herdr recipes](#herdr-recipes)
- [The totem protocol](#the-totem-protocol)
- [Big features: spec → plan → execute](#big-features-spec--plan--execute)
- [External-system discipline](#external-system-discipline)
- [Pitfalls seen in the wild](#pitfalls-seen-in-the-wild)

---

## Choosing the level

Decide the level by how much the *prompt* has to decide. Walk it top-down and
stop at the first "yes":

```
Does the task decide WHAT the goal is?          → L0  The Architect  (you, keep it — Fable-class)
Does it decide WHETHER as well as HOW?          → L1  The Forger     (strong model — Opus)
Is the outcome fully specifiable up front?      → L2  The Point Man  (fast model — Sonnet)
```

Fable is the scarcest tier: it stays topside and is never dealt as a level.
A level that misses the bar gets **re-leveled, not re-prompted** — the task
sat one level too deep or the brief was unclear. Move it up or fix the brief;
never repeat the same prompt louder.

Concrete calls from a real session:

| Task | Level | Why |
|------|-------|-----|
| Rank the audit findings by value-per-effort | L0 | Decides the objective and the order of battle. |
| Judge 31 lint warnings one by one, fix or suppress each | L1 | Every warning is a separate *whether*, not just a *how*. |
| Author a test suite for the most fragile hook | L1 | Requires taste about what's worth asserting. |
| Rename a symbol across 40 files; dedupe two shuffle impls | L2 | Outcome is pinned; only the mechanics remain. |
| Delete a dead speculative feature and its imports | L2 | Pure deletion, CI-verifiable, zero design risk. |
| Decide whether the cosmetics economy ships at all | L0 | That's a product call — it never leaves you. |

Miscasting is expensive both ways: a fast model fumbles judgment work, and a
strong model burns time and tokens on a mechanical rename. Match the level.

## The fenced brief

Every task you hand a level — sub-agent or herdr pane — carries **five**
things. Drop any of them and you get drift, projections, or a confident "done"
that isn't.

```md
## Task: <one-line goal>            (level: L1 forger | L2 point-man)

**Repo:** /abs/path/to/repo   (quote paths with spaces; note env gotchas —
  e.g. use `gtimeout` not `timeout`; zsh needs quoted "[bracket]" globs)

**Why:** <one paragraph of context>. A paragraph of intent beats a page of
  instructions — it lets the agent make the small calls correctly. Include
  what you already ruled out, and point at files instead of paraphrasing them.

**Do this:** <the actual work, specified to the level — exact steps for L2,
  goal + constraints for L1>.

**Fences — do NOT touch:**
  - src/useOnlineGame.ts  ← L1 forger owns this (writing its tests)
  - any file under migrations/  ← sequenced after you, not concurrent
  (You may ONLY write: <explicit allowlist>.)

**Verify before reporting:** run `<typecheck cmd>` and `<test cmd>`; both must
  pass. Do not report green on an unrun check.

**Report back:** open with your level tag · files changed · net lines ± ·
  verification results (each command + the tail of its output) · anything
  suspicious you noticed but didn't fix.
```

**Mind the gap.** The stronger the Architect, the wider the gap to the level
reading the brief. A level knows only what the brief says — what feels obvious
topside does not exist a level down. Write for the reader's tier (exact steps
for L2; goal, constraints and the bar for L1), and if you can't write the brief
clearly, the task isn't ready to leave L0. When you want something *tested* —
a review, a diagnosis — give the code and the symptom, not your theory: an
agent handed a conclusion returns it confirmed.

The fence is the load-bearing part. **No two concurrent levels may write the
same file.** If work overlaps, either sequence it (one after the other) or fence
one side out and give that slice to a single owner. Name the owning level in
the brief so the agent knows a locked file isn't a bug — it's someone else's
dream.

## herdr recipes

The dream levels: one pane per level, the model bound per pane with
`claude --model`. You're already inside a herdr session (the Architect is your
live pane), so you spawn the levels as named agents split off it — no separate
`attach` step, they appear right in your UI. Name every pane by its level so
the UI tells you who is where.

### Deal the levels

```bash
# The Architect (you) writes each brief into .inception/ first, then deals the
# levels as named agents. --split places each new pane; --model binds its tier.
herdr agent start l1-forger    --split right -- claude --model opus   "$(cat .inception/l1-forger.md)"
herdr agent start l2-point-man --split down  -- claude --model sonnet "$(cat .inception/l2-point-man.md)"

# Several levels of the same kind? Keep the level prefix and number them:
herdr agent start l2-point-man-2 --split down -- claude --model sonnet "$(cat .inception/l2-point-man-2.md)"
```

Target agents by the name you gave them — no pane-index bookkeeping.
`herdr agent focus l1-forger` jumps to one; `herdr agent send l1-forger "…"`
interjects.

### Interactive vs. headless

- `claude --model … "$(cat brief.md)"` — **interactive**. The pane stays live;
  you can interject, answer a question, or redirect mid-dream. Use when the
  work is exploratory or you want a hand on the wheel.
- `claude -p --model … "$(cat brief.md)"` — **headless print mode**. Runs to
  completion, prints the result, exits. Use for fire-and-forget mechanical work
  whose output you collect at the end. Redirect it:
  `… -p … > .inception/l2-point-man.out`.

### Watch, then collect

```bash
herdr agent read l2-point-man --source recent --lines 40   # snapshot a level's output
herdr agent wait l2-point-man --status idle                # block until it surfaces
herdr session stop default                                  # strike the set when done
```

`agent wait --status idle` is the herdr advantage over tmux: block until a
level actually finishes instead of polling `capture-pane` on a timer.

When every pane reports done, control returns to the Architect: you spin the
totem on each level's work (next section) and kick it yourself, one commit per
level.

## The totem protocol

A level's report is a dream, not a fact. Before **anything** an agent touched
enters a commit:

1. **Re-run the full gate yourself** — the project's typecheck *and* full test
   suite, not the subset the agent ran. Green one level down is not green
   topside.
2. **Grep-count multi-site edits.** After a `replace_all` or a cross-file
   rename, count the occurrences that should remain and confirm the number.
   Silent misses on differently-indented duplicates are the classic failure.
3. **Diff for scope creep.** Confirm the agent only wrote inside its allowlist.
   A file changed outside the fence is a projection or a misunderstanding —
   catch it before it compounds.
4. **Kick per level.** One level's work = one commit, message says what
   changed *and why*. Pull-rebase before every push if anything else writes the
   branch (CI bots, platform syncs, teammates).

Only after your own totem falls does the work exist. Committing on an agent's
say-so is how a bad edit reaches `main`.

## Big features: spec → plan → execute

Cleanups can go straight to fan-out. *Features* get a design gate first — code
before design is how scope explodes.

1. **Brainstorm** with the user: one question at a time, multiple-choice where
   you can. Decompose an oversized ask into shippable slices.
2. **Spec** the approved design into a committed doc. Flag every legal or
   irreversible decision explicitly — the user must own those, not you.
3. **Plan** with exact files, complete code per step, and a per-task
   *Interfaces: consumes / produces* block so tasks compose without a meeting.
4. **Execute** the plan — inline or fanned out, one commit per task, tests first
   where the logic warrants it.

## External-system discipline

When state lives outside the repo — a managed DB, a deploy platform, a
third-party API — the repo files are a hypothesis, not the truth.

- **Assume drift.** Verify what's actually live before reasoning from repo files.
  Write ownership facts into persistent memory the first time you learn them.
- **Hand-off prompts to external operators** (a platform chatbot applying SQL,
  say) must end with *"exactly as written, do not modify anything else"* and
  include the **verification queries** to run afterwards. A vague hand-off
  invites helpful corruption.
- **Long API jobs:** dry-run a small sample first, then pace *under* the
  documented rate limit. Steady beats hammer-and-backoff — the latter gets your
  IP flagged and is slower overall. Persist progress incrementally so a restart
  isn't a total loss.

## Pitfalls seen in the wild

- **`replace_all` skipping differently-indented duplicates.** Grep-count the
  result after any multi-site edit.
- **Duplicated code diverges.** Two toast systems, two shuffles, three
  Levenshteins — consolidation is bug-hunting, not hygiene. Test the survivor
  against a reference implementation.
- **Dynamic-import blind spots.** A static-import grep misses
  `await import('pkg')`. Search the bare package name before declaring a
  dependency dead.
- **Kicking before your own totem falls.** The whole point of the protocol
  above.
- **Sleeping on stale roadmaps.** Check items off the moment they ship; the next
  session inherits whatever the docs say.
- **Blaming a projection.** When an agent reports a failure "someone else
  caused," the usual suspect is your own concurrent edit. Check yourself first.

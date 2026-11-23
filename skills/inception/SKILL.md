---
name: inception
description: Use when the user asks to "orchestrate", "conduct", "go deeper", invoke "inception", or tackle a large multi-part coding task (audits, refactor sweeps, multi-feature sessions, "clean this up and fix everything") — anything too big for one linear pass. Runs the layered orchestration workflow: The Architect (Level 0, you) designs the levels and verifies, The Forger (Level 1, strong model) handles work needing judgment, The Point Man (Level 2, fast model) executes the fully specified — fanned out in parallel with fenced scopes, nothing committed until the Architect's own totem falls.
---

# Inception — Layered Multi-Agent Orchestration

One job, several levels. You are **The Architect — Level 0**: you design the
levels, you stay topside, you never go down yourself. Agents are the levels
below you. The core insight: match the level to task difficulty, fence each
level so projections stay calm, and never believe a report until your own
totem falls.

## The levels

| Level | Who | Gets |
|-------|-----|------|
| **L0 — The Architect** | You (the top model in the session — ideally Fable-class, e.g. Fable 5.2) | Planning, prioritization, judgment calls, integration, final verification, anything security- or money-adjacent |
| **L1 — The Forger** (`l1-forger`) | Strong model (e.g. Opus) | Refactors needing taste, test-suite authoring for complex code, case-by-case fixes (each lint warning judged individually) — work where the agent must decide *whether* |
| **L2 — The Point Man** (`l2-point-man`) | Fast model (e.g. Sonnet) | Well-specified transformations: migrations, renames, dedups, wiring changes with exact instructions — the outcome is already pinned |

**Default to Level 2.** Start every task at `l2-point-man` (Sonnet); raise it to
`l1-forger` only when the brief truly can't pin the outcome. Rule of thumb: if
the prompt can fully specify the outcome, it's Level 2 work.
If the agent must decide *whether* as well as *how*, it needs Level 1. If it
decides *what the goal is*, it stays at Level 0 — you.

With a Fable-class model topside the ladder is one tier per level:
Fable → Opus → Sonnet. Fable is the scarcest tier — it stays at L0 and is
never dealt as a level. If the Architect is itself Opus, L1 is a peer, not a
subordinate: brief it as one.

The deeper the level, the more happens per topside minute. That time dilation
is the whole point: several levels running at once, each on the cheapest
model that can hold the dream.

## The loop

### 1. Scope with an audit, not a guess
Before delegating anything, produce a **ranked findings list** yourself
(read the tree, measure, count usages). Rank by value-per-effort, biggest
cut first. Pure deletions rank above refactors — they're verifiable with
CI and carry zero design risk. Do trivial items (file deletions, one-liners)
inline; only real work earns a level.

### 2. Build the levels with fenced scopes
Spawn independent levels together so they run concurrently. Two modes:

- **herdr dream levels** (default) — one herdr pane per level, each its own
  `claude` process with the model bound per pane. Use this whenever
  `herdr status server` reports `running`. Recipe:
  1. Write each brief to `.inception/<name>.md` (add `.inception/` to
     `.git/info/exclude`). A herdr pane only sees its brief, not the agent files, so every
     brief ends with this budget block:
     > Budget: past ~150 turns or ~300k context, stop. Write what's done and
     > what's left to `.inception/<name>.report.md` and end, unless the
     > Architect has extended your budget. Your LAST action is always writing
     > that report.
  2. Deal the level, Sonnet by default:
     `herdr agent start l2-point-man --cwd "<repo>" --split down --no-focus -- claude --model sonnet "$(cat .inception/l2-point-man.md)"`
     Use `--model opus` only for an `l1-forger` pane. Never `--model fable`.
     Add `--advisor opus` to Sonnet panes so a point-man gets an Opus second
     opinion at decision points (skip it if the user's settings already set
     `advisorModel`).
     Number repeats: `l2-point-man-2`, `l1-forger-2`.
  3. Herd: `herdr agent wait <name> --status idle --timeout 1800000`, then read
     `.inception/<name>.report.md`. The report file is the completion signal;
     pane status alone can read idle too early. Status `blocked` means the pane
     waits on a prompt (a permission, or "trust this folder?" for a folder Claude
     hasn't opened before): tell the user which pane.
  4. A report that says "budget reached" is a decision for you, not a
     failure. Usually deal a fresh pane with a brief for what's left. If the
     context the worker built up is worth keeping, extend it instead:
     `herdr agent send <name> "Budget extended to ~<N> turns. Continue, same report rules."`
     then press Enter in that pane (`herdr pane send-keys <pane_id> Enter`).
     Same for sub-agents: SendMessage the extension.
  5. Strike the pane once its final report is in: `herdr pane close <pane_id>`
     (id from `herdr agent get <name>`). One pane = one task; deal a new pane
     for the next task instead of sending more work to an old one.
  6. **Other quotas.** If `ext-worker` is on PATH, a level can run outside the
     Claude quota with the same brief/report files:
     `ext-worker <lane> <name>` (brief `.inception/<name>.md`). Lanes:
     `agy-sonnet` / `agy-opus` (Claude on the Google Pro quota, edits in the
     worktree `.inception/wt/<name>`, branch `ext/<name>`; you merge),
     `gemini` (screenshots, video, huge context, second-family review; no
     edits), `hermes` (free NVIDIA model: logs, summaries, drafts; spot-check
     its numbers), `codex` (scarce: one independent review of a risky diff).
     Herd them by the report file, not pane status; close the pane after.
     Prefer `agy-sonnet` for an L2 level when the Claude quota runs tight.
  Full recipes: `docs/orchestration.md`.
- **In-session sub-agents** (fallback when herdr isn't running) — the typed
  agents `l2-point-man` (Sonnet) and `l1-forger` (Opus) with the model
  pre-bound by the agent definition.

Every brief must contain:

- **Repo path + environment gotchas** (quote paths with spaces, etc.)
- **The task with the *why*** — one paragraph of context beats none
- **Explicit fences**: files it must NOT touch, and *which level owns them*
  ("do NOT touch useOnlineGame.ts — L1 is writing tests for it")
- **Verification commands it must run** before reporting (typecheck, tests)
- **A required report format**: level tag, files changed, net lines,
  verification results (the command and the tail of its output, not
  "passes"), anything suspicious found
- **Style constraints** (shortest working diff, match existing code, no new
  abstractions)

**Mind the gap.** The stronger the Architect, the wider the gap to the level
reading the brief — and the more of what you know is missing from it. A level
knows only what the brief says; what feels obvious topside does not exist a
level down.

- **Write for the reader's tier.** L2 gets exact steps and a pinned outcome;
  L1 gets the goal, the constraints, and the bar. If you can't write the
  brief clearly, you don't understand the task well enough to delegate it —
  it stays at L0.
- **Say what you already ruled out**, so the level doesn't re-walk it. Point
  at files and docs instead of paraphrasing them — your paraphrase drops
  detail.
- **Don't hand over your hypothesis when you want it tested.** An agent given
  a conclusion returns it confirmed. For reviews and diagnoses, give the code
  and the symptom, not your theory.
- **Demand evidence, not adjectives.** Wrong reports arrive in the same
  confident register as right ones; quoted command output is the only part
  you can check.
- **Re-level, don't re-prompt.** A level that misses the bar means the task
  sat one level too deep or the brief was unclear. Move it up a level or fix
  the brief — never repeat the same prompt louder.

**Token budget.** Every agent call re-reads the agent's whole context, so a
long-lived level costs more with every turn it lives (measured: forgers kept
running for ~200 turns grew to 400–760k context and cost ~46M tokens each).
- One level = one task. Spawn a fresh level for the next task; never
  send new work to a level that has already reported. Extending a level's
  budget for the SAME task is allowed when you judge it cheaper than a
  fresh brief.
- Brief with file paths, not pasted file content.
- At most 4 levels per wave.
- Fable never runs as a level. A Fable worker the user explicitly asks for
  needs their manual confirmation.

Never give two concurrent levels write access to the same file. Shared files
are how projections turn on you. If work overlaps, sequence it or fence one
side out.

### 3. Keep every level out of limbo
- Agent goes idle without a report → **message it and demand the report**.
  Silence is limbo, not success.
- Agent spawn fails → retry once; if it fails again, do that task inline
  yourself rather than blocking.
- An agent reports a failure "someone else caused" → **check whether it was
  you.** (Concurrent edits by the Architect are the usual suspect.)
- **Messages cross between levels.** Async agents compose replies before
  reading your latest directive. When you supersede a spec mid-flight, the
  new message must name the old spec DEAD in one unmissable line and demand
  the agent repeat the binding spec back before it dispatches further work.
  Watch the next status for stale wording — it means the correction never
  landed.

### 4. Spin the totem, then kick
An agent's "all green" is a dream until your totem falls. Before committing
its work, re-run the full verification yourself (typecheck + full test suite
minimum, plus the project's **domain gates** — benchmarks, golden hashes,
visual checks — not just the generic suite; a level's green tests can coexist
with a wrecked benchmark). Two totems that lie:

- **Verify the path you're shipping.** Flipping a default/config/flag means
  the gate must exercise the NEW path. A benchmark that still runs the old
  path is vacuously green. (Real case: a default-ON flip whose bench had only
  ever measured the OFF path — the ON path hid a 74× force regression.)
- **Read the artifact.** When the deliverable is a generated artifact (PDF,
  image, bundle, export), open and inspect the artifact itself — and check
  its SIZE. (Real case: a "verified" PDF was 55MB because nobody looked at
  the number next to the filename.)

The kick is the commit: one level's verified work = one commit, with a
message that says what changed *and why*. Pull-rebase before every push if
anything else writes to the branch (CI bots, platform syncs, teammates).

### 5. Wake up clean
- Restate agent findings that matter (bugs discovered, behavior changes,
  deliberate limitations) in your summary to the user — the user never saw
  what happened a level down.
- Log the session (what shipped, what was learned, open TODOs) somewhere
  durable before context runs out.
- Update stale project docs (roadmaps, checklists) the moment you notice
  they lie — future sessions inherit them.

## Big features: spec → plan → execute

For feature work (not cleanups), insert a design gate before any code:

1. **Brainstorm** with the user — one question at a time, multiple-choice
   where possible; decompose oversized asks into shippable slices.
2. **Spec** the approved design to a committed doc. Flag legal/irreversible
   decisions explicitly (the user must own those).
3. **Plan** with exact files, complete code in steps, and per-task
   "Interfaces: consumes/produces" blocks so tasks compose.
4. **Execute** the plan inline or via levels, one commit per task, tests first
   where logic warrants it.

## Data & spend safety (iron rules for every brief)

Two rules that must appear in any brief whose work touches live-ish systems.
Both were learned the expensive way:

- **Levels never mutate real data.** Reproducing a bug or verifying a fix
  against real user-owned rows/files/sets is forbidden — the executor CREATES a
  disposable fixture (clone a record under a throwaway name, seed a test
  entity), operates on that, and deletes it after. A brief that says
  "reproduce first" without naming a disposable fixture is a loaded gun.
  (Real case: a diagnosis phase re-ran a destructive operation across 12 real
  production records; the overwrites were irreversible.)
- **Every external call is PAID until proven free.** Provider selection often
  hides in env vars — the "mock" provider you assume is active may be the
  real, billed API. Verify which provider the environment resolves before any
  agent triggers generation/inference/sending. (Real case: "free mock" QA
  burned real API credits because a key in .env.local silently selected the
  paid provider.)

Corollary: a **destructive user-facing operation** (overwrite-in-place,
regenerate, migrate) should get its undo/backup designed BEFORE agents are
allowed to exercise it. If one click can destroy state, an agent will
eventually click it.

## External-system discipline

When a system outside the repo owns state (a managed DB, a deploy platform,
a third-party API):

- **Assume drift.** Verify what's actually live before reasoning from repo
  files. Write down ownership facts in persistent memory the first time you
  learn them.
- Hand-off prompts to external operators (e.g. a platform chatbot applying
  SQL) must say **"exactly as written, do not modify anything else"** and
  include **verification queries** to run afterwards. Vague prompts invite
  helpful corruption.
- Long API jobs: **dry-run a small sample first**, then pace *under* the
  documented rate limit — steady beats hammer-and-backoff, which gets your
  IP flagged and is slower overall. Persist progress incrementally so a
  restart isn't a total loss.

## Pitfalls observed in the wild (avoid these)

- `replace_all` edits silently missing differently-indented duplicates —
  **grep-count the result** after multi-site edits.
- Two toast systems / two shuffle implementations / three Levenshteins:
  duplicated code *diverges*; consolidation is bug-hunting, not just hygiene.
  Test the survivor against a reference implementation.
- Static-import grep missing dynamic `await import('pkg')` consumers —
  search for the bare package name before declaring a dependency dead.
- Committing a level's work before your own totem has fallen.
- Sleeping on stale roadmaps: check items off the moment they ship.
- Unanchored patterns in ignore files (`.vercelignore`, `.dockerignore`)
  matching nested dirs — a bare `supabase` also excludes `lib/supabase/` and
  breaks the build. Anchor to root (`/supabase`).
- Retrying a stuck upload/deploy without killing the previous attempt —
  orphaned processes compete for bandwidth and every retry gets slower.
  `pkill` first, then retry once, clean.
- Test timeouts tuned on your machine flaking on CI's 2-core runners — a
  timeout is a hang guard, not a perf gate; size it generously (30s local
  work can take 35s+ on CI).
- Users watch the hot-reloading dev server while levels edit — mid-flight
  WIP looks like shipped garbage. Warn the user, or fix the visible page
  first.
- Wiring a credential into automation without testing it in its exact
  consumer — a token that authenticates raw API calls can still be rejected
  by a CLI's `--token`. If you flag a risk in your own work, that's a test
  you owe immediately, not a disclaimer.
- Secret-setting commands that normally prompt (`gh secret set`) go
  silent-EMPTY under a non-interactive shell and exit 0. Verify a secret took
  effect by exercising it, never by exit code.
- Fixing from an unverified hypothesis. Instrument first: tally the actual
  failure classes before touching code. (Real case: a root-cause hypothesis
  that survived code review died on the first empirical tally — the true
  cause was elsewhere.)

## Depth

Scale the depth to the task. A one-file fix needs zero levels. A
two-workstream task needs two levels and no ceremony. Only a genuine
multi-front session (audit + features + infra) earns the full loop.
Every level is overhead you must earn back in time dilation.

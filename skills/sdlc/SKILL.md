---
name: sdlc
description: Runs a full software development lifecycle in the current project — architect breaks work into tasks, assigns each to the sdlc-developer agent, pipes it through review, then builder/tester/builder-teardown, then a committer agent commits it. Use when the user wants to plan a feature/bugfix as tracked tasks, run the dev/review/test pipeline, or check the task board.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
  - Agent
---

# /sdlc — Architect-led development pipeline

Arguments passed: `$ARGUMENTS`

While this skill is active, **you are the architect**: a high-level
technical lead who breaks work into well-specified tasks, assigns them,
and gates them through review and test before calling anything done. You
do not write feature code yourself — that's the `sdlc-developer` agent's
job, for every task regardless of size. You design task breakdowns, judge
review/test verdicts, and route work between stages.

Dispatch on the first word of `$ARGUMENTS`:

- `help` — print the pipeline summary below (folders, stages, complexity
  rule, commands) so the user can re-orient without reading this file
- `init` — bootstrap the task board (see below)
- `plan <description>` — turn a request into task files
- `run` — advance tasks through the pipeline
- `status` — show the board
- no args / anything else — figure out intent from context; if the user
  just described a piece of work in plain language, treat it as `plan`.

### `help` output

Print concisely:

```
Pipeline: tasks/active -> tasks/review -> tasks/test -> tasks/done -> commit
  active  : pending work + any task kicked back for rework
  review  : sdlc-reviewer checks correctness/simplicity/spec match; complexity:
            standard tasks also get a second, independent pass from a plain
            haiku-model subagent before continuing
  test    : builder stands up the env -> sdlc-tester verifies behavior
            against it -> builder tears it back down
  done    : shipped, then the committer agent stages + commits it

Every task, regardless of complexity, is implemented by the sdlc-developer
agent — complexity is still recorded (it shapes how detailed the task spec
should be at `plan` time, and gates the second-review rule below). Model
per stage:
  developer / reviewer     : sonnet for complexity:standard, haiku otherwise
  builder / tester / committer : always haiku, regardless of complexity

Task types: bug | feature | test (verification-only tasks — see
templates/test-task.md). Both complexity:simple and complexity:standard
tasks are coded by sdlc-developer itself; complexity:standard tasks are
additionally reviewed twice (sdlc-reviewer + a second independent
haiku-model pass).

Commands: /sdlc init | plan "<description>" | run | status | help

Only the architect (this chat) moves files between folders or edits
frontmatter. Sub-agents only fill in their own body sections (`builder`
and `committer` are the exceptions — infra lifecycle and the commit itself
are their jobs).
```

All state lives in `tasks/` at the project root (find it with
`git rev-parse --show-toplevel`, fall back to cwd if not a git repo). This
directory and its contents are the only source of truth for task state —
don't track task status anywhere else.

---

## `init` — bootstrap

Create, if missing:

```
tasks/active/    — pending tasks, and tasks currently in development or rework
tasks/review/    — tasks awaiting/undergoing code review
tasks/test/      — tasks that passed review, awaiting/undergoing testing
tasks/done/      — completed tasks
```

Nothing else to do. Safe to run even if the folders already exist.

---

## `plan <description>` — break work into tasks

1. Ensure `tasks/{active,review,test,done}` exist (run `init` logic if not).
2. Think through the request like an architect: what are the discrete units
   of work? Split along natural seams (one coherent change per task, each
   independently reviewable and testable) — don't create one giant task,
   and don't fragment trivially related changes into ten tiny ones.
3. For each unit, decide:
   - **Type**: `bug`, `feature`, or `test`. Use the matching template from
     `~/.claude/skills/sdlc/templates/{bug,feature,test}-task.md`. Use
     `test` for verification-only work (auditing/exercising existing
     behavior, no new feature or known defect) — its Acceptance Criteria
     require any defect found to be filed as its own `BUG-XXX` task rather
     than fixed inline.
   - **Complexity**: `simple` or `standard`. Both are implemented by the
     same `sdlc-developer` agent (see `run` below) — this field no longer
     picks an engine, but still matters for how much you spell out at plan
     time: `simple` (a localized, well-defined change — one or two files,
     no architectural decisions, low blast radius) can get a terser spec,
     while `standard` (touches multiple files/modules, needs a design
     decision, touches shared/core logic, or carries real risk) warrants a
     fuller Requirements/Proposed-Solution-guidance writeup up front so the
     agent isn't guessing at scope.
4. Assign the next ID: `ls tasks/*/*.md 2>/dev/null | grep -oE '(BUG|FEAT|TEST)-[0-9]+'`,
   take the max per prefix, increment, zero-pad to 3 digits (`BUG-001`,
   `FEAT-001`, `TEST-001`).
5. Copy the template, fill in `id`, `type`, `title`, `complexity`,
   `created` (today's date), and as much of the body as you can specify
   up front (Summary, Steps to Reproduce / Requirements, Acceptance
   Criteria, Test Plan, Out of Scope for features). Leave Root
   Cause/Proposed Solution/Implementation Notes/Review Notes/Test Notes
   for the agents that own those stages — don't pre-fill them yourself.
6. Write the file into `tasks/active/<ID>-<slug>.md`, status `pending`.
7. After creating all tasks, show the user a short table: ID, type,
   complexity, title. Ask whether to start `run` now or stop here.

---

## `run` — advance the pipeline

**Process one task at a time, start to finish, before picking up the
next.** The stages are sequential and each depends on the previous one's
output — don't parallelize across tasks or stages.

Pick the oldest task file in `tasks/active/` with status `pending` or
`changes-requested`. If none, report the board is clear (or fully in
review/test — see `status`) and stop.

### 1. Assign & develop

- Set status to `in-development`, append a history entry, save.
- Invoke the `sdlc-developer` agent (Agent tool, `run_in_background: false`
  so you block on it) with the full task file contents and its path as the
  prompt — every task goes through this same agent regardless of
  `complexity`. Include, if this is a re-pass, the latest Review/Test Notes
  explaining what needs fixing.
- **Model**: pass `model: "sonnet"` in the Agent tool call when the task's
  `complexity: standard` (it needs real design judgment); omit the
  override for `complexity: simple` so it falls back to the agent
  definition's default (`haiku`) — a localized, well-defined change doesn't
  need the more expensive model.
- If the agent reports a blocker (missing information, contradictory
  requirements, a prerequisite that doesn't exist), don't route it to
  review — treat it per the Loop section below.
- After it reports back, read the task file to confirm `Root
  Cause`/`Proposed Solution` and `Implementation Notes` were actually
  filled in and the described files actually changed (`git status`/`git
  diff`) before moving on — don't take a clean summary on faith.
- Set `assignee: sdlc-developer` in the frontmatter.

### 2. Review

- Move the task file from `tasks/active/` to `tasks/review/`, set status
  `in-review`, append history, save.
- Invoke the `sdlc-reviewer` agent (foreground) with the task file contents
  and path. Same model rule as development: pass `model: "sonnet"` for
  `complexity: standard`, omit the override (defaults to `haiku`) for
  `complexity: simple` — reviewing a real design decision needs more
  judgment than reviewing a narrow, well-defined change.
- Read its verdict from the `## Review Notes` section it appended.
  - **CHANGES_REQUESTED** → move the file back to `tasks/active/`, set
    status `changes-requested`, append a history entry summarizing the
    ask, and go back to step 1 (re-develop) for this same task.
  - **PASS**, `complexity: simple` → continue to Test.
  - **PASS**, `complexity: standard` → don't continue to Test yet — run
    the second review pass below first.

### 2b. Second review: independent haiku pass (standard tasks only)

Standard-complexity tasks get a genuinely independent second opinion before
they reach test — a fresh subagent with no memory of the first review, not
another pass of the same `sdlc-reviewer` agent. Simple tasks skip this; one
review is enough for a narrow, well-defined change.

- Invoke the `general-purpose` agent (Agent tool, `run_in_background: false`
  so it blocks the pipeline, `model: "haiku"` so it's a cheap, genuinely
  separate pass) with read-only review tools in mind — it has edit
  capability by default, so the prompt must constrain it (see below).
- The prompt should give it the same brief `sdlc-reviewer` got: the full
  task file contents *including the sdlc-reviewer's own Review Notes* (so
  it isn't re-deriving ground already covered — tell it explicitly what
  the first pass already checked), the same rubric (correctness,
  simplicity/YAGNI, consistency with repo conventions, duplication/shared
  abstraction, acceptance criteria walked item by item), and an explicit
  ask to end its answer with a clear `Verdict: PASS` or `Verdict:
  CHANGES_REQUESTED` line plus specifics for any issue found.
- **The prompt must explicitly tell it not to edit, fix, or "improve" any
  file — read and report only.** State it as a hard constraint (e.g. "You
  are reviewing only. Do not edit, create, or modify any file for any
  reason — if you believe a change is warranted, describe it in your
  response instead of making it").
- **After it returns, check `git status`/`git diff` against what it was
  before this call** (compare against the diff snapshot from the
  `sdlc-reviewer` pass) to confirm it didn't modify anything despite the
  instruction. If it did, don't silently keep or silently discard the
  edit — note it explicitly in the task's history (what changed, whether
  you're keeping it because it's genuinely harmless/beneficial or reverting
  it), since this is exactly the kind of "reviewer behaved unexpectedly"
  case that should be visible, not quietly absorbed.
- This subagent doesn't edit the task file — you (the architect) append its
  response as a new dated entry under the existing `## Review Notes`
  section yourself, labeled `### second review pass`.
- Route on the combined result:
  - Both PASS → continue to Test.
  - Either CHANGES_REQUESTED → move the file back to `tasks/active/`, set
    status `changes-requested`, append a history entry naming which
    review(s) asked for what, and go back to step 1.

### 3. Test

Three agents now, in sequence — this stage is split so the tester's whole
budget goes to *verification*, not to infrastructure it re-derives every
time.

- Move the task file from `tasks/review/` to `tasks/test/`, set status
  `in-test`, append history, save.
- **Build**: invoke the `builder` agent (Agent tool, `run_in_background:
  false`) with the task file contents and path. Keep track of its
  `agentId` — you'll resume this exact agent for teardown later, since it
  needs to remember what it stood up. Read its report: what's running, how
  to reach it, anything already set up.
- **Test**: invoke the `sdlc-tester` agent (foreground) with the task file
  contents and path, *plus the builder's environment report* verbatim so
  it doesn't waste effort re-deriving what's already running or how to
  reach it. In the prompt, also tell it what the reviewer already
  confirmed statically (build/vet/typecheck passing, specific files
  traced) so it doesn't redundantly redo that either — its whole budget
  should go to the one thing neither builder nor reviewer did: watching
  the system actually behave correctly.
- **Model**: both `builder` and `sdlc-tester`'s jobs (stand it up / hit it
  and observe) don't scale in difficulty with `complexity` the way design
  judgment does — always leave both on their agent definition's default
  (`haiku`), regardless of the task's `complexity`. Don't override either
  to `sonnet` even for `standard` tasks.
- Read the tester's verdict from the `## Test Notes` section it appended.
- **Teardown**: regardless of PASS or FAIL, resume the `builder` agent
  (SendMessage to its `agentId` from the build step) and ask it to tear
  down everything it stood up. Read its report to confirm cleanup actually
  happened before moving on — don't skip this because the task already
  has its verdict.
  - **PASS** → move the file to `tasks/done/`, set status `done`, append
    history. Invoke the `committer` agent (see "Commit on done" below) to
    stage and commit the task. Report the task as complete to the user
    with a one-line summary.
  - **FAIL** → move the file back to `tasks/active/`, set status
    `changes-requested`, append a history entry summarizing what failed,
    and go back to step 1 for this same task.

### Commit on done

Once a task's file is in `tasks/done/`, invoke the `committer` agent
(Agent tool, `run_in_background: false`) with the task file's full
contents and its path — it handles staging and committing on its own from
just that (cross-referencing `git status`/`git diff` against the task's
`Affected Areas`/`Implementation Notes`, applying the repo's established
default of including a whole file rather than attempting risky partial-hunk
surgery when a file's changes are inseparably entangled with unrelated WIP,
and verifying the resulting `HEAD` still builds). This is deliberately not
something the architect does inline anymore — it's mechanical, doesn't need
the architect's full context, and always runs on `committer`'s cheaper
default model regardless of the task's `complexity` (same rule as the
tester: never override it to `sonnet`).

Read the committer's report before moving on:
- If it committed cleanly, done — the report already names what rode along
  on the entanglement default, if anything; no need to re-verify yourself.
- If it flagged something it couldn't safely resolve (conflicting changes
  to a file, or a build failure after its commit), that's a real blocker —
  investigate yourself or escalate to the user rather than re-running it
  and hoping for a different answer.

### Loop

After finishing a task (done, or a hard blocker reported by an agent that
you can't resolve by routing), pick the next eligible task in
`tasks/active/` and repeat. Stop and summarize when the active queue is
empty, or immediately if any agent reports a blocker that needs the user's
input (missing information, contradictory requirements, infra you can't
access) — don't guess past a real blocker.

---

## `status` — show the board

List each task file across all four folders with: ID, type, complexity,
title, status. Group by folder. Flag anything that's been in
`changes-requested` more than twice (same task bounced back repeatedly) —
that's a signal the task itself is underspecified, worth surfacing to the
user rather than silently re-looping.

---

## Notes

- You (the architect) are the only one who moves files between folders and
  edits frontmatter. Sub-agents only edit the body sections they own (the
  `committer` agent is the sole exception to "sub-agents don't touch git
  state" — its whole job is the commit, not the task-file routing). This
  keeps routing logic in one place and prevents a bad agent run from
  corrupting task state.
- **Before any `mv` between task folders, `mkdir -p
  tasks/{active,review,test,done}` first.** Git doesn't track empty
  directories, so `tasks/review/`/`tasks/test/` can silently vanish (e.g.
  after a `git stash`/`pop` cycle elsewhere in the session) and break a
  bare `mv` with "No such file or directory" — cheap to guard against,
  annoying to hit mid-pipeline.
- Every stage transition gets a history entry:
  `{date, stage, by, note}` — keep it terse (one line).
- **When something in this pipeline fails or behaves unexpectedly** (an
  agent invocation errors, a tool has a sharp edge, a skill's instructions
  turn out to be wrong or produce a bad result), don't just work around it
  in the moment and move on — fix the actual global config responsible
  (this file, the relevant agent's `.md` in `~/.claude/agents/`, or
  whatever skill/tool config is at fault) so the same failure doesn't
  recur in a future `/sdlc run` or a different project. A one-off inline
  workaround only helps this run; the fix belongs in the source the next
  run reads from.

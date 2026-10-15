# Execution patterns — surviving interruption

AFK work runs in environments that interrupt it: hosted VMs are replaced
without notice, desktop sessions close, processes are killed, context
windows compact. The patterns below are how agent runtimes — hosted
assistants, coding agents, cloud software-engineering agents, durable
workflow frameworks — keep long work alive across those interruptions.
They are environment-agnostic; the notes after the catalog map them to
specific runtimes. SKILL.md §5 states the load-bearing rules; this file is
the catalog behind them.

## Make progress durable

**Checkpoint + resume marker.** The job writes a progress file as it goes:
items done, items remaining, and a marker naming the job and where it
stopped. Any later instance checks for the marker before starting related
work and resumes, skipping completed items. *Defeats: total loss on
restart; duplicated work on resume.*

**The durable medium is the state.** State kept in a repo (commits, the
branch, the PR body), in a service's API, or in files outlives the
session; state in process memory or in the context window does not.
Commit early and often during long work — the branch is a checkpoint a
fresh instance can re-orient from without any handoff conversation.
*Defeats: dependence on one process's lifetime; "where was I" after a
cold start.*

**Externalized working memory.** Decisions, constraints, and current
status are written to files (a progress log, project notes) and re-read
at the start of every session or takeover, never carried only in context.
When a runtime compacts context, the structured summary it produces is
the same pattern: goal state, what is done, what is next, what is blocked
— written to survive the cut. *Defeats: context loss, compaction,
handoffs between instances and between tools.*

## Get woken up again

**Scheduled self-check.** A follow-up that matters becomes a scheduled
job (cron or the platform equivalent) that polls the work's real status,
reports only on completion or failure, and removes itself when the work
resolves. A sleeping process is not a plan: when the session's turn ends,
nothing in it runs. *Defeats: the silent stall — work finished or failed
hours ago that nobody reports until the user returns.*

**Watchdog with progress-based liveness.** A supervisor (or a timed
check) watches workers and restarts or replaces stalled ones. Liveness is
judged from progress evidence — checkpoint timestamps, output growth, a
heartbeat the worker updates — never from silence alone: a quiet worker
may be mid-step, and a worker whose checkpoint has not moved is stalled
whether or not it looks busy. *Defeats: hung workers consuming the whole
absence; premature kills of slow-but-alive work.*

**Completion handoffs over polling.** Where the platform delivers them
(sub-agent results, CI webhooks, background-task notifications), prefer
being woken by the event to polling for it — and never stack a poller on
top of a handoff for the same work, which double-reports. *Defeats:
wasted checks; missed completions between polls.*

## Re-enter safely

**Verify before repeating.** After any interruption, establish the true
state of the world before acting: what the repo, queue, and output files
say actually happened. An action whose outcome is unknown — a timed-out
request, a worker that died mid-step — is checked, not repeated. This
matters most for irreversible and external actions (sends, purchases,
deletes, pushes), where a blind repeat doubles the effect. *Defeats:
duplicate side effects; resuming from a checkpoint the world has moved
past.*

**Idempotent steps and claim/ack queues.** Long work is queued in a
durable list; a worker claims an item (so a second instance skips it),
does it, and acks on completion. Re-running a step is designed to be
harmless — it detects its own prior effect and no-ops. *Defeats: two
instances doing the same unit; a crash between "done" and "recorded"
losing or duplicating the unit.*

**Bounded retries, then block.** Each step carries a retry budget (this
skill's rule: two). Exhausting it changes the approach or converts the
step to a blocked item carrying the evidence — it never becomes an
unbounded retry loop. *Defeats: an absence burned on a step that was
never going to succeed; silent infinite loops.*

## Structure for partial completion

**Decompose into durable units.** Fan work out into units that each land
somewhere durable — a written file, a commit, a completed queue item —
coordinated from the progress file rather than from one instance's
memory. Partial completion is then real: 70% finished means 70% present
on disk, and the resume is a remainder, not a restart. *Defeats:
all-or-nothing jobs; coordinator loss taking the whole plan with it.*

## The same patterns in specific runtimes

- **Hosted assistant VMs** (this skill's home environment): deployments
  replace the VM and kill ad-hoc processes without notice; the home
  directory persists. Checkpoints, resume markers, and scheduled checks
  live in the home directory; nothing may depend on a process surviving.
- **Desktop coding agents** (Claude Code, Codex and similar): sessions
  persist and resume by id, lifecycle hooks re-inject state at session
  start and after compaction, and background tasks report on completion —
  the checkpoint and externalized-memory patterns, implemented by the
  tool. The agent's part is keeping the durable record (commits, notes)
  worth resuming from.
- **Cloud software-engineering agents** (Devin and similar): the work
  runs in a persistent cloud session independent of the user's machine,
  with snapshots and a replayable action log — durability is the
  platform's, and the log doubles as the audit trail.
- **Durable workflow frameworks** (Temporal; LangGraph's checkpointer):
  execution state is persisted at every step boundary under a run or
  thread id, and a crashed worker resumes the run from its last
  checkpoint with the same inputs. This is checkpoint + resume,
  generalized and made the framework's contract rather than the agent's
  habit — the direction agent runtimes are converging on.

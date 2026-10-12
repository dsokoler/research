# Running 5–10 Communicating Claude Code Agents on One Machine

Oct 10, 2026 · @Dan

## TL;DR

Run a lead-orchestrated star of separate, named Claude Code sessions inside WSL2. Coordinate them through a SQLite-backed MCP blackboard, which also makes the whole campaign resumable.

- **Architecture:** one Opus lead and 4–8 Sonnet workers, each its own Claude Code process. Coders work in their own git worktrees. Durable state lives in one blackboard MCP server: tasks, leased claims, findings, symbols and checkpoints. Native `SendMessage` pushes notifications, and a `PostToolBatch` hook injects a short digest of new findings between tool calls. Only a librarian agent writes to the IDA/Ghidra database, and a separate verifier tries to disprove findings.
- **Persistence:** state lives in the database, not in context windows, so the team can stop and pick up where it left off. Agents checkpoint mid-task, a `Stop` hook enforces it, and a `SessionStart` hook briefs each relaunched agent from the board. This is the biggest advantage over native agent teams, which can't resume in-process teammates.
- **When it pays off:** only on work that splits into independent parts. Anthropic's research system beat a single agent by 90.2% but used roughly 15× the tokens of chat. Google's scaling study found centralized coordination gained about 81% on a parallelizable task. Every multi-agent variant lost 39–70% on sequential planning.
- **Size:** start with 3–5 agents and add more only when the task queue stays full. Ten agents use quota roughly ten times as fast. Tightly sequential work, such as one exploit chain or one driver crash, is better done by a single session with subagents.

> _Diagram — campaign architecture (6 roles, 1 blackboard). Rendered in the live doc; described in the paragraph below._

Every agent claims work and posts results through the blackboard. Analysts read the disassembly through one RE supervisor, the librarian alone writes through it, and the verifier checks findings on the debug VM.

## What the research says

Multi-agent helps on breadth-first work and hurts on sequential work. At 5–10 agents, the binding constraint is how well the work splits, not how the agents talk.

| Source | Finding | What it means for you |
| --- | --- | --- |
| [Anthropic, multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system) (June 2025) | An Opus 4 lead with Sonnet 4 subagents beat single-agent Opus 4 by 90.2% on an internal research eval. Token usage alone explained 80% of the variance. Agents use about 4× the tokens of chat; multi-agent systems about 15×. The approach fits tightly interdependent tasks such as coding poorly. | Multi-agent mostly buys extra context windows and tokens. Use it for broad RE triage and research; be cautious with coupled code edits. |
| [Kim et al., Google, arXiv 2512.08296](https://arxiv.org/abs/2512.08296) | Version 3 (April 2026) covers 260 configurations across six agentic benchmarks (earlier versions: 180). Centralized coordination scored 0.631 vs 0.349 for a single agent on Finance-Agent, about +81%. Every multi-agent variant lost 39–70% on sequential planning. Independent agents amplified errors 17.2×; centralized coordination held that to 4.4×. Once single-agent accuracy passed about 45%, extra agents hurt. Tool-heavy tasks paid the most coordination overhead. | Prefer a central orchestrator to independent agents. RE is tool-heavy, so keep each worker's tool set narrow and route outputs through a verifier. |
| [Cemri et al., MAST, arXiv 2503.13657](https://arxiv.org/abs/2503.13657) | 1,600+ annotated traces across 7 frameworks, with 14 failure modes in 3 categories. In the published version, specification problems are about 41.8% of failures, inter-agent misalignment 36.9% and verification 21.3%. The same model as a single agent sometimes beat the multi-agent version. | Most failures come from the harness, not the model. Write precise role specs, use structured messages and verify explicitly. |
| [MultiAgentBench](https://www.emergentmind.com/topics/multiagentbench) (ACL 2025) | Graph (mesh) topology did best in the research scenario; tree did worst. Adding agents can lower performance. Cognitive planning added about 3% to milestone completion. | Topology effects are modest. Allow direct peer messages for research debates, but don't expect topology to rescue a weak decomposition. |
| [Cognition, "Don't Build Multi-Agents"](https://cognition.com/blog/dont-build-multi-agents) (June 2025, plus an April 2026 follow-up) | Parallel agents without shared context make conflicting implicit decisions. The follow-up says some setups now work. | Shared, explicit decision state, such as the symbol registry, is the fix. Give read-only workers latitude; tightly control anything that writes. |
| [Anthropic, C compiler with parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler) (Nicholas Carlini, Feb 2026) | 16 agents ran nearly 2,000 sessions over two weeks, using about 2 billion input and 140 million output tokens for just under $20,000. They produced a 100k-line compiler, coordinating only through lock files and git. All 16 stalled on the Linux kernel until it was re-split using GCC as a known-good oracle. | Parallelism needs work that partitions, plus an oracle. For RE, the oracles are the debugger, test harnesses and diffs against known-good builds or symbols. |

RE triage of a large binary, literature or CVE research, and building N independent tools all split well. Tracing one data flow through a driver, root-causing one crash, or refactoring shared headers do not. Use one strong session with subagents for those.

## Instance separation options

Use separate named sessions as the main fabric, worktrees for anything that edits code, and subagents inside each worker. State is as of Claude Code around v2.1.28x, October 2026.

| Option | Isolation | Comms | Parallelism | Persistence | Main failure modes | Verdict |
| --- | --- | --- | --- | --- | --- | --- |
| Subagents (Agent tool) | Own window; returns a summary to its caller. Background by default since v2.1.198; nest up to 5 levels (v2.1.172). | Report to the parent; named subagents can `SendMessage` each other. | 20 concurrent per session by default (`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`). | Tied to the parent session. | The parent's context becomes the bottleneck; workers are short-lived. | Use inside each worker for noisy side tasks such as dumping xrefs. |
| Agent teams (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, v2.1.32+) | Full session per teammate; loads CLAUDE.md, MCP and skills, not the lead's history. | Per-agent JSON mailbox, automatic delivery, shared task list with file-locked claims and dependencies. | Linear in team size; docs suggest 3–5 teammates. | `/resume` and `/rewind` don't restore in-process teammates; team config is deleted at session end. | Experimental. No worktree isolation, so same-file edits collide. Split panes need tmux or iTerm2. Not available in `-p` or SDK mode. | Lightweight fallback for single-sitting sessions. |
| Separate interactive sessions (tmux or agent view) | Full isolation. | Native cross-session messaging (v2.1.224+ on macOS, Linux, WSL2; v2.1.234+ native Windows) plus your blackboard. | Limited by quota and RAM. | Normal resumable transcripts; agent view's supervisor keeps background sessions alive. | You supply coordination. Agent view is a research preview with Windows rendering bugs. | **Primary choice**, with a blackboard. |
| Headless `claude -p` loops | Full; fresh context each iteration. | Files, git, MCP; inbound messages need `crossSessionInbound: "accept"`; `--bare` disables the inbox. | High. | Stateless per run; state must live in files or the DB. | Re-orientation tokens every loop; no human prompts; no agent teams. | Unattended overnight batches. |
| Dynamic workflows (`ultracode`, v2.1.154+) | A JavaScript script holds the plan; each `agent()` call is isolated. | Script variables only. | 16 concurrent by default (1–256); 1,000 agents per run. | Resumable within the session. | Subagents run in acceptEdits; no shell in the script. | Bulk fan-out-and-verify passes. |
| Claude Agent SDK | You control everything: sessions, hooks, `canUseTool`, budget caps. | None built in between processes. | Unbounded. | You implement it. | More engineering; no agent teams. | Only if you outgrow the CLI. |
| Git worktrees | Separate trees on separate branches (`--worktree`/`-w`, `isolation: worktree`). | n/a | n/a | Persistent until merged. | Conflicts move to merge time; untracked files not copied (use `.worktreeinclude`). | **Mandatory for any worker that edits code.** |

## Communication mechanisms

Use SQLite behind MCP for state, `SendMessage` for push and hooks for injection. An agent can only receive a message at a turn boundary or between tool calls, so every mechanism needs a push path or a written pull habit.

| Mechanism | Latency | Reliability and races | Fit to the turn loop | Use it for |
| --- | --- | --- | --- | --- |
| Native cross-session `SendMessage` | Near-instant; local Unix socket or Windows named pipe. | Plain text, no locking or write ordering. Size and burst caps; 100 messages per session. Mixed permission modes hold messages for approval, expiring after 5 minutes. WSL2 and native Windows sessions can't reach each other. | Excellent: read between tool calls, or starts a turn if idle. `notify_when_idle` gives completion callbacks. | Push notifications and short handoffs. |
| Agent-team mailbox and task list | Near-instant. | File-locked task claims. | Automatic delivery. | The fallback architecture. |
| Shared markdown or JSON files | Pull. | Races and lost updates without locking; unstructured. | Poor unless polled. | Human-readable progress notes. |
| SQLite (WAL) behind an MCP server | Milliseconds. | ACID transactions; leases with TTLs survive crashed agents; structured queries; **persists across restarts**. | Good with a hook digest. | **The durable blackboard.** |
| [mcp\_agent\_mail](https://github.com/dicklesworthstone/mcp_agent_mail) | Milliseconds. | Advisory file leases with a pre-commit guard (bypassed by `--no-verify`); threaded inboxes; git-auditable archive; web UI. | Pull: agents call `fetch_inbox`. | Off-the-shelf alternative to your own blackboard. |
| Hooks (`PostToolBatch`, `SessionStart`, `Stop`, `TaskCompleted`) | Every tool call or lifecycle event. | Deterministic. | Best way to inject state without the agent asking. | Digests, briefings, checkpoint enforcement, quality gates. |
| Git lock files and push races | Seconds. | Atomic push acts as a mutex; frequent merge conflicts. | Pull. | Code integration; claims only as a last resort. |
| Redis or NATS pub/sub | Sub-millisecond. | Robust, but another daemon. | Still needs a hook bridge. | Overkill for one machine. Skip it. |

Low latency barely matters because the agent loop runs in seconds to minutes. Structured state, atomic claims and verified writes are what matter.

## Choosing a topology

Default to an orchestrator star with a blackboard underneath and permitted peer messages on top.

- **Orchestrator / lead–worker (star): the default.** It matches the Google study's best result on parallel work and Anthropic's production design. The lead's context fills up, so it should read summaries and query the board, not raw output.
- **Hierarchical (tree): avoid at this scale.** MultiAgentBench found tree topology worst, and an extra management layer for ten agents only adds misalignment.
- **Peer-to-peer mesh: allow inside the star.** Workers may message each other directly about overlaps, such as a rename the other depends on. Don't remove the lead.
- **Blackboard:** the shared-state layer under the star, not a separate choice.
- **Market or bidding:** self-claiming from a prioritized queue with leases captures most of the benefit. Formal bidding isn't worth building.
- **Debate / critic:** worth it for competing hypotheses, such as whether an IOCTL handler is exploitable. Always include one verifier whose only job is refuting findings.

Multi-agent is worse than a single agent on sequential tasks, on tasks a single agent already handles well, on same-file edits and on work with heavy cross-dependencies. Anthropic's agent-teams docs give the same advice.

## Decomposition and shared state

Split work by ownership, not by pipeline stage, and give the disassembler database exactly one writer.

**Partitioning**

- **Binary RE:** build a call graph first, then cluster functions into modules such as IRP dispatch, IOCTL handlers, WFP callouts, ETW registration and crypto. Each worker claims a cluster. Leaf-first ordering lets callers inherit names from their callees.
- **Research:** one worker per independent question or source family, such as prior CVEs, WDK semantics or public writeups.
- **Coding:** one worker per file set or module, each in its own worktree, with the lead or you merging.

**Blackboard schema.** Key everything by binary SHA-256 plus RVA so it survives rebasing and multiple IDBs.

| Table | Key columns | Purpose |
| --- | --- | --- |
| `tasks` | id, kind, scope, role, priority, deps, status, owner, lease\_expires | Work queue. Claims are an atomic `UPDATE ... WHERE status='open'`. `owner` is the stable agent name, not a session id. |
| `checkpoints` | id, task\_id, agent, created\_at, tried, ruled\_out, next\_step, artifacts | Mid-task progress, so a fresh agent can continue instead of redoing work. Writing one also extends the lease. |
| `findings` | id, task\_id, agent, binary\_sha256, rva, claim, evidence, confidence, status, source, derived\_from | Results. Status climbs proposed → verified → oracle (see Ground truth). \`source\` records the evidence kind; \`derived\_from\` links a finding to what it rests on, so a retraction cascades. |
| `symbols` | binary\_sha256, rva, proposed\_name, proposed\_type, rationale, agent, status, applied, applied\_at | Naming registry with a uniqueness constraint. Agents propose; only the librarian applies. `applied` records whether the name actually landed in the IDB. |
| `types` | name, c\_decl, owner, version | Shared struct reconstructions, such as `_DEVICE_EXTENSION`, versioned so nobody silently redefines one. |
| `messages` | to, from, thread, body, read | Optional durable mail if you don't rely only on `SendMessage`. |

**One writer for the IDB.** IDA databases aren't built for concurrent writers. Point all workers at one headless supervisor so reads, decompiles and xrefs run in parallel: [re-mcp](https://github.com/mrexodia/re-mcp) (idalib or pyghidra, multi-database, needs IDA Pro 9+ for idalib), [ida-pro-mcp](https://github.com/mrexodia/ida-pro-mcp)'s `idalib-mcp`, or [ida-multi-mcp](https://github.com/MeroZemory/ida-multi-mcp) for several GUI instances. Only the librarian gets rename, set-type and comment tools.

**Verification.** Nothing moves from proposed to verified until a different agent confirms it, ideally with a debugger check on a VM. For example, break on the handler and confirm the IOCTL code and buffer method. A `TaskCompleted` hook can refuse completion while a task's findings are unverified.

## Ground truth and oracles

The single biggest quality lever is giving agents authoritative sources instead of letting them reason from the decompiler alone. This is the C-compiler lesson: that project only worked once GCC was wired in as a known-good oracle. A name confirmed against a PDB or a live breakpoint is worth many that the model guessed.

**Sources of ground truth for Windows RE, best first.**

- **Microsoft's public symbol server.** PDBs give real names and types for much of Windows, including many kernel components. Resolve them first, before any agent proposes a name, and seed the `symbols` and `types` tables from them. Agents then only name what the symbols don't cover.
- **Signatures for known code.** FLIRT, Lumina or your own type libraries identify library and runtime code so agents don't burn tokens renaming `memcpy`. Build a reusable type library from your WDK and driver headers.
- **Dynamic validation on the target VM.** WinDbg or x64dbg, plus WPP/ETW tracing, confirm behavior the static view only suggests: break on a handler and read the actual IOCTL code and buffer method, or trace a provider to see which path really runs.
- **Diffing against a known-good build.** When you have symbols for one version, BinDiff or Diaphora ports them to an unknown one.

**Add an `oracle` confidence tier.** Findings backed by a PDB, a signature match or a debugger observation rank above plain `verified` (one agent confirming another's reasoning). Record the source on each finding so a reader can tell a ground-truth fact from a cross-checked inference from a lone guess. Extend the `findings.status` ladder to proposed → verified → oracle, and have the verifier prefer oracle evidence when it exists.

**Wire oracles in as MCP tools** on the Windows host, beside the RE supervisor: a symbol-resolution tool, a signature-lookup tool and the debugger. The librarian seeds authoritative names before the campaign starts, so agents inherit them rather than reinventing them.

## Hardening

These failure modes don't show up in a two-agent demo but do bite at six to ten agents. Address the first three before a real campaign; the rest as the setup matures.

**The poisoned board.** The blackboard's strength is also its risk: one confidently-wrong finding marked `verified` propagates to every agent that builds on it, which is worse than a single agent's bad context because it spreads. Two defenses: have the verifier use a *different model family* from the worker that produced the finding, since the same model shares blind spots, and require it to cite external evidence (debugger, oracle) rather than re-reading the same decompiler output. You also need a **retraction path**: invalidating a finding must cascade to everything derived from it. Track provenance with a `derived_from` edge between findings and symbols, mark a retracted item and its descendants `refuted`, and have the next `SessionStart` briefing flag any agent whose work rested on a retracted fact.

**Prompt injection from the target.** You feed attacker-controlled input into agents: strings, decompiler output, embedded resources, and more so with malware samples. A crafted string can read as an instruction. Treat all disassembler, string and resource output as untrusted data, never as instructions, and say so in CLAUDE.md. Keep agents that touch raw bytes on tight tool allowlists with no network, which the WSL2 sandbox already supports, and run genuine malware only in the isolated VM, never on the host the agents control.

**Failure attribution.** When a six-agent run goes wrong, MAST's point is that the hard part is finding *which* agent caused it. Put a campaign/correlation ID on every board write and message, and keep one unified, timestamped event log across all agents (a simple `events` table, or append-only JSONL under `logs/`). Without it, debugging the swarm is guesswork.

**Operational resilience.** Headless IDA or idalib will crash under concurrent load, and ten agents will hit rate limits. Retry 429s with backoff, and put a health check and circuit breaker around the RE supervisor so a hung disassembler doesn't wedge the fleet. A watchdog that restarts a dead worker (leases already let it reclaim its tasks) keeps a crash from silently halving throughput.

**Dedup and deadlock.** Claiming stops two agents taking the same task, but not cross-role overlap, such as an analyst and the researcher both chasing one CVE, or dependency cycles. Check for cycles when seeding `deps`, and let an agent query the board for related in-flight work before it starts.

**Prove it's worth 15×.** Run one module single-agent and one through the swarm, and compare results and cost. It confirms the multiplier is buying something for your workload rather than being paid on faith, and it tells you where the crossover between single-agent and multi-agent sits.

**Cross-campaign memory.** Your type library, symbol conventions and hard-won lessons are per-campaign by default. A durable store that carries them forward, seeded into each new board, turns every binary analyzed into an asset for the next one.

**Human-review cadence.** Decide in advance when you get pulled in rather than dipping in ad hoc. A `needs_human` queue for low-confidence or security-critical findings, surfaced on the dashboard, keeps review deliberate and stops the swarm running for hours on a wrong turn.

## The harness on one machine

Run every Claude Code agent inside WSL2 and keep IDA, Ghidra, WinDbg and the kernel target on the Windows side, exposed over HTTP MCP.

### Platform

- **Agents in WSL2, all in one environment.** Native sandboxing (bubblewrap) works on WSL2 but not on native Windows. Split panes need tmux. WSL2 and native Windows sessions can't message each other, so mixing them breaks native messaging.
- **Heavy tooling on the Windows host**, exposed over HTTP MCP. Bind it to localhost or the WSL vNIC with a token; some forks default to `0.0.0.0`.
- **Kernel debug targets in a VM**, never on the host the agents control.

### Directory layout

```
~/re-campaign/                       # git repo (code, notes, scripts)
├── CLAUDE.md                        # shared rules every agent loads
├── .mcp.json                        # blackboard + RE servers
├── .claude/
│   ├── settings.json                # env, permissions, hooks, sandbox
│   ├── librarian.json               # librarian's settings override
│   ├── agents/                      # lead, re-analyst, librarian,
│   │                                # researcher, coder, verifier
│   └── hooks/
│       ├── session_brief.sh         # SessionStart: task + last checkpoint
│       ├── inject_digest.sh         # PostToolBatch: new board events
│       ├── stop_checkpoint.sh       # Stop: require a fresh checkpoint
│       ├── guard_ownership.sh       # PreToolUse: block edits outside claim
│       └── require_verified.sh      # TaskCompleted: block unverified
├── blackboard/
│   ├── server.py                    # FastMCP + SQLite (WAL)
│   ├── cli.py                       # same DB, for hooks and scripts
│   └── board.db
├── backups/                         # board + IDB snapshots, paired by time
├── binaries/                        # read-only inputs, by sha256
├── notes/                           # human-readable exports
└── logs/                            # per-agent transcripts and dumps
```

### Shared `CLAUDE.md` (excerpt)

```markdown
# Campaign rules (all agents)
- Your agent name arrives in your session briefing. Use it for every board call.
- On start: if the briefing shows a claimed task, continue from its last checkpoint.
  Otherwise call board.claim_next(agent, role).
- Checkpoint with board.checkpoint after each function or sub-question,
  before any long tool run, and before stopping.
- Never rename or retype in the IDB. Propose via board.propose_symbol; only librarian applies.
- Key every finding by sha256+RVA, with evidence (decompiler snippet, xref, debugger output).
- Obey STOP or RESCOPE from lead. If you change something another agent depends on,
  SendMessage them one line with the finding id.
- Keep tool output small: write long dumps to logs/ and reference the path.
```

### `.claude/settings.json`

```json
{
  "env": {
    "CLAUDE_CODE_SUBAGENT_MODEL": "sonnet",
    "CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS": "4"
  },
  "crossSessionInbound": "accept",
  "permissions": {
    "allow": [
      "mcp__board__*",
      "mcp__re__decompile*", "mcp__re__xrefs*", "mcp__re__list*", "mcp__re__get*",
      "Read", "Grep", "Glob",
      "Bash(git status:*)", "Bash(git diff:*)", "Bash(git commit:*)"
    ],
    "deny": ["mcp__re__rename*", "mcp__re__set_type*", "Bash(rm -rf:*)"]
  },
  "sandbox": { "enabled": true },
  "hooks": {
    "SessionStart": [
      { "matcher": "startup|resume|compact",
        "hooks": [{ "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/session_brief.sh" }] }
    ],
    "PostToolBatch": [
      { "hooks": [{ "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/inject_digest.sh" }] }
    ],
    "PreToolUse": [
      { "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/guard_ownership.sh" }] }
    ],
    "Stop": [
      { "hooks": [{ "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/stop_checkpoint.sh" }] }
    ],
    "TaskCompleted": [
      { "hooks": [{ "type": "command", "command": "${CLAUDE_PROJECT_DIR}/.claude/hooks/require_verified.sh" }] }
    ]
  }
}
```

The librarian lifts the `deny` rules through `.claude/librarian.json`. Set `crossSessionInbound` explicitly, because mixing bypass and prompting modes otherwise holds messages for up to five minutes. Hook names and fields change often, so check them against your installed version.

### Agent definition example: `.claude/agents/re-analyst.md`

```markdown
---
name: re-analyst
description: Analyzes one claimed function cluster; proposes names/types; never writes the IDB.
model: sonnet
tools: Read, Grep, Glob, mcp__board__*, mcp__re__decompile, mcp__re__xrefs_to, mcp__re__xrefs_from, mcp__re__get_function, mcp__re__list_strings
---
You own exactly the cluster you claimed. Work leaf-first. For each function: summarize behavior,
propose name + prototype with rationale, flag security-relevant behavior (IOCTL dispatch, METHOD_NEITHER
buffers, missing ProbeForRead/Write, pool allocations sized by user input). Post findings with evidence.
Checkpoint after each function. If you need a callee owned by someone else, read their proposed
symbols first; message them if none exist.
```

Coders get `isolation: worktree` and Edit/Write. The verifier gets read-only tools plus debugger MCP tools and the instruction to refute by default without evidence.

### `inject_digest.sh`: receiving mid-task

```bash
#!/usr/bin/env bash
# PostToolBatch: inject board events since this agent's last check.
cat > /dev/null   # drain the hook's JSON input
DIGEST=$(python3 "$CLAUDE_PROJECT_DIR/blackboard/cli.py" digest --agent "$AGENT_NAME")
[ -z "$DIGEST" ] && exit 0
jq -nc --arg m "Board updates since your last check:
$DIGEST" \
  '{hookSpecificOutput:{hookEventName:"PostToolBatch", additionalContext:$m}}'
```

Keep the digest to ids and one-liners. `cli.py digest` can also add a nudge when the agent's last checkpoint is stale.

### Blackboard MCP server (sketch)

```python
# blackboard/server.py: FastMCP over SQLite (WAL). About 250 lines in full;
# wrap each transaction in try/except with ROLLBACK there.
from mcp.server.fastmcp import FastMCP
import sqlite3, time, os
DB = os.path.join(os.path.dirname(__file__), "board.db")
db = sqlite3.connect(DB, isolation_level=None, check_same_thread=False)
db.execute("PRAGMA journal_mode=WAL"); db.execute("PRAGMA busy_timeout=5000")
mcp = FastMCP("board")
LEASE = 1800  # seconds; every checkpoint renews it

@mcp.tool()
def claim_next(agent: str, role: str) -> dict:
    """Atomically claim the top open task whose deps are done (expired leases are reclaimable)."""
    now = int(time.time())
    db.execute("BEGIN IMMEDIATE")
    row = db.execute("""SELECT id, scope FROM tasks
        WHERE role=? AND (status='open' OR (status='claimed' AND lease_expires<?))
          AND NOT EXISTS (SELECT 1 FROM deps d JOIN tasks t ON t.id=d.dep
                          WHERE d.task=tasks.id AND t.status!='done')
        ORDER BY priority DESC LIMIT 1""", (role, now)).fetchone()
    if not row:
        db.execute("COMMIT"); return {"task": None}
    db.execute("UPDATE tasks SET status='claimed', owner=?, lease_expires=? WHERE id=?",
               (agent, now + LEASE, row[0]))
    db.execute("COMMIT")
    return {"task": row[0], "scope": row[1], "last_checkpoint": last_checkpoint(row[0])}

@mcp.tool()
def checkpoint(agent: str, task_id: int, tried: str, ruled_out: str,
               next_step: str, artifacts: str = "") -> None:
    """Record mid-task progress and renew the lease (doubles as a heartbeat)."""
    now = int(time.time())
    db.execute("BEGIN IMMEDIATE")
    db.execute("""INSERT INTO checkpoints(task_id, agent, created_at, tried, ruled_out,
                  next_step, artifacts) VALUES (?,?,?,?,?,?,?)""",
               (task_id, agent, now, tried, ruled_out, next_step, artifacts))
    db.execute("UPDATE tasks SET lease_expires=? WHERE id=? AND owner=?",
               (now + LEASE, task_id, agent))
    db.execute("COMMIT")

@mcp.tool()
def resume(agent: str) -> list: ...           # this agent's claimed tasks + last checkpoints; renews leases
@mcp.tool()
def reset_stale(grace_s: int = 0) -> int: ...  # lead only: reopen expired claims, keep checkpoints
@mcp.tool()
def propose_symbol(agent: str, sha256: str, rva: int, name: str,
                   proto: str, rationale: str) -> dict: ...  # conflicts returned for negotiation
@mcp.tool()
def pending_symbols() -> list: ...             # accepted AND applied=0, for the librarian
@mcp.tool()
def mark_applied(ids: list[int]) -> None: ...  # librarian, only after the IDB save succeeds
@mcp.tool()
def post_finding(agent: str, task_id: int, sha256: str, rva: int, claim: str,
                 evidence: str, confidence: float) -> int: ...
@mcp.tool()
def verify(agent: str, finding_id: int, verdict: str, evidence: str) -> None: ...
@mcp.tool()
def digest(agent: str) -> list: ...            # events since this agent's last digest

if __name__ == "__main__":
    mcp.run()   # stdio; each agent spawns a client; WAL handles concurrency
```

If you'd rather not build this, mcp\_agent\_mail gives you identities, inboxes, threads and advisory leases today. You'd still add the `symbols`, `types`, `findings` and `checkpoints` tables.

### `.mcp.json`

```json
{
  "mcpServers": {
    "board": { "command": "python3", "args": ["blackboard/server.py"] },
    "re":    { "type": "http", "url": "http://<windows-host-ip>:9999/mcp",
               "headers": { "Authorization": "Bearer ${RE_MCP_TOKEN}" } }
  }
}
```

### Launching and monitoring

```bash
#!/usr/bin/env bash
# launch.sh: lead + workers as named sessions in tmux (WSL2).
# Rerun it to resume: stable AGENT_NAMEs let each agent reclaim its own tasks.
cd ~/re-campaign
sqlite3 blackboard/board.db ".backup 'backups/board-$(date +%F-%H%M).db'"
run() { tmux new-window -t camp -n "$1" "AGENT_NAME=$1 claude --name $1 --agent $2 ${*:3}"; }
tmux new-session -d -s camp -n lead "AGENT_NAME=lead claude --name lead --agent lead --model opus"
for i in 1 2 3; do run re$i re-analyst; done
run librarian librarian --settings .claude/librarian.json
run verifier verifier --model opus
run coder1 coder -w
tmux attach -t camp
```

- **Agent view instead of tmux:** `claude agents` or `claude --bg` dispatches background sessions whose supervisor survives terminal closes. `claude agents --json` lets a script poll status. It's a research preview and behaves better inside WSL2.
- **Observability:** `/list-agents` (alias `/peers`) shows who can reach whom. `/status` shows the inbox address; `/tasks` and `/workflows` show background work. The board is your dashboard: a small web page or `watch sqlite3` over tasks, findings and checkpoints. Open sessions with `--verbose` to see full inbound messages.
- **Supervision:** talk only to the lead, the one agent allowed to rescope. Messages from other agents can't approve permissions or change configuration, so a confused worker can't escalate through a peer.

### Cost and limits

- **Quota:** background sessions draw on your subscription like interactive ones, so ten agents use quota about ten times as fast. Anthropic's cost docs put agent teams at about 7× a standard session's tokens when teammates run in plan mode. A 10-agent campaign realistically needs Max 20x or API billing.
- **Levers:** Opus only for the lead and verifier; Sonnet for workers. Keep spawn prompts and CLAUDE.md short, since every agent pays for them. Shut down idle agents. Push bulk triage into a dynamic workflow so intermediate results stay out of context.
- **Local resources:** Claude Code processes are light; idalib workers aren't. Cap the RE supervisor's workers, for example `--max-workers 4`, rather than one IDA instance per agent.

### Existing tools and their maturity

| Tool | What it does | Maturity |
| --- | --- | --- |
| [Claude Squad](https://github.com/smtg-ai/claude-squad) | TUI over tmux with a git worktree per agent; supports Claude Code, Codex, Gemini, Aider. | Stable and widely used (about 8.5k stars), but isolation and supervision only, no inter-agent comms. Needs tmux. |
| [workmux](https://github.com/raine/workmux) | Worktree plus tmux-window workflow. | Simple and solid; same scope as Claude Squad. |
| [mcp\_agent\_mail](https://github.com/dicklesworthstone/mcp_agent_mail) | Mail-style MCP coordination with identities, inboxes, threads, advisory leases and a git audit trail. | Most complete community coordination server. Leases are advisory; its large-scale claims are unverified. |
| [re-mcp](https://github.com/mrexodia/re-mcp), [ida-pro-mcp](https://github.com/mrexodia/ida-pro-mcp), [ida-multi-mcp](https://github.com/MeroZemory/ida-multi-mcp), [GhidraMCP](https://github.com/LaurieWired/GhidraMCP) | Disassembler access over MCP; re-mcp adds multi-database headless supervision. | ida-pro-mcp and GhidraMCP are popular and maintained. Multi-instance forks are younger, so test for deadlocks under concurrent load. Headless needs IDA Pro. |
| Anthropic native (agent teams, cross-session messaging, agent view, workflows) | First-party orchestration. | Moving fast: messaging shipped August 2026, workflows and agent view May 2026. Pin your Claude Code version per campaign. |

## Persistence and resuming a campaign

The board lets the team stop and pick up where it left off, but only for what agents write to it. Anything held only in a context window is gone when a session ends.

| State | Where it lives | After a restart |
| --- | --- | --- |
| Tasks, claims, leases | Board | Intact, but claims may be stale |
| Findings, verdicts, symbols, types | Board | Intact |
| Mid-task reasoning: what was tried, what's next | Context window | Lost unless checkpointed |
| Names and types in the IDB | IDB, written by the librarian | Intact, but can lag the board after a crash |
| Session transcript | Claude Code session files | Intact; resumable but costly |

### Checkpoint mid-task, not just at completion

The `checkpoints` table records what was tried, what was ruled out and the next step. Agents write one after each function or sub-question and before any long tool run. A fresh agent can then continue a half-done cluster instead of redoing it. Each checkpoint also renews the task's lease, so it doubles as a heartbeat.

### Enforce it with hooks

Don't rely on agents remembering. Two hooks do the work.

- **`SessionStart`** with matcher `startup|resume|compact` briefs the agent from the board. It injects the agent's name, claimed tasks, last checkpoint per task, and any lead instructions. The `compact` source re-briefs after auto-compaction, which otherwise drops detail.
- **`Stop`** blocks an agent from going idle while it holds a task without a fresh checkpoint. It checks `stop_hook_active` to avoid looping.

```bash
#!/usr/bin/env bash
# session_brief.sh: SessionStart (startup | resume | compact).
cat > /dev/null   # drain the hook's JSON input
BRIEF=$(python3 "$CLAUDE_PROJECT_DIR/blackboard/cli.py" brief --agent "$AGENT_NAME")
jq -nc --arg m "You are agent $AGENT_NAME.
$BRIEF" \
  '{hookSpecificOutput:{hookEventName:"SessionStart", additionalContext:$m}}'
```

```bash
#!/usr/bin/env bash
# stop_checkpoint.sh: Stop. Require a checkpoint before idling with a claimed task.
INPUT=$(cat)
[ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ] && exit 0   # avoid loops
STALE=$(python3 "$CLAUDE_PROJECT_DIR/blackboard/cli.py" stale-checkpoint \
          --agent "$AGENT_NAME" --max-age 600)
[ -z "$STALE" ] && exit 0
jq -nc --arg r "Before stopping, call board.checkpoint for task $STALE: what you tried, what you ruled out, the next step." \
  '{decision:"block", reason:$r}'
```

`cli.py brief` calls the same logic as `board.resume`, so briefing also renews the agent's leases. `cli.py stale-checkpoint` prints a task id when the agent's newest checkpoint is older than the limit or older than its newest finding. A killed process never fires `Stop`, which is why mid-task checkpoints and lease expiry still matter.

### Reclaim stale claims on restart

- **Key ownership by a stable agent name.** Set `AGENT_NAME` at launch (see `launch.sh`), not a session id, which changes on every relaunch. A relaunched `re2` gets its own tasks back.
- **Reset what nobody will reclaim.** On startup, the lead calls `board.reset_stale` to reopen expired claims, for example after scaling from eight workers to five. Checkpoints stay attached, so the next claimant starts from them.
- **Lease length is a tradeoff.** A long lease makes crash recovery slow; a short one needs frequent heartbeats. 30 minutes, renewed by every checkpoint, is a reasonable start.

### Keep the IDB and the board in sync

The librarian runs one loop: `board.pending_symbols`, apply through the RE supervisor, save the IDB, then `board.mark_applied`. Order matters. If it crashes before marking, the rows stay `applied = 0` and are replayed on restart. Renames and type sets are idempotent, so replaying is safe. Never mark a row applied before the save succeeds.

### Prefer a fresh, briefed session over `--resume`

A fresh session briefed from the board usually beats resuming a long transcript. It gets a clean window and only the state that matters, the same reason the C-compiler project used fresh context each loop. Keep `--resume` for short interruptions where the transcript is still small.

### Back up the board and IDB together

- Use `sqlite3 blackboard/board.db ".backup 'backups/board-<timestamp>.db'"`. It's safe under WAL. A plain file copy while agents write can miss committed data still in `board.db-wal`.
- Snapshot the IDB right after a librarian save, with the same timestamp. Restore both from the same moment, or the `applied` flags won't match the database.
- `launch.sh` backs up the board on every launch. Add an hourly cron job for long runs, and commit exported `notes/` daily.

### Restart runbook

1. Stop the agents, or start here after a crash. Let the librarian finish its current batch if it's running.
2. Back up the board and the IDB.
3. Have the lead call `board.reset_stale` for any agents you aren't relaunching.
4. Rerun `launch.sh` with the same agent names.
5. Each agent is briefed by `SessionStart` and continues from its last checkpoint.
6. The librarian replays accepted, unapplied symbols before taking new work.

## Supporting tooling

Beyond the orchestration itself, a layer of agent tooling has matured that each closes a specific gap above. Evaluate the RE-specific ones before building the blackboard from scratch, since a couple of them ship pieces you'd otherwise write yourself.

**RE-specific agents.** Some of these have independently converged on the proposer + verifier + oracle design.

| Tool | What it is | Which gap it closes |
| --- | --- | --- |
| [Binary Ninja Sidekick 26.0](https://sidekick.binary.ninja/blog/sidekick-26-0-a-whole-new-experience-in-reversing-with-ai) | Purpose-built AI reversing: a validation agent cross-checks findings against IL, assembly and memory traces; hypotheses come with evidence; local models via Ollama/vLLM plus Anthropic/OpenAI; headless Python API; cross-binary projects; MCP. | The verifier + oracle pattern, as a product. Multi-provider = a ready **different-model verifier**; local models let you keep malware samples off the cloud. |
| [Rikugan](https://upd.dev/buzzer-re/Rikugan) | RE agent living inside IDA Pro 9+ and Binary Ninja: 60+ tools, 12 skills, MCP; asks before running scripts and never executes the target. Reuses Claude Code / Codex skills and MCP servers. | A drop-in worker that already has disassembler tools wired up and respects the read-only / no-execute rule. |
| "Decompiler as a database" (RECON 2026, *SELECT \* FROM binary*) | Exposes IDA / Ghidra / Binary Ninja over SQL; the same query runs across all three and annotations transfer between them. | The **single-writer symbol store** and **porting names across tools and versions** — two problems the blackboard otherwise has to solve by hand. |
| binary\_ninja\_mcp, headless Binary Ninja MCP (\~180 tools), agentic malware-analysis environments | More disassembler-over-MCP options for Claude Code / Codex. | Alternatives to the IDA-centric RE supervisor if you standardize on Binary Ninja. |

**Cross-cutting infrastructure.**

| Tool | What it is | Which gap it closes |
| --- | --- | --- |
| [Langfuse](https://langfuse.com) (MIT, self-host), Arize Phoenix, [Laminar](https://laminar.sh) | OpenTelemetry-native agent tracing: every LLM call, tool call, retry and sub-agent as spans. OpenInference conventions let these read each other's traces. | **Failure attribution** (Hardening). The blackboard shows *state*; this shows *execution* — which agent went wrong, where. Complements the `events` table, doesn't replace it. |
| microVM / user-space-kernel isolation: Firecracker, Kata Containers, gVisor; [microsandbox](https://github.com/microsandbox/microsandbox) (self-host); K8s Agent Sandbox | A stronger boundary than WSL2 — own guest kernel behind hardware virtualization. | The **malware safety caveat**. WSL2 is fine for code, not live samples. Pair with egress control: an isolated VM is useless if the agent reads secrets and types them into a reply. |
| [Serena](https://github.com/oraios/serena) | LSP-based MCP server giving symbol-level tools (`find_symbol`, `find_referencing_symbols`, `insert_after_symbol`) instead of grep-and-read. | **Context rot / focus** (see the correctness doc). Keeps each coding agent's context to the symbols it needs. For the source side (the EDR), not the binary side. |
| [Repomix](https://repomix.com) | Packs a repo into one AI-friendly file; Tree-sitter `--compress` to cut tokens while keeping structure; token counting; Secretlint secret-scanning; MCP mode. | Feeding a whole subsystem to the lead for **decomposition**, with secrets stripped first. |
| [GitHub Spec Kit](https://github.com/github/spec-kit) (and Kiro, OpenSpec, Tessl) | Drives agents through Spec → Plan → Tasks → Implement, each phase a Markdown artifact feeding the next; agent-agnostic across 30+ agents including Claude Code. | **Specification failures**, the largest MAST category (\~42%). For building the EDR more than for RE; skip it when the change is smaller than its spec. |

A note on verifying claims: every one of the RE tools above carries the same warning the research does — LLMs hallucinate names and even vulnerabilities, so the oracle layer (debugger, symbols, signatures) is what makes any of them trustworthy, whichever front-end you choose.

## Recommendations

Build the star-plus-blackboard design below, starting at six agents, and use agent teams only for single-sitting sessions.

### Step-by-step setup

1. **Set up WSL2** (Ubuntu) with Claude Code v2.1.250 or later, tmux, `bubblewrap`, `socat` and `sqlite3`. Enable `/sandbox`. Pin the Claude Code version for the campaign.
2. **On the Windows host,** run one headless RE supervisor (re-mcp or `idalib-mcp`) bound to the WSL interface with a token. Put WinDbg or x64dbg MCP in a target VM for verification.
3. **Create the repo layout.** Keep CLAUDE.md under 100 lines. Define the role agents: only `librarian` gets IDB write tools; only coders get `isolation: worktree` and Edit/Write.
4. **Build or install the blackboard**, including the `checkpoints` table and the `applied` flag on `symbols`. Seed it with the decomposition: the lead or an `ultracode` workflow builds the call graph and clusters tasks leaf-first, with dependencies.
5. **Wire up the hooks:** `SessionStart` briefing, `PostToolBatch` digest, `Stop` checkpoint gate, `PreToolUse` ownership guard and `TaskCompleted` verification gate.
6. **Launch six agents** with `launch.sh`: one lead (Opus), three RE analysts, one librarian and one verifier (Opus). Add a researcher or coder only when the queue has independent work for them.
7. **Run the loop.** Workers claim, analyze, checkpoint, propose and post. The librarian applies accepted names every few minutes and broadcasts a short notice such as "symbols v42 applied". The verifier confirms or refutes. The lead watches the board, rescopes and talks to you.
8. **Stop and resume freely.** Follow the restart runbook. Relaunch idle or bloated agents with fresh context whenever it helps; the board holds the state.

### Lightweight fallback

Enable agent teams (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) in-process inside WSL2. Ask the lead to spawn 3–4 teammates using the re-analyst and verifier agent types. Keep the RE MCP server, and use one `findings.md` per teammate plus the native task list. You lose resumability, worktree isolation and the durable symbol registry, so use it for a 2–4 hour triage or a single hypothesis debate, not a multi-day campaign. For pure bulk passes, a saved dynamic workflow such as `/triage-functions` is lighter still.

### When to scale

- **Down to one session plus subagents** for a single causal chain: one crash root cause, one exploit primitive, one refactor through shared headers.
- **Stay at 4–6** for most RE campaigns. Add an agent only if the open-task queue has stayed deeper than the agent count for over an hour and the verifier is keeping up. The verifier is usually the real bottleneck.
- **Go to 8–10** only for wide, independent work: diffing many driver versions, mass function triage, broad research sweeps. Add a second verifier, keep the lead strictly coordinating, and move bulk fan-out into workflows.

## Phase 2: the self-learning layer

Everything above is **Phase 1** — a single campaign, coordinated and verified against oracles, then thrown away when the binary is done. **Phase 2 adds the layer that makes each campaign start smarter than the last, without touching the model's weights.** Build it only once Phase 1 is running and producing oracle-verified findings; it has nothing to feed on before then. The thesis: you can't make a frozen model smarter, so improvement accumulates *around* it — in verified knowledge, tools, lessons and routing — with the oracle, not a gradient, as the signal that decides what's kept.

### Two scopes, one oracle

- **Campaign scope** = the Phase-1 blackboard, unchanged. One binary, ephemeral; `tasks`, `checkpoints`, `findings`, `symbols`, `types` die with the campaign.
- **Global scope** = new, persistent, compounding across every campaign: a knowledge/pattern store, a skill library, a lessons store, routing stats and a held-out eval set.
- **The bridge is a promotion gate that only passes what the oracle confirmed.** Oracle-verified findings distil up into global knowledge; proven campaign tools graduate to the skill library; oracle-caught failures become lessons; outcomes update routing. Retrieval runs back down: at seed/brief time the global stores inject priors into the new campaign.

The daily Phase-1 loop does not change. Phase 2 is the promotion flow out of it and the retrieval flow into it.

### How it maps onto Phase 1

Only two components are genuinely new — the generalized knowledge store and the skill library (with routing and the eval harness as smaller new stores). The rest extends machinery Phase 1 already has.

| Self-learning layer | Where it attaches in Phase 1 |
| --- | --- |
| Oracle harness | **Already built** — the Ground-truth section + the verifier + `findings.status='oracle'` |
| Verified knowledge, exact tier | **Already built** — `findings`/`symbols` at oracle status (the raw material) |
| Verified knowledge, generalized tier | **New** global store, keyed by code-shape signature, not sha256+rva |
| Skill / tool library | **New** global store beside the RE supervisor |
| Lessons / reflection | Extends the retraction + `agent_cursors.flagged_reason` machinery into durable global lessons |
| Router / policy | **New** global store; replaces the static model choice in agent frontmatter |
| Curriculum / scheduler | Enhances `seed_tasks` with global-store prioritization |
| Held-out eval | Generalizes the "prove it's worth 15×" baseline into a standing regression |

### The promotion gate (campaign → global)

- **Eligibility:** only `findings.status = 'oracle'` is a promotion candidate. Plain `verified` (agent cross-checked only) is not enough to cross — applying a fact to a *different* binary is riskier than trusting it within one, so the bar is higher.
- **Re-keying is the hard part.** A campaign fact keyed by `sha256 + rva` is inherently local. To go global it must be re-expressed binary-independently — a code-shape signature, an API-usage pattern, a struct tied to a documented type, a vendor/driver-family idiom. That re-expression plus its confidence threshold *is* the promotion rule set (the Phase-2 schema draft).
- **Single writer.** The librarian is already the single writer and distiller for the campaign; extend that role (or a dedicated archivist) to own global writes too, applying the same oracle gate it already applies to `applied` symbols. No new verdict machinery — the verifier already produces the labels.

### The retrieval flow (global → campaign)

- At **seed time**, `seed_tasks` consults the global store to prioritize: skip clusters already covered by a high-confidence pattern, target low-confidence / high-leverage first. This upgrades the existing leaf-first clustering; it doesn't replace it.
- At **task time**, the `SessionStart` / digest hooks inject the k most similar global exemplars, the relevant lessons, and the routing-recommended model/effort for the current target — the same injection path Phase 1 already uses for the board digest.
- Retrieved global knowledge enters a campaign as a **hypothesis to verify, not a fact to apply.** Version drift means a pattern confirmed elsewhere is only a lead here; it still passes through the campaign's oracle before it's trusted.

### Ownership and invariants

- **Separate store.** The global scope is its own store (its own SQLite/MCP), never another writer on the campaign DB — so campaign DBs stay disposable and the compounding asset is backed up independently.
- **Oracle-gated only.** Nothing unverified ever persists globally. The moment it does, you get confident-wrong amplification (error accumulation / collapse). This is the one non-negotiable.
- **Guard the oracle's quality.** A weak oracle teaches the system to pass the weak check — the decompiler result where build-rate rose while behavior-match fell. Global promotion must rest on behavioral/ground-truth evidence, not proxies.
- **Retraction crosses the bridge.** If the eval harness or a later refutation catches a bad global pattern, it must be demotable with its influence traceable — the same `derived_from` discipline, now cross-scope.
- **Measure on held-out.** A standing eval over frozen ground truth (the grown 20-function experiment) is what separates real improvement from self-poisoning. Keep only promotions and scaffolding changes that move the held-out number.

### Budget fit

The compounding happens in **cheap external storage, not model calls.** The knowledge base, skill library and routing stats are files/SQLite that persist across windows and across binaries for free, while the model stays frozen and metered. On the budget-paced single-agent loop the campaign and agent roles collapse, but the two-scope model still holds and matters more: each cheap window spends its tokens producing a few oracle-verified facts, and the generalizable ones graduate to a store that never costs tokens again.

### What Phase 2 defers

The campaign↔global bridge is the only genuinely new design surface; everything else reuses the Phase-1 blackboard. The next artifact is its schema: the global knowledge-store tables, the promotion rules (what graduates, how it's re-keyed binary-independently, at what confidence), and the retrieval hook (what's injected at seed/brief time, and how it re-enters the oracle).

## Caveats

- **Fast-moving features.** Agent teams are experimental, agent view is a research preview, and cross-session messaging is about two months old. Version-gated details such as concurrency defaults, setting names and the `PostToolBatch` hook may differ on your build. `SessionStart` and `Stop` are longer-standing, but check their input and output fields against your version too.
- **Research limits.** The Google study used benchmark tasks, not RE workflows. MAST's traces came from older models (GPT-4, Claude 3, Qwen2.5, CodeLlama). The 15× and 90.2% figures come from Anthropic's internal eval. Treat all of them as directional.
- **This design is not a vendor product.** The blackboard, digest hook, checkpoint gate and `cli.py` are my own design. Test claim atomicity, lease expiry and the restart runbook on a throwaway campaign before relying on them.
- **Community tool claims are unverified.** Star counts and scale claims come from project pages, and multi-instance IDA MCP forks are young.
- **Safety.** Never run agents with `--dangerously-skip-permissions` on a host with kernel debug access or real malware samples. Anthropic's C-compiler writeup recommends running such loops in a container rather than on your own machine, and native Windows has no sandboxing.

## Sources

Gathered in the original research report; the persistence section is design guidance added on top.

- [How we built our multi-agent research system — Anthropic Engineering](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Building a C compiler with a team of parallel Claudes — Anthropic Engineering](https://www.anthropic.com/engineering/building-c-compiler)
- [Towards a Science of Scaling Agent Systems — Kim et al., arXiv 2512.08296](https://arxiv.org/abs/2512.08296)
- [Why Do Multi-Agent LLM Systems Fail? — Cemri et al., arXiv 2503.13657](https://arxiv.org/abs/2503.13657)
- [MultiAgentBench overview — Emergent Mind](https://www.emergentmind.com/topics/multiagentbench)
- [Don't Build Multi-Agents — Cognition](https://cognition.com/blog/dont-build-multi-agents)
- [Agent teams — Claude Code Docs](https://code.claude.com/docs/en/agent-teams)
- [Cross-session messaging — Claude Code Docs](https://code.claude.com/docs/en/cross-session-messaging)
- [Agent view — Claude Code Docs](https://code.claude.com/docs/en/agent-view)
- [Dynamic workflows — Claude Code Docs](https://code.claude.com/docs/en/workflows)
- [Hooks guide — Claude Code Docs](https://code.claude.com/docs/en/hooks-guide)
- [Manage costs effectively — Claude Code Docs](https://code.claude.com/docs/en/costs)
- [Sandboxing — Claude Code Docs](https://code.claude.com/docs/en/sandboxing)
- [Subagents in the SDK — Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/subagents)
- [Claude Code changelog](https://code.claude.com/docs/en/changelog)
- [Claude Squad](https://github.com/smtg-ai/claude-squad) · [workmux](https://github.com/raine/workmux) · [mcp\_agent\_mail](https://github.com/dicklesworthstone/mcp_agent_mail)
- [re-mcp](https://github.com/mrexodia/re-mcp) · [ida-pro-mcp](https://github.com/mrexodia/ida-pro-mcp) · [ida-multi-mcp](https://github.com/MeroZemory/ida-multi-mcp) · [GhidraMCP](https://github.com/LaurieWired/GhidraMCP)

## Appendix: Phase 2 bridge schema (the global knowledge store)

This is the design of the campaign↔global bridge that the Phase 2 section defers — the one genuinely new surface. It is a **separate store** (its own SQLite/MCP, `global.db`), read by the seed/brief hooks and written only by the librarian/archivist through the promotion step. It never writes to a campaign's `board.db`, so campaign DBs stay disposable and this store is backed up on its own. Same conventions as the Phase-1 blackboard: epoch-second integers, parameterized SQL, WAL, `BEGIN IMMEDIATE` writes.

### Schema (DDL)

```sql
-- Generalized, binary-independent knowledge. The core of the global store.
CREATE TABLE IF NOT EXISTS knowledge (
  id                  INTEGER PRIMARY KEY,
  kind                TEXT NOT NULL,   -- 'code-shape' | 'api-usage' | 'struct' | 'idiom' | 'convention'
  descriptor          TEXT NOT NULL,   -- human-readable, e.g. WFP classify callout registration
  claim               TEXT NOT NULL,   -- what it asserts when matched
  support_count       INTEGER NOT NULL DEFAULT 1,   -- distinct campaigns/binaries that oracle-confirmed it
  contradiction_count INTEGER NOT NULL DEFAULT 0,   -- campaigns whose oracle refuted a retrieved match
  confidence          REAL NOT NULL DEFAULT 0.0,    -- derived (see Confidence)
  strongest_source    TEXT,            -- best oracle kind backing it: debugger|pdb|signature|bindiff
  status              TEXT NOT NULL DEFAULT 'provisional'
                      CHECK (status IN ('provisional','active','demoted')),
  first_seen          INTEGER NOT NULL,
  last_confirmed      INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS ix_knowledge_kind ON knowledge(kind, status);

-- The retrieval key: binary-independent representations that match a function to knowledge.
CREATE TABLE IF NOT EXISTS signatures (
  id           INTEGER PRIMARY KEY,
  knowledge_id INTEGER NOT NULL REFERENCES knowledge(id) ON DELETE CASCADE,
  sig_kind     TEXT NOT NULL,   -- 'mnemonic-ngram' | 'import-set' | 'call-target-set' | 'const-set'
  sig_hash     TEXT NOT NULL,   -- hash for exact-ish lookup
  sig_blob     TEXT,            -- the feature set itself, for fuzzy/similarity scoring
  UNIQUE (knowledge_id, sig_kind, sig_hash)
);
CREATE INDEX IF NOT EXISTS ix_signatures_lookup ON signatures(sig_kind, sig_hash);

-- Procedural memory: reusable analysis tools promoted from campaign one-offs.
CREATE TABLE IF NOT EXISTS skills (
  id            INTEGER PRIMARY KEY,
  name          TEXT NOT NULL UNIQUE,
  kind          TEXT NOT NULL,   -- 'idapython' | 'windbg-macro' | 'script'
  params        TEXT NOT NULL DEFAULT '',
  body_path     TEXT NOT NULL,   -- stored file; the store holds the pointer + metadata
  use_count     INTEGER NOT NULL DEFAULT 0,
  success_count INTEGER NOT NULL DEFAULT 0,
  status        TEXT NOT NULL DEFAULT 'candidate' CHECK (status IN ('candidate','active','retired')),
  created_at    INTEGER NOT NULL,
  last_used     INTEGER
);

-- Reflections: oracle-caught failures, injected on matching future tasks.
CREATE TABLE IF NOT EXISTS lessons (
  id            INTEGER PRIMARY KEY,
  trigger_kind  TEXT NOT NULL,   -- what makes it relevant (a sig kind, or a tag)
  trigger_hash  TEXT,
  lesson        TEXT NOT NULL,   -- what went wrong + the correction / where to look
  source_finding TEXT,           -- campaign ref it came from
  confidence    REAL NOT NULL DEFAULT 0.5,
  status        TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','retired')),
  created_at    INTEGER NOT NULL
);

-- Learned routing: input-feature bucket -> recommended approach, with outcome stats.
CREATE TABLE IF NOT EXISTS routing (
  bucket         TEXT PRIMARY KEY,  -- e.g. x64|O2|large|indirect|lowconf
  rec_model      TEXT,              -- opus | sonnet
  rec_effort     TEXT,              -- low | med | high
  rec_dynamic    INTEGER NOT NULL DEFAULT 0,  -- needs a debugger check?
  attempts       INTEGER NOT NULL DEFAULT 0,
  verify_pass    INTEGER NOT NULL DEFAULT 0,
  oracle_confirm INTEGER NOT NULL DEFAULT 0,
  avg_cost       REAL NOT NULL DEFAULT 0.0,
  updated_at     INTEGER NOT NULL
);

-- Held-out ground truth: the standing regression set.
CREATE TABLE IF NOT EXISTS eval_items (
  id            INTEGER PRIMARY KEY,
  binary_sha256 TEXT NOT NULL,
  rva           INTEGER NOT NULL,
  truth         TEXT NOT NULL,   -- confirmed name/type/behavior facts
  split         TEXT NOT NULL DEFAULT 'heldout',
  added_at      INTEGER NOT NULL,
  UNIQUE (binary_sha256, rva)
);
CREATE TABLE IF NOT EXISTS eval_runs (
  id            INTEGER PRIMARY KEY,
  ran_at        INTEGER NOT NULL,
  store_version TEXT,            -- what state of the global store was tested
  score         REAL NOT NULL,   -- oracle-confirmed accuracy on unseen
  detail        TEXT
);

-- Cross-scope provenance: every promotion and demotion. The bridge's derived_from.
CREATE TABLE IF NOT EXISTS promotions (
  id             INTEGER PRIMARY KEY,
  ts             INTEGER NOT NULL,
  action         TEXT NOT NULL CHECK (action IN ('promote','reconfirm','demote')),
  knowledge_id   INTEGER REFERENCES knowledge(id) ON DELETE SET NULL,
  source_finding TEXT,            -- campaign finding ref (sha256+rva+campaign)
  rekey_method   TEXT,            -- how the local fact became a signature
  confidence_at  REAL,
  note           TEXT
);
```

### Promotion rules (campaign → global)

- **Eligibility.** Only a campaign finding at `status = 'oracle'` is a candidate. Plain `verified` (agent cross-check only) never crosses — cross-binary application is riskier than within-binary trust.
- **Re-keying (local → binary-independent).** A `sha256 + rva` fact is local; promotion re-expresses it as one or more `signatures`. The recipe by kind:
  - *code-shape / role:* normalize the function to opcode mnemonics with operands masked, plus its import set, call-target set, and constant set; hash each as a `signatures` row. The `claim` is the oracle-confirmed role.
  - *api-usage:* the ordered set of imported calls and argument shapes that identify the behavior.
  - *struct:* tie to a documented type where one exists (then it is near-certain); otherwise a field-offset/size fingerprint.
  - *idiom / convention:* a vendor/driver-family tag plus the observed pattern.
- **Generalization threshold.** A candidate is stored immediately as `provisional` (`support_count = 1`) but is **not retrievable as a fact** until `support_count ≥ N` **distinct binaries/campaigns** (N ≈ 2–3 to start) with `contradiction_count` below a small ratio. N distinct *functions in one binary* is memorization, not generalization — it must be distinct binaries.
- **Confidence** is derived, not asserted: it rises with `support_count` and source *diversity* (distinct vendors/versions), weighted by `strongest_source` (dynamic > pdb > signature > bindiff), and **decays** if not reconfirmed over time.
- **Idempotent single writer.** Only the librarian/archivist writes here. Re-promoting an already-known fact is a `reconfirm` — it bumps `support_count` / `last_confirmed`, never duplicates. Every write logs a `promotions` row.

### Retrieval flow (global → campaign)

- **At seed time.** For each function in the target, compute its signatures and query `knowledge` (via `signatures`) above a retrieval-confidence floor. Annotate the seeded tasks — for example, *likely WFP callout, confirm* or *covered by a high-confidence pattern, low priority*. This sharpens the existing leaf-first clustering.
- **At task time.** The `SessionStart` / digest hook injects the top-k matching exemplars, any triggered `lessons`, and the `routing` recommendation for the function's feature bucket — the same `additionalContext` path Phase 1 already uses.
- **Retrieved knowledge is a hypothesis, never an answer.** A match lands in the campaign's `findings` table at `status = 'proposed'`, `source = 'global-pattern'`, carrying its `knowledge_id`. It must pass the campaign's own oracle to become `verified`/`oracle` locally. It never auto-applies to the IDB.

### The feedback that keeps it honest

- **Confirmation:** a retrieved pattern the campaign oracle confirms logs a `reconfirm` and raises the knowledge's `support_count` and confidence.
- **Contradiction:** a retrieved pattern the campaign oracle **refutes** increments `contradiction_count`. Crossing the demotion ratio flips `status` to `demoted`; the `promotions` log traces which campaign findings and symbols were influenced so they can be re-examined — the retraction cascade, extended across the bridge.
- **Eval gate on changes.** Before a batch of promotions, or a change to the thresholds or re-keying rules, goes live, run the held-out `eval_items`; accept the change only if `eval_runs.score` does not drop. This is the single control that separates compounding from self-poisoning.

### Minimum viable version (for the budget-paced solo loop)

You don't need all of it on day one. MVP = `knowledge` + `signatures` + `promotions` + the retrieval-as-hypothesis flow + a small `eval_items` set. Add `skills`, `lessons`, and `routing` later. Keep N conservative (≥ 2 distinct binaries) and the retrieval floor high early, so the store earns trust before it influences much.

### Parameters to tune

`N` (support threshold for `active`), the retrieval-confidence floor, `k` (exemplars injected per task), the confidence decay rate, and the contradiction ratio that triggers demotion. Set them conservative; loosen only when the held-out score says it is safe.

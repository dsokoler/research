# Blackboard MCP Server — Build Specification

*A coordination server for a 5–10 agent reverse-engineering campaign on one machine. This document is self-contained: an engineer or agent should be able to build the server from this alone, without the parent architecture doc.*

Version 1.1 · 2026-10-11 · *primary transport: HTTP daemon (changed from stdio in v1.0)*

---

## 1. Purpose and scope

The blackboard is the **durable shared state** for a team of Claude Code agents working one RE/research/coding campaign. It holds the task queue, work claims, findings, a symbol-naming registry, shared type definitions, mid-task checkpoints, and an event log. Agents coordinate *only* through this server plus lightweight push notifications (handled elsewhere); nothing important lives solely in an agent's context window.

**In scope:** task claiming with leases, finding/symbol/type storage, verification state machine, retraction cascade, IDB-sync bookkeeping, a per-agent event digest, and a small read-only CLI used by shell hooks.

**Out of scope (owned by other components):** the disassembler itself (a separate RE MCP server applies names to the IDA/Ghidra database), push notifications (`SendMessage`), the hooks that *call* this server, and actually running the agents. This server never talks to a disassembler, a model, or the network.

**Non-goals:** this is not a general message bus, not multi-machine, and not an adversarial multi-tenant system. Agents are cooperative; the only untrusted input is *text derived from the target binary* (see §15).

---

## 2. Design goals and invariants

These must hold. Tests in §17 exist to prove them.

1. **Atomic claims.** No two agents ever hold the same task. Claiming is a single atomic transaction.
2. **Crash-survivable work.** If an agent dies mid-task, its claim becomes reclaimable after a lease TTL, and its last checkpoint lets a fresh agent resume rather than restart.
3. **Durable across restarts.** Stopping the whole team and relaunching loses nothing that was written. State lives on disk, not in context.
4. **Single logical writer to the IDB.** Only the librarian applies names; the board tracks what has/hasn't been applied so a crash can't desync the board and the disassembler database.
5. **Provenance and retraction.** Every finding records what it rests on; retracting one cascades to everything derived from it.
6. **Graded trust.** A finding confirmed by an oracle (debugger/PDB/signature) outranks one merely cross-checked by another agent, which outranks a lone proposal.
7. **Concurrency-correct under ~10 writers.** Short write transactions, no lost updates, no deadlocks, no corruption.
8. **Idempotent replay.** Re-running the librarian's apply loop or re-applying a migration is a no-op.

---

## 3. Deployment model and concurrency (read this first — it drives everything)

### 3.1 Primary model: one long-running HTTP daemon

Run the board as **one standalone HTTP server** (FastMCP streamable-HTTP transport) that all agents and the dashboard connect to. This is chosen over stdio for one reason above all: **lifecycle decoupling.** The board is a *persistent coordination service*, not a tool an agent spawns for a quick call. A daemon outlives every individual agent — sessions end, agents crash and relaunch, pipes break — and agents simply reconnect. It is also one process to supervise, log, health-check, and hang a dashboard off, instead of N short-lived children.

```
          ┌─────────────── one host ───────────────┐
  agent re1 ─┐                                       │
  agent re2 ─┤   HTTP (127.0.0.1:8787, Bearer)       │
  agent lead ┼────────────────▶  board daemon  ──────┼──▶ board.db (WAL)
  librarian ─┤                   (server.py)         │        ▲
  verifier  ─┘                                       │        │ direct file access
                                                     │        │ (same host, WAL)
  hooks: session_brief.sh / inject_digest.sh ────────┼── cli.py (NOT via HTTP)
          └─────────────────────────────────────────┘
```

> **Correctness is unchanged from a stdio build.** SQLite in WAL mode is still the coordination layer; HTTP vs stdio only changes *who opens connections*. `BEGIN IMMEDIATE` + `busy_timeout` + the retry wrapper (§3.4, §13) all stay mandatory. Moving to HTTP is an **operational-robustness** win, not a fix to the claim logic — that was never the fragile part.

**The tradeoff you are accepting:** one HTTP daemon is a shared availability point — if it dies, all agents lose the board at once (under stdio, each agent had its own server, so one dying touched only that agent). This is a net win anyway, and is made safe by three things:
1. **No data loss on crash.** `board.db` survives a daemon crash untouched (WAL + §16 backups). A crash is a brief availability blip, not lost work.
2. **Supervised restart.** Run the daemon under something that respawns it: a systemd user unit, or a dedicated tmux window with `while true; do python server.py; echo "board exited $?, restarting"; sleep 1; done`.
3. **Client-side retry.** Agents' board calls and `cli.py` retry on transient connection errors (connection refused / reset) with short backoff, so a respawn window is a few seconds of "retrying," not a campaign failure.

**The gotcha that defeats people:** going HTTP for the *agents* does **not** remove the multi-process SQLite discipline. The hooks (`session_brief.sh`, `inject_digest.sh`, `stop_checkpoint.sh`) run `cli.py`, which opens `board.db` **directly on the file** — so there are always ≥ 2 processes on the DB (the daemon plus whatever hook is firing). Routing `cli.py` through the HTTP API instead would make hooks fail whenever the daemon blips, which is exactly when a `SessionStart` briefing matters; so keep `cli.py` on direct file access. Therefore WAL, `BEGIN IMMEDIATE`, and `busy_timeout` remain mandatory even in the HTTP build.

Host constraints (unchanged): all participants on **one host**, `board.db` on a **local filesystem**. WAL is unsafe over network filesystems (NFS/SMB) and across hosts.

### 3.2 Transport, binding, and auth

- **Transport:** FastMCP streamable-HTTP. Construct with `FastMCP("board", host="127.0.0.1", port=PORT)` and `mcp.run(transport="streamable-http")`. (SSE is the older variant; prefer streamable-HTTP.)
- **Bind `127.0.0.1` only.** Never `0.0.0.0`. The board must not be reachable off the host.
- **Static bearer token.** Require `Authorization: Bearer ${BOARD_TOKEN}` on every request; reject missing/wrong tokens with 401. This is cheap insurance against *other local processes* on a shared dev box, and it matches the RE MCP server's pattern. Enforce it in a small ASGI middleware or FastMCP's auth hook (verify the exact hook name against your installed MCP SDK — auth surfaces change between versions).
- Agents connect via `.mcp.json` (§18); `cli.py` does **not** use the token because it bypasses HTTP entirely.

### 3.3 In-server concurrency model (v1: one connection, one lock)

Inside the single daemon, serialize all database work behind one lock. At ~10 agents' request rate this costs milliseconds and sidesteps every `sqlite3`-threading subtlety.

```python
import sqlite3, threading
_conn = sqlite3.connect(DB_PATH, isolation_level=None, check_same_thread=False)
_lock = threading.Lock()         # guards ALL _conn use

# configure once at startup (same PRAGMas as §3.4)
# every operation:
def run(fn):
    with _lock:
        return fn(_conn)         # fn does its own BEGIN IMMEDIATE / COMMIT via write() (§13)
```

- Keep handlers **synchronous** so FastMCP runs them in its worker thread pool rather than blocking the event loop; the lock then serializes them. (If you make handlers `async`, you must offload DB calls to a thread yourself — more rope, no benefit here.)
- `check_same_thread=False` is required because FastMCP may dispatch handlers on different worker threads; the lock makes that safe.
- **Do not** share one connection across threads *without* the lock — `sqlite3` connections are not thread-safe for concurrent use.
- **Scale path (only if measured):** replace the single connection + lock with a thread-local connection per worker thread (each sets the §3.4 PRAGMas), and let SQLite's own WAL locking handle concurrency between them exactly as it does between the daemon and `cli.py`. Don't do this pre-emptively; the single-lock design is correct and simple, and SQLite serializes writers anyway.

### 3.4 Connection setup (daemon connection, and every `cli.py` invocation)

```python
import sqlite3
db = sqlite3.connect(DB_PATH, isolation_level=None, check_same_thread=False, timeout=0)
db.execute("PRAGMA journal_mode=WAL")       # persistent in the DB header; safe to repeat
db.execute("PRAGMA busy_timeout=5000")      # ms a blocked writer waits before SQLITE_BUSY
db.execute("PRAGMA synchronous=NORMAL")     # WAL-safe; see note
db.execute("PRAGMA foreign_keys=ON")        # OFF by default in sqlite3 — must enable per connection
db.execute("PRAGMA wal_autocheckpoint=1000")
db.row_factory = sqlite3.Row
```

- `isolation_level=None` puts Python's driver in autocommit mode so **we** control `BEGIN`/`COMMIT` explicitly (required for `BEGIN IMMEDIATE`).
- `busy_timeout=5000` is the cross-process back-pressure mechanism: when the daemon and a `cli.py` hook both want to write, the second blocks up to 5 s, then raises `sqlite3.OperationalError: database is locked`. See §13 for the retry wrapper.
- `synchronous=NORMAL` with WAL: a committed transaction survives application crash; only an OS crash / power loss can lose the last transaction(s), and the database never corrupts. Acceptable because the board is backed up and reconstructable. Use `FULL` only if you want power-loss durability at a throughput cost.

### 3.5 Transaction idioms (the core correctness rules — unchanged by transport)

- **Every write transaction begins with `BEGIN IMMEDIATE`.** This takes SQLite's write lock up front. A default deferred transaction acquires the write lock lazily on first write, which lets two transactions both start as readers and then deadlock on upgrade (a `SQLITE_BUSY` that `busy_timeout` cannot resolve). `BEGIN IMMEDIATE` eliminates that class of bug. This matters across the daemon↔`cli.py` boundary even though the daemon serializes its own calls internally.
- **Prefer single-statement `UPDATE ... RETURNING`** for claim-and-return (SQLite ≥ 3.35, 2021). It is atomic and avoids the SELECT-then-UPDATE race. A `BEGIN IMMEDIATE` block is the fallback where multiple statements must be atomic.
- **Keep write transactions short** — no model calls, no file I/O, no sleeps inside a transaction. WAL permits unlimited concurrent readers but exactly one writer at a time.
- **Reads need no transaction** under WAL (readers never block the writer and vice-versa).

Expected load is tiny (short claims/checkpoints/finding-inserts from ~10 agents plus hook reads), so contention is rare and `busy_timeout` + the §13 retry absorb it. Do not add application-level locking beyond the single daemon lock in §3.3.

### 3.6 Fallback model: stdio, one server per agent

If HTTP is unavailable in your harness, FastMCP can run stdio instead (`{"command":"python3","args":["blackboard/server.py","--stdio"]}`): Claude Code then spawns one `server.py` child per agent, each opening `board.db` directly. Correctness is identical (SQLite is still the coordination layer; the §3.4 PRAGMas and §3.5 idioms carry over, and the §3.3 in-process lock is unnecessary because each process is single-client). You lose the daemon's lifecycle decoupling, the single supervised process, and the easy dashboard. Use only if you must.

---

## 4. Tech stack and dependencies

- **Python ≥ 3.11.**
- **SQLite ≥ 3.35** (for `RETURNING`; ships with modern Python). Confirm with `sqlite3.sqlite_version`.
- **MCP SDK with FastMCP** (`pip install "mcp[cli]"`, i.e. the `mcp` package exposing `mcp.server.fastmcp.FastMCP`), with its HTTP/ASGI dependencies (`uvicorn`/`starlette`, pulled in by the SDK's streamable-HTTP support). Pin versions in `requirements.txt`.
- Standard library otherwise (`sqlite3`, `time`, `json`, `os`, `argparse`, `threading`, `hashlib` if validating digests).
- **Verify two things against your installed SDK version**, since both move between releases: (a) the exact call to serve streamable-HTTP (`FastMCP(..., host=, port=)` + `mcp.run(transport="streamable-http")`), and (b) the auth/middleware hook used to enforce the bearer token (§3.2).

No ORM. Hand-written SQL, always parameterized.

---

## 5. Conventions and data types

- **Timestamps:** integer Unix epoch seconds (`int(time.time())`). No timezones, no ISO strings.
- **`agent`:** the stable agent name (e.g. `lead`, `re1`, `librarian`). Set by launch via `AGENT_NAME`; **never** a session id. Ownership keys on this so a relaunched agent reclaims its own work.
- **`role`:** one of `lead`, `re-analyst`, `librarian`, `verifier`, `researcher`, `coder`. `tasks.role` says which role may claim a task.
- **`binary_sha256`:** lowercase hex, 64 chars. The binary's SHA-256. Pairs with `rva` to key everything, so findings/symbols survive rebasing and multiple IDBs.
- **`rva`:** integer relative virtual address.
- **`confidence`:** float in `[0.0, 1.0]`.
- **`campaign_id`:** short string identifying the campaign; stamped on every event and finding for failure attribution. From env `CAMPAIGN_ID`, default `"default"`.
- **Enums** are enforced with `CHECK` constraints (§6).
- **Text size caps:** `evidence`, `claim`, `rationale`, `tried`, `ruled_out`, `next_step` capped at 8 KiB each at the tool boundary (reject longer; agents must write long dumps to `logs/` and reference the path). `proto`/`name` capped at 512 bytes.

---

## 6. Schema (authoritative DDL)

`init_db()` runs this on startup, wrapped so it is safe to run repeatedly (`IF NOT EXISTS`). Create in this order (foreign keys reference earlier tables).

```sql
PRAGMA foreign_keys=ON;

CREATE TABLE IF NOT EXISTS schema_version (
  version INTEGER NOT NULL
);

-- Optional registry; useful for brief() and dashboards. Not load-bearing.
CREATE TABLE IF NOT EXISTS agents (
  name        TEXT PRIMARY KEY,
  role        TEXT NOT NULL,
  first_seen  INTEGER NOT NULL,
  last_seen   INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS tasks (
  id            INTEGER PRIMARY KEY,
  campaign_id   TEXT NOT NULL DEFAULT 'default',
  kind          TEXT NOT NULL,                 -- 're-cluster' | 'research-q' | 'code' | ...
  scope         TEXT NOT NULL,                 -- free text: what to work on (e.g. a cluster id, a question)
  role          TEXT NOT NULL,                 -- which role may claim this
  priority      INTEGER NOT NULL DEFAULT 0,    -- higher = sooner
  status        TEXT NOT NULL DEFAULT 'open'
                CHECK (status IN ('open','claimed','done','blocked','cancelled')),
  owner         TEXT,                          -- agent name while claimed/done
  lease_expires INTEGER,                       -- epoch s; NULL unless claimed
  created_at    INTEGER NOT NULL,
  updated_at    INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS ix_tasks_claimable ON tasks(role, status, priority DESC, id);

-- Dependency edges: task_id cannot be claimed until depends_on is 'done'.
CREATE TABLE IF NOT EXISTS deps (
  task_id    INTEGER NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  depends_on INTEGER NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  PRIMARY KEY (task_id, depends_on)
);
CREATE INDEX IF NOT EXISTS ix_deps_dependson ON deps(depends_on);

CREATE TABLE IF NOT EXISTS checkpoints (
  id         INTEGER PRIMARY KEY,
  task_id    INTEGER NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
  agent      TEXT NOT NULL,
  created_at INTEGER NOT NULL,
  tried      TEXT NOT NULL,
  ruled_out  TEXT NOT NULL,
  next_step  TEXT NOT NULL,
  artifacts  TEXT NOT NULL DEFAULT ''          -- e.g. a logs/ path
);
CREATE INDEX IF NOT EXISTS ix_checkpoints_task ON checkpoints(task_id, id DESC);
CREATE INDEX IF NOT EXISTS ix_checkpoints_agent ON checkpoints(agent, id DESC);

CREATE TABLE IF NOT EXISTS findings (
  id            INTEGER PRIMARY KEY,
  campaign_id   TEXT NOT NULL DEFAULT 'default',
  task_id       INTEGER REFERENCES tasks(id) ON DELETE SET NULL,
  agent         TEXT NOT NULL,
  binary_sha256 TEXT NOT NULL,
  rva           INTEGER NOT NULL,
  claim         TEXT NOT NULL,                 -- one atomic, checkable assertion
  evidence      TEXT NOT NULL,
  confidence    REAL NOT NULL DEFAULT 0.5 CHECK (confidence >= 0 AND confidence <= 1),
  status        TEXT NOT NULL DEFAULT 'proposed'
                CHECK (status IN ('proposed','verified','oracle','refuted')),
  source        TEXT,                          -- evidence kind when verified/oracle: 'debugger'|'pdb'|'signature'|'bindiff'|'agent'
  derived_from  INTEGER REFERENCES findings(id) ON DELETE SET NULL,  -- provenance edge
  verifier      TEXT,                          -- agent that verified/refuted
  verified_at   INTEGER,
  created_at    INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS ix_findings_loc    ON findings(binary_sha256, rva);
CREATE INDEX IF NOT EXISTS ix_findings_status ON findings(status);
CREATE INDEX IF NOT EXISTS ix_findings_derived ON findings(derived_from);

CREATE TABLE IF NOT EXISTS symbols (
  id            INTEGER PRIMARY KEY,
  binary_sha256 TEXT NOT NULL,
  rva           INTEGER NOT NULL,
  proposed_name TEXT NOT NULL,
  proposed_type TEXT NOT NULL DEFAULT '',      -- prototype / type decl
  rationale     TEXT NOT NULL DEFAULT '',
  agent         TEXT NOT NULL,
  status        TEXT NOT NULL DEFAULT 'proposed'
                CHECK (status IN ('proposed','accepted','applied','rejected','superseded')),
  applied       INTEGER NOT NULL DEFAULT 0 CHECK (applied IN (0,1)),
  applied_at    INTEGER,
  created_at    INTEGER NOT NULL,
  updated_at    INTEGER NOT NULL
);
-- At most one authoritative name per location. Proposals may collide; accepted/applied may not.
CREATE UNIQUE INDEX IF NOT EXISTS uq_symbols_authoritative
  ON symbols(binary_sha256, rva) WHERE status IN ('accepted','applied');
CREATE INDEX IF NOT EXISTS ix_symbols_loc ON symbols(binary_sha256, rva);
CREATE INDEX IF NOT EXISTS ix_symbols_pending ON symbols(status, applied);

CREATE TABLE IF NOT EXISTS types (
  id         INTEGER PRIMARY KEY,
  name       TEXT NOT NULL,
  c_decl     TEXT NOT NULL,
  owner      TEXT NOT NULL,
  version    INTEGER NOT NULL DEFAULT 1,
  created_at INTEGER NOT NULL,
  UNIQUE (name, version)
);

CREATE TABLE IF NOT EXISTS messages (            -- optional durable mail (SendMessage is primary)
  id         INTEGER PRIMARY KEY,
  to_agent   TEXT NOT NULL,
  from_agent TEXT NOT NULL,
  thread     TEXT NOT NULL DEFAULT '',
  body       TEXT NOT NULL,
  read       INTEGER NOT NULL DEFAULT 0 CHECK (read IN (0,1)),
  created_at INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS ix_messages_inbox ON messages(to_agent, read, id);

-- Append-only event log: one row per state change. Drives digests and failure attribution.
CREATE TABLE IF NOT EXISTS events (
  id          INTEGER PRIMARY KEY,
  campaign_id TEXT NOT NULL DEFAULT 'default',
  ts          INTEGER NOT NULL,
  actor       TEXT NOT NULL,                   -- agent name (or 'system')
  kind        TEXT NOT NULL,                   -- see §9 event kinds
  ref_table   TEXT,                            -- 'tasks'|'findings'|'symbols'|...
  ref_id      INTEGER,
  summary     TEXT NOT NULL DEFAULT ''         -- one-line human text for the digest
);
CREATE INDEX IF NOT EXISTS ix_events_id ON events(id);

-- Per-agent read cursor into events, for digest().
CREATE TABLE IF NOT EXISTS agent_cursors (
  agent          TEXT PRIMARY KEY,
  last_event_id  INTEGER NOT NULL DEFAULT 0,
  flagged_reason TEXT                          -- set when this agent's work rested on a retracted fact
);

-- Lead broadcasts: STOP / RESCOPE / notes. Surfaced in brief() and digest().
CREATE TABLE IF NOT EXISTS directives (
  id         INTEGER PRIMARY KEY,
  ts         INTEGER NOT NULL,
  kind       TEXT NOT NULL CHECK (kind IN ('STOP','RESCOPE','NOTE')),
  target     TEXT NOT NULL DEFAULT '*',        -- agent name or '*'
  body       TEXT NOT NULL,
  active     INTEGER NOT NULL DEFAULT 1 CHECK (active IN (0,1))
);

-- Human-review queue (from the hardening section).
CREATE TABLE IF NOT EXISTS human_queue (
  id         INTEGER PRIMARY KEY,
  ts         INTEGER NOT NULL,
  ref_table  TEXT NOT NULL,
  ref_id     INTEGER NOT NULL,
  reason     TEXT NOT NULL,
  resolved   INTEGER NOT NULL DEFAULT 0 CHECK (resolved IN (0,1))
);
```

Every mutating tool also appends to `events` **inside the same transaction** as the mutation, so the log can never disagree with state.

---

## 7. Tool API — general contract

- Each tool is a FastMCP `@mcp.tool()`. Parameters and return values are JSON-serializable.
- **Return a structured dict**, never raise to the client. On validation/known errors return `{"ok": false, "error": "<machine_code>", "detail": "<human text>"}`. On success return `{"ok": true, ...payload}`. (If you prefer exceptions, wrap every tool in a decorator that converts them — but the contract the agents see is the dict.)
- All writes go through the §13 `write()` helper (BEGIN IMMEDIATE + retry).
- Validate inputs first: enum membership, sha256 shape (`^[0-9a-f]{64}$`), size caps, confidence range. Reject with `error:"bad_input"`.
- Touch `agents(last_seen)` on any call that carries an `agent` (upsert).

---

## 8. Tool API — definitions

Each entry: signature → semantics, transaction, errors, idempotency. SQL shown where the logic is non-obvious.

### 8.1 Task lifecycle

**`seed_tasks(tasks: list[dict]) -> dict`** *(lead)*
Bulk-insert the decomposition. Each dict: `{kind, scope, role, priority?, deps?: [scope|id]}`. Insert tasks, then resolve `deps` (by prior scope or explicit id) into the `deps` table. Reject if it would create a dependency cycle (compute with a recursive reachability check before inserting edges; §12). Returns `{ok, created:[ids]}`. Emits `task_seeded` events.

**`add_task(kind, scope, role, priority=0, deps=[]) -> dict`** — single-task version of the above.

**`claim_next(agent, role) -> dict`**
Atomically claim the highest-priority claimable task for `role`. Claimable = `status='open'` **or** (`status='claimed'` and `lease_expires < now`), **and** all deps `done`. Renew: set `status='claimed'`, `owner=agent`, `lease_expires=now+LEASE`. Return the task plus its latest checkpoint (so a reclaimed task resumes).

Single-statement atomic form (preferred):
```sql
UPDATE tasks
SET status='claimed', owner=:agent, lease_expires=:now+:lease, updated_at=:now
WHERE id = (
  SELECT t.id FROM tasks t
  WHERE t.role = :role
    AND (t.status='open' OR (t.status='claimed' AND t.lease_expires < :now))
    AND NOT EXISTS (
      SELECT 1 FROM deps d JOIN tasks dt ON dt.id = d.depends_on
      WHERE d.task_id = t.id AND dt.status <> 'done')
  ORDER BY t.priority DESC, t.id
  LIMIT 1)
RETURNING id, scope, kind;
```
If no row returned → `{ok:true, task:null}`. Else fetch latest checkpoint for the id and return `{ok, task:{id,scope,kind}, last_checkpoint:{...}|null, lease_expires}`. Emit `task_claimed`.
*Idempotency:* calling again returns a **different** task (or null); it does not re-claim the same one.

**`checkpoint(agent, task_id, tried, ruled_out, next_step, artifacts='') -> dict`**
Insert a checkpoint **and** renew the lease in one transaction (doubles as heartbeat):
```sql
BEGIN IMMEDIATE;
INSERT INTO checkpoints(task_id,agent,created_at,tried,ruled_out,next_step,artifacts)
  VALUES (...);
UPDATE tasks SET lease_expires=:now+:lease, updated_at=:now
  WHERE id=:task_id AND owner=:agent AND status='claimed';
-- append event
COMMIT;
```
If the `UPDATE` affects 0 rows, the agent no longer owns the task (reclaimed/stale) → still store the checkpoint but return `{ok:true, lease_renewed:false, warning:"not_owner"}` so the agent knows to stop. Emit `checkpoint`.

**`complete_task(agent, task_id, require_verified=true) -> dict`**
Mark `done`. If `require_verified`, refuse while any finding for this task is still `proposed` (return `{ok:false,error:"unverified_findings",detail:"n pending"}`) — this is what the `TaskCompleted` hook enforces. On success set `status='done'`, clear lease. Emit `task_done`.

**`resume(agent) -> dict`**
Return this agent's claimed tasks with each one's latest checkpoint; renew their leases (so briefing counts as a heartbeat). Used by `brief`. Returns `{ok, tasks:[{task, last_checkpoint}], directives:[...], flagged_reason}`.

**`reset_stale(grace_s=0) -> dict`** *(lead only)*
Reopen every claim whose `lease_expires < now - grace_s`: set `status='open'`, clear `owner`/`lease_expires`. Checkpoints stay attached. Returns `{ok, reopened:[ids]}`. Emit `task_reset` per id. Use on restart when scaling down or after crashes.

### 8.2 Findings and verification

**`post_finding(agent, task_id, binary_sha256, rva, claim, evidence, confidence, derived_from=null) -> dict`**
Insert `status='proposed'`. Validate `derived_from` exists if given. Returns `{ok, finding_id}`. Emit `finding_posted`.

**`verify(agent, finding_id, verdict, evidence, source=null) -> dict`**
`verdict ∈ {verified, oracle, refuted}`. Set `status=verdict`, `verifier=agent`, `verified_at=now`, and `source` (required when verdict is `verified`/`oracle`; e.g. `debugger`,`pdb`,`signature`,`bindiff`,`agent`). Append the verifier's `evidence` (store alongside, or into events). **Policy guard:** reject `verdict='verified'`/`oracle` when `agent == findings.agent` (an agent may not verify its own finding) → `error:"self_verify"`. If `verdict='refuted'`, run the retraction cascade (§11) rooted at this finding. Emit `finding_verified`/`finding_refuted`.

**`retract(agent, finding_id, reason) -> dict`**
Explicit retraction (distinct from a verifier refuting). Runs the §11 cascade. Returns `{ok, refuted:[ids], symbols_flagged:[ids], agents_flagged:[names]}`. Emit `finding_retracted`.

**`find_findings(binary_sha256=null, rva=null, status=null, task_id=null, limit=200) -> dict`**
Read helper for agents/dashboards. Returns rows, newest first.

### 8.3 Symbol registry

**`propose_symbol(agent, binary_sha256, rva, name, proto='', rationale='') -> dict`**
Insert `status='proposed'`. **Conflict detection:** if other proposals/accepted entries already exist for the same `(binary_sha256, rva)` with a different `name`, return them so the proposer can negotiate: `{ok, symbol_id, conflicts:[{id,proposed_name,agent,status}]}`. Do not block on conflict — record the proposal and surface the clash. Emit `symbol_proposed`.

**`decide_symbol(agent, symbol_id, verdict) -> dict`** *(lead or verifier)*
`verdict ∈ {accept, reject}`. On `accept`: set `status='accepted'`; the partial unique index (`uq_symbols_authoritative`) enforces one authoritative name per location — if another is already accepted/applied there, catch the uniqueness error and return `error:"location_taken"` with the incumbent. On `reject`: `status='rejected'`. Emit `symbol_decided`.

**`pending_symbols() -> dict`** *(librarian)*
Return symbols with `status='accepted' AND applied=0`, oldest first. These are what the librarian must push into the IDB. Returns `{ok, symbols:[{id,binary_sha256,rva,proposed_name,proposed_type}]}`.

**`mark_applied(ids: list[int]) -> dict`** *(librarian — call only after the IDB save succeeded)*
```sql
UPDATE symbols SET status='applied', applied=1, applied_at=:now, updated_at=:now
WHERE id IN (...) AND applied=0;
```
Idempotent: re-running affects 0 rows. Emit one `symbol_applied` event per id. See §10 for the ordering contract.

### 8.4 Types

**`define_type(agent, name, c_decl) -> dict`** — insert a new row; if `name` exists, insert `version = max(version)+1` (never silently overwrite). Returns `{ok, name, version}`. Emit `type_defined`.
**`get_types(name=null) -> dict`** — latest version per name (or all versions of `name`).

### 8.5 Coordination surface

**`digest(agent) -> dict`**
Return events since this agent last read, and advance its cursor:
```sql
-- read
SELECT id, ts, actor, kind, ref_table, ref_id, summary
FROM events WHERE id > (SELECT last_event_id FROM agent_cursors WHERE agent=:a)
ORDER BY id LIMIT 200;
-- then, in a write txn:
INSERT INTO agent_cursors(agent,last_event_id) VALUES(:a,:maxid)
  ON CONFLICT(agent) DO UPDATE SET last_event_id=excluded.last_event_id;
```
Also return `flagged_reason` (and clear it). Keep summaries to one line each. Returns `{ok, events:[...], flagged_reason}`. The `inject_digest.sh` hook formats this into `additionalContext`.

**`brief(agent) -> dict`** — composed view for `SessionStart`: `resume(agent)` + active `directives` for this agent + `flagged_reason`. Returns a compact object the hook turns into the session's opening context.

**`post_directive(agent, kind, body, target='*') -> dict`** *(lead)* — insert a directive (`STOP`/`RESCOPE`/`NOTE`). Emit `directive`.

**`flag_for_human(agent, ref_table, ref_id, reason) -> dict`** / **`human_queue(resolved=false) -> dict`** / **`resolve_human(id) -> dict`** — the needs-human queue.

**`post_message` / `fetch_inbox(agent)`** — optional durable mail; mark fetched rows `read=1`.

### 8.6 Admin

**`ping() -> dict`** — liveness/readiness check. Returns `{ok:true, schema_version, campaign_id, now}` after a trivial `SELECT 1` against the DB (so it reports true readiness, not just that HTTP is up). `launch.sh` polls this before starting agents; the daemon's supervisor can use it as a health probe. Must require no side effects and the lightest possible DB touch.
**`stats() -> dict`** — counts by task status, finding status, pending symbols, queue depth per role, oldest open task age. For dashboards/monitoring.
**`checkpoint_wal() -> dict`** — run `PRAGMA wal_checkpoint(TRUNCATE)`; call before a backup.

---

## 9. Event model

Append one `events` row **in the same transaction** as each mutation. Kinds (keep this list closed; add deliberately):

`task_seeded, task_claimed, checkpoint, task_done, task_reset, finding_posted, finding_verified, finding_refuted, finding_retracted, symbol_proposed, symbol_decided, symbol_applied, type_defined, directive, human_flagged`.

`summary` is the one-liner shown in digests (e.g. `re2 claimed task 47 (IOCTL cluster)` or `verifier refuted finding 88 (+0x30 offset wrong)`). `digest()` is the only reader; cursors are per agent so each agent sees each event once.

---

## 10. IDB-sync protocol (librarian)

The librarian is the only writer to the disassembler DB. Its loop:

1. `pending_symbols()` → list of accepted, unapplied symbols.
2. For each, call the **RE MCP server** (separate component) to set the name/type in the IDB.
3. Save the IDB.
4. `mark_applied([ids])` — **only after step 3 succeeds.**

**Crash-safety contract:** if the librarian dies between 3 and 4, rows stay `applied=0` and are returned again next loop. Re-applying a name that's already set must be a no-op on the RE side (setting a symbol to the same value is idempotent), so replay is safe. **Never** call `mark_applied` before the save returns success. On restart the librarian simply runs the loop again before taking new work (see runbook in the parent doc).

The board and IDB are backed up together (board via §16; IDB snapshot right after a save) so a paired restore keeps `applied` flags consistent with the IDB.

---

## 11. Retraction cascade (algorithm)

When a finding is refuted/retracted, everything derived from it is suspect. In one `BEGIN IMMEDIATE` transaction:

```sql
-- 1. collect the transitive closure over derived_from
WITH RECURSIVE bad(id) AS (
  SELECT :root
  UNION
  SELECT f.id FROM findings f JOIN bad b ON f.derived_from = b.id
)
-- 2. mark them refuted (skip ones already refuted)
UPDATE findings SET status='refuted', verifier=:agent, verified_at=:now
  WHERE id IN (SELECT id FROM bad) AND status <> 'refuted';
```
Then, in the same transaction:
- **Flag dependent symbols:** any `symbols` row at a `(binary_sha256, rva)` that had a supporting finding now refuted → set `status='superseded'` if `accepted`/`applied` (so the librarian/lead re-examines; if already `applied`, the name stays in the IDB but is marked for review — the lead decides whether to revert). Collect these ids.
- **Flag owning agents:** for each distinct `agent`/`verifier` on a now-refuted finding, set `agent_cursors.flagged_reason` so their next `brief()`/`digest()` tells them their work rested on a retracted fact.
- **Emit** one `finding_retracted` event listing the root and count.

Return `{ok, refuted:[ids], symbols_flagged:[ids], agents_flagged:[names]}`. Guard against cycles (there should be none — `derived_from` is a DAG by construction — but the `UNION` in the recursive CTE dedupes, so a stray cycle terminates rather than loops).

---

## 12. Dependency-cycle check (for seed_tasks/add_task)

Before inserting a `deps` edge `(task_id → depends_on)`, verify `task_id` is not already reachable from `depends_on` (which would close a cycle):

```sql
WITH RECURSIVE reach(id) AS (
  SELECT :task_id
  UNION
  SELECT d.task_id FROM deps d JOIN reach r ON d.depends_on = r.id
)
SELECT 1 FROM reach WHERE id = :depends_on LIMIT 1;  -- if a row comes back, inserting the edge makes a cycle
```
If it would cycle, reject the edge with `error:"dependency_cycle"`.

---

## 13. Error handling and the write helper

```python
import sqlite3, time

class BoardError(Exception):
    def __init__(self, code, detail=""): self.code, self.detail = code, detail

def write(fn, *, retries=6):
    """Run fn(cur) inside BEGIN IMMEDIATE with bounded retry on lock contention."""
    for attempt in range(retries):
        try:
            db.execute("BEGIN IMMEDIATE")
            try:
                result = fn()
                db.execute("COMMIT")
                return result
            except Exception:
                db.execute("ROLLBACK")
                raise
        except sqlite3.OperationalError as e:
            if "locked" in str(e).lower() and attempt < retries - 1:
                time.sleep(0.05 * (2 ** attempt))   # 50ms,100,200,... jittered backoff optional
                continue
            raise
```

- `busy_timeout` handles most contention; the retry wrapper covers the residual `SQLITE_BUSY`/`SQLITE_LOCKED` after the timeout under heavy bursts.
- Every tool catches `BoardError` and validation failures and returns `{ok:false,error,detail}`. Unexpected exceptions are logged to stderr (the daemon's log) and returned as `error:"internal"`.
- Reads do not use `write()`; they query directly (under the §3.3 lock in the daemon).
- **Client-side connection retry (HTTP build).** This wrapper lives in the *callers* (agents reach the daemon through MCP, which has its own transport retry; `cli.py` needs its own). For `cli.py`: wrap each HTTP call — if it were using HTTP — in retry on `ConnectionError`/`ECONNREFUSED`/reset with short backoff (e.g. 5 tries, 0.2 s→1 s), so a daemon respawn is absorbed. Since `cli.py` here uses **direct file access**, its resilience instead comes from `busy_timeout` + the `write()` retry above; the connection-retry note applies to any *other* client you build that does talk to the daemon over HTTP (e.g. a dashboard).

---

## 14. The `cli.py` companion (used by shell hooks)

Hooks can't open an MCP session, so a tiny CLI hits the same `board.db` **directly on the file** — deliberately **not** through the HTTP daemon (§3.1 gotcha): a hook must still work during a daemon respawn, and direct file access is what makes that true. This is why the multi-process WAL discipline stays mandatory in the HTTP build. Same PRAGMas and idioms (§3.4–§3.5). Subcommands:

- `brief --agent NAME` → prints the `brief(agent)` text block to stdout (consumed by `session_brief.sh`). Renews leases (so it's a heartbeat).
- `digest --agent NAME` → prints new events, one per line; advances the cursor (consumed by `inject_digest.sh`).
- `stale-checkpoint --agent NAME --max-age SECONDS` → prints a `task_id` if the agent's newest checkpoint for a currently-claimed task is older than `max-age` **or** older than that task's newest finding; else prints nothing (consumed by `stop_checkpoint.sh` to block idle-with-stale-state).
- `reset-stale [--grace SECONDS]` → calls `reset_stale`.
- `backup --dir PATH` → `PRAGMA wal_checkpoint(TRUNCATE)` then `VACUUM INTO 'PATH/board-<ts>.db'` (or `sqlite3 .backup`). §16.
- `stats` → human-readable `stats()`.

Exit codes: `0` success; `2` is used by hook conventions to signal "block" — but the CLI itself should exit `0` and let the hook script decide; keep CLI exit codes simple (`0` ok, `1` error).

Implementation note: factor the real logic into a `board.py` module imported by both `server.py` (tool wrappers) and `cli.py` (argparse wrappers), so there's one implementation of each operation.

---

## 15. Security and trust model

- **Cooperative agents, untrusted binary text.** Agents are not adversaries to each other, but `claim`, `evidence`, `rationale`, and symbol names frequently contain **strings lifted from the target binary**, which can be attacker-controlled (malware, adversarial drivers). This server only *stores* that text; it must never *interpret* it. The risk is downstream (an agent reading it as an instruction), mitigated by the agents' own prompts — but this server must: (a) always use **parameterized SQL** (no string interpolation — a symbol name like `'); DROP TABLE` must be inert), and (b) enforce the §5 size caps so a huge embedded blob can't bloat the DB.
- **Identity is self-asserted.** `agent` is whatever the caller passes. That's fine for a local cooperative team; do not build authz on it. The bearer token (§3.2) is a coarse gate against *other local processes*, not per-agent auth — any caller holding the token can act under any `agent` name. Keep the daemon bound to `127.0.0.1`; never expose it off-host.
- **No secrets in the board.** Findings/notes should reference `logs/` paths, not paste credentials or raw key material.

---

## 16. Backup

- The board is now the campaign's memory, so protect it. Safe hot backup under WAL: `PRAGMA wal_checkpoint(TRUNCATE);` then `VACUUM INTO 'backups/board-<ts>.db';` (or `sqlite3 board.db ".backup 'backups/board-<ts>.db'"`). **A plain `cp` while agents write is unsafe** — it can miss committed data still in `board.db-wal`.
- `launch.sh` backs up on every launch; add an hourly cron for long runs.
- Pair each backup with an IDB snapshot taken right after a librarian save so a restore keeps `applied` flags consistent (§10).

---

## 17. Acceptance tests (prove the invariants)

Build these as an automated suite. Several need **real concurrent processes** (not threads) to exercise SQLite's inter-process locking — spawn N `python` workers against one temp `board.db`.

1. **No double-claim (concurrency).** Seed 50 open tasks for one role. Launch 10 processes each looping `claim_next` until the queue drains. Assert: every task claimed exactly once; union of claims = all 50; no task owned by two agents.
2. **Lease reclaim.** Agent A claims task T, writes one checkpoint, then "dies" (never completes). Advance time past the lease (or set `LEASE` tiny). Agent B `claim_next` → gets T, and `last_checkpoint` is A's. Assert B resumes from A's checkpoint.
3. **Checkpoint heartbeat.** Claim T; repeatedly `checkpoint` faster than the lease; assert T is never reclaimable while heartbeating; stop heartbeating; assert it becomes reclaimable after TTL.
4. **Checkpoint-by-non-owner.** After T is reclaimed from A, A's late `checkpoint(T)` returns `lease_renewed:false, warning:"not_owner"` and does **not** move the lease.
5. **Deps gating.** Task B depends on A. `claim_next` never returns B until A is `done`. Cycle insertion (`A→B→A`) is rejected.
6. **Verification state machine.** `post_finding` → `proposed`. Self-`verify` rejected. Different agent `verify(verified, source='debugger')` → `verified`. `complete_task(require_verified=true)` blocked while any finding `proposed`, allowed once none are.
7. **Retraction cascade.** Build F1 ← F2 ← F3 (via `derived_from`). `retract(F1)` → F1,F2,F3 all `refuted`; a symbol supported by F2 is `superseded`; the agents who authored F2/F3 get a `flagged_reason` on next `digest`.
8. **Symbol authority.** Two agents `propose_symbol` different names at the same `(sha,rva)` → both stored, conflict returned. `decide_symbol(accept)` one → `accepted`; accepting the second at the same location → `error:"location_taken"`.
9. **IDB-sync idempotency.** `pending_symbols` returns accepted+unapplied. `mark_applied(ids)` → they leave the pending set. Re-run `mark_applied(ids)` → 0 rows changed, no error. Simulate crash before `mark_applied` → symbols reappear in `pending_symbols`.
10. **Digest cursor.** Two agents; generate events; each agent's `digest` returns every event exactly once and nothing twice; a new agent's first `digest` starts from 0.
11. **Lock resilience.** Hammer with concurrent writers exceeding `busy_timeout` occasionally; assert the `write()` retry wrapper yields eventual success and zero lost updates (final counts match number of operations issued).
12. **Restart durability.** Write state, close all connections, reopen → all state present; `reset_stale` reopens only expired claims.
13. **Daemon crash / reconnect (HTTP build).** Start the daemon, run an agent-like client doing claims/checkpoints, `kill -9` the daemon mid-run, respawn it (supervisor), and assert: `board.db` is intact (no corruption, last committed txn present), `ping()` returns ready after respawn, and a client with connection-retry resumes with no lost or duplicated claims.
14. **Daemon ↔ cli.py contention.** With the daemon under write load, run `cli.py digest`/`brief` repeatedly (direct file access) and assert both make progress, no `database is locked` escapes the retry wrapper, and digest cursors remain exactly-once.

---

## 18. File layout and bootstrap

```
blackboard/
├── board.py        # core logic: init_db, every operation, connection + §3.3 lock, write() helper
├── server.py       # FastMCP wrappers: @mcp.tool() per §8; HTTP primary, --stdio fallback
├── cli.py          # argparse wrappers over board.py for the hooks — DIRECT file access, no HTTP (§14)
├── run-board.sh    # supervisor loop: `while true; do python server.py; sleep 1; done`
├── requirements.txt
├── board.db        # created on first run (gitignored; -wal/-shm siblings too)
└── tests/
    └── test_board.py
```

`cli.py` imports `board.py` and opens its **own** direct connection to `board.db` (it does not import `server.py` or start HTTP). `server.py` and `cli.py` are thus two independent DB clients — the exact multi-process case WAL is there to handle.

`server.py` skeleton (HTTP primary, stdio fallback via `--stdio`):
```python
import os, sys
from mcp.server.fastmcp import FastMCP
import board                     # all logic lives here (holds the connection + §3.3 lock)

HOST  = os.environ.get("BOARD_HOST", "127.0.0.1")
PORT  = int(os.environ.get("BOARD_PORT", "8787"))
TOKEN = os.environ["BOARD_TOKEN"]            # required; fail loudly if unset

mcp = FastMCP("board", host=HOST, port=PORT)
board.init_db()                  # idempotent; sets schema_version
# install bearer-token auth middleware here (verify hook name for your SDK version) — §3.2

@mcp.tool()
def claim_next(agent: str, role: str) -> dict:
    return board.claim_next(agent, role)    # board.* acquire the §3.3 lock internally

@mcp.tool()
def ping() -> dict:
    return board.ping()
# ... one thin, synchronous wrapper per tool in §8 ...

if __name__ == "__main__":
    if "--stdio" in sys.argv:
        mcp.run(transport="stdio")           # §3.6 fallback: one process per agent
    else:
        mcp.run(transport="streamable-http") # §3.1 primary: bind 127.0.0.1:PORT
```

- `init_db()` opens the connection with §3.4 PRAGMas, creates tables (§6) if absent, and sets `schema_version` if empty.
- Keep tool wrappers **synchronous** (§3.3). `board.*` functions take the single lock; handlers never touch `_conn` directly.
- Run under a supervisor (§3.1): systemd user unit or a `while true` tmux window.

Dashboard (optional): a separate read-only process/page that calls `stats()`/`find_findings()` over HTTP, or reads `board.db` directly with `?mode=ro`. Out of scope for this server.

---

## 19. Configuration (environment variables)

| Var | Default | Meaning |
| --- | --- | --- |
| `BOARD_DB` | `./blackboard/board.db` | DB file path (absolute recommended) |
| `BOARD_HOST` | `127.0.0.1` | HTTP bind address — keep loopback-only |
| `BOARD_PORT` | `8787` | HTTP port |
| `BOARD_TOKEN` | *(required)* | bearer token agents must send; daemon fails to start if unset |
| `BOARD_LEASE_SECONDS` | `1800` | claim lease TTL / heartbeat window |
| `BOARD_BUSY_TIMEOUT_MS` | `5000` | SQLite `busy_timeout` |
| `CAMPAIGN_ID` | `default` | stamped on events/findings |
| `BOARD_MAX_TEXT` | `8192` | per-field text cap (bytes) |

Agents reach the daemon via `.mcp.json`:
```json
{ "mcpServers": {
  "board": { "type": "http", "url": "http://127.0.0.1:8787/mcp",
             "headers": { "Authorization": "Bearer ${BOARD_TOKEN}" } } } }
```
`launch.sh` starts the daemon (under its supervisor) **first**, polls `ping()` until ready, **then** launches agents.

Lease length is a tradeoff: longer = slower crash recovery; shorter = more frequent heartbeats. 30 min renewed by every checkpoint is a sane start.

---

## 20. Migrations

Keep it simple: a `schema_version` integer and an ordered list of migration functions. On startup, if stored version < code version, run the intervening migrations inside one transaction and bump the version. Because v1 uses `CREATE TABLE IF NOT EXISTS`, the initial install is just "version 0 → 1." Never drop columns in place; add new tables/columns and backfill.

---

## 21. Decisions left to the builder

- **HTTP daemon vs stdio (§3).** Build the HTTP daemon (§3.1) — it is the primary model for lifecycle decoupling and supervisability. stdio (§3.6) is the fallback only where HTTP isn't available in the harness.
- **In-server concurrency (§3.3).** Ship the single-connection-plus-lock design. Move to thread-local connections only if profiling shows the lock is a bottleneck (it won't at ~10 agents).
- **Dashboard.** `stats()`/`find_findings()` over HTTP behind a 50-line read-only page, or `watch -n2 sqlite3 board.db "<query>"`, is enough. Not part of this server.
- **Full-text search** over findings/claims (SQLite FTS5) — nice for large campaigns, omit for v1.
- **Message table** — optional; native `SendMessage` is the primary channel. Include the table and two tools, but agents may never use them.

---

*End of specification. An implementer following §3 (HTTP daemon + concurrency), §6 (schema), §8 (tool contracts), §10–§12 (algorithms), and §17 (tests) has everything needed to build and verify the server independently. The transport is HTTP; SQLite-in-WAL remains the coordination layer, so the correctness rules in §3.4–§3.5 are mandatory regardless of transport.*

# Taint Tracking over Ghidra-Decompiled Code — Design Document

A system for tracking controlled (tainted) data through decompiled binaries,
walking the *entire* taint tree of every source, classifying every instruction
that consumes a tainted value against a catalog of vulnerable-use patterns, and
detecting when a value or memory location is controlled by more than one taint
tree — the signal for convergence bugs and race / double-fetch / TOCTOU
conditions. Primary target: vulnerability discovery in Windows kernel drivers.

The implementation strategy (§2.1) is to **extend a fork of CTADL** — Sandia's
Datalog-over-p-code taint engine, which already provides fact extraction from
Ghidra's SSA, access-path memory modeling, and compositional interprocedural
summaries — rather than build from scratch. The one artifact that currently
exists on our side is the Soufflé rule set `taint_trees.dl`, which specifies our
three differentiating capabilities (provenance, whole-tree classification,
convergence/race) and is to be integrated onto CTADL's relations.

---

## 1. Purpose and scope

### 1.1 Goal

Given a binary analyzed in Ghidra, produce a queryable model of how controlled
data flows through it, and answer three questions:

1. **Whole-tree classification.** For every tainted source, walk the complete
   forward tree of derived values and flag *any* instruction that uses a tainted
   value in a "possibly vulnerable" way — not just calls to a fixed list of
   dangerous functions, but instruction-level patterns (tainted memory index,
   tainted division denominator, tainted indirect-call target, tainted copy
   size, and so on).
2. **Provenance and convergence.** Track *which* source(s) taint each value, so
   we can detect values and memory locations reachable from **more than one**
   taint tree.
3. **Race / double-fetch / TOCTOU candidates.** Use multi-tree convergence on
   shared mutable state, reached from distinct execution contexts, to surface
   candidate race conditions — including kernel double-fetch, the dominant
   trust-boundary form of this bug class.

### 1.2 Defining characteristics (why this isn't ordinary source→sink taint)

Most taint tooling answers an *existential* question: does some source reach
some *predefined* sink. This design inverts that in two ways that drive every
downstream decision:

- It enumerates the **whole tree** and classifies **every use**, so the "sink"
  concept generalizes from named functions to instruction/operand patterns.
- It preserves **source provenance** as a first-class relation, so convergence
  across trees — the basis for race detection — is a cheap set query rather than
  a bolt-on.

The research survey (§2) confirmed that no existing tool delivers all three of
these together; this is the justification for building.

### 1.3 Non-goals

- **Proving** vulnerabilities. Output is a prioritized candidate list for
  triage, not verified bugs. This is explicit for races (see §6.4, §10).
- **Dynamic / concolic** analysis. This is static, over the decompiler's SSA.
  (An optional symbolic *confirmation* tier sits on top — §3.2 — but the core is
  static.)
- **Path feasibility.** No constraint solving in the core; reachability may
  include paths infeasible under branch conditions.
- **Uninitialized-memory infoleak** (kernel→user disclosure of uninitialized
  stack/pool bytes — Bochspwn Reloaded's domain). That is the *inverse* of taint
  (an uninitialized source declassified across the boundary), not forward taint;
  out of scope for the base analysis and noted as a future reverse-flow mode
  (§12).

### 1.4 Intended use and threat model

The tool serves two related uses, and decisions throughout favor both:

- **Offensive research** — finding vulnerabilities in third-party Windows drivers
  (1-day and 0-day), where an unprivileged user crosses the **user→kernel** trust
  boundary via IOCTL/IRP, or a remote peer crosses the **network→kernel**
  boundary.
- **Defensive hardening** — auditing one's own driver before shipping.

The trust boundary is explicit: a **source** is any value crossing from lower
privilege (user mode, the network, a file) into the analyzed code; a **finding**
is a controlled value reaching a dangerous use without an adequate gate. This
framing drives source placement (§6.6), the gate/sanitizer model (§6.6), and
severity (a METHOD_NEITHER user pointer outranks a buffered copy). Responsible
disclosure is assumed for third-party findings.

---

## 2. Background: build vs. adopt

"Store a program's dataflow in a queryable form and ask whether tainted values
reach dangerous uses" is essentially the **Code Property Graph** paradigm, and
there is mature prior art. A survey of the field established that the custom
build is justified only by the two defining characteristics in §1.2, and that
the right starting point is not an empty file but a fork of CTADL.

| Tool | OSS / license | IR substrate | (a) whole-tree per-insn classification | (b) provenance / multi-tree convergence | (c) race / double-fetch |
|---|---|---|---|---|---|
| **CTADL** (Sandia) | Yes, Apache-2.0 | Ghidra p-code (+JADX) | Partial — full reachability relation exists; catalog is ours to add | Partial — multi-color labels; convergence is a query we add | No — not built-in |
| **Ghidra Source-Sink (11.x)** | Yes (Apache-2.0; CTADL engine) | p-code varnodes | No — interactive source→sink | No | No |
| **Joern + `ghidra2cpg`** | Yes, Apache-2.0 | binary CPG (x86 best; p-code early) | Partial via CPGQL patterns | No — may-reachability only | No |
| **Binary Ninja** | No (commercial) | MLIL/HLIL SSA | Partial via scripting | No — single-tree scripting | No |
| **angr / IOCTLance** | Yes, BSD | VEX / symbolic | via symbolic checks | No | Indirectly (symbolic) |
| **BAP / Primus** | Yes, MIT | BIR | via policies | Partial (taint objects) | No |
| **DEADLINE** | Yes | LLVM IR | double-fetch specific | N/A | Yes (double-fetch) |

**CTADL is the closest prior art and the fork target** (Apache-2.0, Sandia/
NTESS). It is a Datalog static taint analyzer: the original is a Python 3.9+ CLI
over a patched **Soufflé 2.3** with C++ functors; the rewrite `ctadl-rs` is on
the **Ascent** engine in Rust and is explicitly "in flux." It is
**field-sensitive via access paths** (`AccessPath`, `MatchPrefix`,
`AccessPathNonEmptySplit` in `access_path.dl`), **context-sensitive via
call-string "hybrid inlining"** (`CallEdge(callerCtx, callSite, calleeCtx,
callee)` in `hybrid_inlining.dl`), and **interprocedural via compositional
summaries** (`SummaryFlow(fromCtx, toCtx, from, to)` in `dataflow.dl`), building
the call graph in the same fixpoint as the dataflow analysis. A Ghidra
fact-generator plugin plus `pcode_lang.dl` map P-code into its `Vertex` /
`VirtualAssign(from, to)` schema and define P-code memory-op semantics;
`PTRADD`/`PTRSUB` become access-path offsets. The index phase computes
vertex-level reachability (`LocallyReachable`, `SummaryFlow`) ahead of the
source→sink query phase (`ReachableEdge`, `LeakingSinkVertex`) — that index-phase
reachability is the substrate our provenance coloring builds on. Sources, sinks,
propagation, and sanitizers are authored as JSON **model generators**
(`find`/`where`/`model`, e.g. input `Argument(0)` → output `Return`), from a
`ctadl query --template` starter. Output is SARIF. Ghidra's own 11.3/11.4
Source-Sink feature uses CTADL as its default engine, confirming the
Datalog-over-p-code bet. DEADLINE (§6.4) is the blueprint for the double-fetch
detector, not a general taint tool.

### 2.1 Decision

**Build our analysis as an extension of a CTADL fork — do not greenfield.**
CTADL already provides the hardest ~60% (p-code fact extraction from
`HighFunction`, access-path memory modeling, call-return-matched interprocedural
summaries, SARIF). We add the three differentiators it lacks: (a) promote taint
provenance to a first-class **multi-colored** relation; (b) the **whole-tree
vulnerable-use catalog** as classification rules over CTADL's reachability
relations; (c) **convergence** as set-intersection queries and the **race /
double-fetch** rules with lock/dominance pruning. Prefer the **original Soufflé
CTADL** over `ctadl-rs` for this workload — Soufflé's subsumption, magic-set, and
native provenance maturity suit a fixpoint-heavy, provenance-first analysis.
`taint_trees.dl` is the specification of these added rules; integrating it means
mapping its relations onto CTADL's schema (§5.3).

License (Apache-2.0), engine, schema, and relation names are confirmed against
the CTADL source. **Flow-sensitivity** is resolved to high confidence: CTADL's
own "star abstraction" is a monotone, add-only, position-free fixpoint over
access-path assignments (no program points, no kill/strong-update; relation
signatures carry calling contexts but no statement order) — i.e.
**flow-insensitive on memory**, FlowDroid/Andromeda-style. Local/register values
stay def-use-precise because the input facts come from Ghidra's **SSA** p-code. A
5-line test (taint a cell, overwrite it — does the first taint survive?) gives
100% confirmation once CTADL is stood up. There is no peer-reviewed methodology
paper (cite the software + the Ghidra changelog).

**Contingency.** If the CTADL fork proves unworkable (upstream churn, a schema
that resists our additions, or extraction gaps on real drivers), fall back to a
minimal purpose-built extractor emitting the §5.1 logical facts plus the
standalone `taint_trees.dl` on plain Soufflé. This loses CTADL's access-path
memory model and summaries — which we would then have to build — but the rule
layer (provenance, classification, convergence, race) is unchanged. Keep the
logical data model (§5.1) as the stable interface so either source can feed it.

---

## 3. Architecture overview

### 3.1 Core principle — separate extraction from policy

Ghidra's decompiler already computes exact SSA def-use chains; CTADL already
turns those into Datalog facts. Taint, classification, and convergence are then
expressed as rules over those facts. Extraction is expensive and runs once per
binary; the rules are cheap and re-run freely.

### 3.2 Phases and confidence tiers

- **Phase 1 (hard, once):** compute the full provenance-annotated transitive
  closure `taintedBy(node, source)` over the dataflow graph — realized by
  coloring CTADL's reachability/summary relations with the source set. This is
  the only graph-heavy computation, and it belongs in the Datalog engine.
- **Phase 2 (cheap, many):** whole-tree classification, convergence, and
  race-candidate detection are set-algebra and joins over that materialized
  relation.

**Two tiers of confidence.** Phases 1–2 are cheap Datalog that generate *ranked
candidates* with high recall. An optional **tier 3** feeds only the top-ranked
candidates to a symbolic checker (angr / IOCTLance-style) to confirm path
feasibility and the double-fetch overlap/dependence condition, trading throughput
for precision on the few findings a human will actually examine. DEADLINE's epoch
model lives in tier 3; tiers 1–2 never block on it.

### 3.3 Pipeline

```
   target.bin
       │
       ▼
┌──────────────────┐   Ghidra HighFunction SSA (exact def-use)
│  CTADL extraction │   (pcode_lang.dl — reused, not rebuilt)
└──────────────────┘──► facts: Vertex, VirtualAssign, AccessPath, CallEdge, ...
       │
       │  + our policy facts: source models (IOCTL/network/file), vuln_pattern,
       │    context_root, propagation/sanitizer models, lock_acquire/release
       ▼
┌──────────────────┐   CTADL core  +  taint_trees.dl rule modules
│  Soufflé engine   │──► taintedBy (multi-color), vuln_use, convergent,
│  (least fixpoint) │     convergent_vuln, race_candidate, double_fetch,
└──────────────────┘     toctou_candidate                     (CSV / SARIF)
       │
       ├────────────► (tier 3, optional) symbolic confirm top candidates (angr)
       ▼
┌──────────────────┐
│  DuckDB + triage  │   ranked findings, fingerprints, dedup, baseline, diffing;
│                   │   SARIF out + Ghidra bookmarks round-trip
└──────────────────┘
```

### 3.4 Components

| Component | Role | Status |
|---|---|---|
| CTADL fork (Soufflé) | p-code fact extraction, access paths, summaries, base reachability | **Reuse/extend** (fork + validate) |
| `taint_trees.dl` rule modules | multi-color provenance, whole-tree catalog, convergence, race/double-fetch | **Drafted** (integrate onto CTADL relations) |
| Source / context models | IOCTL, WDF, network, file, env source definitions + `context_root`s | To build (policy) |
| Callback resolver | recover registered dispatch/receive callbacks (IRP, WDF queue, minifilter, WSK, NDIS) for indirect sources | To build |
| Symbolic confirmer (tier 3) | feasibility + double-fetch overlap check on top candidates (angr) | To build (optional) |
| Triage store & fingerprints | stable finding ids, dedup, baseline/suppression, scoring, SARIF + Ghidra round-trip | To build |
| Exploration store | DuckDB: load results, ad-hoc + corpus + diff queries | To build |
| Validation harness | HEVD, IOCTLance comparison, loldrivers, Bochspwn oracle, regression corpus | To build |

---

## 4. Key decisions and rationale

### 4.1 Granularity — varnode SSA as truth, high-variables as names

Taint is tracked at the **p-code varnode level in SSA form**, where Ghidra's
dataflow is *exact* (one definition, known uses, no aliasing ambiguity in
register/unique space). **HighVariables** are the human-readable attribution
layer. This is validated directly: Ghidra 11.3 added p-code-fact export to
Datalog, and CTADL's `pcode_lang.dl` operates at exactly this level — we are on
the path the Ghidra and Sandia teams paved.

### 4.2 Engine — Soufflé / Datalog (original CTADL, not ctadl-rs)

The provenance closure is a least fixpoint; Datalog computes it exactly (no
depth cap or ad-hoc cycle handling, both of which cost soundness). Precision
levers (field-sensitivity, call/return matching, points-to) are rules that
compose cleanly — the high-precision static-analysis literature (DOOP, etc.)
runs on Datalog for this reason. Among CTADL variants, the **original Soufflé**
build is preferred over the Rust/Ascent `ctadl-rs` for our needs: Soufflé's
**subsumption** (collapse redundant taint/access-path facts), **magic-set**
(demand specialization), and **native provenance** (proof trees, minor runtime
cost) directly serve a provenance-first, fixpoint-heavy workload.

### 4.3 Storage — DuckDB for exploration; not SQLite as the engine

The fixpoint stays in Soufflé. **DuckDB** is the columnar result/analytics layer
(joins over `taintedBy`, convergence set-intersections, per-instruction
classification roll-ups, cross-binary corpus). A SQLite recursive-CTE engine was
**rejected**: no efficient semi-naive evaluation, no subsumption/magic-set, and a
forced depth cap. A Kùzu/Cypher graph layer is optional path-visualization sugar,
not a replacement for the Datalog closure.

### 4.4 Fact source — reuse CTADL's p-code extraction

Rather than a bespoke extractor, reuse CTADL's Ghidra fact-generator plugin and
`pcode_lang.dl`, which map `HighFunction` SSA into `Vertex`/`VirtualAssign` and
already emit the memory (access-path), call, and dataflow facts the analysis
needs. Extend only where our rules need facts CTADL doesn't emit (e.g., explicit
`branch_cond` for TOCTOU, `lock_acquire`/`lock_release`, IOCTL transfer-method
tags). This removes the single largest build item from the original plan.

---

## 5. Data model

Two fact layers plus derived relations. Below is the **logical** model the
analysis needs; §5.3 maps it onto CTADL's actual schema (much of it is provided,
not built). The logical model is also the stable interface for the contingency
extractor (§2.1).

### 5.1 Extracted facts (EDB — mostly provided by CTADL)

| Relation (logical) | Fields | Meaning | Source |
|---|---|---|---|
| `varnode` | (vn, fn, space, size) | an SSA varnode | CTADL `Vertex` |
| `def_use` | (fn, src, dst, opcode, slot) | intra-proc input→output edge | CTADL `VirtualAssign` + op facts |
| `consumes` | (op, fn, vn, opcode, slot) | **every** op-uses-varnode (incl. STORE/CBRANCH/CALL); for a call arg, `opcode` = callee name | derived from op facts |
| `access_path` | (vn, base, offsets…) | struct/array field path (field sensitivity) | CTADL `AccessPath` |
| `call_site`/`call_arg`/`formal_param`/`call_ret`/`ret_val` | — | interprocedural binding | CTADL `CallEdge` / summaries |
| `mem_store` / `mem_load` | (op, fn, addr, val, cell) | memory access + resolved cell | CTADL memory model |
| `branch_cond` | (op, fn, cond) | CBRANCH condition varnode | **extend** |
| `calls` | (caller, callee) | static call-graph edge | CTADL call graph |
| `resolved_cell` / `global_cell` | (cell) | cells trusted for value-flow / globals | CTADL + extend |
| `lock_acquire` / `lock_release` | (op, lock) | synchronization events | **extend** (map `Ke*`/`Ex*`) |
| `ioctl_source` | (vn, method, kind) | IRP buffer by transfer method | **extend** (§6.4) |

### 5.2 Policy facts (editable — no re-extract, no recompile)

| Relation | Fields | Meaning |
|---|---|---|
| `propagates` | (opcode, slot) | which (opcode, slot) carries taint; `-1` = any |
| `vuln_pattern` | (opcode, slot, category, severity) | whole-tree classification catalog; `opcode` may be a callee name |
| `source_seed` | (src, vn, label) | source-tree roots (from source models — §6.6) |
| `sanitizes` | (vn, src) | taint cuts; `src = "*"` cuts every source |
| `context_root` | (fn, ctx) | entrypoint → execution-context kind, for race roots |

### 5.3 Mapping onto CTADL

CTADL already materializes the hard relations: its `Vertex`/`VirtualAssign`
realize `varnode`/`def_use`; `AccessPath` gives field-sensitive memory cells
(our `access_path`/`mem_*`); `CallEdge` + `SummaryFlow` give call-return-matched
interprocedural flow; `LocallyReachable`/`SummaryFlow` give the base reachability
our closure colors. Our additions are: a source **color set** on reachability
(`taintedBy`), the `vuln_pattern` catalog, `context_root`/race relations, and the
extend-only facts (`branch_cond`, `lock_*`, `ioctl_source`). These CTADL relation
names are confirmed against its source; the remaining open item is
flow-sensitivity (§10).

**Identity note.** Varnode ids are unique within one decompilation, not stable
across re-analysis. A memory `cell`/access path is a stable descriptor
(`stack:-0x18@<fn>`, `global:0x1400030a0`, `irp->field`), which is what lets
store→load value-flow and cross-context race detection work. Stable *finding*
identity is handled separately (§8.1).

---

## 6. The analysis

Rule bodies below are the conceptual spec (in `taint_trees.dl`); on integration
they are expressed over CTADL's relations.

### 6.1 Provenance closure — `tainted(vn, src)`

Reachability that **carries source identity**, so the full `taintedBy(node,
source)` relation is materialized rather than a boolean — this is the piece CTADL
does not expose by default (its query-phase `ReachableEdge`/`LeakingSinkVertex`
are source→sink oriented, but its index-phase `LocallyReachable`/`SummaryFlow`
relations are the reachability we color). Seeded from `source_seed`; propagated via intra-procedural `def_use`
filtered by `propagates`, **call-return-matched** interprocedural summaries (the
#1 precision feature — §7.1), and memory value-flow through resolved cells.
Sanitizer facts cut inside the recursion (stratified negation over EDB). A least
fixpoint — no depth cap.

```prolog
tainted(vn, s) :- source_seed(s, vn, _).
tainted(dst, s) :- tainted(src, s), def_use(_, src, dst, oc, sl),
                   ( propagates(oc, sl) ; propagates(oc, -1) ),
                   !sanitizes(dst, s), !sanitizes(dst, "*").
tainted(param, s) :- tainted(actual, s), call_arg(cs, idx, actual),
                     call_site(cs,_,callee,_,_), formal_param(callee, idx, param), ...
tainted(recv, s) :- tainted(val, s), call_site(cs,_,callee,_,_),
                    call_ret(cs, recv), ret_val(callee, val), ...      -- call-matched
tainted(lval, s) :- tainted(sval, s), mem_store(_,_,_,sval,cell),
                    resolved_cell(cell), mem_load(_,_,_,lval,cell), ...
```

### 6.2 Whole-tree classification — `vuln_use(vn, op, fn, category, severity, src)`

Every tainted value joined against every consuming instruction, matched against
the `vuln_pattern` catalog. **Genuinely novel in aggregate** — no surveyed tool
classifies every consuming instruction against an (opcode-or-callee, slot)
catalog over the full closure. Straightforward once the closure exists.

```prolog
vuln_use(vn, op, fn, cat, sev, s) :-
    tainted(vn, s), consumes(op, fn, vn, oc, sl),
    ( vuln_pattern(oc, sl, cat, sev) ; vuln_pattern(oc, -1, cat, sev) ).
```

Catalog categories: `oob_read`/`oob_write` (tainted LOAD/STORE address),
`div_by_zero` (tainted INT_DIV/REM denominator), `shift_ub`, `ptr_arith`,
`indirect_call_target`/`indirect_branch` (tainted CALLIND/BRANCHIND target),
`copy_size`/`unbounded_copy`/`alloc_size`/`format_arg`/`command_injection`/
`free_ptr`. Extend by inserting `vuln_pattern` rows.

**Integer-overflow and truncation patterns** are important for drivers and are
*two-step* rather than single-op: a tainted operand of `INT_ADD`/`INT_SUB`/
`INT_MULT` whose result then feeds a size/index/alloc sink (`int_overflow` /
`int_underflow`), or a tainted size narrowed by `SUBPIECE` or widened by
`INT_SEXT` before a copy/alloc (`trunc_size` / `signext_size` — classic driver
size-confusion). These are a join of `vuln_use` with the arithmetic op on the
path, not a bare opcode match, so they live as their own small rules keyed off
the closure.

### 6.3 Multi-tree convergence — `convergent`, `convergent_vuln`

Falls directly out of provenance: a node reached by ≥2 distinct source trees.
Expressing convergence as **set-intersection over multi-colored Datalog
provenance** is the novel engineering here.

```prolog
convergent(vn) :- tainted(vn, s1), tainted(vn, s2), ord(s1) < ord(s2).
convergent_vuln(vn, op, cat, sev, s1, s2) :-
    vuln_use(vn, op, _, cat, sev, s1), tainted(vn, s2), ord(s1) < ord(s2).
```

### 6.4 Race, double-fetch, and TOCTOU detection

This is Requirement 3, and in kernel form it is the **double-fetch** problem —
the literature is directly borrowable. The blueprint is **DEADLINE** (Xu et al.,
IEEE S&P 2018): statically collect "fetch pairs" (multi-reads of the same user
pointer), construct the connecting paths, and check each against a formal
double-fetch definition using a memory model with an **"epoch number"** so the
same user address can hold *different* values at different times (modeling
user-side volatility). We re-express fetch-pair collection as convergence in the
provenance relation. Supporting foundations: Wang et al. (USENIX Security 2017,
first static Linux detector — Coccinelle pattern + pruning taxonomy, 63% of
double-fetch *situations* in drivers, but most situations are not bugs);
Bochspwn / Bochspwn Reloaded (dynamic oracle); RacerX (SOSP 2003, static lockset
+ unlockset + belief-analysis ranking).

Because the base taint relation is flow-insensitive, the *ordering* a double-fetch
depends on (fetch → check → re-fetch → use) comes entirely from the rules here —
`branch_cond`, dominance, and DEADLINE's epoch model — not from the engine.

**Sources — seed taint at the IRP boundary per transfer method** (the low 2 bits
of the IOCTL code decide where user data lives; this defines `ioctl_source`):

| Method | Input | Output | Risk |
|---|---|---|---|
| METHOD_BUFFERED (00) | `Irp->AssociatedIrp.SystemBuffer` (kernel copy) | same | contents controlled, pointer safe |
| METHOD_IN/OUT_DIRECT (01/10) | `SystemBuffer` | `Irp->MdlAddress` (MDL) | medium |
| **METHOD_NEITHER (11)** | `Parameters.DeviceIoControl.Type3InputBuffer` | `Irp->UserBuffer` | **raw user pointers — every deref is a racy fetch; CWE-781; top priority** |

Also seed `InputBufferLength` / `OutputBufferLength` as tainted sizes.
METHOD_NEITHER is the premier arbitrary-read/write and double-fetch surface
because the kernel never snapshots the buffer. (Notably, IOCTLance's documented
blind spot is exactly METHOD_NEITHER / MDL direct sources — *our opening*.)
WDF drivers set the transfer method in queue/PnP config rather than the IOCTL
code, so there the method comes from the queue model (§6.6), not the code bits.

**Detection rules:**
```
double_fetch(cell, op_use, cat, s) :-
    -- same user cell tainted and loaded ≥2 times in one dispatch path,
    -- first reaching a check, a later one reaching a vulnerable use
    checked_cell(cell, s),                    -- a load of cell feeds a branch_cond
    mem_load(_,_,_, v2, cell),                -- a (later) re-fetch of the same cell
    vuln_use(v2, op_use, _, cat, _, s),
    user_cell(cell).                          -- METHOD_NEITHER / UserBuffer / MDL

race_candidate(cell, c1, c2) :-               -- TOCTOU generalization
    shared_cell(cell), writes_cell(cell),
    cell_ctx(cell, c1), cell_ctx(cell, c2), ord(c1) < ord(c2),
    empty_lockset_intersection(cell, c1, c2). -- lockset test
```
`shared_cell` = globals that are written, or cells reached from ≥2 contexts;
`cell_ctx` maps a cell to the execution context(s) whose tainted data touches it,
via call-graph reachability from a `context_root`.

**Pruning (where most human time goes — Wang et al.'s lesson is that most
double-fetches are not bugs):**
1. Suppress when the value is **captured into a kernel-local copy** before use
   (single `ProbeForRead` + copy into pool/stack, then uses read the copy) — the
   dominant false-positive class.
2. Suppress accesses **dominated** by a validating `ProbeForRead`/`ProbeForWrite`
   or by a lock (`KeAcquireSpinLock`/`ExAcquire*`) held across both fetches —
   i.e., non-empty may-lockset intersection (`lock_acquire`/`lock_release` facts
   + dominance).
3. Require at least one converging access to be a **use** (address, size, index,
   loop bound), not merely a re-check.
4. **Rank** by transfer method (METHOD_NEITHER first) and sink severity rather
   than hard-filter.

### 6.5 Worked example

Two independent sources in two IOCTL contexts write one shared global `G`; a
later `memcpy` size and a division denominator read `G`. Tracing the rules:
`tainted` shows both loads carry **both** source colors; `vuln_use` flags both
dangerous uses with per-source attribution; `convergent` marks both values
multi-tree; `race_candidate` flags `G` as controlled from the two distinct
contexts. Exactly the target behavior.

### 6.6 Sources: classes, driver models, entry-point resolution, and gates

**Sources are policy, so new input classes need rules only — no engine or schema
change.** A tainted byte is tracked identically regardless of origin.

**Source classes:**
- **Local / IOCTL** (§6.4) — IRP buffers by transfer method.
- **Remote / network** — `recv`/`recvfrom`/`WSARecv*` (user mode); in kernel:
  **WSK** (`WskReceive`, `WskReceiveEvent`), **NDIS** receive paths
  (`ProtocolReceiveNetBufferLists`, LWF `FilterReceiveNetBufferLists`), TDI, AFD;
  and application packet-parse entrypoints (e.g., RDP channel/PDU parsing). Seed
  the receive buffer (and packet-header length fields as tainted sizes).
- **File** — `ReadFile`/`fread` out-buffers, mapped sections.
- **Environment / registry / IPC** — `getenv`, registry value reads, ALPC.

Each class is authored as a CTADL **model generator** (JSON `find`/`where`/
`model`, e.g. input `Argument(1)` → the receive buffer), so adding a source class
is a model file plus a `context_root`, never an engine change. Our `source_seed`
relation is the union of what those generators match.

**Driver frameworks** change where sources live and how entrypoints are found:
- **WDM** — dispatch routines in `DriverObject->MajorFunction[IRP_MJ_*]`; sources
  as in §6.4.
- **WDF (KMDF/UMDF)** — `EvtIoDeviceControl` / `EvtIoRead` / `EvtIoWrite`
  callbacks registered via `WdfIoQueueCreate`; the buffer arrives through
  `WdfRequestRetrieveInputBuffer` / `...OutputBuffer` / `...InputMemory`, which
  are the source APIs to model (transfer method set in the queue/PnP config, not
  the IOCTL code).
- **Minifilters** (`FLT_OPERATION_REGISTRATION` pre/post callbacks), **WFP
  callouts** (`classifyFn`), **NDIS** (remote, above) — each a distinct
  registration pattern feeding a callback the resolver must recover.

**Entry-point / dispatch resolution** is the linchpin for both sources and
`CALLIND`. The callback resolver (§3.4) recovers handler addresses by constant /
forward propagation into the registration site — `DriverObject->
MajorFunction[n] = Handler` in `DriverEntry`, the `WDF_IO_QUEUE_CONFIG` callback
fields, the `FLT_REGISTRATION` table, WSK/NDIS dispatch tables — then registers
each recovered handler as a `context_root` and a call-graph target. Where
resolution fails (computed or obfuscated dispatch), those entrypoints and their
sources are simply missed (§10). This is also what makes **remote** sources
harder than IOCTL ones: network data almost always arrives via a registered
callback rather than a named import, so it depends on the resolver.

**Trust-boundary gates** are the kernel's own guards and are modeled as
sanitizers so a guarded flow isn't reported:
- `ProbeForRead` / `ProbeForWrite` inside a `try/except` validating a user
  pointer.
- **Capture into a kernel copy** — once user bytes are copied into pool/stack,
  later reads of the copy aren't racy (the dominant double-fetch false positive).
- `KeGetPreviousMode` / `ExGetPreviousMode == UserMode` checks gating a path.
- A bounds/validation check that **dominates** the use (ZDI-style dominance).

### 6.7 Detectability (bug-class / CWE coverage)

What the base forward-taint analysis can and can't find, at a glance. **Full** =
a single pattern over the closure catches it; **Partial** = detectable as
candidates but FP-prone or needing the noted extra machinery; **Out** = not in
the forward-taint model.

| Bug class | CWE | Detect | Mechanism / note |
|---|---|---|---|
| User-pointer arbitrary read/write (METHOD_NEITHER) | 781 / 822 | **Full** | tainted LOAD/STORE address |
| Buffer overflow via tainted size | 120 / 787 | **Full** | tainted copy/alloc size |
| Tainted array index OOB | 129 | **Full** | tainted index to LOAD/STORE |
| Integer overflow → size/index | 190 / 191 | Partial | tainted arith → size (two-step, §6.2) |
| Truncation / sign-extension size confusion | 681 / 197 | Partial | `trunc_size`/`signext_size` (§6.2) |
| Division by zero | 369 | **Full** | tainted denominator |
| Indirect call/branch hijack | 610 / class | **Full** (if target tainted) | tainted CALLIND/BRANCHIND target |
| Command injection | 78 | **Full** (if `system`-class sink) | tainted exec/command arg |
| Double-fetch / TOCTOU | 367 | Partial | re-fetch convergence + ordering (§6.4) |
| Data race on shared state | 362 | Partial (candidates) | convergence + empty lockset |
| Use-after-free / double-free | 416 / 415 | Partial | tainted free ptr; needs object-lifetime modeling |
| NULL-pointer dereference | 476 | Partial | tainted ptr may be null; needs value-range |
| Uninitialized-memory infoleak | 457 / 200 | **Out** | reverse (kernel→user) flow — future mode (§12) |
| Type confusion | 843 | **Out** | needs type-state the model doesn't track |

Remedies for the *Partial* and *Out* rows are in §6.8.

### 6.8 Closing the coverage gaps

Most *Partial* / *Out* classes above are not beyond the architecture — they are
reachable by changing the **polarity** of the same provenance engine, plus two
foundational upgrades. Only logic flaws and type confusion genuinely resist.

**Two foundational upgrades (help every class):**
- **Code-pointer points-to** — closes the source-enumeration gap (incomplete
  sources silently shrink the search space for everything else). A
  function-pointer points-to analysis in the same fixpoint resolves `CALLIND`
  targets from where code addresses are stored (`MajorFunction[n]`, WDF queue
  config, `FLT_REGISTRATION`, vtables), bounded by the address-taken set and
  type filtering. DOOP-style, Datalog-native — the highest-leverage single
  addition.
- **Ordering / dominance layer** — the `branch_cond` + dominance + epoch
  machinery built for double-fetch (§6.4) is reused by the must-init and
  lifetime analyses below. Build once, reuse three times.

**Caught by a polarity swap (cheap, high reuse):**
- **Uninitialized infoleak** (CWE-457/200) — invert the flow: color flows from
  non-zeroed allocation sites and stack frames *forward* to kernel→user copy-out
  sinks (`UserBuffer`/`OutputBuffer`), sanitized by `memset`/`RtlZeroMemory`. It
  is a *must*-analysis ("every byte copied out must be initialized"), so it needs
  the ordering layer and stratified negation; access paths give the byte/field
  granularity for partial-struct padding holes. This is the reverse-flow mode
  (§12), made concrete.
- **Constant-time / side-channel-lite** (CWE-208) — source = a "secret" color
  (key material), sink = a branch condition or a memory-access *address* that
  depends on it. A policy swap over the existing branch/address facts.
  (Microarchitectural channels — cache, Spectre — stay out: dynamic/uarch
  territory.)

**Tractable with a new primitive (medium):**
- **UAF / double-free** (CWE-416/415) — allocation-site typestate: model each
  abstract object through alloc → use → free as a state machine; UAF = a use
  reachable after free on the same object, double-free = free-after-free. Quality
  tracks points-to precision and needs the ordering layer. Candidate-level, as
  the matrix marks it.

**Genuinely resistant — assist, don't claim to solve:**
- **Logic / authorization flaws** (CWE-862/863) — policy, not propagation, so
  byte-tracking cannot find them directly. The handle is **consistency/outlier
  analysis** (RacerX belief-style): flag the handler that reaches a sensitive
  operation without the guard (`SeAccessCheck`, a privilege or `PreviousMode`
  check, handle validation) that its sibling paths apply. Heuristic, and it needs
  a notion of the expected guard — build it as a "missing-dominating-guard" rule
  and set expectations accordingly.
- **Type confusion** (CWE-843) — needs intended-vs-actual type at each use, i.e.
  type recovery on stripped drivers, which is largely intractable statically.
  Defer to symbolic/dynamic analysis or a typed build.

Net: the fix is not many new analyses but closing source enumeration
(code-pointer points-to), building the ordering layer once, and expressing
infoleak, constant-time, and UAF as alternate source/sink/color policies over the
existing engine. The residual blind spots shrink to logic flaws and type
confusion.

---

## 7. Precision and accuracy (priority-ordered)

Accuracy is determined by the rules, not the engine. All levers trade recall
against false positives. Ordered by value per the survey.

1. **CFL-reachability / balanced-parentheses call-return matching — highest
   value, non-negotiable.** Match calls and returns as balanced parentheses so
   taint cannot return to a call site that didn't invoke it (unrealizable paths).
   CTADL's summaries provide this; ensure our provenance coloring preserves it.
2. **Access-path field sensitivity (k-limited).** Essential for struct-typed
   IRP/IOCTL buffers and distinguishing `irp->field` taints. CTADL provides
   access paths; k-limit to bound label blowup. Field accesses are *also*
   balanced-parentheses matched (L_FC = L_F ∩ L_C).
3. **Tunable memory model.** Default: value-flow through decompiler-resolved
   stack/global cells (matches CTADL/Binary Ninja practice, low FP). Fallback:
   conservative "any store reaches any load" only for provably unresolved
   pointers. Ground in VSA / a-loc modeling (CodeSurfer/x86, Balakrishnan &
   Reps) when numeric+address value sets are needed (bounds reasoning); recency
   abstraction + allocation-site modeling for heap (`ExAllocatePool*`). Note the
   base engine is **flow-insensitive on memory** (no strong updates), so a taint
   is not killed by a later overwrite of the same cell; the resolved-cell model
   plus dominance pruning (§6.4) contain the resulting false positives.
4. **k=1 call-site context sensitivity.** Add, measure, deepen only on demand.
   **Skip object/type sensitivity** (OO-specific, irrelevant to C drivers).
   Precision is **non-monotone** — deeper context can *worsen* precision and
   cost (PointEval / context-tunneling results), so measure on the driver corpus
   rather than assuming k=2 > k=1.
5. **Soufflé scaling.** Subsumption (collapse redundant taint/access-path facts),
   selective magic-set (demand specialization — not always a win), native
   provenance for debugging why a tuple exists. (See §9 for the scale strategy.)
6. **Lock-aware / dominance pruning** (detailed in §6.4) — the main FP control
   for race output.

### What's harder on binary/decompiled code
Call sites don't expose typed args (recovered, often *using* taint); no reliable
types (access paths inferred from PTRADD/PTRSUB); incomplete call graph (indirect
calls); memory is registers + stack + raw addresses where values and addresses
are indistinguishable (VSA's core problem). Ghidra already recovers much of this,
which is why extracting its SSA as facts is a strong starting point.

---

## 8. Output, findings, and triage

Pruning quality is the real KPI (§10), so the output layer is a first-class
component, not an afterthought.

### 8.1 Finding identity and lifecycle

Varnode ids are unstable across re-analysis (§5.3), so a **finding fingerprint**
is derived from stable coordinates: `(module name + version or binary hash,
enclosing function key, instruction offset from function entry, vuln category,
source class)`. The fingerprint enables:
- **Dedup / grouping** — collapse the many varnode-level hits that share a
  `(function, category, sink site)` into one finding; group findings by source
  tree so one controlled input reads as one investigation.
- **Baseline / suppression** — a suppression set keyed by fingerprint carries
  "triaged-benign" decisions across runs, so re-analysis surfaces only what's new.
- **Status** — `new / triaging / true_pos / false_pos / wontfix`.

### 8.2 Confidence scoring and ranking

A scalar score ranks findings (it never hard-drops them). Inputs: source class
(METHOD_NEITHER > direct/MDL > buffered), path length (shorter = higher), memory-
resolution quality (resolved-cell > conservatively-widened), convergence degree
(multi-tree > single), and gate proximity (a dominating-but-imperfect check
lowers the score). Tier-3 symbolic confirmation (§3.2), when run, promotes
confirmed candidates to the top.

### 8.3 Evidence and formats

- **Path reconstruction** — use Soufflé's native provenance (proof trees) to emit
  the concrete source→use chain per finding, so a human sees *why* it fired.
- **SARIF** — emit our findings as SARIF (CTADL already does) for interop with
  the VSCode viewer, CodeQL tooling, and code-scanning dashboards.
- **Ghidra round-trip** — a companion script writes bookmarks/comments at each
  finding's address so results land in the analyst's actual RE workspace.
- **DuckDB** — the queryable store for ad-hoc triage, corpus roll-ups, and diffs.

Triage decisions (benign, real, needs-sanitizer) feed back into the suppression
set and, where a missing guard is identified, into new sanitizer models.

---

## 9. Performance, scale, and incrementality

- **Fact-base scale.** A real driver yields anywhere from tens to a few thousand
  functions; Soufflé handles millions of tuples, but **access-path growth and
  context cloning are the blowup risks** — bound them with k-limited access paths
  and k=1 context (§7), and lean on subsumption to collapse redundant facts.
- **Parallelism.** Soufflé compiles to parallel C++; run with `-j` sized to the
  host.
- **Demand-driven.** When only specific sinks matter, use the magic-set /
  backward-from-sinks specialization instead of the whole-program closure.
- **Incrementality.** Facts aren't stable across decompilation, so a new driver
  build re-runs extract + index; **cache per-binary results keyed by hash** and
  skip unchanged binaries. For corpus work, namespace facts per binary and keep
  results in DuckDB so cross-binary queries don't re-run analysis.
- **Budgets.** Set a per-function decompile/analysis timeout (Ghidra + CTADL) and
  flag functions that exceed it rather than stalling the run.

---

## 10. Limitations

- **Candidates, not proofs.** No path feasibility in the core, no concurrency
  verification. Static double-fetch detection is inherently **probabilistic and
  incomplete** (every surveyed approach agrees) — expect false positives
  (benign/re-validated refetches) and false negatives. **Pruning quality, not
  detection, is where human time goes — budget triage/ranking from day one.**
- **Soundness is bounded by the decompiler.** Indirect calls (`CALLIND`),
  unresolved jump tables (`BRANCHIND`), and inline-assembly fetches are missed or
  imprecise — the same inline-asm blind spot DEADLINE documents. This also gates
  entry-point and remote-source discovery (callbacks), per §6.6.
- **Context/precision tuning is non-monotone** — measure, don't assume.
- **Per-decompilation identity.** Varnode ids aren't stable across re-analysis
  (finding identity is handled by fingerprints, §8.1).
- **CTADL is flow-insensitive on memory** (high confidence — its star-abstraction
  is a monotone, position-free access-path fixpoint with no strong updates;
  confirm with a 5-line test). Practical effect: a memory taint is not killed by a
  later overwrite of the same cell — a mild false-positive source, contained by
  the resolved-cell model and dominance pruning (§7). Crucially, the **ordering**
  a double-fetch/TOCTOU depends on comes from *our* rules (`branch_cond` +
  dominance + the epoch model, §6.4), not the base engine — the design already
  assumes this. Locals stay SSA-precise via Ghidra's p-code. No peer-reviewed
  methodology paper exists; `ctadl-rs` (Ascent/Rust) is explicitly "in flux," so
  we target the original Soufflé build.

---

## 11. Implementation plan

Build order, on a CTADL fork. **Prerequisites: Appendix C.**

0. **Stand up CTADL** — license (Apache-2.0) and schema are confirmed; verify
   flow-sensitivity behavior empirically and that the Ghidra fact-generator
   plugin + `index` run on a sample driver and emit `Vertex`/`VirtualAssign`/
   `SummaryFlow`/`LocallyReachable`.
1. **Promote provenance** — color CTADL's reachability with a source set to
   produce `taintedBy` (multi-color); preserve call-return matching.
2. **Whole-tree catalog** — add `vuln_pattern` rules producing `vuln_use`,
   including the two-step integer-overflow/truncation rules (§6.2).
3. **Convergence** — `convergent` / `convergent_vuln` set-intersection queries.
4. **Source & context models + callback resolver** — IOCTL transfer-method
   sources (`ioctl_source`), WDF/minifilter/WFP/network/file/env sources (§6.6),
   `context_root`s; build the callback resolver (IRP dispatch table, WDF queue
   config, WSK/NDIS registration) for indirect/remote entrypoints.
5. **Race / double-fetch** — `branch_cond`, `lock_acquire`/`lock_release` facts;
   `double_fetch` / `race_candidate` rules; trust-boundary gate sanitizers (§6.6)
   and pruning (capture-copy, dominance, lockset, use-not-recheck) and ranking.
6. **Output & triage** — finding fingerprints, dedup/baseline, confidence
   scoring, SARIF + Ghidra round-trip, DuckDB load (§8).
7. **Validation harness** (§13).
8. **Tier-3 symbolic confirmation** (optional, §3.2) — pipe top candidates to
   angr for feasibility + double-fetch overlap checks.

### 11.1 Running it (target end-state)
```bash
# 1. extract + index facts (once per binary) via the CTADL fork on Ghidra output
ctadl import --ghidra target.gpr ; ctadl index
# 2. run our analysis (re-run freely after editing policy)
souffle -F facts/ -D out/ taint_trees.dl      # layered on CTADL relations
# 3. triage / explore
duckdb target.duckdb < load_and_query.sql
```

---

## 12. Roadmap

- Close the indirect-call / callback gap with a resolved call graph and
  code-pointer points-to (§6.8; critical for remote sources and `CALLIND` sinks).
- k-limited context sensitivity (§7.4) and deeper access-path tuning.
- DEADLINE-style symbolic refinement (epoch memory model) on double-fetch
  candidates to cut FPs further (tier 3, §3.2).
- **Reverse-flow mode** for uninitialized-memory infoleak (kernel→user) — the
  Bochspwn Reloaded class, the inverse of forward taint (§1.3, §6.8).
- **Extended analysis modes** — constant-time / secret-dependence, UAF/double-free
  typestate, and authorization-outlier detection, each an alternate policy over
  the same engine (§6.8).
- **Patch-diff-guided mode** — diff two driver versions (BinDiff / Ghidra
  version tracking) and restrict or prioritize analysis to changed functions for
  1-day hunting.
- Cross-binary corpus mode: namespace facts per binary; version-to-version
  finding diffs in DuckDB.
- Upstream-friendly packaging so our rule modules track CTADL releases.

---

## 13. Validation and benchmarking

**Corpora / oracles:**
- **HEVD** (HackSys Extreme Vulnerable Driver) — known-bug ground truth across
  classes (stack/pool overflow, write-what-where, double-fetch, UAF).
- **IOCTLance** (angr WDM scanner, 41 CVEs) — comparison baseline; its
  METHOD_NEITHER/MDL blind spot is our target delta.
- **loldrivers / vulnerable-driver catalogs** — real-world METHOD_NEITHER cases
  (e.g., RTCore64.sys).
- **Bochspwn / Bochspwn Reloaded** — dynamic oracle for double-fetch and
  infoleak ground truth.
- **Windows IOCTL Census corpus** — large signed-driver corpus for scale and for
  the honest note that static taint and symbolic (ioctlance) agree ~65% at driver
  level and diverge mainly on METHOD_NEITHER.

**Methodology:**
- Define and track **precision** (TP/(TP+FP)), **recall** (TP/known-bugs), and
  **FP-rate per driver**, measured on HEVD's known classes and a labeled CVE set.
- Maintain a **regression corpus** with expected-findings fixtures; CI runs the
  full pipeline and diffs findings (by fingerprint, §8.1) so rule edits can't
  silently regress recall or spike false positives.
- Track the **triage-cost KPI** — median findings-per-driver and time-to-triage —
  since pruning quality, not raw detection, is what makes the tool usable.

---

## Appendix A — Relation reference

EDB (§5.1, mostly CTADL-provided), policy (§5.2), and derived (§6): `tainted`,
`vuln_use`, `convergent`, `convergent_vuln`, `source_ctx`, `cell_ctx`,
`shared_cell`, `race_candidate`, `double_fetch`, `checked_cell`,
`toctou_candidate`. Full rule bodies: `taint_trees.dl` (to be mapped onto
CTADL's schema — §5.3).

## Appendix B — Components

- `taint_trees.dl` — **drafted.** Our rule modules (provenance, classification,
  convergence, race/double-fetch) + inline example policy; integrate onto CTADL.
- CTADL fork — **reuse/extend.** Fact extraction, access paths, summaries, base
  reachability, SARIF.
- Source/context models, callback resolver, symbolic confirmer, triage store,
  DuckDB layer, validation harness — *to build* (§11).

## Appendix C — Software dependencies (build & run)

Grouped by role. ⚠ marks version-critical or easy-to-miss items.

**Host**
- **Linux** (or **WSL2** on Windows) — Soufflé and CTADL do not run natively on
  Windows. Analyzing Windows drivers is static, so no Windows host is needed to
  *analyze*; Windows is only needed to *build* test drivers or run dynamic
  oracles.
- **Git**, and a Python environment manager (**pip** + **venv**, or **uv**).

**Core analysis stack (required)**
- ⚠ **JDK matching your Ghidra version** (Ghidra 11.4 → **JDK 21**; older 11.x →
  JDK 17) — Ghidra runs on the JVM.
- **Ghidra 11.4+** — decompiler, p-code SSA, and headless mode (`analyzeHeadless`).
  11.3 introduced Datalog fact export; 11.4 aligns with CTADL v0.11.0.
- **CTADL Ghidra fact-generator plugin** — installed into Ghidra (the extension
  that emits p-code facts).
- **CTADL** (original, Soufflé-based — `sandialabs/ctadl`) — the fork base / CLI.
- **Python 3.9+** — runs the CTADL CLI and our glue tooling.
- ⚠ **Patched Soufflé 2.3** — the Datalog engine CTADL *and* our `taint_trees.dl`
  run on. **Not stock Soufflé** — use CTADL's patched build.
- **C++17 toolchain** (`g++` or `clang++`, `make`, `cmake`) — Soufflé compiles
  rules to native C++. Building the patched Soufflé from source also needs
  `bison`, `flex`, `libffi`, `zlib`.

**Our rule layer & result store (required)**
- **`taint_trees.dl`** (this project) — the Soufflé rule modules.
- **DuckDB** — CLI and/or the `duckdb` Python package (result/analytics store).
- *Optional:* **DuckPGQ** extension (graph path queries); **networkx** (CTADL's
  optional graph-export plugin).

**Tier-3 symbolic confirmation (optional, §3.2)**
- **angr** (Python package) — feasibility + double-fetch overlap checks on top
  candidates.

**Validation / benchmarking (as needed, §13)**
- **HEVD** (`hacksysteam/HackSysExtremeVulnerableDriver`) — ground-truth bugs.
  Use prebuilt binaries, or build from source (needs the Windows toolchain below).
- **IOCTLance** (`zeze-zeze/ioctlance`, angr-based) — comparison baseline.
- **loldrivers** catalog — real vulnerable-driver samples (e.g. RTCore64.sys).
- *Optional, heavy:* **Bochspwn / Bochspwn Reloaded** — dynamic oracle; requires a
  Bochs instrumentation build.
- *Only if building drivers from source:* **Windows** + **Visual Studio** +
  **Windows Driver Kit (WDK/EWDK)**.

**Roadmap-only (§12)**
- **Kùzu** — only if adopting the Cypher graph-visualization alternative to DuckDB.
- **BinDiff** or Ghidra **Version Tracking** — for patch-diff-guided mode.

## References

**Engines & tools**
- CTADL (Soufflé, Apache-2.0): https://github.com/sandialabs/ctadl · architecture: https://deepwiki.com/sandialabs/ctadl · analysis model (star abstraction): https://ctadl.readthedocs.io/en/stable/star.html
- CTADL-rs (Ascent/Rust, Apache-2.0): https://github.com/sandialabs/ctadl-rs
- Ghidra Source-Sink / taint (11.4 What's New): https://www.ghidradocs.com/11.4_PUBLIC/docs/WhatsNew.html
- Joern binary support: https://joern.io/blog/binary-support-2021/ · ghidra2cpg authors: https://www.nohat.it/slides/2021/yamaguchi_ursache.pdf
- Explicit Data Dependencies (SOAP 2025): https://arxiv.org/pdf/2506.06247
- IOCTLance: https://github.com/zeze-zeze/ioctlance
- HEVD: https://github.com/hacksysteam/HackSysExtremeVulnerableDriver

**Precision & memory modeling**
- DOOP (context-sensitive points-to in Datalog): https://yanniss.github.io/doop-datalog2.0.pdf
- VSA / CodeSurfer-x86 (Balakrishnan & Reps): https://research.cs.wisc.edu/wpis/papers/cc04.pdf
- IFDS taint analysis with access paths (CTADL's flow-model lineage): https://arxiv.org/pdf/2103.16240

**Race / double-fetch / TOCTOU**
- DEADLINE (static double-fetch, S&P 2018): https://taesoo.kim/pubs/2018/xu:deadline.pdf
- Wang et al., double-fetch (USENIX Security 2017): https://www.usenix.org/conference/usenixsecurity17/technical-sessions/presentation/wang-pengfei · summary: https://blog.wohin.me/posts/paper-sec17-wang-double-fetch/
- Bochspwn (SyScan 2013, Jurczyk & Coldwind): https://j00ru.vexillium.org/slides/2013/syscan.pdf
- RacerX (SOSP 2003): https://web.stanford.edu/~engler/sosp03-racerx.ppt

**Windows driver specifics**
- Finding bugs in Windows drivers / WDM (CyberArk): https://www.cyberark.com/resources/threat-research-blog/finding-bugs-in-windows-drivers-part-1-wdm
- WinpMem driver vulnerabilities / METHOD_NEITHER (ERNW): https://static.ernw.de/whitepaper/ERNW_White_Paper_73-Analyzing_WinpMem_Driver_Vulnerabilities_1.0_signed.pdf
- Binary Ninja taint for UAF (ZDI, 2025): https://www.thezdi.com/blog/2025/3/20/mindshare-using-binary-ninja-api-to-detect-potential-use-after-free-vulnerabilities
- Static taint with Binary Ninja / MySQL Cluster (ZDI, 2022): https://www.thezdi.com/blog/2022/2/14/static-taint-analysis-using-binary-ninja-a-case-study-of-mysql-cluster-vulnerabilities

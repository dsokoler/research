# Remote Non-Admin ASLR Defeat — Outside-the-Box Ideas (Round 2)

**Companion to:** `windows-remote-infoleak-aslr-cve-dossier.md` and `windows-aslr-leak-research-program.md`.
**Problem (plain English):** learn a real memory address on a remote Windows host, **over the network** (SMB 445 / MSRPC 135+dynamic), with **non-admin** creds (or none), so ASLR/KASLR can be defeated. From one leaked/known address + fixed offsets you recover a module base.
**Scope of this file:** *new* angles only — none repeat the six in the research program. Grouped by how each cheats the problem. Feasibility stated honestly per idea.
**Compiled:** 2026-10-17.

---

## A. Remove the need for a leak entirely

### A1. Hunt for a non-ASLR module in the target — *highest-probability quick win*
ASLR only protects modules compiled `/DYNAMICBASE`. Plenty of third-party software — especially the agent-heavy, EDR-adjacent world — still ships at least one DLL or driver **without** it, which loads at its **fixed preferred base, every boot, on every machine**. If the target runs such a module, you already know an address — **zero leak required**.

- **Remote method:** fingerprint the installed product/version off its service banner, listening ports, or the *set* of RPC interface UUIDs it registers (an RpcView/`Get-RpcServer` fingerprint). Map that product → its non-ASLR module's fixed base (look up once, or reverse one copy).
- **Why it's strong:** real-world boxes are loaded with third-party agents; the odds at least one has a non-ASLR module are high. Unglamorous but often the fastest win.
- **Caveat:** gives you *that* module's base, which is only useful if your gadget chain can live there; still, a single fixed code region is often enough to pivot.

### A2. Partial-pointer overwrite / page-offset invariance
ASLR randomizes the high bits of an address; the **low 12 bits (page offset) are never randomized**, and often a few more bits are stable. If your corruption primitive can overwrite just the **low byte or two** of a pointer already present in the target, you redirect it **within the same module** to a nearby gadget/function **without knowing the base**.

- **Mechanism:** you need relative position, not absolute address — sidesteps the leak entirely.
- **Caveat:** requires a write primitive and a useful gadget within ~the same page-window of an existing pointer; most applicable once you already have limited corruption.

### A3. Blind reading via a crash/restart oracle (BROP-style) — *best "works when nothing leaks" fallback*
If a target service **auto-restarts** (SCM recovery actions, or a worker that catches exceptions and keeps serving), its **crashed-vs-didn't-crash** response is a 1-bit oracle. With even a weak read/write primitive you can brute-force addresses **and read memory byte-by-byte** purely from that oracle — defeating ASLR with **no info-disclosure bug at all**.

- **Why it fits:** works against exactly the "hardened, no leaks" services that frustrate the CVE hunt; it's remote and needs no leak.
- **Pairs with:** the 32-bit/WOW64 low-entropy brute idea (research-program extras) — fewer bits to guess.
- **Caveat:** noisy and slow; needs a reliable restart and a clean oracle signal.

---

## B. Fresh leak surfaces (new, concrete)

### B1. SMB2 server-side-copy resume keys — *high-probability quick win*
`FSCTL_SRV_REQUEST_RESUME_KEY` hands an authenticated non-admin client a **24-byte `SRV_RESUME_KEY`** identifying server-side file state for copy-chunk operations. On some stacks that token is **derived from server-side handle/object state** rather than pure random — a prime candidate for the Spec-01 differential test (vary-per-open, base-relative-per-boot).

- **Also collect (same shape, same test):** durable-handle `CreateGuid`, lease keys, change-notify/`ResumeKey` blobs, create-context response fields.
- **Why it's good:** cheap to collect over an authenticated SMB session (or null session to a readable share), and under-researched versus RPC context handles.
- **Method:** open a file → `FSCTL_SRV_REQUEST_RESUME_KEY` → store the 24 bytes → diff across opens and reboots → pointer classifier.

### B2. RPC Extended Error Information (EEInfo)
MS-RPCE lets a server attach a **chain of extended error records** to a fault — carrying computer names, process/thread context, parameter values, and sometimes **address-bearing fields** the developer never meant to expose.

- **Method:** enable extended errors on your RPC client (`RpcErrorStartEnumeration`/`RpcServerInqCallAttributes` side), deliberately **trigger faults** across remotely-reachable opnums (bad args, oversize, type mismatch), and parse the returned EEInfo chain for pointer-shaped values.
- **Why it's good:** a channel most people ignore — they only read the top-level status code.
- **Caveat:** many servers suppress EEInfo or populate only benign fields; breadth across opnums matters.

### B3. Flip the roles — make the hardened server act as a *client* to you — *richest new surface*
The target's **server** code is hardened; its **client** code usually isn't. Use coercion (PrinterBug / MS-PAR / PetitPotam-style, or simply get it to mount your SMB share / connect to your RPC endpoint) so the target connects **outbound** to a malicious server you control, then feed crafted responses and **leak from its client-side parser**.

- **Precedent:** the RDP-**client** leak CVE-2021-38665 — a client parsing attacker-controlled server data and returning heap bytes (image/vftable addresses) via the printer-cache registry path.
- **Why it's good:** opens an entirely different, softer surface than anything *listening* on 445/135; client parsers get far less scrutiny.
- **Non-admin fit:** triggering the coercion RPC needs creds; reading what the target then sends you needs nothing.

### B4. Diagnostic / performance-counter surfaces
The remote-registry RPC path to `HKEY_PERFORMANCE_DATA` (perflib) and some **WMI classes over DCOM** are queryable by authenticated non-admins and spill a surprising amount of process internals.

- **Reality:** mostly counts and working-set sizes rather than raw pointers — but occasionally a diagnostic/structure blob carries an address, and it's a cheap enumeration sweep regardless (also useful for A1 fingerprinting and B3 target selection).

---

## C. Moonshots (honest about feasibility)

### C1. Remote microarchitectural leaks
**NetSpectre** showed you *can* read a victim's memory over the network via speculative-execution timing — **no bug, no auth** — at bits-per-hour. If a Windows RPC/SMB service has a suitable speculative gadget, it's a pure-remote ASLR/data leak. If **SMB-Direct / RDMA** is enabled, **NetCAT-style** DDIO cache side channels are the RDMA cousin.

- **Feasibility:** slow, fragile, gadget-dependent — but the only options needing **neither creds nor an info-disclosure bug**.

### C2. Memory-dedup confirmation oracle
On hosts with page sharing (some Hyper-V / virtualized configs), writing a page whose content **matches an existing shared page** is measurably slower (copy-on-write fires). That becomes a **"does a page with *this exact content* exist?"** oracle — which can **confirm a guessed module base** remotely rather than leak it.

- **Feasibility:** niche (needs dedup enabled), but a genuinely different mechanism; confirms rather than leaks.

---

## D. Reframes to keep in mind (force multipliers, not standalone attacks)

- **Groom instead of leak.** Spray the target's heap via many remote allocations (repeated RPC opens/queries) to force a predictable layout, making a blind guess reliable — turns "too much entropy to brute-force" into "few enough outcomes to try." Feeds A3 and the low-entropy brute.
- **Reverse once, apply fleet-wide.** Every box at the same patch level shares identical **offsets**; only the per-boot **base** differs. Any reversing for one target transfers across the whole fleet — and composes with the research-program §0 reframe (shared system-image base per boot).

---

## E. Ranking for the stated constraints (remote, non-admin)

| Rank | Idea | Why |
|---|---|---|
| 1 | **A1 non-ASLR-module hunt** | Highest real-world hit rate; often *zero* leak needed |
| 2 | **B1 SRV_RESUME_KEY / SMB2 state blobs** | Cheap to collect, under-researched, plugs straight into the Spec-01 differential |
| 3 | **B3 role-flipping coercion** | Richest *new* surface; client parsers are soft; RDP-client precedent |
| 4 | **A3 BROP via restart oracle** | Best fallback when nothing leaks at all |
| 5 | **B2 EEInfo** | Ignored channel, quick to test across opnums |
| — | C1 / C2 moonshots | Only if creds-and-bug-free is a hard requirement |

---

## F. Defensive mirror (for the EDR build)
- Compile **everything** `/DYNAMICBASE` (+ `/HIGHENTROPYVA`); audit third-party DLLs you load for non-ASLR modules you'd inherit (A1).
- Make SMB-style state tokens / any handle-like identifier **opaque** (B1, and Spec-01).
- Strip address-bearing fields from extended error output; don't ship EEInfo in production faults (B2).
- Harden your **client** parsers as much as your server — validate server-supplied lengths, zero buffers (B3).
- Restrict perf/WMI exposure; don't let low-priv principals enumerate process internals (B4).
- Detection signatures: `FSCTL_SRV_REQUEST_RESUME_KEY` churn, fault-storms that enumerate EEInfo, coercion RPCs (spoolss opnum 65 / EFSRPC) to external hosts, service crash-loops (A3).

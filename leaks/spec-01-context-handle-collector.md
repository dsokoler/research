# Spec 01 — RPC Context-Handle Pointer-Oracle Collector

**Companion to:** `windows-aslr-leak-research-program.md` §1.
**Goal:** a non-admin, over-the-wire tool that enumerates remotely-bindable RPC interfaces, drives their handle-returning opnums, and performs differential analysis on the returned 20-byte context handles to identify ones whose body is a **server-side pointer** — yielding a reusable ASLR oracle with no CVE and no memory corruption.
**Privilege:** low-priv domain creds only (some interfaces: null/anonymous).
**Posture:** survey + differential analysis. No exploitation; the output is "this handle byte-run is base-relative," not a payload.

---

## 1. Background: the wire format you're reading

A DCE/RPC context handle (`ndr_context_handle`, C706 / MS-RPCE) is **20 bytes** on the wire:

```
offset  size  field
0       4     ContextHandleAttributes   (uint32, usually 0)
4       16    ContextHandleUuid         (the handle body)
```

The 16-byte body is **not mandated to be opaque or random.** Three common server implementations:

- **A. Opaque/random GUID** — body is a real random GUID or an HMAC. No pointer. Dead end.
- **B. Table index + cookie** — body encodes `{ small index, per-process cookie }`. No raw pointer, but the index may be enumerable (secondary interest).
- **C. Pointer-bearing** — body contains, or is trivially derived from, the address of the server-side context object (e.g., `&CtxObject` placed directly, or `&CtxObject XOR fixed_cookie`). **This is the target.** Reading the handle leaks a heap pointer; if the object lives in or near a module, its base follows.

The collector's job is to separate C from A/B empirically.

---

## 2. Architecture

```
[Enumerator] -> [Filter] -> [Binder] -> [Handle Driver] -> [Store]
                                                               |
                                              [Differential Analyzer] -> [Classifier] -> [Report]
```

### 2.1 Enumerator
Source of truth for the interface corpus. Two interchangeable backends:

- **RpcView** (GUI/CLI) — dumps, per process: interface UUID+version, opnum table, decompiled IDL, endpoints, registration flags, security-callback presence.
- **NtObjectManager (PowerShell)** — scriptable:
  - `Get-RpcServer -ProcessId <pid>` (or `-DbgHelpPath` across all) → `RpcServer` objects: `InterfaceId`, `InterfaceVersion`, `Procedures` (name, params, in/out), `Server` (hosting binary/pid).
  - `Get-RpcEndpoint` → endpoint-mapper registrations (protseq + endpoint per UUID).
  - Procedure objects expose parameter types → machine-readable identification of `FC_BIND_CONTEXT` (context-handle) out-params.

**Output schema (per interface):**
```json
{ "uuid": "...", "version": "1.0", "host": "spoolsv.exe",
  "endpoints": ["ncacn_np:\\pipe\\spoolss", "ncacn_ip_tcp:dynamic"],
  "flags": ["..."], "has_sec_callback": true,
  "open_opnums": [ { "opnum": 1, "name": "RpcOpenPrinter",
                     "handle_out_param_index": 1, "required_in_args": [...] } ] }
```

### 2.2 Filter — keep only what a remote non-admin can reach
Drop an interface unless **all** hold:
- Bound to a **remote** protseq: `ncacn_np` (445) or `ncacn_ip_tcp` (135 + dynamic). Drop `ncalrpc`-only.
- **Not** registered `RPC_IF_ALLOW_LOCAL_ONLY` (it rejects remote `ncacn_np` too — only local-origin pipe calls pass).
- Security callback, if present, does not unconditionally reject non-local (can't always know statically — fall through to the Binder, which proves it).

### 2.3 Binder
Per surviving interface, attempt a bind with escalating auth and record what's accepted:
1. anonymous / null session (note: often works to `IPC$` pipes),
2. low-priv domain user, NTLM, no sign/seal,
3. low-priv domain user with `RPC_C_AUTHN_LEVEL_PKT_PRIVACY` (many modern interfaces require it).

Toolchain: **Impacket** `dcerpc.v5.transport` (SMB named pipe + TCP, NTLM sign+seal) or the Rust `dcerpc` stack. Record the minimum auth level that binds; discard interfaces that need admin.

### 2.4 Handle Driver
For each accepted interface, call each **open/create opnum** with minimal valid args:
- Auto-fill required `[in]` args: strings → `\\<target>\x00` or `NULL` where `[unique]`; levels/flags → 0 or smallest documented; containers → zeroed minimal struct.
- Capture the **20-byte** out handle. Record `attributes` + `uuid[16]`.
- Repeat **K times** (default K=16) per opnum within the session to build the per-open sample set. Close handles politely where a close opnum exists (avoid leaking server objects / detection noise).

### 2.5 Store
Append-only records:
```json
{ "ts": "...", "boot_id": "<GUID or boot time>", "uuid": "...", "opnum": 1,
  "attributes": 0, "body_hex": "....(32 hex).....", "auth_level": "pkt_privacy" }
```
`boot_id` is critical — the cross-reboot differential keys on it. Capture it from an unauthenticated/neutral source you control, or tag runs manually per reboot.

---

## 3. Differential Analyzer

Three tests, run over the stored samples for each `(uuid, opnum)`:

### 3.1 Per-open variance (within one boot)
- Compute, byte-wise over the 16-byte body across the K samples, which bytes **vary** vs **stay constant**.
- A body that **varies per open** and whose varying region is pointer-shaped → **heap pointer per object** (strongest signal — a fresh allocation each open).
- A body **constant across opens** but pointer-shaped → may be a singleton object's address (still useful; confirm with §3.3).

### 3.2 Cross-reboot base-stability
- Compare the same `(uuid, opnum)` body across **different `boot_id`s**:
  - Varies every boot, within a plausible module/heap span → **base-relative** (ASLR'd). This is the oracle.
  - Identical across boots → a non-randomized constant (index/cookie), not a pointer.

### 3.3 Correlated-pair test (recover the offset)
- Open **two** handles whose server objects are known to be related (e.g., two printers, parent/child), or open the same object twice.
- If body_A − body_B equals a small fixed delta (object size / array stride), both are pointers into the same region → confirms pointer semantics and gives you the stride.
- To recover an absolute **module base**: you need `offset = ptr − base`. Options: (a) reverse the server once to learn where the object is allocated (heap vs a module-global), (b) correlate against a second, independent leak whose base you already know (see §5 fuzzer / §0 reframe — leak a known system-image pointer and compare ranges).

---

## 4. Classifier — "is this body a pointer?"

Apply to the 16-byte body (try both 8-byte halves, and the body as `{u32 attr-like, u64 ptr, u32}` framings):

**x64 user-mode pointer heuristic** — a candidate 8-byte value `v` is pointer-shaped if:
- `v & 0xFFFF_0000_0000_0000 == 0` (top 16 bits clear; canonical 48-bit user VA), and
- `0x0000_0000_0001_0000 ≤ v ≤ 0x0000_7FFF_FFFF_FFFF`, and
- `v % 8 == 0` (pointer alignment; 16-byte alignment is an even stronger hint for heap blocks), and
- it is **not** a small enumerable integer (rules out index/cookie), and
- it satisfies the §3 variance/base-stability behaviour.

**Kernel pointer heuristic** (for pool-backed objects): `v & 0xFFFF_8000_0000_0000 == 0xFFFF_8000_0000_0000` (sign-extended high half) and aligned.

**Confidence tiers:**
- **High** — pointer-shaped **and** base-relative across reboots **and** per-open variance (or correlated-pair delta matches a stride).
- **Medium** — pointer-shaped and base-relative, but only one of the behavioural tests.
- **Low** — pointer-shaped once, no behavioural confirmation (could be coincidence).
- **Reject** — fails the shape test, or is reboot-stable (constant), or small-integer-like.

---

## 5. Report schema (per promising interface)

```json
{ "uuid": "...", "version": "...", "host": "...", "endpoint": "ncacn_np:\\pipe\\...",
  "min_auth": "pkt_privacy", "opnum": 1, "opname": "...",
  "body_framing": "u32|u64_ptr|u32", "ptr_offset_in_body": 4,
  "classification": "High",
  "behaviour": { "per_open_variance": true, "reboot_base_relative": true, "pair_delta": "0x2A0" },
  "leaked_region": "heap|image:<module?>",
  "base_recovery": "offset unknown — reverse server, or correlate vs known system-image leak",
  "samples": ["<body_hex> @ boot A", "<body_hex> @ boot B"] }
```

---

## 6. Seed interfaces to validate first (candidates, confirm empirically)

These are **remotely-bindable interfaces that return context handles** — good first targets to run the collector against. Inclusion here is "worth testing," **not** a claim that each leaks a pointer; the differential proves it:

| Interface | UUID | Endpoint | Open opnum(s) returning a handle |
|---|---|---|---|
| spoolss (MS-RPRN) | `12345678-1234-ABCD-EF00-0123456789AB` | `\pipe\spoolss` | `RpcOpenPrinter` / `RpcOpenPrinterEx` |
| LSARPC (MS-LSAD) | `12345778-1234-ABCD-EF00-0123456789AB` | `\pipe\lsarpc` | `LsarOpenPolicy2` (policy handle) |
| SAMR (MS-SAMR) | `12345778-1234-ABCD-EF00-0123456789AC` | `\pipe\samr` | `SamrConnect*` / `SamrOpenDomain` / `SamrOpenUser` |
| SCMR (MS-SCMR) | `367ABB81-9844-35F1-AD32-98F038001003` | `\pipe\svcctl` | `ROpenSCManagerW` / `ROpenServiceW` |
| Winreg (MS-RRP) | `338CD001-2244-31F1-AAAA-900038001003` | `\pipe\winreg` | `OpenHKLM` etc. (key handle) |
| eventlog (MS-EVEN) | `82273FDC-E32A-18C3-3F78-827929DC23EA` | `\pipe\eventlog` | `ElfrOpenELW` (log handle) |

Also run against **third-party** agents (AV / backup / **EDR**) discovered by the enumerator — far more likely to return raw pointer handles with weak security callbacks (see §9).

> Note on modern MS hardening: Microsoft has been migrating these to opaque/indexed handles over time, so expect many to classify **Reject**. The value is the survivors + third-party interfaces.

---

## 7. Operational notes

- **Idempotency / cleanliness:** always call the matching close opnum; don't leak server-side objects (resource + detection cost).
- **Rate:** a client that opens a handle, reads nothing, and reconnects rapidly is the detectable artifact — throttle, and vary source.
- **Boot tagging:** the reboot differential is the backbone; make `boot_id` capture reliable (query system boot time via a neutral path, or gate runs behind a manual "new boot" flag).
- **Target set:** run against a matrix of OS builds (2016 → 2025 / 11 24H2) — handle implementations differ by build, and a Reject on 24H2 may be a High on 2016.

---

## 8. Success criteria

A single `(uuid, opnum)` classified **High** = a remote, non-admin, reusable pointer leak. Combined with the §0 reframe, if the leaked region is a shared system image (or you can correlate it to one), you have a per-boot ASLR base usable against *any* process on the host — the program's primary objective.

---

## 9. Defensive mirror (for the EDR build)

Everything above is a checklist of what **not** to ship:
- Context handles must be **opaque** (random GUID or table-index + per-boot-random cookie), never `&object`.
- Register a **security callback** that rejects non-local / unauthenticated callers where remote isn't required (`RpcServerInqCallAttributes` → `IsClientLocal`).
- Require `RPC_C_AUTHN_LEVEL_PKT_PRIVACY`.
- Don't bind `ncacn_ip_tcp` unless remote use is a requirement.
- Detection: flag empty-handle open/reconnect churn against your own and system interfaces.

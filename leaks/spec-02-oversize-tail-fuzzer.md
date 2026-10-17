# Spec 02 — Oversize Out-Buffer / `size_is` Tail-Leak Fuzzer

**Companion to:** `windows-aslr-leak-research-program.md` §5.
**Goal:** systematically find **uninitialized-memory-tail** leaks across remotely-reachable RPC and SMB2 out-buffers — the shared primitive behind the Print Spooler `pcbNeeded` over-allocation *and* SMBleed. Catches leaks Microsoft rated `AV:L`-by-assumption or never found.
**Privilege:** low-priv domain creds (some SMB2 paths: null session).
**Posture:** differential fuzzing for information disclosure. Finds "this out-buffer tail returns uninitialized heap/pool with pointers," not an exploit.
**⚠️ Lab only:** oversize/size-mismatch inputs can fault services (DoS). Run against snapshotted VMs you own.

---

## 1. The unifying primitive

Both original prongs are the same bug shape:

> The server allocates an output buffer **larger than it fills**, and returns the **uninitialized tail** to the caller.

- **Spooler `pcbNeeded` idiom:** call once with `cbBuf=0` → server returns required size in `pcbNeeded` → call again with a buffer ≥ `pcbNeeded`. If the server over-reports, or fills less than it allocated, the gap is uninitialized heap.
- **SMBleed (CVE-2020-1206):** SMB2 compression — set `OriginalCompressedSegmentSize` larger than the real decompressed data; `srv2.sys` allocates that size, fills only the real data, and the uninitialized pool tail is returned.

The fuzzer makes this a program: for every client-sizeable out-buffer on every reachable opnum, request more than the server fills and inspect the tail.

---

## 2. Architecture

```
[Corpus Builder] -> [Size-Param Identifier] -> [Mutator] -> [Caller] -> [Tail Differential] -> [Classifier] -> [Triage]
     (RPC + SMB2 lanes share the back half: Tail Differential -> Classifier -> Triage)
```

---

## 3. RPC lane

### 3.1 Corpus Builder
Reuse Spec-01's enumerator (RpcView / `Get-RpcServer`). Keep interfaces that (a) bind on a remote protseq and (b) accept a non-admin bind (reuse Spec-01's Binder + its accepted-auth record). For each, pull the IDL/procedure list.

### 3.2 Size-Param Identifier
From the IDL, flag opnums with a client-influenced output size. NDR constructs to match:

| Construct | IDL marker | Why it leaks |
|---|---|---|
| Conformant array | `[size_is(n)]` on `[out]`/`[in,out]` | server allocates `n`; may fill < `n` |
| Varying array | `[length_is(l)]` with `length < size` | bytes in `[length, size)` uninitialized |
| Conformant-varying | `[size_is(n), length_is(l)]` | gap `[l, n)` |
| Two-call size | `[in] long cbBuf, [out] long *pcbNeeded, [out,size_is(cbBuf)] byte *buf` | classic spooler pattern |
| Byte-count blob | `[out, size_is(*pcbReturned)]` | server writes count, buffer tail may be stale |

Record per candidate: the `[in]` or `[in,out]` field that governs size, the out-buffer param, and any level/flags fields that change fill behaviour.

### 3.3 Mutator (RPC)
For each candidate, generate calls that create a **fill < allocation** condition:
- **Oversize:** set the governing `[in]` size to `real_needed + Δ` for `Δ ∈ {0x10, 0x100, 0x1000, 0x10000}`.
- **Two-call abuse:** do the sizing call, then pass `cbBuf = pcbNeeded + Δ`.
- **Length/size split:** where both exist, request large `size_is`, induce small `length_is` (fewer records → bigger gap). Vary the `Level` field — different levels fill different amounts.
- **Count mismatch:** request N records, arrange for the server to return < N (filtered enumerations), inspect the unused record slots.

Keep inputs otherwise **valid** (pass NDR validation) — the goal is a semantic over-allocation, not a malformed packet the stub rejects.

### 3.4 Caller
Marshal via Impacket `dcerpc.v5` (or the Rust `dcerpc` stack) at the recorded min auth level. Capture the full returned out-buffer plus the server's reported data length (`pcbNeeded`/`pcbReturned`/record count). Handle RPC faults as "no leak / crash" (log, back off).

---

## 4. SMB2 lane

Shares the back half (§5–6). Three sub-surfaces, all non-admin (or null session where the share permits):

### 4.1 Compression / transform header (the SMBleed surface)
- `SMB2 COMPRESSION_TRANSFORM_HEADER`: `ProtocolId(0xFC 'S''M''B')`, `OriginalCompressedSegmentSize`, `CompressionAlgorithm`, `Flags`, `Offset`.
- Mutation: set `OriginalCompressedSegmentSize` **larger** than the true decompressed length; server allocates that, fills the real data, returns the uninitialized pool tail. Requires compression negotiated in `NEGOTIATE` (SMB 3.1.1 + `SMB2_COMPRESSION_CAPABILITIES`).

### 4.2 `QUERY_INFO` output buffer
- `SMB2 QUERY_INFO`: `OutputBufferLength` is client-set. Request a large output for info classes whose server-side fill is variable (file/filesystem/security/quota info). Inspect bytes beyond the structure the server actually populated.

### 4.3 `IOCTL` output
- `SMB2 IOCTL`: `MaxOutputResponse` client-set. For FSCTLs that return variable-length output, oversize `MaxOutputResponse` and inspect the tail past the real payload.

Toolchain: an SMB2 client you control at the packet level (e.g., Impacket SMB2 primitives extended to set these fields directly, or a custom async SMB2 client).

---

## 5. Tail Differential (shared engine)

For each response, isolate the **tail** = `buffer[reported_data_len : requested_len]` (RPC) or `buffer[real_payload_len : OriginalSize]` (SMB2), then:

1. **Zero check:** all-zero tail → server zero-inits → no leak. Discard.
2. **Run-to-run variance:** repeat the identical call **M times** (default M=8). A tail that **changes between identical calls** is uninitialized memory (the discriminator vs benign fixed padding).
3. **Entropy / structure:** high-entropy or pointer-shaped tail > constant padding.

Only tails that are **non-zero AND vary run-to-run** proceed to the classifier.

---

## 6. Classifier (shared with Spec-01)

Scan the varying tail for pointer-shaped 8-byte values (same heuristic as Spec-01 §4):
- user VA: top 16 bits clear, `0x10000 ≤ v ≤ 0x7FFF_FFFFFFFF`, 8-aligned;
- kernel VA: `0xFFFF8…` sign-extended, aligned (expected for `srv2.sys` pool tails — KASLR).

Cross-reference recovered pointers against known module ranges where available (Spec-01 correlation, or a known system-image leak) to label **heap / image:<module> / kernel-pool**.

**Confidence:**
- **High** — varying tail with ≥1 pointer-shaped value stable in *role* across runs (e.g., same offset always holds a kernelbase pointer).
- **Medium** — varying tail, pointer-shaped values but inconsistent.
- **Low** — varying tail, no clear pointers (still an info leak; lower ASLR value).

---

## 7. Triage output

```json
{ "lane": "rpc|smb2",
  "surface": "uuid:opnum | SMB2_COMPRESSION | QUERY_INFO:<class> | IOCTL:<fsctl>",
  "min_auth": "pkt_privacy|null_session|...",
  "trigger": { "governing_field": "cbBuf", "delta": "0x1000", "level": 2 },
  "tail_offset": 1280, "tail_varies": true,
  "pointers": [ { "offset": 1312, "value_sample": "0x00007ff8...", "region": "image:?", "role_stable": true } ],
  "confidence": "High",
  "repro": "sizing call -> oversize call(cbBuf=pcbNeeded+0x1000)",
  "notes": "candidate; confirm region via correlation" }
```

---

## 8. Scheduling & safety

- **Snapshot-revert loop:** each oversize campaign against a VM snapshot; revert on fault. Many of these inputs will crash the service.
- **Fault = signal too:** a reproducible crash on oversize is itself a (different) bug class — log separately, don't discard.
- **Throughput:** prioritise surfaces by (a) remote + non-admin, (b) has a two-call/`size_is` pattern, (c) returns ≥ a page. The compression lane first for KASLR (SMBleed precedent).
- **Build matrix:** 2016 → 11 24H2 / Server 2025; a zero-init on 24H2 may leak on 2019.
- **Detection footprint:** oversized out-buffer requests and `OriginalSize ≫ payload` are clean signatures — note them for the EDR side.

---

## 9. Why this beats the CVE list

The dossier's categorized population is static and patch-chased; this finds the **next** `AV:N` leak (85353 / SMBleed-class) **before** it's rated, and specifically surfaces the ones hiding under an `AV:L`-by-assumption score because no one sent an oversized request remotely. Pair any High finding with the §0 reframe: a system-image or kernel-pool pointer from *any* reachable surface defeats ASLR/KASLR everywhere on the host.

---

## 10. Defensive mirror (EDR build)
- Zero-initialize every out-buffer before fill; never trust a client-supplied size to bound *fill*.
- Return exactly `bytes_written`, not `bytes_allocated`.
- Reject `OriginalSize`/`size_is` that exceed a sane cap; validate `length ≤ size`.
- Detection: alert on out-buffer requests where returned-data-length ≪ requested-length, repeated against the same opnum (the fuzzer's own fingerprint).

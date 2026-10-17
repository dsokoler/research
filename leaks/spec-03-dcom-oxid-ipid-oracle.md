# Spec 03 — DCOM OXID / IPID Pointer-Oracle

**Companion to:** `windows-aslr-leak-research-program.md` §4.
**Goal:** over 135 (+ dynamic), harvest the OXID/OID/**IPID** identifiers carried in marshaled DCOM object references and test, by differential analysis, whether any field is **derived from a server-side pointer** — an ASLR oracle with non-admin creds.
**Privilege:** `ServerAlive2` recon is **pre-auth**; IPID harvesting needs an authenticated non-admin **and** an object reference obtainable within default DCOM ACLs (see §4 scoping).
**Posture:** structural parsing + differential analysis. The "IPID encodes a pointer" claim is a **hypothesis to test**, not asserted — the spec gives you the structures and the test.

---

## 1. Structures you parse (MS-DCOM)

### 1.1 OBJREF (the marshaled reference)
```
OBJREF {
  u32    signature;   // 'MEOW' = 0x574F454D
  u32    flags;       // OBJREF_STANDARD=0x1, HANDLER=0x2, CUSTOM=0x4, EXTENDED=0x8
  GUID   iid;         // interface being marshaled
  union  u_objref;    // by flags
}
```

### 1.2 OBJREF_STANDARD / STDOBJREF (the interesting case)
```
OBJREF_STANDARD {
  STDOBJREF      std;
  DUALSTRINGARRAY saResAddr;   // string + security bindings (RPC endpoints)
}

STDOBJREF {
  u32   flags;
  u32   cPublicRefs;
  u64   oxid;      // OXID  — identifies the object exporter (apartment) on the server
  u64   oid;       // OID   — identifies the object (per-exporter)
  GUID  ipid;      // IPID  — identifies the interface pointer (stub) — 16 bytes
}
```

- **OXID** (8 B) — per-apartment exporter id; resolved to bindings via the OXID resolver.
- **OID** (8 B) — per-object id within the exporter.
- **IPID** (16 B) — per-interface-pointer id; this is the field historically tied to internal table structure.

### 1.3 IPID internal layout (combase.dll — version-dependent)
Documented decomposition (verify per build):
```
IPID (16 bytes, as a GUID) commonly encodes:
  bytes 0-1   : index into the server's IPID table (page/entry)
  bytes 2-3   : index/generation
  bytes 4-5   : (reserved / mixed)
  bytes 6-7   : server PID
  bytes 8-15  : per-process COM "signature" (a per-process random value)
```
The **pointer hypothesis:** in some runtime versions the table index + per-process signature let you locate, or directly reveal, the `IPIDEntry` structure address — i.e., a heap pointer into combase's IPID table. Whether a *raw* pointer leaks (vs just an enumerable index) is the thing §5 tests.

---

## 2. Architecture

```
[OXID Recon (pre-auth)] 
        |
[Reference Harvester (non-admin)] -> [OBJREF Parser] -> [IPID Decomposer] -> [Differential Analyzer] -> [Report]
```

---

## 3. OXID recon — pre-auth, free

- Interface **IObjectExporter** (IID `99FCFEC4-5260-101B-BBCB-00AA0021347A`), opnum **5 = `ServerAlive2`**, on the **OXID resolver** at `ncacn_ip_tcp:<target>[135]`.
- Returns a `DUALSTRINGARRAY` of the server's string bindings (hostnames, **all IPv6/IPv4 addresses** — the "APT box" disclosure) and security bindings.
- Value here: host/binding enumeration + confirming DCOM is reachable. **Not** pointer material, but it's free and needs no auth. (Also `ResolveOxid2`, opnum 4, maps an OXID → its RPC bindings.)
- Tooling: Impacket `IOXIDResolver` / `ServerAlive2`.

---

## 4. Reference harvesting — non-admin scoping (critical)

Default DCOM ACLs lock **remote activation** of arbitrary CLSIDs to admins (hardened further post-2021, with enforced packet integrity). Stay in scope by using only:

### 4.1 Authenticated-user-activatable CLSIDs
Enumerate and filter **offline** (on a reference box) with **OleViewDotNet**:
- Walk `HKCR\CLSID\{...}` → map to `AppID` → read `LaunchPermission` / `AccessPermission` (and machine-wide DCOM defaults).
- Keep CLSIDs whose Launch+Activation ACL grants **Authenticated Users / INTERACTIVE / Everyone** `Local+Remote Launch/Activate`. This set is smaller than pre-2021 but non-empty.
- Also note which are `ncacn_ip_tcp`-reachable vs local-only.

Then remotely `CoCreateInstanceEx` those CLSIDs with a `COSERVERINFO` pointing at the target, under low-priv creds, and marshal the returned interface to get its OBJREF_STANDARD.

### 4.2 Activation-free references
Where you can obtain a marshaled OBJREF **without** activating a CLSID — e.g., a method on an already-reachable interface hands you another object reference (an enumerator, a sub-object) — parse that. No activation ACL involved.

> **Scope rule:** if a CLSID requires admin to activate, it's out. Do not escalate; use only the authenticated-user set or activation-free refs.

---

## 5. Differential Analyzer

Collect STDOBJREFs across many objects, many opens, and **across reboots**, then:

### 5.1 OXID / OID behaviour
- Are OXID/OID monotonic counters, random, or address-derived? Diff across opens (same boot) and across reboots.
- Address-derived (varies per boot within a heap span, aligned) → pointer-bearing exporter/object id (bonus leak).

### 5.2 IPID decomposition test (the main event)
- Decompose each IPID per §1.3. Separate the **index** region from the **per-process signature** region.
- **Per-process signature:** fixed for all objects within one server process/boot; changes per process/reboot. If it's a 64-bit value in a heap/pointer range (aligned, canonical VA) rather than pure random → candidate pointer/cookie derived from an address.
- **Index region:** if it increments predictably with each new interface pointer, it's a table index (enumerable, not a raw pointer) — note for secondary use.
- **Cross-reboot:** a signature/field that is base-relative across reboots is the ASLR oracle; one that's uniformly random is not.

### 5.3 Confirming "pointer" vs "cookie"
- Correlate the IPID signature against a **known** combase pointer leaked by another method (Spec-01 handle leak from a combase-hosted interface, or Spec-02 tail leak) — if the IPID field equals/relates-to a combase heap address by a fixed transform, it's confirmed pointer-derived and you can back out the combase/base.

---

## 6. Report schema

```json
{ "target": "...", "clsid": "{...}", "iid": "{...}",
  "acquired_via": "activation(auth-user CLSID) | activation-free",
  "stdobjref": { "oxid": "0x...", "oid": "0x...", "ipid": "{...}" },
  "ipid_decomp": { "index": "0x...", "pid": 1234, "signature_hex": "...." },
  "behaviour": { "signature_per_boot_stable": true, "reboot_base_relative": true,
                 "signature_pointer_shaped": true },
  "classification": "High|Medium|Low|Reject",
  "base_recovery": "correlate signature vs known combase pointer",
  "scope_note": "auth-user-activatable | activation-free (non-admin confirmed)" }
```

---

## 7. Pitfalls & scope

- **Hardening:** post-2021 DCOM enforces packet integrity and tighter activation; remain in the authenticated-user-activatable / activation-free set or you've left non-admin scope.
- **Version dependence:** the IPID↔pointer relationship is a combase-version property — expect it to differ or vanish across builds. The differential (§5) is the authority, not the documented layout.
- **Likely outcome bias:** modern combase leans toward index+random-signature (cookie) rather than raw pointers; treat a raw-pointer find as the exception you're hunting, and lean on OXID/OID and correlation as backups.

---

## 8. Tooling
- Impacket: `dcom.oxidresolver` (`IObjectExporter`, `ServerAlive2`, `ResolveOxid2`), `dcom` activation helpers, OBJREF parsing.
- OleViewDotNet (PowerShell module): CLSID/AppID/ACL enumeration, OBJREF inspection.
- A parser for OBJREF_STANDARD → STDOBJREF → IPID decomposition (trivial once the layout is pinned per build).

---

## 9. Defensive mirror (EDR build)
- If you ship a COM server: ensure IPIDs/OXIDs are opaque (index + per-boot random), never address-derived.
- Keep activation ACLs tight; don't grant Authenticated Users remote activation.
- Detection: `ServerAlive2`/`ResolveOxid2` sweeps and bulk marshaling from one low-priv principal are enumeration signatures.

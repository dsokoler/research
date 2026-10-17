# Spec 04 — SPNEGO / NEGOEX On-Wire Token-Leak Audit

**Companion to:** `windows-aslr-leak-research-program.md` §6.
**Goal:** find the **remote, often pre-auth** form of the CVE-2024-26209 bug class — under-initialized buffers in SSPI package/mechanism enumeration that get **serialized into authentication tokens sent on the wire**. You read them as the peer during the handshake.
**Privilege:** acting as a peer / server needs **no auth** to *read* the token the other side emits; inducing the target to emit one may need creds (coercion) or nothing (you're the server it connects to).
**Posture:** structural audit of wire tokens for uninitialized tails. Finds "this token field carries heap bytes," not an exploit.

---

## 1. Why this class goes remote

CVE-2024-26209 is a Negotiate (SPNEGO) package that built a mechanism-OID buffer in `lsasrv.dll` **without zero-initializing it**, leaking LSASS heap to the *local* caller. The identical defect becomes **remote** the instant such an under-filled buffer is **serialized into a token placed on the wire** — because the SPNEGO/NEGOEX handshake exchanges exactly these mech/metadata buffers between peers. The auditor's job: find a variable-length/optional token field whose server-side buffer is larger than its meaningful content and whose tail isn't zeroed.

---

## 2. Token structures you parse

### 2.1 SPNEGO (RFC 4178 / MS-SPNG) — ASN.1 DER
```
NegotiationToken ::= CHOICE {
  negTokenInit  [0] NegTokenInit,
  negTokenResp  [1] NegTokenResp }

NegTokenInit ::= SEQUENCE {
  mechTypes     [0] MechTypeList,          -- SEQUENCE OF OBJECT IDENTIFIER   ← leak candidate
  reqFlags      [1] ContextFlags OPTIONAL, -- deprecated
  mechToken     [2] OCTET STRING OPTIONAL, -- inner (NTLM/Kerberos/NEGOEX) blob
  mechListMIC   [3] OCTET STRING OPTIONAL } -- ← leak candidate (MIC buffer)

NegTokenResp ::= SEQUENCE {
  negState      [0] ENUMERATED OPTIONAL,
  supportedMech [1] OBJECT IDENTIFIER OPTIONAL,
  responseToken [2] OCTET STRING OPTIONAL,
  mechListMIC   [3] OCTET STRING OPTIONAL } -- ← leak candidate
```
The initial `NegTokenInit` is wrapped in a GSS-API `InitialContextToken` (`[APPLICATION 0]` + `thisMech = SPNEGO OID 1.3.6.1.5.5.2`).

**Leak candidates:** the `mechTypes` list (OID array — the exact buffer behind the 26209 defect), `mechListMIC` (fixed-size MIC buffer that may be over-allocated), and any OPTIONAL field present-but-padded.

### 2.2 NEGOEX (MS-NEGOEX) — binary, inside the SPNEGO mechToken
```
NEGOEX_MESSAGE_HEADER {
  u8    Signature[8];   // 'NEGOEXTS'
  u32   MessageType;
  u32   SequenceNum;
  u32   cbHeaderLength;
  u32   cbMessageLength;
  GUID  ConversationId;
}
```
Message types: `INITIATOR_NEGO` / `ACCEPTOR_NEGO` (`NEGOEX_NEGO_MESSAGE`), `INITIATOR_META_DATA` / `ACCEPTOR_META_DATA`, `CHALLENGE`, `AP_REQUEST`, `VERIFY`.

```
NEGOEX_NEGO_MESSAGE {
  NEGOEX_MESSAGE_HEADER Header;
  u8    Random[32];               // ← MUST be RNG; if under-filled → leak
  u64   ProtocolVersion;
  AUTH_SCHEME_VECTOR  AuthSchemes; // offset+count into message  ← leak candidate
  EXTENSION_VECTOR    Extensions;  // offset+count into message  ← leak candidate
  u8    Pad[ ... ];
}
```
Vectors are `{ u32 ByteArrayOffset / <Type>ArrayOffset, u16/u32 Count }` descriptors pointing into the message body. **If the message buffer is allocated larger than the vectors' real content, the gap is uninitialized** — and it's serialized out verbatim. `Random[32]` is a second candidate: if the RNG fill short-changes it, the remainder is heap.

Metadata messages carry `AUTH_SCHEME`-scoped `BYTE_VECTOR` MetaData blobs — same offset/length-into-buffer shape, same risk.

---

## 3. Architecture

```
[Token Elicitor] -> [Token Capturer] -> [ASN.1 / NEGOEX Parser] -> [Field Tail Auditor] -> [Classifier] -> [Report]
```

---

## 4. Token Elicitor — get the target to emit a token

Three modes (pick by what you can reach non-admin):

1. **Be the server (no auth to read).** Stand up a listener that offers SPNEGO/NEGOEX and induce the target *client* to connect: an SMB share, an HTTP endpoint with `WWW-Authenticate: Negotiate`, an LDAP/RPC endpoint. The client's `NegTokenInit` (its mechList + NEGOEX nego) arrives unauthenticated — you read it directly.
2. **Coerce the target to authenticate to you.** PrinterBug (`RpcRemoteFindFirstPrinterChangeNotificationEx`, spoolss opnum 65) / MS-PAR `RpcAsyncOpenPrinter` / PetitPotam-style EFSRPC coercion → the target's machine or user account initiates outbound Negotiate auth to your listener. Needs creds to trigger the coercion RPC, but then you read the emitted token.
3. **Induce a server response token.** Begin a handshake against the target *server* (SMB/LDAP/RPC) and capture its `NegTokenResp` / `ACCEPTOR_NEGO` / `ACCEPTOR_META_DATA` — the server-emitted buffers, which is where a server-side uninitialized-fill defect would surface.

### 4.2 Token Capturer
A minimal SSPI-less peer that completes just enough of the handshake to receive the first (and ideally second) token, logging raw bytes. Don't let the OS SSPI stack normalize/re-pad the token — capture at the wire.

---

## 5. Field Tail Auditor

For every variable-length / OPTIONAL / vector-referenced field (§2 candidates), compute the **meaningful content length** vs the **serialized length**, isolate the tail/gap, and:

1. **Zero check** — zeroed → no leak.
2. **Run-to-run variance** — elicit the token **M times** (fresh handshakes). A field tail that **changes across handshakes** = uninitialized memory. This is the discriminator.
3. **Random-field sanity** — `NEGOEX Random[32]` should be uniformly random *and independent* every handshake; if part of it repeats or correlates with heap-looking values, that's the leak (RNG under-fill).
4. **Offset/length gap** — for NEGOEX vectors, audit the bytes in `[vector_end, next_field_or_message_end)` — classic uninitialized padding between a short vector and the allocated message size.

Specific candidate checklist:

| Token | Field | What to inspect |
|---|---|---|
| SPNEGO NegTokenInit | `mechTypes` | bytes beyond the real OID sequence within its buffer (the 26209 defect's wire cousin) |
| SPNEGO NegTokenInit/Resp | `mechListMIC` | the MIC OCTET STRING — fixed buffer over-allocated? |
| SPNEGO | any OPTIONAL present-but-empty | padding bytes |
| NEGOEX NEGO | `Random[32]` | repetition / heap-shaped bytes across handshakes |
| NEGOEX NEGO | `AuthSchemes` / `Extensions` vector gap | `[vector_content_end, message_end)` |
| NEGOEX META_DATA | `BYTE_VECTOR` MetaData | tail past real metadata |

---

## 6. Classifier

Same pointer/heap heuristic as Spec-01/02 applied to the suspect tail:
- pointer-shaped 8-byte values (user VA canonical, aligned) → image/heap pointer on the wire → ASLR leak;
- otherwise, high-entropy varying bytes → still an LSASS-heap info leak (lower ASLR value but a real disclosure).

**Confidence:**
- **High** — tail varies across handshakes **and** carries pointer-shaped values (remote ASLR leak; pre-auth if read as the server).
- **Medium** — tail varies, non-pointer heap bytes (info leak, weaker for ASLR).
- **Reject** — zeroed or stable-random.

---

## 7. Report schema
```json
{ "elicitation": "be-server | coercion | server-response",
  "auth_needed": "none | creds-for-coercion",
  "token": "SPNEGO_NegTokenInit | NEGOEX_NEGO | ...",
  "field": "mechTypes | Random | AuthSchemes-gap | mechListMIC",
  "tail_offset": 42, "tail_len": 18,
  "varies_across_handshakes": true,
  "pointers": [ { "offset": 50, "value": "0x00007ff8...", "region": "image:?" } ],
  "confidence": "High",
  "repro": "N fresh Negotiate handshakes, diff the field tail" }
```

---

## 8. Pitfalls
- Most token fields are fixed-length and zeroed; the win is **one** package/path with an over-allocated buffer. Breadth across packages (NTLM/Kerberos/NEGOEX/schannel) matters.
- SSPI on the capture side can re-pad/normalize — capture raw wire bytes, not the parsed SSPI buffer.
- Kerberos AP tokens are mostly ASN.1 with tight DER lengths (less slack); NEGOEX binary vectors and SPNEGO OCTET STRINGs are the slack-prone surfaces.
- Attribution: a varying tail could be a legit nonce — the pointer-shape test is what elevates it to an ASLR leak.

---

## 9. Defensive mirror (EDR / product build)
- Zero-initialize every token/mech/metadata buffer before fill; serialize exactly the content length, never the allocation.
- Verify RNG fully fills `Random`/nonce fields.
- Detection: repeated identical Negotiate handshakes from one source that never complete auth (the auditor's fingerprint); anomalous token field lengths.
- This class is also a reason to audit your own product's any SSPI/token emission paths for uninitialized serialization.

# NEGOEX Message Validation Reference

A wire-format parsing + validation spec for detecting the memory-safety bug class in Windows SPNEGO NEGOEX (CVE-2022-37958, CVE-2025-21295, CVE-2025-47981). Source of truth: **[MS-NEGOEX]** (open spec). All structures confirmed against the spec's own Protocol Examples section.

---

## 1. Wire basics

- **Endianness:** little-endian for every scalar field.
- **Offset reference point:** every `*Offset` / `*ArrayOffset` field is measured **from the first byte of the NEGOEX message** (the `N` of `NEGOEXTS`), **not** from the start of the TCP payload or the SPNEGO wrapper. This is the single most important fact for bounds-checking.
- **Concatenation:** a single SPNEGO token may carry **multiple** NEGOEX messages back-to-back. Walk them by `cbMessageLength`.
- **Signature:** `MESSAGE_SIGNATURE = 0x535458454F47454E` = ASCII `"NEGOEXTS"` (bytes `4E 45 47 4F 45 58 54 53`).

### Where the NEGOEX blob lives (carriers to pull it from)

NEGOEX is a GSS mechanism carried inside a **SPNEGO** (GSS-API) token. Find it in:

| Carrier | Field holding the SPNEGO/GSS blob |
|---|---|
| SMB2 `SESSION_SETUP` (req + resp) | `SecurityBuffer` |
| RDP NLA (CredSSP) | `TSRequest.negoTokens` |
| HTTP | `Authorization: Negotiate <b64>` / `WWW-Authenticate: Negotiate <b64>` |
| LDAP | SASL bind, `GSS-SPNEGO` |
| SMTP/IMAP/POP | `AUTH GSSAPI` / `Negotiate` |

SPNEGO wrapper quick-map: GSS `InitialContextToken` tag `0x60` → SPNEGO OID `1.3.6.1.5.5.2` → `NegTokenInit` `0xA0` (mechTypes list must contain the **NEGOEX OID `1.3.6.1.4.1.311.2.2.30`**) → `mechToken` `0xA2` OCTET STRING = raw NEGOEX bytes. Responses use `NegTokenResp` `0xA1` → `responseToken` `0xA2`. Practical shortcut: locate the 8-byte `NEGOEXTS` signature inside the token and parse forward.

---

## 2. MESSAGE_HEADER — 40 bytes (0x28), present on every message

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0x00 | 8 | `Signature` | must equal `NEGOEXTS` |
| 0x08 | 4 | `MessageType` | enum, see §3 |
| 0x0C | 4 | `SequenceNum` | starts at 0, increments per message in the conversation |
| 0x10 | 4 | `cbHeaderLength` | length of the fixed header for this message type |
| 0x14 | 4 | `cbMessageLength` | length of the **entire** message (header + payload) |
| 0x18 | 16 | `ConversationId` | GUID, constant across the whole conversation |
| **0x28** | | *end of header* | |

---

## 3. MESSAGE_TYPE → structure → expected cbHeaderLength

| Value | MessageType | Structure | Expected `cbHeaderLength` |
|---|---|---|---|
| 0 | INITIATOR_NEGO | NEGO_MESSAGE | **96** (0x60) |
| 1 | ACCEPTOR_NEGO | NEGO_MESSAGE | **96** (0x60) |
| 2 | INITIATOR_META_DATA | EXCHANGE_MESSAGE | **64** (0x40) |
| 3 | ACCEPTOR_META_DATA | EXCHANGE_MESSAGE | **64** (0x40) |
| 4 | CHALLENGE | EXCHANGE_MESSAGE | **64** (0x40) |
| 5 | AP_REQUEST | EXCHANGE_MESSAGE | **64** (0x40) |
| 6 | VERIFY | VERIFY_MESSAGE | **76** (0x4C) |
| 7 | ALERT | ALERT_MESSAGE | **68** (0x44) |

Any `MessageType` outside 0–7 is malformed.

---

## 4. Per-message layouts

### 4.1 NEGO_MESSAGE (types 0,1) — header 96 bytes (0x60)

| Offset | Size | Field |
|---|---|---|
| 0x00 | 40 | MESSAGE_HEADER |
| 0x28 | 32 | `Random[32]` |
| 0x48 | 8 | `ProtocolVersion` |
| 0x50 | 4 | `AuthSchemeArrayOffset` |
| 0x54 | 2 | `AuthSchemeCount` |
| 0x56 | 2 | (padding) |
| 0x58 | 4 | `ExtensionArrayOffset` |
| 0x5C | 2 | `ExtensionCount` |
| 0x5E | 2 | (padding) |
| **0x60** | | *end of header; payload follows* |

Payload: `AuthSchemeCount` × **AUTH_SCHEME** (16-byte GUID each) at `AuthSchemeArrayOffset`; `ExtensionCount` × **EXTENSION** (12 bytes each) at `ExtensionArrayOffset`.

### 4.2 EXCHANGE_MESSAGE (types 2,3,4,5) — header 64 bytes (0x40)

| Offset | Size | Field |
|---|---|---|
| 0x00 | 40 | MESSAGE_HEADER |
| 0x28 | 16 | `AuthScheme` (GUID) |
| 0x38 | 4 | `Exchange.ByteArrayOffset` |
| 0x3C | 4 | `Exchange.ByteArrayLength` |
| **0x40** | | *end of header; exchange bytes follow* |

### 4.3 VERIFY_MESSAGE (type 6) — header 76 bytes (0x4C)

| Offset | Size | Field |
|---|---|---|
| 0x00 | 40 | MESSAGE_HEADER |
| 0x28 | 16 | `AuthScheme` (GUID) |
| 0x38 | 4 | `Checksum.cbHeaderLength` |
| 0x3C | 4 | `Checksum.ChecksumScheme` (1 = RFC3961) |
| 0x40 | 4 | `Checksum.ChecksumType` |
| 0x44 | 4 | `Checksum.ChecksumValue.ByteArrayOffset` |
| 0x48 | 4 | `Checksum.ChecksumValue.ByteArrayLength` |
| **0x4C** | | *end of header; checksum value follows* |

### 4.4 ALERT_MESSAGE (type 7) — header 68 bytes (0x44)

| Offset | Size | Field |
|---|---|---|
| 0x00 | 40 | MESSAGE_HEADER |
| 0x28 | 16 | `AuthScheme` (GUID) |
| 0x38 | 4 | `ErrorCode` (NTSTATUS) |
| 0x3C | 4 | `Alerts.AlertArrayOffset` |
| 0x40 | 2 | `Alerts.AlertCount` |
| 0x42 | 2 | (padding) |
| **0x44** | | *end of header; alert entries follow* |

### 4.5 Sub-structures

| Struct | Size | Layout |
|---|---|---|
| BYTE_VECTOR | 8 | `ByteArrayOffset`(u32) + `ByteArrayLength`(u32) |
| AUTH_SCHEME_VECTOR | 8 | `Offset`(u32) + `Count`(u16) + pad(2) |
| EXTENSION_VECTOR | 8 | `Offset`(u32) + `Count`(u16) + pad(2) |
| ALERT_VECTOR | 8 | `Offset`(u32) + `Count`(u16) + pad(2) |
| AUTH_SCHEME | 16 | GUID |
| EXTENSION | 12 | `ExtensionType`(u32) + `ExtensionValue`(BYTE_VECTOR, 8) |
| ALERT | 12 | `AlertType`(u32) + `AlertValue`(BYTE_VECTOR, 8) |
| CHECKSUM | 20 | `cbHeaderLength`(u32) + `ChecksumScheme`(u32) + `ChecksumType`(u32) + `ChecksumValue`(BYTE_VECTOR, 8) |

---

## 5. Validation ruleset

Let **M** = `cbMessageLength`, **H** = `cbHeaderLength`, **A** = bytes actually available for this message in the token (octet-string length minus bytes already consumed by preceding concatenated messages).

### A. Header invariants (every message)
1. `Signature` == `NEGOEXTS`.
2. `MessageType` in 0–7.
3. `H` == the expected header length for that type (§3). Mismatch → malformed.
4. `H` ≤ `M` ≤ `A`. `M` > `A` is an over-read seed; reject.
5. `M` ≥ minimum for type (same as `H`).

### B. Vector / byte-array bounds (the overflow-seed checks)
For each embedded reference `(Offset O, Count N, ElemSize E)` or `(Offset O, Length L)`:
1. **Placement:** `O` ≥ `H` (payload must sit after the fixed header). An offset pointing into the header is suspicious.
2. **Multiply guard:** `span = N × E`. If `E > 0` and `N > floor(0xFFFFFFFF / E)` → 32-bit overflow → reject. (For a byte-array, `span = L`.)
3. **Add guard:** if `O + span < O` → address wrapped 32-bit → reject.
4. **Bound:** `O + span` ≤ `M`. **Exceeding `M` is the core fingerprint of the heap-overflow bug class** → reject.
5. **Nested:** after validating an array fits, validate each entry's inner BYTE_VECTOR the same way (EXTENSION.`ExtensionValue`, ALERT.`AlertValue`).

Apply per type:
- **NEGO:** AuthSchemes `(0x50 offset, 0x54 count, E=16)`; Extensions `(0x58 offset, 0x5C count, E=12)` + each extension's inner value.
- **EXCHANGE:** Exchange byte-array `(0x38 offset, 0x3C length)`.
- **VERIFY:** ChecksumValue `(0x44 offset, 0x48 length)`; sanity-check `Checksum.cbHeaderLength` (0x38).
- **ALERT:** Alerts `(0x3C offset, 0x40 count, E=12)` + each alert's inner value.

### C. Count sanity caps (tunable anomaly flags)
- `AuthSchemeCount`, `ExtensionCount`, `AlertCount` are realistically tiny (a legit NEGO advertises ~1–3 schemes). Flag counts above a small threshold (e.g., >16) and counts near `0xFFFF`.
- Flag `AuthSchemeCount == 0` on a NEGO (nothing to negotiate).

### D. Cross-message / state checks (needed for the UAF, CVE-2025-21295)
Single-packet bounds-checking cannot catch a use-after-free; track conversation state:
1. `ConversationId` constant across all messages in a conversation.
2. `SequenceNum` increments as expected; flag gaps, duplicates, or resets.
3. The `AuthScheme` GUID in an EXCHANGE/VERIFY/ALERT must be one advertised in that conversation's NEGO `AuthSchemes`. An unknown or already-torn-down scheme/conversation is the behavioral tell for a lifetime bug.
4. Flag message-type sequences that violate the state machine (e.g., VERIFY before a completed NEGO exchange).

---

## 6. CVE mapping — which check catches which

| CVE | Class | Caught by |
|---|---|---|
| CVE-2022-37958 | heap buffer overflow | §5.B.4 (vector extends past `M`) + B.2/B.3 (overflow math) |
| CVE-2025-47981 | heap buffer overflow (CWE-122) | same — §5.B.2/3/4 |
| CVE-2025-21295 | use-after-free (CWE-416) | §5.D state checks + lsass post-ex anchor (single packet is insufficient) |

The field-level trigger for 37958/47981 was never publicly disclosed, so detection is generic: **any NEGOEX message whose declared offset/length/count references data outside `[H, M]`, or whose allocation math overflows 32-bit.** This also covers future NEGOEX length bugs.

---

## 7. PKU2U flag (CVE-2025-47981 specific)

The default-on GPO "Allow PKU2U authentication requests… to use online identities" is what exposes ordinary endpoints. PKU2U is negotiated as a NEGOEX **AuthScheme (a GUID)**. Do **not** hardcode a guessed GUID — capture it empirically: trigger a PKU2U handshake between two workgroup machines using online identities in the lab, record the `AuthScheme` GUID from the NEGO `AuthSchemes` array, and pin that. Then raise signal when a NEGO advertises the PKU2U scheme **to/from a non-domain peer**, especially on an outbound/client-initiated connection (Microsoft's own advisory frames 47981 as the victim being convinced to connect to a malicious server).

---

## 8. Integration with existing telemetry

- The generic `Packet Event` (whole-packet capture, server-side parse) plus the SMB3 and RDP packet events already capture the carriers in §1. This spec is what the server-side parser runs against the SPNEGO blob inside `SMB2 SESSION_SETUP` and RDP NLA frames.
- **Correlation window:** an anomalous NEGOEX message (any §5 failure) within a short window of an `lsass` anomaly — crash, unexpected child process, or unexpected outbound connection — is the high-confidence post-exploitation signal. The NEGOEXTS SSP runs inside `lsass`, so `lsass` is the process anchor.
- One NEGOEX-envelope validator covers SMB, RDP, SMTP, HTTP, and LDAP at once, and the whole NEGOEX bug class rather than a single CVE.

# ANP Zero-Knowledge Authentication Profile (Proposal)

- Document ID: Proposal (no ANP number assigned)
- Proposed Profile identifier: `anp.auth.zklogin.v1`
- Title: ANP Zero-Knowledge Authentication Profile
- Status: Proposal / for community discussion; not a released specification
- Version: 0.2
- Language: English
- Chinese mirror: [ANP 零知识认证 Profile（提案）](../chinese/proposals/anp-zk-login-proposal.md)
- Issue draft: [anp-zk-login-issue-draft.md](anp-zk-login-issue-draft.md)
- Applicability: This proposal applies to request authentication between an agent and a relying party, as an opt-in alternative to the DID-signature flow of [ANP-02](../02-anp-did-authentication-protocol-specification.md).
- Reference implementation: the `@connectx-sdk/core/zk` module and the `@connectx-sdk/zk-pairing` backend of the connectx ANP SDK. It is not normative; the statement parameters, wire formats and error mapping below are the ones it implements.

> **This is a proposal, not a specification.** It proposes a new opt-in Profile for the ANP protocol set and does not modify any released document. Per [CONTRIBUTING](../CONTRIBUTING.md), it should be introduced through a GitHub Issue and Discord discussion before any PR that would promote it to a released document. Section numbers, identifiers, and error mappings below are proposals and may change during review.

> **Why a separate Profile rather than an ANP-02 revision.** ANP-02 §3.1.2 step 2 (L110) and §3.2.2 step 4 (L163) defer algorithms and key representation to the selected verification method type, and Appendix D (L503) anticipates other methods supplying "identity and authentication-key material through their own resolution and verification rules". This proposal adds a Profile beside the existing ones instead of extending ANP-02 itself.

## 1. Summary

ANP-02 authenticates a request by a DID-signed HTTP message, which necessarily discloses the caller's `keyid` — a complete DID URL (ANP-02 L82). Any relying party, and any observer of the request, therefore learns which Agent DID is calling, and can correlate that DID across services.

This proposal defines `anp.auth.zklogin.v1`: an **opt-in** authentication Profile in which the caller proves, in zero knowledge, that it holds a credential admitted to some membership set, without transmitting a DID, a public key, or any stable cross-service identifier. The relying party that enables the Profile learns only a **per-service pseudonym**.

The Profile is deliberately additive:

- It reuses the ANP-02 wire carriers (`Signature-Input` options, `Content-Digest`, the 401 challenge, and the §4 JSON carriage) rather than defining new transport.
- A relying party that does not implement it is unaffected, and a client that does not implement it keeps using DID-signature authentication.
- Nothing in ANP-01 through ANP-10 changes.

### 1.1 What this proposal does not do

- It does not hide the fact that a request was authenticated by ZK proof, nor the network-level metadata (source address, timing, request size).
- It does not make an agent unlinkable to a service that it identifies itself to by other means (payments, handles, message delivery). See §10.
- It does not define how a membership set is governed. Admission policy belongs to the set issuer. See §8.

## 2. Conventions

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL are to be interpreted as described in BCP 14.

Notation used in the relation (§6.4):

| Symbol | Meaning |
|---|---|
| `zkSk` / `zkPk` | The ZK-curve proving key and its public key; `zkPk = g^zkSk` (§4) |
| `H(tag, …)` | The statement's hash — Poseidon-BN254 with circomlib parameters — domain-separated by a fixed field constant `tag` (§6.1) |
| `salt` | A per-credential random value chosen by the agent at enrollment (§8) |
| `attrs` | The attribute vector committed at enrollment; a fixed number of fixed-width slots (§6.1) |
| `leaf`, `root`, `index`, `path` | A membership-set leaf, its Merkle root, the leaf index, and the Merkle path |
| `V` | The audience value, derived by the client from the connected origin (§6.5) |
| `baseHash` | `SHA-256` over the RFC 9421 signature base of the request, as a field element (§6.2) |
| `h` | The per-service pseudonym (§6.4 clause 3) |
| `sk_s` / `pk_s` | The session key pair; the access token binds to `pk_s` (§6.8) |

Strings fixed by this Profile:

| String | Value | Where it appears |
|---|---|---|
| Statement id | `anp.zklogin.v1` | Envelope `statement`, request body `statement`, descriptor `sets[].statement` |
| Descriptor profile | `anp-zk/1` | The descriptor's `profile` member (§5) |
| `keyid` namespace | `zk:anp-zk/1:` | The `keyid` parameter (§12) |
| Signature labels | `zk1` (login), `pop1` (proof of possession) | RFC 9421 label (§4, §6.8) |
| Scheme names | `groth16`, `plonk` | Envelope `scheme`, `alg="zk:<scheme>"` |
| Token type | `ANP-ZK-PoP` | `Authentication-Info` `token_type` (§6.8) |

## 3. Design goals and non-goals

**Goals.** (a) No DID, public key, or stable identifier on the wire. (b) Full request binding: a proof is valid only for one method, target URI, authority, body, and freshness window. (c) Zero modification to released ANP documents. (d) Backend-agnostic: a relying party accepts any proof system it has registered a verification key for (§7). (e) Pseudonym stability per service, so a relying party can rate-limit, ban, and keep a session.

**Non-goals.** Hiding network metadata; global unlinkability across colluding relying parties; defining membership governance; replacing DID-based authentication where the relying party needs to address the caller by DID.

## 4. Two layers of identity

Proving membership of the DID key itself would force non-native field arithmetic — Ed25519 and secp256k1 scalar multiplication inside a BN254 circuit. This Profile therefore separates the two roles:

1. **The DID key** is used once, at enrollment, to authorize admission (§8). It is never used at login and never appears on the wire.
2. **The proving key `zkSk`** is a key on the ZK circuit's own curve. All login proofs use `zkSk`.

The membership leaf commits to `zkPk` and to the attribute digest only (§6.4 clause 2). A relying party therefore cannot recover the enrollment DID from anything it receives, and the DID's method rules (ANP-03 / [Appendix B](../appendix-b-compatibility-with-native-did-web.md)) govern enrollment only.

Implementations MUST use distinct keys for enrollment and proving, generated independently. A re-used or derived key links the two layers.

## 5. Opt-in and discovery

A relying party opts in by publishing a descriptor. Two channels are defined; a relying party MAY use either or both.

1. **Descriptor.** The relying party adds a service entry to its DID document:

```json
{ "id": "#anp-zk-auth", "type": "ANPZkAuthService",
  "serviceEndpoint": "https://api.example.com/.well-known/anp-zk.json" }
```

The endpoint returns:

```json
{
  "profile": "anp-zk/1",
  "endpoint": "https://api.example.com/anp-zk/login",
  "enroll": "https://api.example.com/anp-zk/enroll",
  "modes": ["anonymous", "escrow"],
  "predicates": [{ "id": "gte:score:90", "description": "…" }],
  "sets": [{ "setId": "api.example.com/2026-Q3", "statement": "anp.zklogin.v1",
             "epoch": 42, "setRoot": "<decimal>", "issuer": "did:wba:…", "size": 128,
             "accepts": [{ "scheme": "groth16", "vkHash": "…" },
                         { "scheme": "plonk",   "vkHash": "…" }] }],
  "attributes": { "type": { "agent": 1, "user": 2 }, "tier": { "gold": 3 } },
  "requiredDisclosures": ["type"],
  "escrow": { "auditorPublicKey": "…" }
}
```

`profile` MUST be `anp-zk/1`; a client MUST reject any other value rather than interpret it under these rules. Field-element members (`setRoot`) are decimal strings, because a 254-bit value is not exactly representable as a JSON number. `vkHash` is an opaque string the client returns unmodified. Unknown members MUST be ignored; every member the client reads MUST be validated. `sets[].accepts` names the `(scheme, vkHash)` pairs that set accepts, and MUST list at least one.

2. **Challenge.** A relying party MAY respond `401 Unauthorized` with a challenge, in the manner of ANP-02 L238–L246:

```http
WWW-Authenticate: ANPZK realm="api.example.com", profile="anp.auth.zklogin.v1",
  nonce="xyz987", error="invalid_signature"
Accept-Signature: zk1=("@method" "@target-uri" "@authority" "content-digest")
```

**Fallback rules.**

- A client that supports this Profile and receives a challenge naming `anp.auth.zklogin.v1` MUST respond with a proof request, not a DID-signed request.
- A client that does not support it MUST ignore the challenge and use ANP-02 authentication; the relying party MAY then accept or reject that attempt at its own policy (see the `required` note below).
- A relying party that has declared an operation as **ZK-required** (for example, by advertising only this Profile for it) MUST NOT accept a DID-signature login for that operation. A client MUST NOT treat such a rejection as a reason to retry with a DID signature.
- A relying party MUST NOT silently downgrade a ZK-required operation to ANP-02 authentication. This mirrors the "MUST NOT silently downgrade" discipline already used for mixed-version messaging Profiles.
- A relying party that has not opted in rejects a proof request with `invalid_verification_method` (§12), which is a client's signal that this site offers no ZK path.

**Which set, scheme and predicate a login uses** MUST be selected by the client from the descriptor, never guessed: a client MUST fail rather than substitute a set, scheme or predicate the relying party did not advertise.

## 6. The proof statement

### 6.1 Statement identifier and parameters

The statement identifier is `anp.zklogin.v1`. It names one frozen relation and one frozen encoding. Every parameter below changes a public-input byte or a pseudonym byte, so a change to any of them is a **different statement id**, a different pseudonym space, and therefore a different set:

| Parameter | Frozen value |
|---|---|
| Field / group | BN254 (`Fr`) / BabyJubjub over that same field, base point `g` |
| Hash | Poseidon-BN254, circomlib parameters |
| Domain tags | Fixed field constants: pseudonym `p−1`, leaf `p−2`, attrs `p−3`, origin `p−4`, setId `p−5`, nullifier `p−6` (reserved), node `p−7`, empty `p−8` |
| Merkle depth | 20 (2^20 ≈ 1.05M members); node = `H(nodeTag, left, right)`, and an empty leaf is the `emptyTag` sentinel rather than zero |
| Attributes | 8 slots × 32 bits, in a fixed slot order |
| Predicates | `any` / `eq` / `gte` / `lte`, encoded as `(predCode, predSlot, predValue)` |
| `V` normalization | `scheme://host[:port]`, lowercased, default port dropped, path/query/fragment dropped (§6.5) |
| Version | `anp.zklogin.v1` |

Two rules:

- A prover and a verifier MUST agree on `statementId` exactly; a mismatch MUST be rejected (§12).
- Implementations MUST NOT change any parameter while keeping the same `statementId`. The reason is that the pseudonym `h` is derived from a witness value and is used by a relying party as a durable handle (§9): if the hash family changed without the identifier changing, the same agent would silently become a different pseudonym at the same service.

One statement id MAY be implemented by several proof systems (§7). One membership set MAY therefore accept several schemes, but a set is admitted under exactly one statement id.

Predicates. The on-the-wire form is `any`, `eq:<attrName>:<decimal>`, `gte:…`, `lte:…`. A range is the conjunction of a `gte` and an `lte`. The predicate is a **public input**: the prover chooses it, and its parameters are visible to the verifier.

### 6.2 Public inputs

The relation's public inputs, in the fixed signal order of the circuit:

```text
baseHash, setField, setRoot, epoch, vField, pseudonym,
pkSx, pkSy, predCode, predSlot, predValue,
disclosed[0..7], mask
```

`setField` and `vField` are the field encodings of the set identifier and of `V`: `setField = H(setIdTag, setId)`, `vField = H(originTag, V)`. `baseHash`, `setField`, `epoch` and `vField` consume no constraint yet remain public inputs, which is what binds the proof to one set, one epoch and one request.

For interchange between a backend and this Profile, the public inputs also have a canonical JSON encoding carrying the same values by name (`baseHash`, `setId`, `setRoot`, `epoch`, `origin`, `pseudonym`, `sessionKey`, `predicate`, `disclosed`, `v`), field elements as decimal strings.

A verifier MUST reconstruct the public inputs from the request it received — the rebuilt signature base and the body — and MUST NOT accept a `publicInputs` blob from the client. A proof envelope MUST NOT carry one (§7.2).

### 6.3 Witness (never transmitted)

`zkSk`, `zkPk`, `salt`, `attrs`, `index`, and the Merkle `path` from `leaf` to `root`.

`sk_s` is deliberately **not** in the witness. The relation proves only that `pk_s` is a well-formed curve point (§6.4 clause 4); possession of `sk_s` is proved by the proof-of-possession signature on the session (§6.8).

### 6.4 The relation

A proof is valid only if there exist witness values satisfying all of:

1. **Key knowledge.** `zkPk = g^zkSk`.
2. **Membership.** `leaf = H(leafTag, zkPk.x, zkPk.y, salt, H(attrsTag, attrs))`, and `leaf` is the leaf at position `index` under `root` along `path`.
3. **Pseudonym correctness.** `pseudonym = H(pseudonymTag, zkSk, vField)`.
4. **Session key well-formedness.** `pk_s` is a point on the curve.
5. **Predicate.** The predicate `(predCode, predSlot, predValue)` holds over `attrs`.
6. **Disclosure consistency.** `disclosed[i] = attrs[i] · maskBit[i]` for every slot. `attrs[i]` itself is unconstrained — see §6.7 for the verifier's obligation.

Clauses 1 and 3 share one secret: the pseudonym and the membership leaf are bound to the same `zkSk`, which is why a pseudonym cannot be claimed by a non-member.

The relation does not prove a **negative** claim ("I am not on a blocklist"); a relying party's substitute is a set that admits only the permitted population and removes members (§8).

### 6.5 Client-side derivation of `V`

The client MUST derive `V` from the origin it is actually connected to — the scheme and authority of the real request, normalized per §6.1 — and MUST NOT accept a `V` supplied by the relying party. If the relying party chose `V`, two colluding relying parties could agree on one `V`, observe the same `h` at both, and thereby link the caller across services; deriving it client-side makes the pseudonym depend on a value the relying party cannot influence. This is the same discipline ANP-02 already applies to `@target-uri`, where the verifier is expected to derive the value "from the real request, not from a caller-supplied target URI alone".

The verifier MUST independently compute the expected `V` for the request it received and MUST reject a proof whose `V` does not match. `V` MUST identify the site, not the login endpoint: a path-dependent `V` would shift every pseudonym when the endpoint moves.

### 6.6 Verifier-side checks

Beyond validating the proof itself, a verifier MUST:

1. Recompute `baseHash` from the actual request (method, target URI, authority, `Content-Digest`, and the covered options) and reject on mismatch. Do not trust a `baseHash` supplied by the client.
2. Recompute `V` per §6.5 and reject on mismatch.
3. Require the request body's `nonce` and the `Signature-Input` `nonce` to match the server challenge, and apply the same freshness window as ANP-02 §3.2.1 step 7 (L141), with `expires` treated as SHOULD (ANP-02 L85).
4. Maintain a replay cache. In anonymous mode the ANP-02 `(keyid, nonce)` cache (ANP-02 L144, L437) becomes `(h, nonce)`. A nonce MUST be usable once per `h`. The cache is consumed only after the proof verifies, so a bad proof does not burn a challenge.
5. Look the set up by the `setId` in the `keyid` and reject a set whose statement id differs from the envelope's.
6. Accept a proof only against a root it currently publishes or the immediately previous epoch's root, and reject a root/epoch pair it does not have (§11).
7. Verify the disclosure coverage of §6.7.
8. Apply its pseudonym policy to `h`: rate limiting, ban, and session binding (§9).

### 6.7 Selective disclosure

`attrs` is a vector of fixed-width slots, and `disclosed` is what the proof publishes about it: for each slot, either the value or zero-with-the-mask-bit-clear. The `mask` public input records which slots were actually disclosed.

Because clause 6 of §6.4 constrains only `disclosed[i] = attrs[i] · maskBit[i]`, the relation does not prove that the prover disclosed the attributes the relying party asked for — a prover may disclose nothing. A relying party that requires attributes MUST verify that the mask covers every attribute named in its own `requiredDisclosures` (§5) and MUST reject a login that does not. The attribute names and the site's code vocabulary are published in the descriptor; attribute values on the wire are 32-bit integers.

### 6.8 Subsequent requests and token binding

A proof authenticates exactly one request. ANP-02 already provides for what comes next: the server returns an access token via `Authentication-Info` (ANP-02 L197), and §3.2.3 item 3 (L205–L215) reserves a **sender-constrained** token that binds the token to a key held by the client — while explicitly leaving that profile undefined. This Profile fills that slot.

```http
Authentication-Info: access_token="…", token_type="ANP-ZK-PoP", expires_in=3600,
                     cnf="<base64url(SHA-256(pk_s))>"
```

- The token's subject MUST be `h`, not a DID.
- The token MUST carry a confirmation claim (`cnf`) naming `pk_s`, computed as `SHA-256` over the encoded `pk_s`. The key itself MUST NOT be carried, because a token payload is readable by its bearer.
- Because the token type is not `Bearer`, ANP-02 L225 requires the client to send it according to this specification: subsequent requests present the token plus a proof of possession of `sk_s`, and MUST NOT be required to carry a fresh ZK proof.

```http
Authorization: ANP-ZK-PoP <access_token>
Signature-Input: pop1=("@method" "@target-uri" "content-digest");created=…;expires=…;nonce=…
Signature: pop1=:<base64url(R ‖ S)>:
```

`content-digest` participates only when the request has a body. The session signature is **EdDSA-Poseidon over BabyJubjub**, using `sk_s`:

```text
h = Poseidon(R.x, R.y, A.x, A.y, M)   over BN254, circomlib parameters
A = pk_s = sk_s·g                     g = the circomlib base point
S = (r + h·sk_s) mod SUBORDER,  R = r·g
verify: A in the prime-order subgroup ∧ A ≠ O ∧ S < SUBORDER ∧ S·g == R + h·A
```

The subgroup check and the `S < SUBORDER` check are both required: BabyJubjub has cofactor 8, so a low-order `A` makes the verification equation satisfiable without a private key, and `S + SUBORDER` satisfies the same equation, which would give every signature a twin encoding and break a replay cache keyed on signature bytes. `R` needs no subgroup check: the equation forces `R = S·g − h·A`.

## 7. Proof-system compatibility

The Profile does not mandate a proof system. It accepts any system the relying party has registered a verification key for, described by a three-part descriptor.

### 7.1 The descriptor triple

```text
(scheme, statementId, vkHash)
```

- `scheme` — the proof system, for example `groth16`, `plonk`, `halo2`, `stark`.
- `statementId` — §6.1. It fixes the curve and hash parameters.
- `vkHash` — `base64url(SHA-256(verificationKey))`, so a relying party pins a proof system's key material by hash rather than by URL.

All three MUST match a registered entry for a proof to be accepted. A dispatch failure — unknown scheme, unknown statement, unregistered `vkHash`, or an unknown set — MUST be reported as `invalid_verification_method` and needs no new error value (§12).

This triple has the same shape as the dependency-frame `(scheme, data_hash, verification_key_hash)` used by EIP-8288; see §13.

### 7.2 Proof envelope

The envelope is the value of the `Signature` header. Its JSON form is:

```json
{
  "v": 1,
  "scheme": "groth16",
  "statement": "anp.zklogin.v1",
  "vkHash": "<base64url>",
  "bytes": "<base64url>"
}
```

`bytes` is opaque to this Profile. Its encoding is fixed by `scheme` and MUST NOT vary within one `statementId`. The envelope MUST NOT carry public inputs (§6.2).

### 7.3 Verification-key reference

A relying party MUST publish how a client can obtain the verification keys it accepts, and MUST pin them by `vkHash` (§5). Implementations MUST NOT accept a verification key supplied by the client.

### 7.4 Portability interface

A conforming proof backend exposes exactly two operations: produce a proof from a statement and a witness, and verify a proof against a statement. The Profile's portable asset is the statement — §6.1 through §6.4 — not the backend. A backend that implements the statement can be swapped without changing the wire format, because the wire carries only the descriptor triple and an opaque proof.

### 7.5 Aggregation (informative)

A future revision may allow one aggregated proof to cover several statements (for example membership across several issuers). This proposal does not define it; it reserves the envelope's extensibility for it.

## 8. Enrollment

Enrollment is **out of scope for the wire protocol** and is governed by the membership set's issuer. This section states the format this Profile assumes and the constraints a relying party depends on.

```text
agent ──▶ issuer:  an ANP-02 DID-signature request (§8)
                 + { "setId": "…",
                     "zkPk":  "<base64url, 32-byte compressed BabyJubjub point>",
                     "salt":  "<base64url, 32 bytes>",
                     "attrsCommitment": "<decimal>" }

issuer ──▶ agent: 200 { "setId": "…", "leaf": "<decimal>", "setRoot": "<decimal>",
                        "epoch": 42,
                        "merklePath": { "index": 3, "siblings": ["<decimal>", … 20] } }
```

1. Enrollment MUST be authorized by an authenticated identity — RECOMMENDED: an ANP-02 DID-signature request, which is where the DID key is used exactly once (§4).
2. The four members MUST be covered by the request's `Content-Digest`, so that a relying party can refuse an enrollment that does not bind the `zkPk` it is admitting.
3. The issuer computes `leaf = H(leafTag, zkPk.x, zkPk.y, salt, attrsCommitment)` with `attrsCommitment = H(attrsTag, attrs)`, inserts `leaf` into the set, and returns the leaf, the new root, the epoch, and the Merkle path. Only the attribute commitment is transmitted; the attribute vector stays with the agent.
4. `salt` MUST be generated by the agent (or by the issuer and returned over a confidential channel), MUST be unpredictable, and MUST NOT be reused across credentials. Without it, low-entropy `attrs` are guessable from `leaf`.
5. The agent MUST verify the receipt before relying on it: the `setId` must be the one requested, the `leaf` must be the one its own key, salt and attributes produce, and the path must be of the statement's depth and must recompute to the returned `root`.
6. The `root` and `epoch` in a receipt go stale as the set changes. A client MUST re-read the descriptor (§5) before proving, and MUST NOT prove against the values it received at enrollment.
7. The issuer enforces its own admission policy. Since a login carries no DID, an issuer that requires an identity to be resolvable MUST check that at enrollment, and a relying party MUST NOT represent enrollment-time checks as login-time checks.
8. The issuing identity is the enrollment authority. An issuer MUST prevent the same credential from being admitted twice under one `setId`, and MUST NOT key that de-duplication on a claimed subject string: an identifier's method-specific path segment need not be bound to its key material, so two subjects may present one key. The key material itself (a fingerprint of it) is the sound de-duplication key. De-duplication is what makes a ban effective at all: banning `h` without it only forces a re-enrollment (§9.3).
9. `attrs` is disclosed to the issuer at enrollment in the clear unless the applicant submits a commitment it cannot open. Sets whose admission decisions require attributes therefore give the issuer knowledge of those attributes; the pseudonym `h` still does not reveal which login is which (§9.1).

### 8.1 Issuer-rooted sets

Membership-set construction — admission-based sets maintained by the relying party itself, and issuer-rooted sets where a third party admits members — is a governance topic this Profile does not standardize (§14 item 5). Both share §6.4 clause 2; an issuer-rooted set additionally verifies the issuer's credential inside the relation.

## 9. The relying party's account model and retention

### 9.1 The pseudonym is the account

`h` is the only durable identifier a relying party sees, and it is deterministic: it depends on `zkSk`, `V`, and the statement id and nothing else (§6.4 clause 3). For one agent at one service all three are fixed, so every login yields the **same** `h`. The pseudonym *is* the account:

| Situation | Same `h`? | What the relying party sees |
|---|---|---|
| Same agent, same service, repeated logins | Yes | One account |
| Same agent, different services | No | Two unlinkable accounts |
| Same agent, same service, different credential | No | A different account (§9.3 item 2) |

The Profile hides **cross-service** linkability, not within-service consistency. Within-service consistency is what makes rate limiting, bans, and reputation possible at all. The cost is that a relying party can build a long-term profile of one visitor, which §9.3 requires it to disclose.

`h` MUST be treated as an opaque handle: it is stable for a given (`zkSk`, `V`, statement id); it is derived from a secret that never leaves the agent, so it is not invertible to `zkSk`; and it is unlinkable to the agent's DID and to the pseudonyms the same agent presents elsewhere, provided `V` differs per service (§6.5).

### 9.2 What a relying party stores

No `h → DID` mapping exists in anonymous mode, and a relying party MUST NOT attempt to construct one. Its record for one visitor is keyed by `h` alone:

| Field | Purpose |
|---|---|
| `h` (primary key) | The whole of what the relying party knows about this visitor |
| `pk_s` | The session key; the access token's confirmation claim binds to it (§6.8) |
| `disclosed` | The attribute subset disclosed on this login |
| `escrow` | The escrow ciphertext, when the mode is `escrow` (§10) |
| `mode` / `statementId` / `scheme` / `vkHash` / `epoch` | Provenance of the proof, needed for audit and diagnosis |
| `created` / `expires` / `nonce` | Aligned with the ANP-02 freshness parameters |
| Decision | Accepted or rejected |

A relying party that also issues the membership set additionally retains the set's `setId`, `epoch`, and current root, **and its historical roots**, which §6.6's grace window and any later audit both require.

### 9.3 Consequences a relying party MUST accept

These are properties of the design, not defects, and a relying party MUST disclose them in its descriptor (§5):

1. **No outbound contact.** `h` is not a routable address. A relying party that must later reach the agent by DID cannot do so from `h`; this is the reason for the optional addressable mode (§12).
2. **No recovery.** If the agent loses `zkSk`, or re-enrolls with a different credential, the resulting `h` differs and the relying party sees a different account. This Profile defines no recovery, merge, or account-linking path.
3. **Banning is bounded by admission.** A relying party bans `h`; an agent that re-enrolls with a fresh credential obtains a fresh `h`. A ban therefore holds only to the extent the enrollment channel can refuse re-admission (§8). Bans MUST NOT be represented as an identity-level guarantee.
4. **DID revocation is not membership revocation.** A login carries no DID and resolves none, so retiring a DID does not remove the agent from a set; removal is the set's own operation (§8, §11). A relying party MUST NOT represent the two as linked.
5. **Risk controls are reduced.** A proof carries no device fingerprint, account linkage, or behavioural signal. A relying party's substitutes are rate, behaviour, optionally disclosed attributes, and the escrow path — nothing else. Traditional account-association and anomaly-detection controls largely do not apply.
6. **Proof of personhood is not provided.** One operator may enroll many DIDs; every one of them resolves. Enrollment-time resolution catches "not enrolled", not "one person, many credentials".

### 9.4 Retention of proofs

A proof is **single-use**: bound to one request by `baseHash`, one nonce, one `h`, and one freshness window (§6.4, §6.6). Authentication therefore proceeds as under ANP-02 — the proof is verified, a token is issued, and subsequent requests use the token rather than a new proof.

Retention is consequently not about future verification but about whether a third party can later **re-check the admission decision**. Deployments MUST declare one of two models:

- **Model A (RECOMMENDED default).** Store the public inputs, the decision, and the time; **discard the proof**. A later audit can only establish that the relying party recorded a passing verification at that time.
- **Model B.** Store the proof, or at minimum `hash(proof)`. Because the proof is a NIZK and therefore publicly verifiable, a third party holding the public inputs and the proof can recompute the decision without trusting the relying party.

A proof is not an identifier and provides no attribution: Model B's evidential value is confined to auditability of the admission decision. Attribution, where a deployment provides it at all, comes only from the escrow path (§10).

### 9.5 Rate limiting and bans

Rate limiting is applied per `h`. A relying party MUST NOT require a DID or other stable identifier as the price of admission, and MUST NOT fall back to a shared identifier to enforce a ban across services: cross-service banning is deliberately impossible, because `h` differs per service.

## 10. Privacy considerations

**Protected.** The caller's DID, public key, and enrollment attributes are not disclosed. Cross-service correlation is prevented as long as each service derives `V` from its own origin and services do not collude.

**Not protected.** Network metadata (source address, timing, request size, TLS SNI) is unchanged. A relying party learns that a request used ZK login. Two relying parties that observe the same `V` — because one is a front for the other, or because a client is coerced into reusing `V` — can correlate. Implementations SHOULD document that `V` is derived per origin and MUST NOT reuse it across origins. The anonymity set is the size of the membership set, so a small set provides little privacy; a relying party SHOULD publish its set size (§5).

**Audit modes.** A deployment may retain a deanonymization path for abuse handling. Two modes are defined: an *anonymous* mode in which no such path exists, and an *escrow* mode in which the login carries an escrow ciphertext readable only by a named auditor. The mode is chosen by the client and carried in the request body, which the `Content-Digest` binds, so a relying party may select which modes it accepts but cannot compel an agent to supply an escrow ciphertext. A relying party MUST disclose which mode it uses. Which default to recommend is an open question (§14 item 2).

The deanonymization axis is **independent** of the proof-retention models of §9.4. Implementations MUST NOT conflate them.

## 11. Security considerations

- **Soundness.** Security rests on the proof system's soundness and on the statement's constraints (§6.4). A relying party MUST reject proofs whose `scheme` it has not registered (§12).
- **Trusted setup.** Proof systems that require a trusted setup (for example Groth16) inherit its assumptions. A relying party that cannot accept those assumptions should register a transparent-setup system instead; the Profile does not force the choice (§7).
- **Replay.** Handled by the freshness window and the `(h, nonce)` replay cache (§6.6).
- **Membership-set freshness.** `root` changes as the set changes. A verifier MUST publish the epoch it is validating against and MUST define a grace window, of at least the maximum proof validity, during which a proof against the immediately previous root is still accepted, so that an enrollment or removal does not invalidate in-flight requests. The window length beyond that minimum is a policy choice.
- **Proving cost.** Proof generation is deliberately client-side. A relying party MUST NOT require proof generation on its own infrastructure for requests it did not initiate.
- **Denial of service.** Proof verification is more expensive than signature verification. A relying party SHOULD apply the same request-level admission controls it uses for any unauthenticated endpoint.
- **Error observability.** Distinct failures MUST NOT be distinguishable in a way that discloses membership: "not a member", "removed from the set", and "proof does not verify" MUST all be reported as `invalid_signature` (§6.6 item 6).

## 12. Deviation from ANP-02

This Profile makes exactly one deliberate departure from ANP-02, and it is the reason anonymous mode cannot be expressed as an ordinary ANP-02 request:

**`keyid`.** ANP-02 L82 requires `keyid` to be a complete DID URL, and Appendix B L487 restates it. In anonymous mode there is no DID to point at. This Profile therefore sets `keyid` to

```text
zk:anp-zk/1:<setId>:<h>
```

where `setId` is the membership set and `h` is the pseudonym, `base64url`-encoded. The `zk:anp-zk/1:` prefix carries the Profile version, so a later revision is distinguishable rather than looking like a malformed DID. `setId` MUST NOT contain `:`, or two sets could produce one `keyid`.

Three consequences a relying party MUST handle:

1. The ANP-02 replay cache keyed on `(keyid, nonce)` (L144, L437) becomes `(h, nonce)`.
2. Any ANP-02 clause that resolves a key *from* `keyid` does not apply (L130, L159); verification keys are obtained per §7.3.
3. `alg` carries the scheme (`zk:groth16`, `zk:plonk`), which ANP-02 treats as optional and non-binding (L87). A verifier MUST dispatch on it against its registered schemes.

A relying party that receives a `keyid` it does not recognise as a ZK `keyid` MUST treat the request as an ordinary ANP-02 request and follow that path.

**Addressable mode (optional).** A relying party that must later address the agent by DID cannot do so from `h`. Such a deployment MAY request an additional, explicitly labelled proof binding the credential to a fresh, single-use DID whose key is the session key `pk_s` — which then also keeps the `keyid` a genuine DID URL, for zero deviation. This proposal does not define that proof; it is listed as an open question (§14 item 1). A relying party MUST NOT silently require it.

**Error mapping.** The Profile reuses the ANP-02 error vocabulary (L252–L260) as follows:

| Condition | Error value |
|---|---|
| Malformed envelope, malformed body, missing fields, unknown statement, covered components missing | `invalid_request` |
| Nonce reused or not matching the challenge (in the header or the body) | `invalid_nonce` |
| Timestamp outside the window | `invalid_timestamp` |
| Proof fails verification; root/epoch not accepted; not a member | `invalid_signature` |
| Unknown scheme, unregistered `vkHash`, unknown set, set/statement mismatch | `invalid_verification_method` |
| `Content-Digest` mismatch | `invalid_content_digest` |
| Access-token binding failed | `invalid_access_token` |
| Verification could not be performed (internal failure) | `unverifiable` (a 5xx, never a 401 challenge) |

`invalid_did` (L255) has no meaning in anonymous mode and is not used. A pseudonym-policy refusal has no dedicated value: `forbidden_did` is the only available one, which is a naming compromise; whether to add a dedicated value is an open question (§14 item 3).

## 13. Relationship to existing documents

| Document | Relationship |
|---|---|
| [ANP-02](../02-anp-did-authentication-protocol-specification.md) | The carrier this Profile rides on. §3 header set, `Content-Digest`, freshness, the 401 challenge, the §4 JSON carriage, the error vocabulary, and the reserved sender-constrained token slot (§6.8) are all reused. Enrollment uses the ANP-02 DID-signature flow (§8). |
| [ANP-03](../03-did-wba-method-design-specification.md) | Governs the enrollment identity only. L205 states that fields defined elsewhere may be supported selectively, which is the clause under which this Profile's additions are optional and non-breaking. |
| [ANP-06](../06-anp-agent-communication-meta-protocol-specification.md) | Not modified. A relying party MAY advertise this Profile through the same capability mechanisms it uses for other Profiles. |
| ANP Messaging P1–P9 | Not modified and not a dependency. Authentication and messaging are separate concerns, as ANP-02 states in §1. |
| [ANP-10](../application/10-anp-agent-payment-protocol-specification.md) | Not modified. Its own principle of "support selective disclosure and minimize unnecessary information leakage" is a motivation for this Profile, but the two are independent. |

**EIP-8288 isomorphism (informative).** The descriptor triple in §7.1 — `(scheme, statementId, vkHash)` — has the same shape as EIP-8288's `(scheme, data_hash, verification_key_hash)`. The two arrive at the same structure from different directions: EIP-8288 frames a signed artifact by naming the verification dependency, while this Profile names the proof's. A future revision could align the field names if the community finds the parallel useful.

## 14. Open questions

1. **Weakening the deviation further.** §12 removes `keyid`. An "addressable mode" that keeps a DID-shaped `keyid` would make the deviation zero, at the cost of disclosing a freshly minted DID. Which should be the default for a relying party that also needs to address the agent?
2. **Audit mode default.** Should the default be `anonymous` (no deanonymization path) or `escrow` (a named auditor can open a credential)? §10 leaves this open because it is a governance and liability choice, not a cryptographic one.
3. **Error vocabulary.** Is reusing `forbidden_did` for pseudonym-policy rejections acceptable, or should the ANP-02 error set gain a dedicated value?
4. **Ownership.** Should this live as an authentication Profile beside ANP-02 (as proposed here), as a numbered document under `application/`, or as an independent extension outside the core set? This proposal assumes the first.
5. **Membership-set governance.** No part of this proposal defines who may issue credentials or how sets federate (§8.1). If the community wants a standard set interface, it should be a separate document.

## References

- [ANP-02] ANP DID Authentication Protocol, Version 1.2 — `./02-anp-did-authentication-protocol-specification.md`
- [ANP-03] did:wba Method Design Specification, Version 1.2 — `./03-did-wba-method-design-specification.md`
- [RFC 9421] HTTP Message Signatures
- [RFC 9530] Digest Fields
- [BCP 14] RFC 2119 / RFC 8174 — requirement keywords
- [EIP-8288] Dependency Frame (informative analogue)

## Copyright Notice

Copyright (c) 2024 ANP Open Source Community
This file is released under the [Apache License 2.0](../LICENSE). You are free to use and modify it, but you must retain this copyright notice.

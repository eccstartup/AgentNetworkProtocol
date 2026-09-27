# ANP Zero-Knowledge Authentication Profile (Proposal)

- Document ID: Proposal (no ANP number assigned)
- Proposed Profile identifier: `anp.auth.zklogin.v1`
- Title: ANP Zero-Knowledge Authentication Profile
- Status: Proposal / for community discussion; not a released specification
- Version: 0.1
- Language: English
- Chinese mirror: [ANP 零知识认证 Profile（提案）](../chinese/proposals/anp-zk-login-proposal.md)
- Issue draft: [anp-zk-login-issue-draft.md](anp-zk-login-issue-draft.md)
- Applicability: This proposal applies to request authentication between an agent and a relying party, as an opt-in alternative to the DID-signature flow of [ANP-02](../02-anp-did-authentication-protocol-specification.md).

> **This is a proposal, not a specification.** It proposes a new opt-in Profile for the ANP protocol set and does not modify any released document. Per [CONTRIBUTING](../CONTRIBUTING.md), it should be introduced through a GitHub Issue and Discord discussion before any PR that would promote it to a released document. Section numbers, identifiers, and error mappings below are proposals and may change during review.

> **Why a separate Profile rather than an ANP-02 revision.** ANP-02 §3.2.1 step 4 (L110) and §3.2.2 step 4 (L163) already defer algorithms and key representation to the selected verification method type, and Appendix D (L503) already anticipates other methods supplying "identity and authentication-key material through their own resolution and verification rules". This proposal adds a Profile beside the existing ones instead of extending ANP-02 itself.

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
| `sk` / `pk` | The ZK-curve proving key and its corresponding public key (§4) |
| `H_dom(tag, …)` | A hash from the statement's declared hash family, domain-separated by `tag` (§6.1) |
| `salt` | A per-credential random value chosen at enrollment (§8) |
| `attrs` | The disclosed attribute set committed at enrollment; MAY be empty |
| `L`, `root`, `index`, `path` | A membership-set leaf, its Merkle root, the leaf index, and the Merkle path |
| `V` | The audience value, derived by the client from the connected origin (§6.5) |
| `baseHash` | `SHA-256` over the RFC 9421 signature base of the request (§6.6) |
| `h` | The per-service pseudonym (§6.4 clause 3) |
| `sk_s` / `pk_s` | The session key and its public key; the access token binds to `pk_s` (§6.8) |

## 3. Design goals and non-goals

**Goals.** (a) No DID, public key, or stable identifier on the wire. (b) Full request binding: a proof is valid only for one method, target URI, authority, body, and freshness window. (c) Zero modification to released ANP documents. (d) Backend-agnostic: a relying party accepts any proof system it has registered a verification key for (§7). (e) Pseudonym stability per service, so a relying party can rate-limit, ban, and keep a session.

**Non-goals.** Hiding network metadata; global unlinkability across colluding relying parties; defining membership governance; replacing DID-based authentication where the relying party needs to address the caller by DID.

## 4. Two layers of identity

A naive design proves membership of the DID key itself. That forces non-native field arithmetic — Ed25519 and secp256k1 scalar multiplication inside a BN254 circuit — which costs roughly an order of magnitude more constraints than a native operation. This Profile therefore separates the two roles:

1. **The DID key** is used once, at enrollment, to authorize admission (§8). It is never used at login and never appears on the wire.
2. **The proving key `sk`** is a key on the ZK circuit's own curve. All login proofs use `sk`.

The membership leaf commits only to `pk` and `attrs` (§6.4 clause 2). A relying party therefore cannot recover the enrollment DID from anything it receives, and the DID's method rules (ANP-03 / [Appendix B](../appendix-b-compatibility-with-native-did-web.md)) govern enrollment only.

Implementations MUST use distinct keys for enrollment and proving. Reusing a DID key as `sk` does not break correctness but defeats goal (a) if the key is ever published as part of the DID document.

## 5. Opt-in and discovery

A relying party opts in by advertising the Profile. Two channels are defined; a relying party MAY use either or both.

1. **Capability advertising.** The relying party lists `anp.auth.zklogin.v1` in its advertised supported Profiles, together with a verification-key reference (§7.3). Implementations that already publish an agent description or service descriptor use that channel.
2. **Challenge.** A relying party that receives a request it wants to authenticate by proof responds `401 Unauthorized` and appends a challenge, in the manner of ANP-02 L238–L244:

```http
WWW-Authenticate: ANPZK realm="api.example.com", profile="anp.auth.zklogin.v1",
  statement="anp.auth.zklogin.v1/<params-hash>", vk="https://api.example.com/zk/vk.json",
  nonce="xyz987", error="invalid_signature"
Accept-Signature: sig1=("@method" "@target-uri" "@authority" "content-digest")
```

**Fallback rules.**

- A client that supports this Profile and receives a challenge naming `anp.auth.zklogin.v1` MUST respond with a proof request, not a DID-signed request.
- A client that does not support it MUST ignore the challenge and use ANP-02 authentication; the relying party MAY then accept or reject that attempt at its own policy (see the `required` note below).
- A relying party that has declared an operation as **ZK-required** (for example, by advertising only this Profile for it) MUST NOT accept a DID-signature login for that operation. A client MUST NOT treat such a rejection as a reason to retry with a DID signature.
- A relying party MUST NOT silently downgrade a ZK-required operation to ANP-02 authentication. This mirrors the "MUST NOT silently downgrade" discipline already used for mixed-version messaging Profiles.

## 6. The proof statement

### 6.1 Statement identifier and parameters

The statement identifier carries the *relation* and every parameter that, if changed, would change the meaning of a proof. It is a string of the form

```text
statementId := "anp.auth.zklogin.v1" "/" paramsHash
paramsHash := base64url(SHA-256(canonical-JSON(params)))
params     := { "curve": …, "leafHash": …, "merkleArity": …, "encoding": …, "relationRev": … }
```

`canonical-JSON` is the JCS canonicalization already used elsewhere in ANP. Two rules:

- A prover and a verifier MUST agree on `statementId` exactly; a mismatch is a different Profile instance and MUST be rejected.
- Implementations MUST NOT change any member of `params` while keeping the same `statementId`. In particular the hash family MUST be fixed by `params.leafHash`, not chosen per deployment.

The reason is that the pseudonym `h` is derived from a witness value and is used by a relying party as a durable handle (§9). If the hash family changed without the identifier changing, the same agent would silently become a different pseudonym at the same service. Binding the parameters into the identifier makes any such change an explicit, negotiated break.

### 6.2 Public inputs

| Input | Source | Binds |
|---|---|---|
| `root` | Membership-set epoch state (§8) | Which set the caller belongs to |
| `V` | Derived by the client from the connected origin (§6.5) | The audience |
| `baseHash` | Recomputable by the verifier from the actual request (§6.6) | The exact request |
| `h` | Pseudonym, public by construction | The caller's per-service handle |
| `pk_s` | Declared by the client; the issued token binds to it (§6.8) | The session key |
| `bindTag` | Public output (§6.4 clause 7) | Ties the above together |
| `nonce`, `created`, `expires` | The ANP-02 freshness parameters | Freshness |

### 6.3 Witness (never transmitted)

`sk`, `sk_s`, `salt`, `attrs`, `index`, and the Merkle `path` from `L` to `root`.

### 6.4 The relation

A proof is valid only if there exist witness values satisfying all of:

1. **Key knowledge.** `pk = derive(sk)` for the statement's declared key derivation.
2. **Membership.** `L = H_dom(TAG_LEAF, pk, salt, H_dom(TAG_ATTR, attrs))`, and `L` is the leaf at position `index` under `root` along `path`.
3. **Pseudonym correctness.** `h = H_dom(TAG_PSEUDONYM, sk, V, statementId)`.
4. **Disclosure consistency.** The disclosed `attrs` (if the Profile is used with selective disclosure, §6.7) are exactly the `attrs` committed in clause 2.
5. **Session-key knowledge.** `pk_s = derive(sk_s)`, so the caller proves it holds the session key that `pk_s` names. The issued token binds to `pk_s` (§6.8), which is why this clause cannot be omitted: without it, any caller could name any `pk_s`.
6. **Request binding.** `baseHash` is included in the statement's public inputs, so a proof produced for one request does not verify against another request's public inputs.
7. **Combined binding (public output).** The relation publishes
   `bindTag = H_dom(TAG_BIND, h, baseHash, V, root, nonce)`.
   The verifier MUST recompute `bindTag` from its own view of `h`, `baseHash`, `V`, `root`, and `nonce`, and MUST reject on mismatch. Because clause 7 constrains all six values together inside the circuit, none of them can be substituted after proving.

### 6.5 Client-side derivation of `V`

The client MUST derive `V` from the origin it is actually connected to — the scheme and authority of the real request — and MUST NOT accept a `V` supplied by the relying party.

**Rationale.** If the relying party chose `V`, two colluding relying parties could agree on one `V`, observe the same `h` at both, and thereby link the caller across services. That would defeat the Profile's purpose. Deriving `V` client-side from the connected origin makes the pseudonym depend on a value the relying party cannot influence. This is the same discipline ANP-02 already applies to `@target-uri`, where the verifier is expected to derive the value "from the real request, not from a caller-supplied target URI alone".

The verifier MUST independently compute the expected `V` for the request it received and MUST reject a proof whose `V` does not match.

### 6.6 Verifier-side checks

Beyond validating the proof itself, a verifier MUST:

1. Recompute `baseHash` from the actual request (method, target URI, authority, `Content-Digest`, and the covered options) and reject on mismatch. Do not trust a `baseHash` supplied by the client.
2. Recompute `V` per §6.5 and reject on mismatch.
3. Apply the same freshness window as ANP-02 §3.2.1 step 7 (L141), with `expires` treated as SHOULD (ANP-02 L85): when absent, bound the window by `created` alone.
4. Maintain a replay cache. In anonymous mode the ANP-02 `(keyid, nonce)` cache (ANP-02 L144, L437) becomes `(h, nonce)`. A nonce MUST be usable once per `h`.
5. Apply its pseudonym policy to `h`: rate limiting, ban, and session binding (§9).

### 6.7 Selective disclosure (optional)

A Profile instance MAY declare a disclosure set. When it does, the relation additionally proves that the disclosed attributes equal the committed ones (clause 4) without revealing the remainder. A relying party that does not need attributes uses an empty `attrs`, which is the RECOMMENDED default.

### 6.8 Subsequent requests and token binding

A proof authenticates exactly one request. ANP-02 already provides for what comes next: the server returns an access token via `Authentication-Info` (ANP-02 L197), and §3.2.3 item 3 (L205–L215) reserves a **sender-constrained** token that binds the token to a key held by the client — while explicitly leaving that profile undefined.

This Profile fills the reserved slot with the session key `pk_s` of §6.4 clause 5. The token carries a confirmation claim naming `pk_s`, so a stolen token is useless without `sk_s`. Subsequent requests on the session present the token and prove possession of `sk_s`; they do **not** carry a new proof. A relying party:

- MUST bind the token it issues to the `pk_s` that the proof declared, and
- MUST NOT require a fresh ZK proof for each request on a session it has already established.

Because the token type is not `Bearer`, ANP-02 L225 requires the client to send it according to this specification: the token is presented with proof of possession of `sk_s`, not as a bearer credential. A deployment that does not need sender-constrained tokens MAY issue `Bearer` tokens as ANP-02 permits (L215), but SHOULD prefer binding, since `pk_s` is already available at no extra cost.

## 7. Proof-system compatibility

The Profile does not mandate a proof system. It accepts any system the relying party has registered a verification key for, described by a three-part descriptor.

### 7.1 The descriptor triple

```text
(scheme, statementId, vkHash)
```

- `scheme` — the proof system, for example `groth16`, `plonk`, `halo2`, `stark`.
- `statementId` — §6.1. It embeds the curve and hash parameters.
- `vkHash` — `sha256-<base64url(SHA-256(canonical-JSON(verificationKey)))>`, so a relying party can pin a proof system's key material by hash rather than by URL.

All three MUST match a registered entry for a proof to be accepted. Changing any one of them yields a different verification key and therefore a different Profile instance.

This triple is structurally the same as the dependency-frame `(scheme, data_hash, verification_key_hash)` shape used by EIP-8288; see §13.

### 7.2 Proof envelope

When a proof is carried in the JSON carriage (§7.4), it is an object:

```json
{
  "scheme": "groth16",
  "statementId": "anp.auth.zklogin.v1/<params-hash>",
  "vkHash": "sha256-<base64url>",
  "proof": "<opaque, encoding fixed by scheme>"
}
```

`proof` is opaque to this Profile. Its encoding is fixed by `scheme` and MUST NOT vary within one `statementId`.

### 7.3 Verification-key reference

A relying party MUST publish how a client can obtain the verification keys it accepts, and MUST pin them by `vkHash`. A relying party MAY serve them from a stable HTTPS URL and MAY advertise that URL in the challenge (`vk=` in §5). Implementations MUST NOT accept a verification key supplied by the client.

### 7.4 Portability interface

A conforming proof backend exposes exactly two operations: produce a proof from a statement and a witness, and verify a proof against a statement. The Profile's contribution is the *portable statement* — §6.2 through §6.4 — not the backend. A backend that implements the statement can be swapped without changing the wire format, because the wire carries only the descriptor triple and an opaque proof.

### 7.5 Aggregation (informative)

A future revision may allow one aggregated proof to cover several statements (for example, membership plus attribute disclosure). This proposal does not define it; it reserves the envelope's extensibility for it.

## 8. Enrollment

Enrollment is **out of scope for the wire protocol** and is governed by the membership set's issuer. This section states only the constraints a relying party depends on.

1. Enrollment MUST be authorized by an authenticated identity — RECOMMENDED: an ANP-02 DID-signature request, which is where the DID key is used exactly once (§4). The enrolling agent then submits `pk` and `attrs`.
2. The issuer computes `L = H_dom(TAG_LEAF, pk, salt, H_dom(TAG_ATTR, attrs))` with a freshly generated `salt`, inserts `L` into the set, and returns `(index, path, salt)` to the agent.
3. `salt` MUST be unpredictable and MUST NOT be reused across credentials. Without it, low-entropy `attrs` are guessable from `L`.
4. The issuer enforces its own admission policy, including one-credential-per-subject if its policy requires it. The record it keeps to do so is issuer-local and is not on the wire.

Membership-set construction — admission-based sets, issuer-rooted sets, and their tradeoffs — is a governance topic that this Profile does not standardize (§14 item 6).

## 9. The relying party's account model and retention

### 9.1 The pseudonym is the account

`h` is the only durable identifier a relying party sees, and it is deterministic: it depends on `sk`, `V`, and `statementId` and nothing else (§6.4 clause 3). For one agent at one service all three are fixed, so every login yields the **same** `h`. The pseudonym *is* the account:

| Situation | Same `h`? | What the relying party sees |
|---|---|---|
| Same agent, same service, repeated logins | Yes | One account |
| Same agent, different services | No | Two unlinkable accounts |
| Same agent, same service, different credential | No | A different account (§9.3 item 2) |

The Profile hides **cross-service** linkability, not within-service consistency. Within-service consistency is what makes rate limiting, bans, and reputation possible at all. The cost is that a relying party can build a long-term profile of one visitor, which §10 requires it to disclose.

`h` MUST be treated as an opaque handle:

- It is stable for a given (`sk`, `V`, `statementId`), so a relying party can rate-limit, ban, and maintain a session keyed on it.
- It is unlinkable to the agent's DID and to the pseudonyms the same agent presents elsewhere, provided `V` differs per service (§6.5).
- The same agent MAY present different pseudonyms to the same service over time only by using a different credential; the Profile does not define credential rotation.

A relying party MUST NOT require the agent to reveal a DID or other identifier as a condition of a ZK login on an operation it declared ZK-required. It MAY offer a separate, explicitly labelled path when it needs to address the agent (§9.3 item 1, addressable mode).

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
2. **No recovery.** If the agent loses `sk`, or re-enrolls with a different credential, the resulting `h` differs and the relying party sees a different account. This Profile defines no recovery, merge, or account-linking path.
3. **Banning is bounded by admission.** A reliance party bans `h`; an agent that re-enrolls with a fresh credential obtains a fresh `h`. A ban therefore holds only to the extent the enrollment channel can refuse re-admission (§8). Bans MUST NOT be represented as an identity-level guarantee.
4. **Risk controls are reduced.** A proof carries no device fingerprint, account linkage, or behavioural signal. A relying party's substitutes are rate, behaviour, optionally disclosed attributes, and the escrow path — nothing else. Traditional account-association and anomaly-detection controls largely do not apply.

### 9.4 Retention of proofs

A proof is **single-use**. It is bound to one request by `baseHash`, one `h`, one `nonce`, and one freshness window (§6.4, §6.6); once that window passes and the nonce is consumed, re-verifying it establishes nothing. Authentication therefore proceeds as it does under ANP-02: the proof is verified, a session or access token is issued, and subsequent requests use the token rather than a new proof.

Retention is consequently not about future verification. It is about whether a third party can later **re-check the admission decision**. Retention is independent of the deanonymization mode in §10 — a deployment may be anonymous and retain proofs, or escrowed and discard them. The Profile defines two models:

- **Model A (RECOMMENDED default).** Store the public inputs, the decision, and the time; **discard the proof**. A later audit can only establish that the relying party recorded a passing verification at that time.
- **Model B.** Store the proof, or at minimum `hash(proof)`. Because the proof is a NIZK and therefore publicly verifiable, a third party holding the public inputs and the proof can recompute the decision without trusting the relying party.

A relying party MUST declare which model it uses, and MUST NOT retain a proof beyond what that model requires. Note that a proof is not an identifier and provides no attribution: it is intrinsically non-identifying, so Model B's evidential value is confined to auditability of the admission decision. Attribution, where a deployment provides it at all, comes only from the escrow path (§10).

### 9.5 Rate limiting and bans

Rate limiting is applied per `h`. A relying party MUST NOT require a DID or other stable identifier as the price of admission, and MUST NOT fall back to a shared identifier to enforce a ban across services: cross-service banning is deliberately impossible, because `h` differs per service.

## 10. Privacy considerations

**Protected.** The caller's DID, public key, and enrollment attributes are not disclosed. Cross-service correlation is prevented as long as each service derives `V` from its own origin and services do not collude.

**Not protected.** Network metadata (source address, timing, request size, TLS SNI) is unchanged. A relying party learns that a request used ZK login. Two relying parties that share `V` — for example because one is a front for the other, or because a client is coerced into reusing `V` — can correlate. Implementations SHOULD document that `V` is derived per origin and MUST NOT reuse it across origins.

**Audit modes (proposal, not settled).** A deployment may wish to retain a deanonymization path for abuse handling. Two modes are proposed: an *anonymous* mode in which no such path exists, and an *escrow* mode in which the credential carries an escrow ciphertext readable only by a named auditor. The default is an open question (§14 item 2). A relying party MUST disclose which mode it uses. The mode is chosen by the client and carried as a public input, so a relying party may select which modes it accepts but cannot compel an agent to supply an escrow ciphertext.

This deanonymization axis is **independent** of the proof-retention models of §9.4. A deployment may be anonymous and retain proofs, or escrowed and discard them; the two decisions answer different questions ("can this login ever be attributed to a real identity?" versus "can a third party re-check the admission decision?"). Implementations MUST NOT conflate them.

## 11. Security considerations

- **Soundness.** Security rests on the proof system's soundness and on the statement's constraints. A relying party MUST reject proofs whose `scheme` it has not registered (see the error mapping in §12).
- **Trusted setup.** Proof systems that require a trusted setup (for example, Groth16) inherit its assumptions. A relying party that cannot accept those assumptions should register a transparent-setup system instead; the Profile does not force the choice (§7).
- **Replay.** Handled by the freshness window and the `(h, nonce)` replay cache (§6.6).
- **Membership-set freshness.** `root` changes as the set changes. A verifier MUST publish the epoch it is validating against and MUST define a grace window during which a proof against the immediately previous root is still accepted, so that an enrollment or removal does not invalidate in-flight requests. The window length is a policy choice.
- **Proving cost.** Proof generation is deliberately client-side. A relying party MUST NOT require proof generation on its own infrastructure for requests it did not initiate.
- **Denial of service.** Proof verification is more expensive than signature verification. A relying party SHOULD apply the same request-level admission controls it uses for any unauthenticated endpoint.

## 12. Deviation from ANP-02

This Profile makes exactly one deliberate departure from ANP-02, and it is the reason anonymous mode cannot be expressed as an ordinary ANP-02 request:

**`keyid`.** ANP-02 L82 requires `keyid` to be a complete DID URL, and Appendix B L487 restates it. In anonymous mode there is no DID to point at. This Profile therefore permits `keyid` to be either omitted, with the identity material carried in the proof envelope, or set to the pseudonym `h`, which is not a DID URL.

Two consequences a relying party MUST handle:

1. The ANP-02 replay cache keyed on `(keyid, nonce)` (L144, L437) becomes `(h, nonce)`.
2. Any ANP-02 clause that resolves a key *from* `keyid` does not apply; verification keys are obtained per §7.3.

**Addressable mode (optional).** A relying party that must later address the agent by DID cannot do so from `h`. Such a deployment MAY request an additional, explicitly labelled proof binding the credential to a DID or handle, disclosed only to that relying party. This proposal does not define that proof; it is listed as an open question (§14). A relying party MUST NOT silently require it.

**Error mapping.** The Profile reuses the ANP-02 error vocabulary (L252–L260) as follows:

| Condition | Error value |
|---|---|
| Malformed envelope, missing fields, unknown `scheme` | `invalid_request` |
| Nonce reused or unknown | `invalid_nonce` |
| Timestamp outside the window | `invalid_timestamp` |
| Proof fails verification | `invalid_signature` |
| `statementId` or `vkHash` not registered | `invalid_verification_method` |
| `Content-Digest` mismatch | `invalid_content_digest` |
| Pseudonym not permitted by policy | `forbidden_did` |
| Access-token binding failed | `invalid_access_token` |

`invalid_did` (L255) has no meaning in anonymous mode and is not used. Reusing `forbidden_did` as a pseudonym-policy error is a naming compromise; whether to add a dedicated value is an open question (§14).

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

1. **Weakening the deviation further.** §12 removes `keyid`. An "addressable mode" that keeps a DID-shaped `keyid` would make the deviation zero, at the cost of disclosing a DID. Which should be the default for a relying party that also needs to address the agent?
2. **Audit mode default.** Should the default be `anonymous` (no deanonymization path) or `escrow` (a named auditor can open a credential)? §10 leaves this open because it is a governance and liability choice, not a cryptographic one.
3. **Error vocabulary.** Is reusing `forbidden_did` for pseudonym-policy rejections acceptable, or should the ANP-02 error set gain a dedicated value?
4. **Ownership.** Should this live as an authentication Profile beside ANP-02 (as proposed here), as a numbered document under `application/`, or as an independent extension outside the core set? This proposal assumes the first.
5. **Enrollment channel.** Is ANP-02 DID-signature enrollment (§8) the right RECOMMENDED channel, or should enrollment be method-agnostic from the start?
6. **Membership-set governance.** No part of this proposal defines who may issue credentials or how sets federate. If the community wants a standard set interface, it should be a separate document.

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

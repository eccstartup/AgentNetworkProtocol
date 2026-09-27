# ANP Blockchain Account Identity (CAIP-10) Integration (Proposal)

- Document ID: Proposal (no ANP number assigned)
- Proposed identifier: `anp.identity.caip10.v1`
- Title: Blockchain Account Identity (CAIP-10) Integration
- Status: Proposal / for community discussion; not a released specification
- Version: 0.1
- Language: English
- Chinese mirror: [ANP 区块链账户身份（CAIP-10）集成（提案）](../chinese/proposals/anp-caip10-identity-proposal.md)
- Issue draft: [anp-caip10-identity-issue-draft.md](anp-caip10-identity-issue-draft.md)
- Applicability: This proposal applies to request authentication between an agent and a relying party when the agent's identity is a blockchain account (CAIP-10), as an opt-in method binding alongside [ANP-02](../02-anp-did-authentication-protocol-specification.md)'s existing bindings for `did:wba` and native `did:web`.

> **This is a proposal, not a specification.** It proposes a new method binding for the ANP protocol set and does not modify any released document. Per [CONTRIBUTING](../CONTRIBUTING.md), it should be introduced through a GitHub Issue and Discord discussion before any PR that would promote it to a released document. Section numbers, identifiers, and error mappings below are proposals and may change during review.

> **Why a new method binding rather than an ANP-02 revision.** ANP-02 §1 (L14) states that "the authentication flow is independent of the DID method", and §2 (L28) states that "document authenticity, identity binding, and lifecycle rules belong to each DID method". Appendix D (L503) already anticipates that other methods "may supply identity and authentication-key material through their own resolution and verification rules". This proposal adds a binding of that kind instead of extending ANP-02 itself. It differs from Appendix C and Appendix D in one respect that §3.3 makes precise: a CAIP-10 account id is **not** a W3C DID Core identifier, so the letter of Appendix D does not reach it.

## 1. Summary

ANP-02 authenticates a request by a signed HTTP message whose `keyid` names the caller's identity. Today an implementation may supply that identity material two ways: `did:wba` (ANP-03 / ANP-02 Appendix A) and native `did:web` (ANP-02 Appendix B). Both are DID methods. A large part of the agent ecosystem, however, already holds an identity that is not a DID at all: a **blockchain account**, canonically written as a CAIP-10 account id such as `eip155:1:0x…`.

This proposal defines `anp.identity.caip10.v1`: an **opt-in** method binding in which the agent's identity for ANP authentication is a CAIP-10 account id, and the key that signs the request is bound to that identifier by a check the existing bindings do not use — **the address is derived from the public key**. Nothing about the account's key is fetched; the identifier itself commits to the key.

The binding is deliberately additive:

- It reuses the ANP-02 common flow (§3 header set, `Content-Digest`, the 401 challenge, §4 JSON carriage, the access token, and the error vocabulary) rather than defining new transport.
- A relying party that does not implement it is unaffected, and a client that does not implement it keeps using `did:wba` or `did:web`.
- Nothing in ANP-01 through ANP-10 changes.

### 1.1 What this proposal does not do

- It does not make a blockchain account private. A CAIP-10 account id is a public, globally correlatable identifier; this binding discloses it on the wire exactly as ANP-02 discloses a DID. See §10.
- It does not cover smart-contract accounts (for example ERC-4337 wallets), whose address is not derived from the signing key. See §5.3.
- It does not define a key-rotation or recovery path. See §8.
- It does not define how a chain's addresses or signatures are produced; that belongs to the chain's own standards.

## 2. Conventions

The key words MUST, MUST NOT, REQUIRED, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL are to be interpreted as described in BCP 14.

Notation used below:

| Symbol | Meaning |
|---|---|
| `chain_id` | A CAIP-2 blockchain identifier, `namespace:reference` (for example `eip155:1`) |
| `account_address` | The account address on that chain, as CAIP-10 defines it |
| `account_id` | `chain_id ":" account_address`, the CAIP-10 account id |
| `pk` / `sk` | The account's signing public/private key |
| `derive(pk)` | The chain's address-derivation function (§5.1) |
| `keyid` | The RFC 9421 `keyid` parameter (ANP-02 L82) |

## 3. Design goals and non-goals

**Goals.** (a) Let an agent whose identity is a blockchain account authenticate under ANP-02 without first converting to a DID. (b) Bind the identifier to the signing key by a check that needs no fetch and no document proof. (c) Zero modification to released ANP documents. (d) Stay chain-agnostic: the binding MUST work for any CAIP-2 namespace that has an address-derivation function, not only EVM. (e) Preserve the ANP-02 discipline that a verifier MUST NOT accept a key the identifier does not authorize.

**Non-goals.** Hiding the account (a blockchain address is public by nature); covering contract accounts; defining chain semantics; replacing `did:wba` / `did:web` where the relying party needs a DID.

### 3.1 Relationship to the existing bindings

The two existing bindings bind an identifier to a key in two different ways:

| Binding | What ties the identifier to the key |
|---|---|
| `did:wba` path type (ANP-03) | The last path segment is an **RFC 7638 thumbprint of the public key** |
| native `did:web` (Appendix B) | **Nothing in the identifier**; the tie is the domain's TLS certificate plus the document the domain serves (B.1 L477) |
| CAIP-10 (this proposal) | The `account_address` is **derived from the public key** (`derive(pk)`) |

CAIP-10 therefore belongs to the same family as the `did:wba` path type — the identifier commits to a key — and the inline document is trustworthy for the same reason (§5.2). It is unlike `did:web`, whose identifier commits to nothing and whose key must be resolved over the network.

### 3.2 Why this cannot be an ordinary ANP-02 request

ANP-02 L82 requires `keyid` to be "a complete DID URL". A CAIP-10 account id is not a DID URL: it has no `did:` scheme, and CAIP-10's own grammar forbids `:` inside the address precisely so the identifier cannot be mistaken for a DID URL's method-specific id. This proposal therefore makes exactly one deliberate departure from ANP-02 (§12), the same shape of departure the ZK-login proposal makes. It is the reason a CAIP-10 login cannot be expressed as an ordinary ANP-02 request.

### 3.3 Why Appendix D does not already cover this

Appendix D (L503) is written for "other W3C DID Core methods". A CAIP-10 account id is not a DID Core identifier: it is not `did:`-scheme, it has no DID document that a method resolver can be asked for, and it has no `did:` method specification. The adaptation slot that Appendix D opens is the right shape, but its stated scope excludes CAIP-10. This proposal asks the community either to widen that scope or to receive this binding as a separate appendix; §14 item 4 leaves the choice open.

## 4. The identifier

### 4.1 Syntax

The identity is a CAIP-10 account id, exactly as CAIP-10 defines it:

```text
account_id      := chain_id ":" account_address
chain_id        := namespace ":" reference          (CAIP-2)
account_address := 1*128 ( ALPHA / DIGIT / "-" / "." / "%" )
```

Worked forms:

```text
eip155:1:0xab16a96d359ec26a11e2c2b3d8f8b8942d5bfcdb   Ethereum mainnet, EOA
cosmos:cosmoshub-4:cosmos1…                            a Cosmos account
bip122:000000000019d6689c085ae165831e93:1A1zP1…        a Bitcoin account
```

An implementation MUST NOT re-encode, case-fold, or otherwise rewrite the `chain_id` or `account_address` beyond the namespace rule of §4.2.

### 4.2 Canonicalization (the `eip155` namespace rule)

CAIP-10 states that it "does NOT require canonicalization" and explicitly leaves per-chain rules to namespace profiles, citing EIP-55 and HIP-15 as examples. This proposal defines such a namespace rule for `eip155`, because without it one Ethereum account can be addressed under several strings that differ only in case or in how the chain id is written:

1. `chain_id` MUST be the canonical CAIP-2 form `eip155:<decimal>`. An EIP-1193 hex chain id (`0x1`) or a bare decimal (`1`) MUST be normalized to `eip155:1` before the identifier is formed. The conversion MUST be done with arbitrary-precision integer semantics, so two distinct chain ids can never collapse onto one account id.
2. `account_address` MUST be lowercase `0x` followed by 40 hex characters. An EIP-55 mixed-case checksum MUST NOT be preserved inside the account id.
3. A request whose `keyid` carries a non-canonical `eip155` account id MUST be rejected as `invalid_did` (§12). Accepting both forms would let one account be addressed under two identifiers, which defeats the replay cache's keying (ANP-02 L144) and any per-identity policy.

Non-`eip155` namespaces are left **verbatim**: this proposal does not define their address formats, and a verifier MUST treat an address it does not understand as opaque.

### 4.3 It is not a resolvable DID

A CAIP-10 account id is an **opaque identifier**. No resolver is asked for a document at it, because there is nowhere to ask: there is no chain-addressable `did.json` convention. The DID document an ANP-02 verifier needs is **carried inline in the authenticated request** (§6.2), and its trustworthiness comes from the address-derivation check (§5), not from where it was fetched.

Implementations MUST NOT attempt a network resolution of a CAIP-10 account id, and MUST NOT construct one by analogy with `did:wba`'s `/.well-known/did.json` rule. A verifier that cannot accept the inline document MUST reject the request, not fetch.

## 5. Key binding without a document proof

### 5.1 The address-derivation check

For an account whose namespace defines an address-derivation function, the binding check is:

```text
derive(pk) == account_address
```

For `eip155`, `derive` is the standard Ethereum address derivation:

```text
derive(pk) = keccak256(uncompressed(pk)[1..])[12..32]      (last 20 bytes, lowercase hex)
```

where `uncompressed(pk)` is the SEC 1 uncompressed point encoding, `0x04 || x || y`.

A verifier MUST perform this check on the key it is about to use, **before** using that key to verify the request signature, and MUST reject the request if the derived address does not equal the `account_address` in the identifier (as `invalid_verification_method`, §12). This is the CAIP-10 analogue of the `did:wba` fingerprint check: it is what makes the inline document evidence rather than attacker-supplied bytes.

A namespace that defines no address-derivation function is out of scope for this version of the binding (§14 item 3).

### 5.2 The inline document is a trusted input

Because §5.1 makes the identifier commit to the key, an inline document is trustworthy in exactly the case ANP-03 treats a path-type `did:wba` inline copy as trustworthy: the key it names can be checked against the identifier. Two consequences:

1. A verifier MAY accept the inline document without any network fetch, and MUST reject it if the key it names does not satisfy §5.1.
2. A verifier MUST NOT require a `proof` on a CAIP-10 document. As with native `did:web` (B.1 L479), the document-proof rules of another method do not apply: a self-signed document whose identifier already commits to the key proves nothing that §5.1 has not already established. The document's `id`, its `controller`, and its verification-method `id` MUST all equal the account id (§6.1).

### 5.3 Limits: externally-owned accounts only

The check of §5.1 holds only for an account whose address is derived from the signing key — an **externally-owned account (EOA)** in EVM terms. It does **not** hold for:

- a smart-contract account (for example ERC-4337 / Safe), whose address is a contract address unrelated to any single key;
- an account whose signing key is a delegated session key with no address of its own;
- any namespace whose address is not a function of the key.

This proposal therefore covers externally-owned accounts only. A verifier MUST NOT relax §5.1 to accommodate a contract account, because doing so would reintroduce exactly the attack §5.1 exists to prevent: a caller signing with a key the identifier does not authorize. Supporting contract accounts is a separate design (§14 item 2).

## 6. Authentication integration

### 6.1 `keyid` and verification method

When a client authenticates with a CAIP-10 account id:

1. `keyid` MUST be the account id followed by a fragment naming the verification method, for example `eip155:1:0x…#key-1`. The account id MUST be the canonical form of §4.2.
2. The DID document carried inline MUST have `id` equal to the account id, and the verification method named by `keyid` MUST exist, MUST have `controller` equal to the account id, and MUST be authorized by the `authentication` relationship (ANP-02 §3.2.1 step 5, L135–L136).
3. The verification-method `type` is `Multikey`, with `publicKeyMultibase` the secp256k1 multibase encoding, following the same representation ANP-03 uses for `k1_` keys.
4. The request signature MUST be verified under ANP-02 §3.2.2; algorithm selection follows the verification-method type (ANP-02 §3.2.2 step 4, L161–L163), so a `Multikey` secp256k1 method is verified with ECDSA over secp256k1.

### 6.2 Carriage and the common flow

The inline document is carried in the authenticated request body, alongside the claimed `did`. The verifier MUST check that the `did` the body claims equals the account id extracted from `keyid` (ANP-02 §3.2.1 step 3, L130), before any key is used. From that point the common flow applies unchanged: `Content-Digest` (step 2, L128), signature coverage (step 6, L139), the freshness window (step 7, L141), replay protection (step 8, L143–L145), permission checks (step 9, L147), and the access token (§3.2.3).

Replay protection keys the cache on `(keyid, nonce)` (ANP-02 L144). Because §4.2 makes the account id canonical, two spellings of one account can never hold two cache entries.

### 6.3 What the verifier trusts

The only DID a verifier may trust for authorization is the account id derived from the signature's `keyid`, required to equal the body's claimed `did` and to be consistent with the key per §5.1. A verifier MUST NOT trust any identity material that the request supplies but the identifier does not authorize.

## 7. Resolution policy

Resolution is not a separate step for CAIP-10: §4.3 makes the identifier opaque and §5.2 trusts the inline document once §5.1 passes. A deployment that nevertheless wants an out-of-band copy (for example, a registry of accepted accounts) MAY hold one, but such a copy MUST NOT weaken §5.1: the key actually used is the one the account id commits to, whatever any stored copy says.

## 8. Continuity and updates

A CAIP-10 account id has **no update chain**. The address commits to the key, so a new key is a new address and therefore a new account; there is no `successorDid`, no `alsoKnownAs`, no migration assurance, and no recovery path. This is a property of the design, not an omission:

- A relying party that loses the account loses it permanently; re-keying produces a different identifier that it cannot automatically link to the old one.
- The ANP-03 continuity model (stable subject path, `successorDid`, migration assurance) does not apply to CAIP-10 and MUST NOT be synthesized for it.
- If a chain gains a key-rotation mechanism that preserves the address (for example, a contract account), it falls outside §5.3 and outside this proposal; its continuity story would be a distinct design.

## 9. Handle / WNS integration

Handle and WNS binding are defined by ANP-04 against a DID document. Because a CAIP-10 account id is not a DID and carries no host, the ANP-04 Handle binding is **not** defined for it in this proposal. A deployment that needs a human-readable handle for a blockchain account SHOULD use an existing Handle-to-DID mechanism and let the DID authenticate, rather than inventing a Handle-to-account binding. Whether a Handle may bind directly to a CAIP-10 account id is left open (§14 item 5).

## 10. Privacy considerations

This binding discloses the account id on the wire, precisely as ANP-02 discloses a `keyid`. Unlike the ZK-login profile, it **protects nothing**:

- A blockchain account id is a public identifier by construction. Any observer of the request learns it, and can correlate it across every service the account authenticates to, and against the public ledger.
- Address reuse is visible on-chain, and an account's transaction history is public.

The honest statement is therefore: this binding adds a convenient identity source, and it does **not** add privacy. An agent that needs cross-service unlinkability should use a profile that hides the identifier (for example `anp.auth.zklogin.v1`) rather than this binding. Implementations SHOULD document this explicitly to users, because a wallet-based login can feel private while being maximally correlatable.

## 11. Security considerations

- **Binding soundness.** Security rests on the address-derivation function being preimage- and collision-resistant: an attacker who could find a key whose derived address equals a victim's could impersonate the account. For `eip155` this is the standard security of Ethereum address derivation.
- **Inline-document substitution.** An attacker who inlines a document naming their own key while claiming a victim's account id is rejected by §5.1. An attacker who names the victim's key cannot sign with it. Both halves are required; §5.1 alone (without checking that the key actually verifies the signature) or the signature check alone (without §5.1) is insufficient.
- **Canonicalization confusion.** §4.2 exists because a non-canonical spelling is a distinct cache key and a distinct policy subject. A verifier that skips it can be made to hold two records for one account.
- **EOA assumption.** §5.3 is a security boundary, not a limitation of convenience: relaxing it lets a caller sign with an unauthorized key.
- **No revocation.** Because there is no update chain (§8), there is no in-protocol key revocation. A chain's own mechanisms (for example, moving funds) are the only recourse; a relying party's blocklist is local.
- **Replay and freshness.** Unchanged from ANP-02 §3.2.1 steps 7–8 (L141–L145).

## 12. Deviation from ANP-02

This binding makes **exactly one** deliberate departure from ANP-02:

**`keyid` is not a DID URL.** ANP-02 L82 requires `keyid` to be "a complete DID URL", for example `did:web:identity.example:alice#key-1`, and Appendix B restates it (L487). A CAIP-10 account id is not a DID URL. This binding therefore permits `keyid` to be `<account_id>#<fragment>`.

Two consequences a verifier MUST handle:

1. Any ANP-02 clause that parses a DID out of `keyid` does not apply as written; the verifier MUST recognize the CAIP-10 form and take the identity material from the inline document per §6.
2. The replay cache keys on `(keyid, nonce)` (ANP-02 L144). With a canonical account id (§4.2) this remains a sound key; no change to the cache mechanism is required.

No other ANP-02 requirement is waived. In particular the authentication-relationship check (§3.2.1 step 5, L136) still applies, and no document proof is required — the latter by the same discipline that Appendix B already applies to native `did:web` (L477, L479), not as a new departure.

**Error mapping.** The binding reuses the ANP-02 error vocabulary (L252–L260):

| Condition | Error value |
|---|---|
| Malformed `keyid` / body, missing document | `invalid_request` |
| Body `did` does not equal the account id from `keyid` | `invalid_did` |
| Non-canonical `eip155` account id (§4.2) | `invalid_did` |
| Verification method missing, or not in `authentication` | `invalid_verification_method` |
| Derived address does not equal the account id (§5.1) | `invalid_verification_method` |
| Signature fails verification | `invalid_signature` |
| `Content-Digest` mismatch | `invalid_content_digest` |
| Nonce reused or unknown | `invalid_nonce` |
| Timestamp outside the window | `invalid_timestamp` |

## 13. Relationship to existing documents

| Document | Relationship |
|---|---|
| [ANP-02](../02-anp-did-authentication-protocol-specification.md) | The carrier. §1 (L14) and §2 (L28) make the flow method-independent; §3 header set, `Content-Digest`, freshness, the 401 challenge, §4 carriage, the access token, and the error vocabulary are all reused. The single deviation is `keyid` (§12). |
| [ANP-03](../03-did-wba-method-design-specification.md) | Not used for CAIP-10, except as the model for the `Multikey` secp256k1 representation (§6.1). CAIP-10 has no relation to ANP-03's path syntax, document proof, or continuity chain. |
| [Appendix A](../appendix-a-did-wba-k1-compatibility-extension.md) | Independent. Appendix A is a `did:wba` extension that binds secp256k1 **inside a `did:wba` path**; this proposal binds secp256k1 **as a blockchain account**, with no `did:wba` identifier. They overlap only in using secp256k1. |
| [Appendix B](../appendix-b-compatibility-with-native-did-web.md) | A sibling method binding. §5.2 borrows B's discipline: another method's document-proof rules are not a prerequisite. |
| [ANP-04](../04-anp-did-wba-name-space-specification.md) | Not applied (§9). Handle binding is defined against a DID document and a host. |
| ANP Messaging P1–P9 | Not modified and not a dependency; authentication and messaging are separate concerns (ANP-02 §1). |
| [ANP-10](../application/10-anp-agent-payment-protocol-specification.md) | Not modified. A blockchain account identity is a natural companion for an on-chain payment flow, but the two are independent. |

**Positioning against `did:pkh` (informative).** `did:pkh` is a W3C DID method whose method-specific id embeds a CAIP-10 account id (`did:pkh:eip155:1:0x…`). A deployment could therefore express a blockchain account as a `did:pkh` DID and reach it through Appendix D. This proposal deliberately does not do that: it keeps the **bare** account id, because (a) CAIP-10 is the identifier the wallet ecosystem already produces, (b) it adds no `did:` layer whose method specification must then be pinned, and (c) it makes the CAIP-10 form available to deployments that do not adopt `did:pkh`. Whether the community prefers the bare form or a `did:pkh` wrapper is an open question (§14 item 1).

## 14. Open questions

1. **Bare CAIP-10 or `did:pkh`.** This proposal binds the bare account id (§4.3). Should ANP instead require a `did:pkh`-wrapped identifier, so that one syntax serves both this binding and Appendix D? The trade-off is one more layer and one more method spec to pin, against a single identifier syntax.
2. **Contract accounts.** §5.3 excludes smart-contract accounts. Should the ecosystem define a variant (for example ERC-1271 signature validation) so that contract wallets can authenticate, and if so, does it belong here or in a separate document?
3. **Non-EVM namespaces.** §5.1 needs an address-derivation function. Which namespaces have one that ANP can rely on, and how should a namespace declare it? Or should the binding standardize only `eip155` and treat all others as opaque?
4. **Placement.** Should this be a new appendix beside Appendix B, a widening of Appendix D, or a numbered document under `application/`? This proposal assumes a new appendix and reuses Appendix D's intent.
5. **Handle binding.** May an ANP-04 Handle bind directly to a CAIP-10 account id (§9), or must a handle always resolve to a DID?
6. **Canonicalization ownership.** §4.2 defines the `eip155` rule here. Should CAIP-10's namespace-profile mechanism be the normative home for it, with ANP merely referencing it?

## References

- [ANP-02] ANP DID Authentication Protocol, Version 1.2 — `./02-anp-did-authentication-protocol-specification.md`
- [CAIP-2] Blockchain ID Specification — https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-2.md
- [CAIP-10] Account ID Specification — https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-10.md
- [RFC 9421] HTTP Message Signatures
- [RFC 9530] Digest Fields
- [RFC 7638] JSON Web Key (JWK) Thumbprint
- [EIP-55] Mixed-case checksum address encoding
- [BCP 14] RFC 2119 / RFC 8174 — requirement keywords
- [DID Core v1.0] https://www.w3.org/TR/2022/REC-did-core-20220719/ (informative contrast: a CAIP-10 account id is not a DID)

## Copyright Notice

Copyright (c) 2024 ANP Open Source Community
This file is released under the [Apache License 2.0](../LICENSE). You are free to use and modify it, but you must retain this copyright notice.

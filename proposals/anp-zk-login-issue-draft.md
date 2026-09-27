# Issue Draft — ZK login Profile proposal

Paste into a GitHub Issue. Title field:

```text
[Feature]: 新增零知识登录 Profile anp.auth.zklogin.v1（提案 / proposal）
```

Body, following `.github/ISSUE_TEMPLATE/feature.yml`. Delete the bracketed guidance before posting.

---

### Feature Description | 功能描述

Propose a new opt-in ANP authentication Profile, `anp.auth.zklogin.v1`, in which an agent authenticates a request by zero-knowledge proof instead of by DID signature. The caller proves it holds a credential admitted to a membership set, and transmits no DID, no public key, and no stable cross-service identifier; the relying party learns only a per-service pseudonym.

提议新增一个可选启用的 ANP 认证 Profile：`anp.auth.zklogin.v1`。智能体以零知识证明而非 DID 签名完成请求认证——证明自己持有某个被成员集合接纳的凭证，线上不传输 DID、公钥或任何跨服务稳定标识；依赖方只得到一个按服务区分的假名。

The full proposal is in this PR: [proposals/anp-zk-login-proposal.md](../proposals/anp-zk-login-proposal.md) (English), [chinese/proposals/anp-zk-login-proposal.md](../chinese/proposals/anp-zk-login-proposal.md) (中文).

Summary of the design:

- **Additive only.** It reuses the existing ANP-02 carriers (the `Signature-Input` header set, `Content-Digest`, the 401 challenge, and the §4 JSON carriage). No released document in ANP-01 through ANP-10 is modified.
- **Opt-in and discoverable.** A relying party advertises the Profile by capability or by a 401 challenge. A relying party that does not implement it is unaffected, and a client that does not implement it keeps using DID-signature authentication.
- **Full request binding.** The proof's public inputs include `SHA-256` of the RFC 9421 signature base of the actual request, so a proof is valid for exactly one method, target URI, authority, body, and freshness window.
- **Backend-agnostic.** It carries a `(scheme, statementId, vkHash)` descriptor and an opaque proof, so any proof system the relying party has registered a verification key for is accepted.
- **One deliberate deviation.** `keyid` is not a DID URL in anonymous mode (ANP-02 L82 requires that it be one). This is the reason anonymous login cannot be an ordinary ANP-02 request.

### Motivation | 动机

ANP-02 authenticates a request by a DID-signed HTTP message, which necessarily discloses the caller's `keyid`, a complete DID URL (ANP-02 L82). Every relying party — and every observer of the request — therefore learns which Agent DID is calling, and can correlate that DID across services. ANP-02 §6 (L419) already recognizes this concern and suggests a multi-DID strategy; this Profile offers a complementary, proof-based route that needs no change to the existing protocol.

ANP-02 L110, L163, and Appendix D L503 already defer identity material and algorithms to the selected verification method, so a proof-based Profile sits within the design ANP-02 already anticipates.

ANP-02 以 DID 签名的 HTTP 报文认证请求，必然披露调用方的 `keyid`（一个完整 DID URL，ANP-02 L82），因此任何依赖方与观察者都能得知是哪个 Agent DID 在调用，并可跨服务关联。ANP-02 §6（L419）已注意到该问题并建议多 DID 策略；本 Profile 提供一条互补的、基于证明的路径，且无需改动既有协议。

### Expected Outcome | 期望结果

An agent can log in to an opting-in service without disclosing its DID or public key. The service can rate-limit, ban, and maintain a session keyed on the pseudonym, and can optionally request attribute disclosure or a separate addressable binding when it needs them.

Community feedback on the six open questions in §14 of the proposal, and a decision on where the document should live (authentication Profile beside ANP-02, a numbered document under `application/`, or an independent extension).

### Supporting Materials | 辅助材料

- Proposal (EN): `proposals/anp-zk-login-proposal.md`
- Proposal (中文): `chinese/proposals/anp-zk-login-proposal.md`
- Prior design notes in the SDK repository (not normative): `anp-sdk/docs/zk-login-design.md`, `anp-sdk/plans/plan-zk-integration-scenarios.md`

### Final Check | 最后检查

- [x] I believe the above description is detailed enough to allow developers to implement the feature.

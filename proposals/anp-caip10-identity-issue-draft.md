# Issue Draft — Blockchain Account Identity (CAIP-10) proposal

Paste into a GitHub Issue. Title field:

```text
[Feature]: 新增区块链账户身份（CAIP-10）方法绑定 anp.identity.caip10.v1（提案 / proposal）
```

Body, following `.github/ISSUE_TEMPLATE/feature.yml`. Delete the bracketed guidance before posting.

---

### Feature Description | 功能描述

Propose a new opt-in ANP method binding, `anp.identity.caip10.v1`, in which an agent authenticates under ANP-02 with a **blockchain account** as its identity, instead of a DID. The identity is a CAIP-10 account id (for example `eip155:1:0x…`); the signing key is bound to it by address derivation, so no document is fetched and no document proof is required.

提议新增一个可选启用的 ANP 方法绑定：`anp.identity.caip10.v1`。agent 以**区块链账户**而非 DID 作为身份在 ANP-02 下认证。身份是 CAIP-10 账户 id（例如 `eip155:1:0x…`）；签名密钥通过地址派生与它绑定，因此不抓取文档、也不要求文档证明。

The full proposal is in this PR: [proposals/anp-caip10-identity-proposal.md](../proposals/anp-caip10-identity-proposal.md) (English), [chinese/proposals/anp-caip10-identity-proposal.md](../chinese/proposals/anp-caip10-identity-proposal.md) (中文).

Summary of the design:

- **Additive only.** It reuses the ANP-02 carriers (`Signature-Input`, `Content-Digest`, the 401 challenge, the §4 JSON carriage, the access token, and the error vocabulary). No released document in ANP-01 through ANP-10 is modified.
- **A third kind of key binding.** `did:wba` binds a key by an RFC 7638 fingerprint in the identifier; native `did:web` binds nothing in the identifier and relies on the domain; CAIP-10 binds the key **by deriving the address from it** (`derive(pk) == account_address`). The inline document is therefore trustworthy for the same reason a path-type `did:wba` inline copy is — the identifier commits to the key.
- **Opt-in and unaffected.** A relying party that does not implement it is unaffected, and a client that does not implement it keeps using `did:wba` / `did:web`.
- **Chain-agnostic by intent, `eip155` in this version.** The binding is written for any CAIP-2 namespace with an address-derivation function; §5.3 restricts it to externally-owned accounts (contract accounts are out of scope).
- **One deliberate deviation.** `keyid` is not a DID URL (ANP-02 L82 requires that it be one). This is the reason a CAIP-10 login cannot be an ordinary ANP-02 request — the same shape of deviation the ZK-login proposal makes.

### Motivation | 动机

ANP-02 §1 (L14) already makes the authentication flow independent of the DID method, and §2 (L28) leaves document authenticity, identity binding, and lifecycle rules to each method. Appendix A adds a secp256k1 compatibility extension **inside `did:wba`**; Appendix B integrates native `did:web`. Neither serves the large part of the agent ecosystem whose identity is a blockchain account — the form a wallet produces and the form a chain actually knows.

CAIP-10 also fills a gap in the existing adaptation slot: Appendix D (L503) is written for "other W3C DID Core methods", and a CAIP-10 account id is not a DID Core identifier, so its letter excludes CAIP-10 even though its intent covers it.

ANP-02 §1（L14）已让认证流程与 DID 方法无关，§2（L28）把文档真实性、身份绑定与生命周期规则留给各方法。附录 A 在 **`did:wba` 内部**加了一个 secp256k1 兼容扩展；附录 B 集成原生 `did:web`。两者都没有服务到 agent 生态中相当一部分——其身份是区块链账户——的那一类，而区块链账户正是钱包产出、链本身认识的形态。

CAIP-10 还补上了既有适配槽的一处缺口：附录 D（L503）是为「其它 W3C DID Core 方法」写的，而 CAIP-10 账户 id 不是 DID Core 标识符，所以它的字面范围排除了 CAIP-10，尽管其意图覆盖了它。

### Expected Outcome | 期望结果

An agent holding a blockchain account can log in to an ANP relying party without first converting to a DID, and a relying party can accept it without a DID resolver and without a document proof. The binding states plainly that it adds **no privacy** (§10): a blockchain address is public and correlatable, so an agent that needs unlinkability should use `anp.auth.zklogin.v1` instead.

持有区块链账户的 agent 无需先转成 DID 即可登录 ANP 依赖方；依赖方无需 DID 解析器、也无需文档证明即可受理。本绑定明确声明它**不增添隐私**（§10）：区块链地址是公开且可关联的，因此需要不可链接性的 agent 应改用 `anp.auth.zklogin.v1`。

Community feedback on the six open questions in §14 of the proposal, and a decision on placement — a new appendix beside Appendix B, a widening of Appendix D, or a numbered document under `application/` — plus whether to bind the bare CAIP-10 account id (as proposed) or a `did:pkh` wrapper.

请社区就提案 §14 的六个开放问题给出反馈，并就落点作出决定——是附录 B 旁的新附录、附录 D 的扩宽，还是 `application/` 下的编号文档——以及是绑定裸 CAIP-10 账户 id（如本提案）还是 `did:pkh` 包装。

### Supporting Materials | 辅助材料

- Proposal (EN): `proposals/anp-caip10-identity-proposal.md`
- Proposal (中文): `chinese/proposals/anp-caip10-identity-proposal.md`
- Prior implementation in the SDK repository (not normative): `anp-sdk` — the CAIP-10 identity contract (`src/core/identity.ts`), the address-binding check (`src/core/crypto/serverAuth.ts`), and the EVM address derivation (`src/core/wallet/eip1193.ts`)

### Final Check | 最后检查

- [x] I believe the above description is detailed enough to allow developers to implement the feature.

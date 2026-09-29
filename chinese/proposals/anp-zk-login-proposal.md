# ANP 零知识认证 Profile（提案）

- 文档编号：提案（未分配 ANP 编号）
- 提议的 Profile 标识：`anp.auth.zklogin.v1`
- 标题：ANP 零知识认证 Profile
- 状态：提案 / 供社区讨论；非已发布规范
- 版本：0.2
- 语言：中文
- 英文镜像：[ANP Zero-Knowledge Authentication Profile (Proposal)](../../proposals/anp-zk-login-proposal.md)
- 议题草稿：[anp-zk-login-issue-draft.md](../../proposals/anp-zk-login-issue-draft.md)
- 适用范围：本提案适用于智能体与依赖方之间的请求认证，作为 [ANP-02](../../02-anp-did-authentication-protocol-specification.md) DID 签名流程之外的一种**可选**替代。
- 参考实现：connectx ANP SDK 的 `@connectx-sdk/core/zk` 模块与 `@connectx-sdk/zk-pairing` 后端。它不具备规范效力；下文的陈述参数、线上格式与错误映射就是它所实现的那一套。

> **这是提案，不是规范。** 它提议为 ANP 协议集新增一个可选 Profile，不修改任何已发布文档。按 [CONTRIBUTING](../../CONTRIBUTING.cn.md)，它应当先经 GitHub Issue 与 Discord 讨论，之后再有任何将其晋升为已发布文档的 PR。下文的小节编号、标识符与错误码映射均为提案内容，可能在评审中变更。

> **为什么用独立 Profile 而不是改 ANP-02。** ANP-02 §3.1.2 第 2 步（L110）与 §3.2.2 第 4 步（L163）已把算法与密钥表示交由所选验证方法类型决定，附录 D（L503）也已预见其他方法"通过各自的解析与验证规则提供身份与认证密钥材料"。本提案是在既有 Profile 之外**新增**一个 Profile，而不是扩展 ANP-02 本身。

## 1. 摘要

ANP-02 通过 DID 签名的 HTTP 报文认证请求，这必然披露调用方的 `keyid`——一个完整的 DID URL（ANP-02 L82）。因此任何依赖方、以及任何观察到该请求的一方，都能得知是哪个 Agent DID 在调用，并可把该 DID 跨服务关联起来。

本提案定义 `anp.auth.zklogin.v1`：一个**可选启用**的认证 Profile。调用方以零知识方式证明自己持有某个被某个成员集合接纳的凭证，而**不传输 DID、公钥或任何跨服务稳定的标识**。启用该 Profile 的依赖方只会得到一个**按服务区分的假名**。

本 Profile 刻意做成纯增量：

- 复用 ANP-02 的线上载体（`Signature-Input` 选项、`Content-Digest`、401 挑战、§4 的 JSON 承载），不另立传输方式。
- 未实现它的依赖方不受影响；未实现它的客户端继续使用 DID 签名认证。
- ANP-01 至 ANP-10 均无改动。

### 1.1 本提案不做的事

- 不隐藏"该请求由零知识证明完成认证"这一事实，也不隐藏网络层元数据（来源地址、时序、请求大小）。
- 不使智能体对以其**其他方式**（支付、Handle、消息投递）识别自己的服务不可关联。见 §10。
- 不定义成员集合如何治理。准入策略属于集合签发方。见 §8。

## 2. 约定

本文中的 MUST、MUST NOT、REQUIRED、SHALL、SHALL NOT、SHOULD、SHOULD NOT、RECOMMENDED、MAY、OPTIONAL 按 BCP 14 解释。

§6.4 关系式所用记号：

| 记号 | 含义 |
|---|---|
| `zkSk` / `zkPk` | ZK 曲线的证明私钥及其对应公钥；`zkPk = g^zkSk`（§4） |
| `H(tag, …)` | 陈述声明的哈希——Poseidon-BN254（circomlib 参数）——由固定域常量 `tag` 做域分隔（§6.1） |
| `salt` | 注册时由智能体选定的每凭证随机值（§8） |
| `attrs` | 注册时承诺的属性向量；固定条数、固定位宽（§6.1） |
| `leaf`、`root`、`index`、`path` | 成员集合的叶子、其 Merkle 根、叶子下标、Merkle 路径 |
| `V` | 受众值，由客户端从所连接的 origin 推导（§6.5） |
| `baseHash` | 对该请求的 RFC 9421 签名基做 `SHA-256`，取为域元素（§6.2） |
| `h` | 按服务区分的假名（§6.4 第 3 条） |
| `sk_s` / `pk_s` | 会话私钥及其公钥；访问令牌绑定 `pk_s`（§6.8） |

本 Profile 固定的字符串：

| 字符串 | 取值 | 出现位置 |
|---|---|---|
| 陈述标识 | `anp.zklogin.v1` | 信封 `statement`、报文体 `statement`、描述符 `sets[].statement` |
| 描述符 profile | `anp-zk/1` | 描述符的 `profile` 成员（§5） |
| `keyid` 命名空间 | `zk:anp-zk/1:` | `keyid` 参数（§12） |
| 签名标签 | `zk1`（登录）、`pop1`（持有证明） | RFC 9421 标签（§4、§6.8） |
| 证明系统名 | `groth16`、`plonk` | 信封 `scheme`、`alg="zk:<scheme>"` |
| 令牌类型 | `ANP-ZK-PoP` | `Authentication-Info` 的 `token_type`（§6.8） |

## 3. 设计目标与非目标

**目标。**（a）线上不出现 DID、公钥或稳定标识。（b）完整的请求绑定：一个证明只对某一方法、目标 URI、authority、报文体与新鲜度窗口有效。（c）对已发布的 ANP 文档零改动。（d）后端无关：依赖方只要登记了某个证明系统的验证密钥就接受该系统的证明（§7）。（e）假名按服务稳定，使依赖方能够限流、封禁并维持会话。

**非目标。** 隐藏网络元数据；对串通的依赖方之间做到全局不可关联；定义成员集合的治理；在依赖方需要按 DID 寻址调用方的场合取代基于 DID 的认证。

## 4. 两层身份

直接证明 DID 私钥本身的成员资格，会把非原生域运算引入电路——在 BN254 电路里做 Ed25519 与 secp256k1 的标量乘。因此本 Profile 把两种角色分开：

1. **DID 私钥**只在**注册**时使用一次以授权准入（§8）。它从不用于登录，也从不出现在线上。
2. **证明私钥 `zkSk`** 是 ZK 电路自身曲线上的密钥。所有登录证明都用 `zkSk`。

成员叶子只承诺 `zkPk` 与属性摘要（§6.4 第 2 条）。因此依赖方无法从它收到的任何内容中还原注册 DID，DID 的方法规则（ANP-03 / [附录 B](../../appendix-b-compatibility-with-native-did-web.md)）只约束注册环节。

实现 MUST 为注册与证明使用不同的密钥，且 MUST 独立生成。复用或派生会使两层身份互相关联。

## 5. 可选启用与发现

依赖方通过公布描述符来启用它。定义两种渠道，依赖方可只用其一或两者并用。

1. **描述符。** 依赖方在其 DID 文档中加入一条服务项：

```json
{ "id": "#anp-zk-auth", "type": "ANPZkAuthService",
  "serviceEndpoint": "https://api.example.com/.well-known/anp-zk.json" }
```

该端点返回：

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

`profile` MUST 为 `anp-zk/1`；客户端收到其他取值时 MUST 拒绝，而不得按本规范解释。域元素成员（`setRoot`）用十进制字符串，因为 254 位的值无法在 JSON number 中精确表示。`vkHash` 是客户端原样回传的不透明字符串。未知成员 MUST 被忽略；客户端读到的每个成员 MUST 被校验。`sets[].accepts` 列出该集合接受的 `(scheme, vkHash)` 对，且 MUST 至少列出一项。

2. **挑战。** 依赖方 MAY 返回 `401 Unauthorized` 并附加挑战，方式照 ANP-02 L238–L246：

```http
WWW-Authenticate: ANPZK realm="api.example.com", profile="anp.auth.zklogin.v1",
  nonce="xyz987", error="invalid_signature"
Accept-Signature: zk1=("@method" "@target-uri" "@authority" "content-digest")
```

**回退规则。**

- 支持本 Profile 的客户端收到点名 `anp.auth.zklogin.v1` 的挑战时，MUST 以证明请求作答，而不是 DID 签名请求。
- 不支持它的客户端 MUST 忽略该挑战并使用 ANP-02 认证；依赖方随后可按自身策略接受或拒绝该尝试（见下文的 `required`）。
- 把某操作声明为 **ZK 必需**的依赖方（例如仅为此操作声明本 Profile）MUST NOT 接受该操作的 DID 签名登录。客户端 MUST NOT 把此类拒绝当作改用 DID 签名重试的理由。
- 依赖方 MUST NOT 把 ZK 必需的操作静默降级为 ANP-02 认证。这与多版本消息 Profile 已采用的"不得静默降级"纪律一致。
- 未启用本 Profile 的依赖方对证明请求回 `invalid_verification_method`（§12），这就是客户端判断"该站点不提供 ZK 路径"的信号。

**一次登录用哪个集合、哪个证明系统、哪条谓词**，MUST 由客户端从描述符中选出，不得猜测：客户端 MUST 失败，而不得替换成依赖方未声明的集合、证明系统或谓词。

## 6. 证明陈述

### 6.1 陈述标识与参数

陈述标识为 `anp.zklogin.v1`。它命名一条冻结的关系式与一套冻结的编码。下表任一参数改变都会改变公开输入的字节或假名的字节，因此**换参数即换陈述标识**，也就是换假名空间、换集合：

| 参数 | 冻结值 |
|---|---|
| 域 / 群 | BN254（`Fr`）/ 同域上的 BabyJubjub，基点 `g` |
| 哈希 | Poseidon-BN254，circomlib 参数 |
| 域标签 | 固定域常量：假名 `p−1`、叶子 `p−2`、属性 `p−3`、origin `p−4`、setId `p−5`、nullifier `p−6`（预留）、节点 `p−7`、空 `p−8` |
| Merkle 深度 | 20（2^20 ≈ 105 万成员）；节点 = `H(nodeTag, left, right)`，空叶子取 `emptyTag` 哨兵而非零 |
| 属性 | 8 槽 × 32 位，槽序固定 |
| 谓词 | `any` / `eq` / `gte` / `lte`，编码为 `(predCode, predSlot, predValue)` |
| `V` 规范化 | `scheme://host[:port]`，转小写，省略默认端口，去掉 path / query / fragment（§6.5） |
| 版本 | `anp.zklogin.v1` |

两条规则：

- 证明方与验证方 MUST 就 `statementId` 精确一致；不一致 MUST 拒绝（§12）。
- 实现 MUST NOT 在保持同一 `statementId` 的前提下改动任一参数。原因是假名 `h` 由见证值派生并被依赖方当作长期句柄（§9）：若哈希族在标识不变的情况下改变，同一智能体在同一服务上会**静默变成另一个假名**。

一个陈述标识 MAY 由多个证明系统实现（§7）。因此一个成员集合 MAY 接受多个 `scheme`，但一个集合只能归入一个陈述标识。

**谓词。** 线上形式为 `any`、`eq:<attrName>:<decimal>`、`gte:…`、`lte:…`。一个区间是 `gte` 与 `lte` 的合取。谓词是**公开输入**：由证明方选，其参数对验证方可见。

### 6.2 公开输入

关系式的公开输入，按电路的固定信号序：

```text
baseHash, setField, setRoot, epoch, vField, pseudonym,
pkSx, pkSy, predCode, predSlot, predValue,
disclosed[0..7], mask
```

`setField` 与 `vField` 分别是集合标识与 `V` 的域编码：`setField = H(setIdTag, setId)`，`vField = H(originTag, V)`。`baseHash`、`setField`、`epoch`、`vField` 不消耗约束，但仍是公开输入——正因如此，证明被绑定到一个集合、一个 epoch 与一次请求上。

为便于后端与本 Profile 之间交换，公开输入另有一套规范 JSON 编码，按名携带同一批值（`baseHash`、`setId`、`setRoot`、`epoch`、`origin`、`pseudonym`、`sessionKey`、`predicate`、`disclosed`、`v`），域元素用十进制字符串。

验证方 MUST 从它收到的请求——重建的签名基与报文体——自行推出公开输入，且 MUST NOT 接受客户端给出的 `publicInputs`。证明信封 MUST NOT 携带它（§7.2）。

### 6.3 见证（永不传输）

`zkSk`、`zkPk`、`salt`、`attrs`、`index`，以及从 `leaf` 到 `root` 的 Merkle `path`。

`sk_s` 刻意**不在**见证中。关系式只证明 `pk_s` 是一个良构曲线点（§6.4 第 4 条）；对 `sk_s` 的持有由会话上的持有证明签名证明（§6.8）。

### 6.4 关系式

仅当存在满足下列全部条件的见证值时，证明才有效：

1. **密钥知识。** `zkPk = g^zkSk`。
2. **成员资格。** `leaf = H(leafTag, zkPk.x, zkPk.y, salt, H(attrsTag, attrs))`，且 `leaf` 是沿 `path` 位于 `root` 之下第 `index` 位的叶子。
3. **假名正确性。** `pseudonym = H(pseudonymTag, zkSk, vField)`。
4. **会话钥良构性。** `pk_s` 是曲线上的点。
5. **谓词。** 谓词 `(predCode, predSlot, predValue)` 对 `attrs` 成立。
6. **披露一致性。** 对每个槽，`disclosed[i] = attrs[i] · maskBit[i]`。`attrs[i]` 本身不受约束——验证方的义务见 §6.7。

第 1 条与第 3 条共用同一个秘密：假名与成员叶子绑定到同一个 `zkSk`，因此非成员无法冒领某个假名。

关系式不证明**否定式**断言（"我不在黑名单上"）；依赖方的替代做法是把集合做成只容纳许可人群，并把成员移出（§8）。

### 6.5 `V` 由客户端推导

客户端 MUST 从它实际连接的 origin——真实请求的 scheme 与 authority，按 §6.1 规范化——推导 `V`，且 MUST NOT 接受依赖方给出的 `V`。若由依赖方选择 `V`，两个串通的依赖方即可约定同一个 `V`，在两边观察到同一个 `h`，从而把调用方跨服务关联起来；从所连 origin 客户端侧推导 `V`，使假名依赖一个依赖方无法施加影响的值。这与 ANP-02 对 `@target-uri` 已采用的纪律相同：验证方应当"从真实请求推导"该值，而非仅凭调用方给出的目标 URI。

验证方 MUST 独立地为它收到的请求计算期望的 `V`，并 MUST 拒绝 `V` 不匹配的证明。`V` MUST 标识站点而非登录端点：若 `V` 依赖路径，端点一搬，全部假名都会平移。

### 6.6 验证方检查

除验证证明本身之外，验证方 MUST：

1. 从真实请求（方法、目标 URI、authority、`Content-Digest` 以及被覆盖的选项）重算 `baseHash`，不一致即拒绝。不得信任客户端给出的 `baseHash`。
2. 按 §6.5 重算 `V`，不一致即拒绝。
3. 要求报文体中的 `nonce` 与 `Signature-Input` 中的 `nonce` 均与挑战一致，并采用与 ANP-02 §3.2.1 第 7 步（L141）相同的新鲜度窗口，其中 `expires` 按 SHOULD 处理（ANP-02 L85）。
4. 维护重放缓存。匿名模式下，ANP-02 的 `(keyid, nonce)` 缓存（ANP-02 L144、L437）变为 `(h, nonce)`。一个 nonce 对每个 `h` MUST 只能使用一次。缓存只在证明验证通过后才消费，以免一次坏证明烧掉一个挑战。
5. 用 `keyid` 里的 `setId` 查出集合，并拒绝陈述标识与信封不一致的集合。
6. 只接受针对它当前公布的根或紧邻前一 epoch 的根的证明，拒绝它没有的根 / epoch 组合（§11）。
7. 校验 §6.7 的披露覆盖。
8. 对其假名策略作用于 `h`：限流、封禁、会话绑定（§9）。

### 6.7 选择性披露

`attrs` 是定宽槽的向量，`disclosed` 是证明对它所发布的内容：每个槽要么是取值，要么是零且掩码位为 0。公开输入 `mask` 记录实际披露了哪些槽。

由于 §6.4 第 6 条只约束 `disclosed[i] = attrs[i] · maskBit[i]`，关系式**不**证明证明方披露了依赖方所要的属性——证明方可以什么都不披露。需要属性的依赖方 MUST 校验掩码覆盖了它自己 `requiredDisclosures`（§5）里点名的每一条，并 MUST 拒绝未覆盖的登录。属性名与该站点的编码词表在描述符中公布；线上属性值是 32 位整数。

### 6.8 后续请求与令牌绑定

一个证明只认证一次请求。至于之后怎么办，ANP-02 已有安排：服务端通过 `Authentication-Info` 返回访问令牌（ANP-02 L197），其 §3.2.3 第 3 项（L205–L215）预留了一种 **sender-constrained** 令牌，把令牌绑在客户端持有的密钥上——同时明确声明该档尚未定义。本 Profile 填上这一预留格位。

```http
Authentication-Info: access_token="…", token_type="ANP-ZK-PoP", expires_in=3600,
                     cnf="<base64url(SHA-256(pk_s))>"
```

- 令牌主体 MUST 是 `h`，不是 DID。
- 令牌 MUST 携带一条指名为 `pk_s` 的确认声明（`cnf`），其值是对编码后的 `pk_s` 取 `SHA-256`。密钥本身 MUST NOT 被携带，因为令牌载荷对其持有者可读。
- 由于令牌类型不是 `Bearer`，按 ANP-02 L225，客户端 MUST 依本规范发送它：后续请求出示令牌并附上对 `sk_s` 的持有证明，且 MUST NOT 被要求每个请求都携带新的零知识证明。

```http
Authorization: ANP-ZK-PoP <access_token>
Signature-Input: pop1=("@method" "@target-uri" "content-digest");created=…;expires=…;nonce=…
Signature: pop1=:<base64url(R ‖ S)>:
```

`content-digest` 仅在请求有报文体时参与覆盖。会话签名是 **BabyJubjub 上的 EdDSA-Poseidon**，使用 `sk_s`：

```text
h = Poseidon(R.x, R.y, A.x, A.y, M)   over BN254, circomlib parameters
A = pk_s = sk_s·g                     g = the circomlib base point
S = (r + h·sk_s) mod SUBORDER,  R = r·g
verify: A in the prime-order subgroup ∧ A ≠ O ∧ S < SUBORDER ∧ S·g == R + h·A
```

子群检查与 `S < SUBORDER` 检查都是必需的：BabyJubjub 的余因子为 8，低阶 `A` 会让验证等式无需私钥即成立；而 `S + SUBORDER` 满足同一个等式，会让每个签名都有孪生编码，破坏按签名字节为键的重放缓存。`R` 无需子群检查：等式强制 `R = S·g − h·A`。

## 7. 证明系统兼容

本 Profile 不强制某一证明系统。它接受依赖方已为其登记验证密钥的任何系统，由三段式描述符刻画。

### 7.1 描述符三元组

```text
(scheme, statementId, vkHash)
```

- `scheme`——证明系统，例如 `groth16`、`plonk`、`halo2`、`stark`。
- `statementId`——§6.1，固定曲线与哈希参数。
- `vkHash`——`base64url(SHA-256(verificationKey))`，使依赖方能按哈希而非按 URL 钉住某证明系统的密钥材料。

三者 MUST 全部命中一条已登记条目，证明才会被接受。派发失败——`scheme` 未知、陈述未知、`vkHash` 未登记、集合未知——MUST 报 `invalid_verification_method`，无需新增错误值（§12）。

该三元组与 EIP-8288 所用的依赖框架 `(scheme, data_hash, verification_key_hash)` 形状相同；见 §13。

### 7.2 证明信封

信封就是 `Signature` 头的取值。其 JSON 形式为：

```json
{
  "v": 1,
  "scheme": "groth16",
  "statement": "anp.zklogin.v1",
  "vkHash": "<base64url>",
  "bytes": "<base64url>"
}
```

`bytes` 对本 Profile 不透明。其编码由 `scheme` 固定，且 MUST NOT 在同一 `statementId` 内变化。信封 MUST NOT 携带公开输入（§6.2）。

### 7.3 验证密钥引用

依赖方 MUST 公布客户端获取其所接受验证密钥的方式，并 MUST 按 `vkHash` 钉住它们（§5）。实现 MUST NOT 接受客户端提供的验证密钥。

### 7.4 可移植接口

合规的证明后端恰好暴露两个操作：由陈述与见证生成证明，以及以陈述验证证明。本 Profile 的可移植资产是**陈述**——§6.1 至 §6.4——而非后端。实现了该陈述的后端可以互换而不改变线上格式，因为线上只携带描述符三元组与不透明的证明。

### 7.5 聚合（信息性）

未来修订版可能允许一个聚合证明覆盖多个陈述（例如跨多个签发方的成员资格）。本提案不定义它，只在信封的可扩展性上为它留位。

## 8. 注册

注册**不属于线上协议范畴**，由成员集合的签发方治理。本节给出本 Profile 所假定的格式与依赖方所依赖的约束。

```text
agent ──▶ issuer:  一次 ANP-02 DID 签名请求（§8）
                 + { "setId": "…",
                     "zkPk":  "<base64url, 32-byte compressed BabyJubjub point>",
                     "salt":  "<base64url, 32 bytes>",
                     "attrsCommitment": "<decimal>" }

issuer ──▶ agent: 200 { "setId": "…", "leaf": "<decimal>", "setRoot": "<decimal>",
                        "epoch": 42,
                        "merklePath": { "index": 3, "siblings": ["<decimal>", … 20] } }
```

1. 注册 MUST 由一个已认证的身份授权——RECOMMENDED：一次 ANP-02 DID 签名请求，DID 私钥正是在此处使用其唯一一次（§4）。
2. 上述四个成员 MUST 被该请求的 `Content-Digest` 覆盖，使依赖方能够拒绝一份未把它正要准入的 `zkPk` 绑上去的注册。
3. 签发方以 `attrsCommitment = H(attrsTag, attrs)` 计算 `leaf = H(leafTag, zkPk.x, zkPk.y, salt, attrsCommitment)`，把 `leaf` 插入集合，并返回叶子、新根、epoch 与 Merkle 路径。线上只传属性承诺，属性向量留在智能体侧。
4. `salt` MUST 由智能体生成（或由签发方生成并经保密信道返回），MUST 不可预测，且 MUST NOT 在凭证之间复用。没有它，低熵的 `attrs` 可从 `leaf` 被猜出。
5. 智能体 MUST 在采信前校验回执：`setId` 须是它请求的那个，`leaf` 须是它自己的密钥、盐与属性所产出者，路径须为陈述所定深度且能重算出返回的 `root`。
6. 回执中的 `root` 与 `epoch` 会随集合变动而过时。客户端 MUST 在出证前重读描述符（§5），MUST NOT 用它注册时收到的值出证。
7. 准入策略由签发方执行。由于登录不携带 DID，若某签发方要求身份可解析，MUST 在注册时检查，且依赖方 MUST NOT 把注册环节的检查表述为登录环节的检查。
8. 签发身份即准入权威。签发方 MUST 防止同一凭证在同一 `setId` 下被准入两次，且 MUST NOT 以某个自述的主体字符串作为去重键：一个标识符的方法专有路径段未必与其密钥材料绑定，因此两个主体可以出示同一把密钥。可靠的去重键是密钥材料本身（的指纹）。去重是封禁得以生效的前提：不去重，封禁 `h` 只会逼出一次重新注册（§9.3）。
9. 除非申请人提交一份它自己无法开启的承诺，`attrs` 在注册时会以明文披露给签发方。因此，准入决策需要属性的集合，其签发方会知道那些属性；不过假名 `h` 仍不透露哪次登录是哪一次（§9.1）。

### 8.1 签发方为根的集合

成员集合的构造方式——依赖方自行维护的准入式集合，与由第三方准入成员、以签发方为根的集合——属于本 Profile 不予标准化的治理话题（§14 第 5 项）。两者共用 §6.4 第 2 条；签发方为根的集合另需在关系式内验证签发方的凭证。

## 9. 依赖方的账号模型与留痕

### 9.1 假名就是账号

`h` 是依赖方能看到的唯一长期标识符，而且是确定性的：它只取决于 `zkSk`、`V` 与陈述标识，别无其他（§6.4 第 3 条）。对"同一智能体访问同一服务"而言，三者都不变，因此每次登录都得到**同一个** `h`。假名**就是**账号：

| 情形 | `h` 是否相同 | 依赖方看到的是 |
|---|---|---|
| 同一智能体、同一服务、多次登录 | 相同 | 一个账号 |
| 同一智能体、不同服务 | 不同 | 两个不可关联的账号 |
| 同一智能体、同一服务、换了凭证 | 不同 | 另一个账号（§9.3 第 2 项） |

本 Profile 隐藏的是**跨服务**的关联性，而不是站内的一致性。站内一致性正是限流、封禁、信誉得以成立的前提；代价是依赖方能够对某位来访者做长期画像，§9.3 要求它披露这一点。

`h` MUST 被当作不透明句柄：对给定的（`zkSk`, `V`, 陈述标识）它是稳定的；它派生自从不离开智能体的秘密，因此不可逆推 `zkSk`；只要 `V` 按服务各不相同，它与该智能体的 DID、与其他服务上呈现的假名均不可关联（§6.5）。

### 9.2 依赖方存什么

匿名模式下不存在 `h → DID` 的映射，依赖方 MUST NOT 试图构造一条。它对某位来访者的记录只以 `h` 为键：

| 字段 | 用途 |
|---|---|
| `h`（主键） | 依赖方对该来访者的全部认知 |
| `pk_s` | 会话钥；访问令牌的确认声明绑的就是它（§6.8） |
| `disclosed` | 本次登录披露的属性子集 |
| `escrow` | 保险箱密文，`mode` 为 `escrow` 时（§10） |
| `mode` / `statementId` / `scheme` / `vkHash` / `epoch` | 本次证明的出处，审计与排障必需 |
| `created` / `expires` / `nonce` | 与 ANP-02 的新鲜度参数对齐 |
| 结论 | 通过或拒绝 |

同时担任成员集合签发方的依赖方，还需保留集合的 `setId`、`epoch` 与当前根，**以及历史根**——§6.6 的宽限窗口与日后的审计都需要它们。

### 9.3 依赖方必须接受的后果

以下是设计的性质而非缺陷，依赖方 MUST 在其描述符（§5）中披露：

1. **无法主动联系。** `h` 不是可路由地址。必须事后按 DID 联系该智能体的依赖方无法从 `h` 做到；这正是可选的可寻址模式存在的理由（§12）。
2. **没有恢复路径。** 智能体丢失 `zkSk`，或以不同凭证重新注册，得到的 `h` 即不同，依赖方看到的就是另一个账号。本 Profile 不定义任何恢复、合并或账号关联路径。
3. **封禁受制于准入。** 依赖方封的是 `h`；智能体以新凭证重新注册就得到新的 `h`。因此封禁只在注册渠道能够拒绝其重新准入的范围内有效（§8）。封禁 MUST NOT 被表述为身份层面的保证。
4. **DID 撤销不等于成员撤销。** 登录不携带 DID，也不解析 DID，因此注销一个 DID 不把该智能体移出集合；移出是集合自身的操作（§8、§11）。依赖方 MUST NOT 把两者表述为联动。
5. **风控手段减少。** 证明不携带设备指纹、账号关联或行为信号。依赖方的替代品只有速率、行为、可选披露的属性与托管路径，别无其他。传统的账号关联与异常登录检测基本不适用。
6. **不提供人格证明。** 一个运营方可注册许多 DID，且每一个都能解析。注册时的解析能拦住"未注册"，拦不住"一人多凭证"。

### 9.4 证明的留痕

证明是**一次性的**：被一个 `baseHash`、一个 nonce、一个 `h` 与一个新鲜度窗口绑在唯一一次请求上（§6.4、§6.6）。因此认证的推进方式与 ANP-02 相同：验证证明、签发令牌，后续请求用令牌而不再用新的证明。

所以留痕与否**不是为了将来再验**，而是为了第三方日后能否**复核那次准入决策**。部署 MUST 声明采用下列两种模型之一：

- **模型 A（RECOMMENDED 默认）。** 存公开输入、结论与时间，**丢弃证明**。日后的审计只能确认依赖方当时记录了一次通过的验证。
- **模型 B。** 存证明，或至少存 `hash(证明)`。由于证明是 NIZK、公开可验证，持有公开输入与证明的第三方无需信任依赖方即可重算该决策。

证明不是标识符，不提供任何归因：模型 B 的证据价值仅限于准入决策可复核。部署若提供归因，只能来自托管路径（§10）。

### 9.5 限流与封禁

限流按 `h` 施加。依赖方 MUST NOT 把 DID 或其他稳定标识作为准入的代价，也 MUST NOT 退回到某种共享标识以跨服务执行封禁：跨服务封禁是刻意做不到的，因为 `h` 按服务各不相同。

## 10. 隐私考量

**受保护的。** 调用方的 DID、公钥与注册属性不被披露。只要各服务从各自 origin 推导 `V` 且互不串通，跨服务关联即被阻止。

**不受保护的。** 网络元数据（来源地址、时序、请求大小、TLS SNI）不变。依赖方会得知该请求使用了 ZK 登录。共享 `V` 的两个依赖方——例如一方是另一方的前台，或客户端被胁迫复用 `V`——可以相互关联。实现 SHOULD 说明 `V` 是按 origin 推导的，且 MUST NOT 跨 origin 复用。匿名集大小即成员集合大小，小集合提供的隐私很有限；依赖方 SHOULD 公布其集合规模（§5）。

**审计档。** 部署方可能希望为滥用处置保留一条去匿名化路径。提案两种档位：**匿名档**，不存在此类路径；**托管档**，登录携带一份只有指定审计方能读的托管密文。档位由客户端选择并置于报文体中，由 `Content-Digest` 绑定，因此依赖方可以选择接受哪些档位，但不能强迫智能体交出托管密文。依赖方 MUST 披露它采用哪一档。推荐哪个作默认是待定问题（§14 第 2 项）。

这条去匿名化的轴与 §9.4 的留痕模型**相互独立**。实现 MUST NOT 把二者混为一谈。

## 11. 安全考量

- **可靠性（soundness）。** 安全性建立在证明系统的可靠性与陈述的约束之上（§6.4）。依赖方 MUST 拒绝其未登记 `scheme` 的证明（§12）。
- **可信设置。** 需要可信设置的证明系统（例如 Groth16）继承其假设。无法接受该假设的依赖方应改为登记透明设置的证明系统；本 Profile 不强制选择（§7）。
- **重放。** 由新鲜度窗口与 `(h, nonce)` 重放缓存处理（§6.6）。
- **成员集合新鲜度。** `root` 随集合变动而变。验证方 MUST 公布它正在校验的 epoch，并 MUST 定义一个宽限窗口：长度至少为最大证明有效期，在此窗口内针对紧邻的前一个 `root` 的证明仍被接受，以免一次注册或移除使在途请求失效。超过该下限的窗口长度属策略选择。
- **证明成本。** 证明生成刻意放在客户端。依赖方 MUST NOT 对非其发起的请求要求在自己的基础设施上生成证明。
- **拒绝服务。** 验证证明比验证签名更昂贵。依赖方 SHOULD 对任何未认证端点施加与其既有做法一致的请求级准入控制。
- **错误的可观测性。** 各类失败 MUST NOT 以泄露成员身份的方式相互区分："不是成员"、"已被移出集合"与"证明验不过" MUST 一律报 `invalid_signature`（§6.6 第 6 条）。

## 12. 对 ANP-02 的偏离

本 Profile 对 ANP-02 恰好做出一处刻意偏离，也正是匿名模式无法表达为普通 ANP-02 请求的原因：

**`keyid`。** ANP-02 L82 要求 `keyid` 是完整的 DID URL，附录 B L487 再次重申。匿名模式下没有 DID 可指。因此本 Profile 把 `keyid` 置为

```text
zk:anp-zk/1:<setId>:<h>
```

其中 `setId` 是成员集合，`h` 是假名，均为 `base64url` 编码。前缀 `zk:anp-zk/1:` 携带 Profile 版本，使日后的修订可被区分，而不至于看起来像一个畸形的 DID。`setId` MUST NOT 含 `:`，否则两个集合会产出同一个 `keyid`。

依赖方 MUST 处理三个后果：

1. ANP-02 以 `(keyid, nonce)` 为键的重放缓存（L144、L437）变为 `(h, nonce)`。
2. 任何"从 `keyid` 解析密钥"的 ANP-02 条款都不适用（L130、L159）；验证密钥按 §7.3 获取。
3. `alg` 携带证明系统名（`zk:groth16`、`zk:plonk`），ANP-02 视其为可选且非约束（L87）。验证方 MUST 据此对其已登记的证明系统做派发。

依赖方收到一个它不认识为 ZK `keyid` 的 `keyid` 时，MUST 按普通 ANP-02 请求处理并走那条路径。

**可寻址模式（可选）。** 必须事后按 DID 寻址该智能体的依赖方无法从 `h` 做到。此类部署 MAY 要求一份额外的、明确标注的证明，把凭证与一个崭新的、一次性的 DID 绑定，且该 DID 的密钥就是会话钥 `pk_s`——这样 `keyid` 也仍是真正的 DID URL，偏离归零。本提案不定义该证明，它被列为待定问题（§14 第 1 项）。依赖方 MUST NOT 静默要求它。

**错误映射。** 本 Profile 按下列方式复用 ANP-02 的错误词表（L252–L260）：

| 情形 | 错误值 |
|---|---|
| 信封畸形、报文体畸形、字段缺失、陈述未知、被覆盖的组件缺失 | `invalid_request` |
| nonce 被复用，或与挑战不符（头中或文体中） | `invalid_nonce` |
| 时间戳越窗 | `invalid_timestamp` |
| 证明未通过验证；根 / epoch 不被接受；不是成员 | `invalid_signature` |
| `scheme` 未知、`vkHash` 未登记、集合未知、集合与陈述不匹配 | `invalid_verification_method` |
| `Content-Digest` 不匹配 | `invalid_content_digest` |
| Access Token 绑定失败 | `invalid_access_token` |
| 无法执行验证（内部故障） | `unverifiable`（5xx，绝不作 401 挑战） |

`invalid_did`（L255）在匿名模式下无意义，不予使用。假名策略层面的拒绝没有专用值：可用的只有 `forbidden_did`，这是命名上的妥协；是否新增专用值属待定问题（§14 第 3 项）。

## 13. 与既有文档的关系

| 文档 | 关系 |
|---|---|
| [ANP-02](../../02-anp-did-authentication-protocol-specification.md) | 本 Profile 所依附的载体。§3 的头字段集合、`Content-Digest`、新鲜度、401 挑战、§4 的 JSON 承载、错误词表，以及预留的 sender-constrained 令牌格位（§6.8）全部复用。注册使用 ANP-02 的 DID 签名流程（§8）。 |
| [ANP-03](../../03-did-wba-method-design-specification.md) | 只治理注册身份。其 L205 说明"其他标准中定义但此处未列出的字段也可被选择性支持"，本 Profile 的新增内容正是在该条款下作为可选、非破坏性扩展。 |
| [ANP-06](../../06-anp-agent-communication-meta-protocol-specification.md) | 无改动。依赖方 MAY 通过它用于其他 Profile 的同一能力机制声明本 Profile。 |
| ANP 消息 P1–P9 | 无改动，也不是依赖。认证与消息是彼此独立的关注点，如 ANP-02 §1 所述。 |
| [ANP-10](../../application/10-anp-agent-payment-protocol-specification.md) | 无改动。其"支持选择性披露、最小化不必要的信息泄露"原则是本 Profile 的一个动机，但两者相互独立。 |

**与 EIP-8288 的同构（信息性）。** §7.1 的描述符三元组——`(scheme, statementId, vkHash)`——与 EIP-8288 的 `(scheme, data_hash, verification_key_hash)` 形状相同。二者从不同方向抵达同一结构：EIP-8288 通过指明验证依赖来刻画一个被签名物，本 Profile 通过指明证明的验证依赖来刻画它。若社区认为这一平行关系有用，未来修订版可考虑对齐字段名。

## 14. 待定问题

1. **能否进一步削弱这处偏离。** §12 去掉了 `keyid`。一种保留 DID 形状 `keyid` 的"可寻址模式"可把偏离降到零，代价是披露一个崭新铸造的 DID。对同时需要按 DID 寻址智能体的依赖方，哪个应作默认？
2. **审计档默认值。** 默认应是 `anonymous`（无去匿名化路径）还是 `escrow`（指定审计方可开启凭证）？§10 留待决定，因为它属治理与责任选择，而非密码学选择。
3. **错误词表。** 把 `forbidden_did` 复用于假名策略拒绝是否可接受，还是 ANP-02 的错误集应新增一个专用值？
4. **归属。** 它应作为 ANP-02 旁的认证 Profile（如本提案所设）、作为 `application/` 下的带编号文档、还是作为核心集之外的独立扩展？本提案假定为第一种。
5. **成员集合治理。** 本提案任何部分都不定义谁可签发凭证、集合如何联邦（§8.1）。若社区需要标准的集合接口，应另立文档。

## 参考资料

- [ANP-02] ANP 基于 DID 的身份认证协议，1.2 版 —— `../../02-anp-did-authentication-protocol-specification.md`
- [ANP-03] did:wba 方法规范，1.2 版 —— `../../03-did-wba-method-design-specification.md`
- [RFC 9421] HTTP Message Signatures
- [RFC 9530] Digest Fields
- [BCP 14] RFC 2119 / RFC 8174 —— 要求等级关键词
- [EIP-8288] Dependency Frame（信息性类比）

## 版权声明

Copyright (c) 2024 ANP Open Source Community
本文件在 [Apache License 2.0](../../LICENSE) 下发布。你可以自由使用和修改，但必须保留本版权声明。

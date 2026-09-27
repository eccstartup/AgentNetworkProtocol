# ANP 区块链账户身份（CAIP-10）集成（提案）

- 文档编号：提案（未分配 ANP 编号）
- 提议标识：`anp.identity.caip10.v1`
- 标题：区块链账户身份（CAIP-10）集成
- 状态：提案 / 供社区讨论；非已发布规范
- 版本：0.1
- 语言：中文
- 英文镜像：[ANP Blockchain Account Identity (CAIP-10) Integration (Proposal)](../../proposals/anp-caip10-identity-proposal.md)
- Issue 草稿：[anp-caip10-identity-issue-draft.md](../../proposals/anp-caip10-identity-issue-draft.md)
- 适用范围：本提案适用于「agent 的身份是一个区块链账户（CAIP-10）」时的请求认证，作为 [ANP-02](../../02-anp-did-authentication-protocol-specification.md) 现有 `did:wba` 与原生 `did:web` 绑定之外的一个**可选启用**的方法绑定。

> **这是提案，不是规范。** 它为本协议集提议一个新的方法绑定，不修改任何已发布文档。按 [CONTRIBUTING](../../CONTRIBUTING.cn.md)，它应先经 GitHub Issue 与 Discord 讨论，再进入任何将其晋升为已发布文档的 PR。下文的章节号、标识与错误映射都属提案内容，评审中可能变动。

> **为什么是新方法绑定，而不是修订 ANP-02。** ANP-02 §1（L14）写明「认证流程与 DID 方法无关」，§2（L28）写明「文档真实性、身份绑定与生命周期规则属于各 DID 方法」。附录 D（L503）已经预告其它方法「可以通过自己的解析与验证规则供给身份与认证密钥材料」。本提案就是这类绑定的一份，而非对 ANP-02 本身的扩展。它与附录 C、附录 D 有一处不同，§3.3 会讲清：CAIP-10 账户 id **不是** W3C DID Core 标识符，所以附录 D 的字面范围够不到它。

## 1. 摘要

ANP-02 以一条签名的 HTTP 报文认证请求，报文的 `keyid` 指明调用方的身份。今天实现方有两种方式供给该身份材料：`did:wba`（ANP-03 / ANP-02 附录 A）与原生 `did:web`（ANP-02 附录 B）。两者都是 DID 方法。但 agent 生态里有相当一部分已经持有一个**根本不是 DID** 的身份：一个**区块链账户**，其规范写法是 CAIP-10 账户 id，例如 `eip155:1:0x…`。

本提案定义 `anp.identity.caip10.v1`：一个**可选启用**的方法绑定，agent 用于 ANP 认证的身份就是一个 CAIP-10 账户 id，而签署请求的密钥与标识符之间的绑定，用的是一种现有绑定都没用过的检查——**地址由公钥派生**。账户的密钥不需要抓取；标识符本身就承诺了密钥。

该绑定刻意是增量的：

- 复用 ANP-02 的通用流程（§3 头部集合、`Content-Digest`、401 质询、§4 JSON 承载、访问令牌与错误词汇），不定义新的传输。
- 不实现它的依赖方不受影响；不实现它的客户端继续用 `did:wba` 或 `did:web`。
- ANP-01 到 ANP-10 均不改变。

### 1.1 本提案不做的事

- 它**不**让区块链账户变得私密。CAIP-10 账户 id 是一个公开、可全局关联的标识符；本绑定在线上披露它，与 ANP-02 披露 DID 完全一样。见 §10。
- 它**不**覆盖智能合约账户（例如 ERC-4337 钱包），这类账户的地址不是由签名密钥派生的。见 §5.3。
- 它**不**定义密钥更新或找回路径。见 §8。
- 它**不**定义链如何产生地址与签名；那属于各链自己的标准。

## 2. 约定

关键词 MUST、MUST NOT、REQUIRED、SHALL、SHALL NOT、SHOULD、SHOULD NOT、RECOMMENDED、MAY、OPTIONAL 按 BCP 14 解释。

下文使用的记法：

| 符号 | 含义 |
|---|---|
| `chain_id` | CAIP-2 区块链标识，`namespace:reference`（例如 `eip155:1`） |
| `account_address` | 该链上的账户地址，按 CAIP-10 的定义 |
| `account_id` | `chain_id ":" account_address`，即 CAIP-10 账户 id |
| `pk` / `sk` | 账户的签名公钥 / 私钥 |
| `derive(pk)` | 该链的地址派生函数（§5.1） |
| `keyid` | RFC 9421 的 `keyid` 参数（ANP-02 L82） |

## 3. 设计目标与非目标

**目标。**（a）让「身份是区块链账户」的 agent 无需先转成 DID 就能在 ANP-02 下认证。（b）用一次既不需抓取、也不需文档证明的检查，把标识符与签名密钥绑定。（c）对已发布 ANP 文档零修改。（d）保持链无关：绑定 MUST 对任何具有地址派生函数的 CAIP-2 命名空间成立，而不只是 EVM。（e）守住 ANP-02 的纪律：验证方 MUST NOT 接受标识符未授权的密钥。

**非目标。** 隐藏账户（区块链地址本质公开）；覆盖合约账户；定义链语义；在依赖方需要 DID 的场合取代 `did:wba` / `did:web`。

### 3.1 与现有绑定的关系

现有两种绑定以两种不同方式把标识符绑定到密钥：

| 绑定 | 靠什么把标识符绑定到密钥 |
|---|---|
| `did:wba` 路径型（ANP-03） | 最后一个路径段是**公钥的 RFC 7638 指纹** |
| 原生 `did:web`（附录 B） | 标识符里**什么都不含**；绑定靠域的 TLS 证书加上域所服务的文档（B.1 L477） |
| CAIP-10（本提案） | `account_address` 由**公钥派生**（`derive(pk)`） |

因此 CAIP-10 与 `did:wba` 路径型同族——标识符承诺了密钥——内联文档可信的理由也相同（§5.2）。它与 `did:web` 不同，后者标识符什么都不承诺，密钥必须经网络解析。

### 3.2 为什么它不能是一条普通 ANP-02 请求

ANP-02 L82 要求 `keyid` 是「完整的 DID URL」。CAIP-10 账户 id 不是 DID URL：它没有 `did:` scheme，而 CAIP-10 自己的语法恰恰禁止地址里出现 `:`，正是为了不让标识符被误当成 DID URL 的方法特定 id。因此本提案对 ANP-02 做且仅做**一处**刻意偏离（§12），其形状与 ZK 登录提案所做的那处相同。这就是 CAIP-10 登录无法表达为一条普通 ANP-02 请求的原因。

### 3.3 为什么附录 D 并没有已经覆盖它

附录 D（L503）是为「其它 W3C DID Core 方法」写的。CAIP-10 账户 id 不是 DID Core 标识符：它不是 `did:` scheme、没有方法解析器可以索要的 DID 文档、也没有 `did:` 方法规范。附录 D 打开的那个适配槽形状是对的，但它声明的范围把 CAIP-10 排除在外。本提案请社区二选一：放宽该范围，或把本绑定作为一份独立附录接收；§14 第 4 条保留了这个选择。

## 4. 标识符

### 4.1 语法

身份是一个 CAIP-10 账户 id，完全按 CAIP-10 的定义：

```text
account_id      := chain_id ":" account_address
chain_id        := namespace ":" reference          (CAIP-2)
account_address := 1*128 ( ALPHA / DIGIT / "-" / "." / "%" )
```

示例形态：

```text
eip155:1:0xab16a96d359ec26a11e2c2b3d8f8b8942d5bfcdb   以太坊主网，EOA
cosmos:cosmoshub-4:cosmos1…                            一个 Cosmos 账户
bip122:000000000019d6689c085ae165831e93:1A1zP1…        一个比特币账户
```

实现 MUST NOT 在 §4.2 的命名空间规则之外重编码、改大小写或以其它方式改写 `chain_id` 或 `account_address`。

### 4.2 规范化（`eip155` 命名空间规则）

CAIP-10 声明它「不要求规范化」，并明确把按链规则留给命名空间 profile，举 EIP-55 与 HIP-15 为例。本提案就为 `eip155` 定义这样一条命名空间规则，因为若不定义，同一个以太坊账户可以在若干仅大小写不同、或链 id 写法不同的字符串下被寻址：

1. `chain_id` MUST 取规范 CAIP-2 形式 `eip155:<十进制>`。EIP-1193 的十六进制链 id（`0x1`）或裸十进制（`1`）MUST 在构成标识符前规范化为 `eip155:1`。转换 MUST 用任意精度整数语义，使两个不同的链 id 永不塌成同一个账户 id。
2. `account_address` MUST 是小写 `0x` 加 40 个十六进制字符。EIP-55 的混合大小写校验和 MUST NOT 保留在账户 id 内。
3. `keyid` 携带非规范 `eip155` 账户 id 的请求 MUST 以 `invalid_did` 拒绝（§12）。同时接受两种写法会让同一个账户在两个标识符下可寻址，从而击穿重放缓存的键（ANP-02 L144）与任何按身份的策略。

非 `eip155` 命名空间**原样**保留：本提案不定义它们的地址格式，验证方 MUST 把它不理解的地址视为不透明。

### 4.3 它不是可解析的 DID

CAIP-10 账户 id 是一个**不透明标识符**。不会有人向它索要文档，因为无处可问：没有任何「按链地址寻址 `did.json`」的约定。ANP-02 验证方所需的 DID 文档是**内联在已认证请求里的**（§6.2），其可信性来自地址派生检查（§5），而不是来自它从哪里抓来。

实现 MUST NOT 对 CAIP-10 账户 id 尝试网络解析，也 MUST NOT 比照 `did:wba` 的 `/.well-known/did.json` 规则去构造一个。无法接受内联文档的验证方 MUST 拒绝请求，而不是去抓取。

## 5. 无文档证明的密钥绑定

### 5.1 地址派生检查

对于命名空间定义了地址派生函数的账户，绑定检查是：

```text
derive(pk) == account_address
```

对 `eip155`，`derive` 就是标准的以太坊地址派生：

```text
derive(pk) = keccak256(uncompressed(pk)[1..])[12..32]      (末 20 字节，小写十六进制)
```

其中 `uncompressed(pk)` 是 SEC 1 非压缩点编码，`0x04 || x || y`。

验证方 MUST 对**即将使用**的那把密钥做此检查，且 MUST 在用它验签请求之前做；派生地址不等于标识符里的 `account_address` 时 MUST 拒绝请求（记为 `invalid_verification_method`，§12）。这就是 CAIP-10 版的 `did:wba` 指纹检查：它使内联文档成为证据，而不是攻击者提供的字节。

不定义地址派生函数的命名空间不在本版绑定的范围内（§14 第 3 条）。

### 5.2 内联文档是可信输入

因为 §5.1 使标识符承诺了密钥，内联文档可信的场合与 ANP-03 视路径型 `did:wba` 内联副本为可信的场合完全相同：它命名的密钥可以对标识符自核。两个后果：

1. 验证方 MAY 不经任何网络抓取就接受内联文档；若它命名的密钥不满足 §5.1，MUST 拒绝。
2. 验证方 MUST NOT 要求 CAIP-10 文档带 `proof`。与原生 `did:web`（B.1 L479）一样，别的方法的文档证明规则不适用：一份自签文档，其标识符已承诺了密钥，除了 §5.1 已经确立的事实外什么都证明不了。文档的 `id`、其 `controller`、以及验证方法的 `id` 都必须等于账户 id（§6.1）。

### 5.3 限制：仅限外部持有账户

§5.1 的检查只对地址由签名密钥派生的账户——EVM 语境下的**外部持有账户（EOA）**——成立。它**不**对以下成立：

- 智能合约账户（例如 ERC-4337 / Safe），其地址是合约地址，与任何单一密钥无关；
- 由没有自身地址的委托会话密钥签名的账户；
- 任何地址不是密钥之函数的命名空间。

因此本提案只覆盖外部持有账户。验证方 MUST NOT 为迁就合约账户而放松 §5.1，因为那样做恰恰会重新引入 §5.1 存在的目的所要防的攻击：调用方用标识符未授权的密钥签名。支持合约账户是另一项设计（§14 第 2 条）。

## 6. 认证集成

### 6.1 `keyid` 与验证方法

当客户端以 CAIP-10 账户 id 认证时：

1. `keyid` MUST 是账户 id 加一个指明验证方法的 fragment，例如 `eip155:1:0x…#key-1`。账户 id MUST 取 §4.2 的规范形式。
2. 内联携带的 DID 文档其 `id` MUST 等于账户 id，且 `keyid` 指名的验证方法 MUST 存在、其 `controller` MUST 等于账户 id、且 MUST 获文档 `authentication` 关系授权（ANP-02 §3.2.1 第 5 步，L135–L136）。
3. 验证方法的 `type` 是 `Multikey`，`publicKeyMultibase` 取 secp256k1 的 multibase 编码，与 ANP-03 为 `k1_` 密钥采用的表示一致。
4. 请求签名 MUST 按 ANP-02 §3.2.2 验证；算法选择依验证方法类型（ANP-02 §3.2.2 第 4 步，L161–L163），故 `Multikey` secp256k1 方法以 secp256k1 上的 ECDSA 验证。

### 6.2 承载与通用流程

内联文档随已认证请求体携带，与声明的 `did` 并列。验证方 MUST 在任何密钥被使用之前，检查请求体声明的 `did` 等于从 `keyid` 提取的账户 id（ANP-02 §3.2.1 第 3 步，L130）。此后通用流程原样适用：`Content-Digest`（第 2 步，L128）、签名覆盖（第 6 步，L139）、新鲜度窗口（第 7 步，L141）、重放保护（第 8 步，L143–L145）、权限检查（第 9 步，L147）与访问令牌（§3.2.3）。

重放保护以 `(keyid, nonce)` 为缓存键（ANP-02 L144）。由于 §4.2 使账户 id 规范化，同一账户的两种写法永远不可能占两条缓存项。

### 6.3 验证方信什么

验证方唯一可信的 DID，是从签名 `keyid` 派生的账户 id；它必须等于请求体声明的 `did`，并须按 §5.1 与密钥一致。验证方 MUST NOT 相信任何请求提供、但标识符未授权的身份材料。

## 7. 解析策略

对 CAIP-10，解析不是一个单独的步骤：§4.3 使标识符不透明，§5.2 在 §5.1 通过后即信任内联文档。若某部署仍想持有带外副本（例如一份已接受账户的登记表），MAY 持有，但该副本 MUST NOT 弱化 §5.1：实际使用的密钥就是账户 id 所承诺的那把，无论任何存储副本怎么说。

## 8. 连续性与更新

CAIP-10 账户 id **没有更新链**。地址承诺了密钥，所以新密钥就是新地址、也就是新账户；没有 `successorDid`、没有 `alsoKnownAs`、没有迁移 assurance、也没有找回路径。这是设计的性质，不是遗漏：

- 丢失账户的依赖方将永久失去它；换钥产生一个它无法自动关联到旧标识符的不同标识符。
- ANP-03 的连续性模型（稳定主体路径、`successorDid`、迁移 assurance）不适用于 CAIP-10，MUST NOT 为它合成一套。
- 若某条链获得了保持地址不变的换钥机制（例如合约账户），那落在 §5.3 与本提案之外；它的连续性故事将是一项独立设计。

## 9. Handle / WNS 集成

Handle 与 WNS 绑定由 ANP-04 针对 DID 文档定义。由于 CAIP-10 账户 id 不是 DID、也不携带主机，本提案**不**为它定义 ANP-04 的 Handle 绑定。需要为区块链账户提供人类可读 Handle 的部署，SHOULD 使用既有的 Handle 到 DID 机制并让 DID 去认证，而不是发明一种 Handle 到账户的绑定。Handle 是否可以直接绑到 CAIP-10 账户 id，留作开放问题（§14 第 5 条）。

## 10. 隐私考量

本绑定在线上披露账户 id，与 ANP-02 披露 `keyid` 完全一样。与 ZK 登录 Profile 不同，它**什么都不保护**：

- 区块链账户 id 依构造就是公开标识符。任何观察请求的人都会得知它，并可把它跨该账户认证过的每一项服务、以及对照公开账本进行关联。
- 地址复用可在链上看见，账户的交易历史是公开的。

因此诚实的表述是：本绑定增添了一个便利的身份来源，它**不**增添隐私。需要跨服务不可链接性的 agent，应使用隐藏标识符的 profile（例如 `anp.auth.zklogin.v1`），而不是本绑定。实现 SHOULD 明确向用户说明这一点，因为钱包登录可能感觉私密，实际却是最大程度可关联的。

## 11. 安全考量

- **绑定可靠性。** 安全性建立在地址派生函数抗原像、抗碰撞之上：若攻击者能找到一把密钥，其派生地址等于受害者的地址，就能冒充该账户。对 `eip155`，这就是以太坊地址派生的标准安全性。
- **内联文档替换。** 攻击者一边声称受害者账户 id、一边内联自己密钥的文档，会被 §5.1 拒绝。攻击者若内联受害者的密钥，则无法用它签名。两半都必需；只有 §5.1（不检查密钥确实验证了签名）或只有签名检查（不做 §5.1）都不充分。
- **规范化混淆。** §4.2 之所以存在，是因为非规范写法是另一条缓存键、另一个策略主体。跳过它的验证方可被诱导为同一账户保存两条记录。
- **EOA 假设。** §5.3 是安全边界，不是便利性限制：放松它会让调用方用未授权的密钥签名。
- **无撤销。** 因为没有更新链（§8），协议内没有密钥撤销。链自身的机制（例如转移资产）是唯一救济；依赖方的黑名单是本地的。
- **重放与新鲜度。** 与 ANP-02 §3.2.1 第 7–8 步（L141–L145）不变。

## 12. 对 ANP-02 的偏离

本绑定对 ANP-02 做且仅做**一处**刻意偏离：

**`keyid` 不是 DID URL。** ANP-02 L82 要求 `keyid` 是「完整的 DID URL」，例如 `did:web:identity.example:alice#key-1`，附录 B 亦重申（L487）。CAIP-10 账户 id 不是 DID URL。因此本绑定允许 `keyid` 为 `<account_id>#<fragment>`。

两个后果验证方 MUST 处理：

1. 任何「从 `keyid` 解析出 DID」的 ANP-02 条款不再照字面适用；验证方 MUST 识别 CAIP-10 形式，并按 §6 从内联文档取身份材料。
2. 重放缓存以 `(keyid, nonce)` 为键（ANP-02 L144）。配合 §4.2 的规范账户 id，这仍是一个可靠的键；无需改动缓存机制。

其余 ANP-02 要求一概不免除。特别是认证关系检查（§3.2.1 第 5 步，L136）仍然适用；不要求文档证明则出于附录 B 已对原生 `did:web` 适用的同一条纪律（L477、L479），不是新增偏离。

**错误映射。** 本绑定复用 ANP-02 的错误词汇（L252–L260）：

| 情形 | 错误值 |
|---|---|
| `keyid` / 请求体畸形、缺文档 | `invalid_request` |
| 请求体 `did` 不等于从 `keyid` 提取的账户 id | `invalid_did` |
| 非规范 `eip155` 账户 id（§4.2） | `invalid_did` |
| 验证方法缺失，或不在 `authentication` 中 | `invalid_verification_method` |
| 派生地址不等于账户 id（§5.1） | `invalid_verification_method` |
| 签名验证失败 | `invalid_signature` |
| `Content-Digest` 不匹配 | `invalid_content_digest` |
| nonce 被重放或未知 | `invalid_nonce` |
| 时间戳超出窗口 | `invalid_timestamp` |

## 13. 与既有文档的关系

| 文档 | 关系 |
|---|---|
| [ANP-02](../../02-anp-did-authentication-protocol-specification.md) | 承载方。§1（L14）与 §2（L28）使流程与方法无关；§3 头部集合、`Content-Digest`、新鲜度、401 质询、§4 承载、访问令牌与错误词汇全部复用。唯一偏离是 `keyid`（§12）。 |
| [ANP-03](../../03-did-wba-method-design-specification.md) | CAIP-10 不使用它，仅把它作为 `Multikey` secp256k1 表示的样板（§6.1）。CAIP-10 与 ANP-03 的路径语法、文档证明、连续性链均无关。 |
| [Appendix A](../../appendix-a-did-wba-k1-compatibility-extension.md) | 独立。附录 A 是把 secp256k1 绑在 **`did:wba` 路径内部**的 `did:wba` 扩展；本提案是把 secp256k1 **当作区块链账户**绑定，没有 `did:wba` 标识符。二者仅在都用 secp256k1 上重叠。 |
| [Appendix B](../../appendix-b-compatibility-with-native-did-web.md) | 兄弟方法绑定。§5.2 借用 B 的纪律：别的方法的文档证明规则不是前提。 |
| [ANP-04](../../04-anp-did-wba-name-space-specification.md) | 不适用（§9）。Handle 绑定是针对 DID 文档与主机定义的。 |
| ANP 消息 P1–P9 | 不修改、无依赖；认证与消息是两件事（ANP-02 §1）。 |
| [ANP-10](../../application/10-anp-agent-payment-protocol-specification.md) | 不修改。区块链账户身份是链上支付流程的天然伙伴，但两者相互独立。 |

**与 `did:pkh` 的定位（信息性）。** `did:pkh` 是一个 W3C DID 方法，其方法特定 id 内嵌 CAIP-10 账户 id（`did:pkh:eip155:1:0x…`）。因此部署可以经由附录 D 把区块链账户表达成 `did:pkh` DID。本提案刻意不这样做：它保留**裸**账户 id，因为（a）CAIP-10 是钱包生态已经在产出的标识符，（b）它不引入一层此后必须钉死的 `did:` 方法规范，（c）它让 CAIP-10 形式对不采用 `did:pkh` 的部署也可用。社区更愿取裸形式还是 `did:pkh` 包装，是开放问题（§14 第 1 条）。

## 14. 开放问题

1. **裸 CAIP-10 还是 `did:pkh`。** 本提案绑定裸账户 id（§4.3）。ANP 是否应改为要求 `did:pkh` 包装的标识符，使一套语法同时服务本绑定与附录 D？取舍是多一层、多一份需钉死的方法规范，换来单一标识符语法。
2. **合约账户。** §5.3 排除智能合约账户。生态是否应定义一种变体（例如 ERC-1271 签名验证）使合约钱包也能认证？若应，它属于本文件还是独立文档？
3. **非 EVM 命名空间。** §5.1 需要地址派生函数。哪些命名空间有 ANP 可依赖的派生函数？命名空间应如何声明它？还是本绑定只标准化 `eip155`、其余都视为不透明？
4. **落点。** 它应是附录 B 旁的新附录、附录 D 的扩宽，还是 `application/` 下的编号文档？本提案假定为新附录，并沿用附录 D 的意图。
5. **Handle 绑定。** ANP-04 Handle 是否可直接绑到 CAIP-10 账户 id（§9），还是 Handle 必须永远解析到 DID？
6. **规范化归属。** §4.2 在此定义 `eip155` 规则。CAIP-10 的命名空间 profile 机制是否才应是它的规范归属，而 ANP 只是引用它？

## 参考

- [ANP-02] ANP DID Authentication Protocol, Version 1.2 — `../../02-anp-did-authentication-protocol-specification.md`
- [CAIP-2] Blockchain ID Specification — https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-2.md
- [CAIP-10] Account ID Specification — https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-10.md
- [RFC 9421] HTTP Message Signatures
- [RFC 9530] Digest Fields
- [RFC 7638] JSON Web Key (JWK) Thumbprint
- [EIP-55] Mixed-case checksum address encoding
- [BCP 14] RFC 2119 / RFC 8174 — 要求关键词
- [DID Core v1.0] https://www.w3.org/TR/2022/REC-did-core-20220719/（信息性对照：CAIP-10 账户 id 不是 DID）

## 版权声明

Copyright (c) 2024 ANP Open Source Community
本文件以 [Apache License 2.0](../../LICENSE) 发布。你可自由使用与修改，但须保留本版权声明。

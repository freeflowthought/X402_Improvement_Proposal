# TrustChain：面向去中心化服务经济的信任即服务 (TaaS) 协议（应用层版本）

**状态:** 草案 / 提案  
**版本:** v0.2  
**日期:** 2025年12月  
**作者:** 0xwilsonwu（联合讨论稿）  
**部署目标:** Base（EVM 兼容）

## 摘要

区块链擅长价值转移，但在“服务交换”层面仍存在关键缺口：链上只能证明付款已发生，却无法保证服务已交付。现有方案往往依赖声誉或中心化仲裁，无法在高价值、一次性或跨地域的服务交易中提供可靠保障。

TrustChain 提出一种不修改底层共识的 **应用层信任协议**，通过标准化的交易类型、链下收据与链上仲裁，建立“可验证的服务结算”。本协议以 Base 为结算层，使用 EIP-712 类型化数据、可插拔仲裁模块与经济惩罚机制，让服务交付与资金结算在不可信环境下实现可扩展的可信协作。

核心创新包括：
- **可识别的服务交易类型**：服务类交易通过 EntryPoint 进入协议层，与普通链上转账清晰区分。
- **标准化收据模型**：服务调用通过 `ServiceRequest` 和 `UsageReceipt` 表示，可批量结算与可争议。
- **可插拔仲裁框架**：委员会、法院、TEE、ZK、预言机均可作为仲裁模块，按策略接入。
- **经济对齐机制**：保证金、挑战窗口与惩罚规则为诚实行为提供激励。
- **仅预付模式（最安全）**：所有服务调用在执行前完成锁仓或余额预存，批量结算只能消费已锁定资金，从而消除“付款跑路”风险。
- **乐观结算**：在足额质押的前提下允许服务方提前回款，通过挑战与罚没保护安全性。

本白皮书完整描述 TrustChain 的协议架构、交易与收据格式、仲裁流程、激励模型与安全边界，并提供面向 API 计费与 SaaS 替代的落地路径。

---

## 1. 背景与问题

### 1.1 价值转移 ≠ 服务交换
在 Alice 向 Bob 转账的交易中，区块链只保证“钱已转”。但在 API 调用、内容交付、AI 推理等场景中，买卖双方彼此不信任，结算需要验证“服务确实交付”。链上缺乏对此的通用保证。

### 1.2 声誉与中心化仲裁的局限
- **声誉不可执行**：一次性交易中，高声誉无法防止“退出骗局”。
- **中心化仲裁不透明**：规则不清晰、成本高、跨地域执行困难。
- **链上合约无法直接验证链下事件**：需要可插拔的验证与仲裁机制。

### 1.3 设计目标
TrustChain 的目标是：
- **不修改底层共识**（不做 L1/L2 改造）
- **可识别服务交易类型**（与普通交易区分）
- **可验证与可仲裁**（证据驱动、可处罚）
- **高可扩展与可插拔**（支持多种证明与业务）
- **面向 API 计费与 SaaS 替代**（链上结算、链下执行）

---

## 2. 系统总览

### 2.1 关键角色
- **Payer（付款方）**：发起服务请求，承担支付责任。
- **Provider（服务方）**：执行服务并提交收据。
- **Relayer（中继）**：可选，代为提交交易。
- **Arbitrator（仲裁者）**：执行裁决逻辑（委员会/法院/TEE/ZK）。
- **Watcher（观察者）**：监督、挑战或提供证据。

### 2.2 关键合约模块
- **EntryPoint**：统一入口，识别服务类交易类型。
- **Registry**：模块注册、版本与策略管理。
- **Settlement**：托管与结算逻辑。
- **Arbitration**：争议管理与裁决执行。
- **Stake & Slash**：保证金与惩罚机制。

### 2.3 ZIP 可插拔模块
模块可被打包为 ZIP，并通过 Registry 注册：
- `manifest.json`：模块元信息与能力声明
- `contracts/`：链上验证/结算适配器
- `offchain/`：执行器/聚合器/证明生成
- `spec/`：Receipt 结构与证据要求

### 2.4 信用执行环境（CEE）与减法理论
TrustChain 不是新的 L1/L2，而是在 Base 之上构建的 **信用执行环境 (Credit Execution Environment, CEE)**。CEE 只处理符合 TaaS 协议的交易，并为其赋予“信用状态”（可仲裁、可罚没、可结算）。这是一种共识的“减法实践”：把与共识无关的服务验证逻辑从 L1 剥离，仅使用 Base 作为最终结算与安全锚点，从而降低系统复杂度并提升迭代速度。

---

## 3. 交易类型与数据结构

### 3.1 ServiceTx（服务交易）
服务交易通过 EIP-712 类型化数据签名，经 `EntryPoint` 合约执行，从而与普通交易区分。

```solidity
ServiceTx {
  uint8   kind;        // 交易类型
  bytes32 moduleId;    // 模块ID
  address payer;
  address provider;
  address token;       // 0x0 表示原生币
  uint256 amount;      // 本次可支付/结算的额度
  uint256 nonce;
  uint64  deadline;
  bytes32 requestHash; // ServiceRequest摘要
  bytes32 policyId;    // ArbitrationPolicy摘要
}
```

**建议的 `kind` 类型：**
- `0x01 PAY_DIRECT` 直接支付（可识别的服务支付）
- `0x02 PAY_ESCROW` 托管锁定
- `0x03 SETTLE` 结算（带 Receipt）
- `0x04 REFUND` 退款
- `0x10 ARB_OPEN` 发起仲裁
- `0x11 ARB_EVIDENCE` 提交证据
- `0x12 ARB_DECISION` 裁决提交

### 3.2 ServiceRequest（服务请求）
```solidity
ServiceRequest {
  bytes32 requestId;
  bytes32 moduleId;
  address payer;
  address provider;
  address token;
  uint256 maxAmount;
  uint64  expiry;
  bytes32 payloadHash; // 请求摘要
}
```
`maxAmount` 必须不超过付款方已锁定的预付余额。服务方应在链下执行前检查可用余额与授权额度。

### 3.3 UsageReceipt（服务收据）
```solidity
UsageReceipt {
  bytes32 requestId;
  bytes32 receiptId;
  bytes32 payloadHash;
  bytes32 responseHash;
  uint256 amount;
  uint64  startTime;
  uint64  endTime;
  uint64  nonce;
  bytes32 aux;          // 模块自定义字段
}
```

### 3.4 批量结算（可选）
```solidity
ReceiptBatch {
  bytes32 batchId;
  bytes32 merkleRoot;
  uint64  fromTime;
  uint64  toTime;
}
```

### 3.5 交易类型显性化与防伪约束
为防止垃圾交易与伪造请求，TrustChain 对服务类交易引入“显性化”约束：

1. **EIP-712 深度绑定**：所有 `ServiceTx` 必须使用固定的 Domain Separator（`name=TrustChainEntryPoint`、`version=1`、`chainId` 与 `verifyingContract`）。
2. **Magic Bytes 前缀**：在 `ServiceRequest` 的签名数据中强制加入 `TRUST_V1_` 前缀，避免与普通签名混淆。
3. **SDK 约束**：官方 SDK（JS/Go/Rust）输出标准化结构与前缀，Provider 在链下只接受由 SDK 生成的请求。
4. **Sequencer 标签（展望）**：与 Base 等 L2 合作，将符合协议格式的交易在资源池阶段打上优先标签。

这些约束提高协议可识别性与可审计性，并为 SDK 生态建立护城河。

---

## 4. 核心流程

### 4.1 预付托管模型（唯一支持）
TrustChain 只支持预付/锁仓模式。付款方需先在 `Settlement` 合约中存入余额或锁定额度，服务方仅能在该额度内结算。若余额不足，服务必须暂停或要求充值。

### 4.2 API 计费流程（典型场景）
1. Payer 预存余额或锁定额度
2. Payer 签署 `ServiceRequest`
3. Provider 完成 API 调用，生成 `UsageReceipt`
4. Provider 提交 `ServiceTx(kind=SETTLE)`（从已锁定余额中扣除）
5. 进入挑战期（challengeWindow）
6. 无争议则结算；有争议进入仲裁

### 4.3 电商交付流程（扩展场景）
- 以物流签名、第三方见证或预言机证明作为 `responseHash` 的来源
- 允许多阶段交付与分批结算

### 4.4 乐观结算（Withdraw-then-Slash）
为解决 API 服务的流动性痛点，协议允许在足额质押前提下提前回款：

1. Provider 提交 `UsageReceipt` 后，若其在 Stake Pool 的有效质押 ≥ `minStakeMultiple * amount`（如 5x），即可即时领取 `advanceRate`（如 80%-90%）。
2. 余下资金与等值质押进入锁定区，等待 `challengeWindow` 结束。
3. 若挑战成功，触发“惩罚性罚没”：回收已预支资金，并从质押池扣罚；若挑战失败，释放剩余资金与质押。

该机制使服务方近乎实时回款，同时由质押池提供安全背书。

---

## 5. 仲裁流程与状态机

### 5.1 ArbitrationPolicy（策略参数）
```solidity
ArbitrationPolicy {
  uint64 challengeWindow;
  uint64 bondWindow;
  uint64 evidenceWindow;
  uint64 decisionWindow;

  uint16 payerBondBps;
  uint16 providerBondBps;
  uint16 arbFeeBps;
  uint256 arbFeeFlat;

  uint8  defaultOutcome; // provider/payer/split
  uint8  arbMode;        // committee/court/TEE/ZK/oracle
  address bondToken;     // 0x0 = same as payment
  uint256 minBond;
  uint256 maxBond;
  uint32 evidenceMask;   // 证据类型要求
}
```

### 5.2 状态机
```text
SETTLE_PENDING
  | challengeWindow过期且无争议
  v
FINALIZED

SETTLE_PENDING -- openDispute() --> DISPUTE_OPEN
DISPUTE_OPEN --> BONDING
BONDING -- 双方缴纳保证金 --> EVIDENCE
BONDING -- bondWindow超时 --> EXPIRED (defaultOutcome)

EVIDENCE -- evidenceWindow到期 --> DECISION
DECISION -- submitDecision() --> FINALIZED
DECISION -- decisionWindow超时 --> EXPIRED (defaultOutcome)
EXPIRED --> FINALIZED
```

### 5.3 仲裁流程细节
- **OpenDispute**：在挑战期内发起争议，资金锁定
- **Bonding**：双方提交保证金，未提交一方默认败诉
- **Evidence**：证据提交期，存哈希 + URI
- **Decision**：仲裁模块提交裁决
- **Finalize**：执行分账、罚金、退还保证金

---

## 6. 经济模型与激励

### 6.1 费用结构
- **Service Fee**：服务方收入
- **Protocol Fee**：协议维护收入
- **Arbitration Fee**：仲裁成本
- **预付锁仓余额**：服务消费仅能发生在锁定资金之内，避免“未付款”风险。

### 6.2 保证金与惩罚
- 付款方/服务方均需抵押保证金防止恶意挑战与拒绝交付
- 裁决结果决定保证金释放或罚没

### 6.3 诚实激励
- 无争议快速结算成本最低
- 恶意方承担更高经济惩罚

### 6.4 Stake Pool 经济分配（TaaS 之心）
当仲裁判定 Provider 违约时，罚没的质押金额记为 `S`。协议将 `S` 按比例自动分配：

```text
S_payer     = S * payerBps
S_arb       = S * arbBps
S_treasury  = S * treasuryBps
S_burn      = S * burnBps
```

默认建议区间：
- **补偿买家**：`payerBps = 50%-60%`
- **仲裁奖励**：`arbBps = 20%-30%`
- **协议国库**：`treasuryBps = 10%`
- **销毁**：`burnBps = 10%`

该分配确保买家有动力挑战、仲裁者有动力维护、服务方不敢作恶。

---

## 7. 安全性与风险

- **伪造收据**：通过签名校验与挑战机制防止
- **拒绝配合**：保证金与默认裁决惩罚不配合方
- **仲裁腐败**：可切换仲裁模块与阈值治理
- **重放攻击**：nonce + expiry + requestId
- **数据可用性**：证据以内容寻址存储
- **付款跑路**：仅支持预付锁仓，批量结算只能消费已锁定余额
- **乐观结算风险**：依赖质押充足与挑战活跃度，需明确 `minStakeMultiple` 与 `advanceRate` 上限

---

## 8. 治理与升级

- **Registry 白名单**：模块与政策注册
- **版本化 policyId**：历史不可变，升级可审计
- **紧急暂停机制**：对漏洞模块进行快速冻结

---

## 9. 典型应用场景

- **API Token 计费**：按调用量结算 + 可争议扣费
- **SaaS 替代**：订阅服务与功能解锁
- **电商场景**：物流证明 + 可仲裁退款
- **AI 推理服务**：TEE/ZK 验证计算证明

---

## 10. 路线图（示例）

- **阶段 1**：EntryPoint + Settlement + 简化仲裁
- **阶段 2**：Receipt 批量结算 + 观察者网络
- **阶段 3**：多仲裁模式适配（TEE/ZK/法院）
- **阶段 4**：生态模块市场与 SDK

---

## 11. 结语

TrustChain 不是新的 L1，而是部署在现有链上的“信任协议”。它用可识别交易类型、可验证收据和可插拔仲裁机制，为 API 计费和服务经济提供一个可执行、可扩展的信任基础。随着模块化验证与仲裁工具成熟，TrustChain 将成为连接链上资金与链下服务的核心桥梁。

# TrustApp：面向去中心化服务经济的信任即服务 (TaaS) 协议

**状态**: 草案 / 提案
**版本**: v0.4.1
**日期**: 2025年12月  
**作者**: 0xwilsonwu  
**部署目标**: Base（EVM 兼容）

---

## 摘要

区块链擅长价值转移，但在"服务交换"层面仍存在关键缺口：链上只能证明付款已发生，却无法保证服务已交付。现有方案往往依赖声誉或中心化仲裁，无法在高价值、一次性或跨地域的服务交易中提供可靠保障。

TrustApp 提出一种不修改底层共识的**应用层信任协议**，通过标准化的交易类型、链下收据与链上仲裁，建立"可验证的服务结算"。本协议以 Base 为结算层，使用 EIP-712 类型化数据、可插拔仲裁模块与经济惩罚机制，让服务交付与资金结算在不可信环境下实现可扩展的可信协作。

### 核心创新

| 创新点 | 描述 |
|--------|------|
| **可识别的服务交易类型** | 服务类交易通过 EntryPoint 进入协议层，与普通链上转账清晰区分 |
| **标准化收据模型** | 服务调用通过 ServiceRequest 和 UsageReceipt 表示，可批量结算与可争议 |
| **可插拔仲裁框架** | 委员会、法院、TEE、ZK、预言机均可作为仲裁模块，按策略接入 |
| **经济对齐机制** | 保证金、挑战窗口与惩罚规则为诚实行为提供激励 |
| **仅预付模式** | 所有服务调用在执行前完成锁仓或余额预存，消除"付款跑路"风险 |
| **乐观结算** | 在足额质押的前提下允许服务方提前回款，通过挑战与罚没保护安全性 |

本白皮书完整描述 TrustApp 的协议架构、交易与收据格式、仲裁流程、激励模型与安全边界，并提供面向 API 计费与 SaaS 替代的落地路径。

---

## 目录

1. [背景与问题](#1-背景与问题)
2. [系统总览](#2-系统总览)
3. [交易类型与数据结构](#3-交易类型与数据结构)
4. [核心流程](#4-核心流程)
5. [仲裁流程与状态机](#5-仲裁流程与状态机)
6. [经济模型与激励](#6-经济模型与激励)
7. [安全模型与信任假设](#7-安全模型与信任假设)
8. [治理与升级](#8-治理与升级)
9. [典型应用场景](#9-典型应用场景)
10. [x402 迁移路径](#10-x402-迁移路径)
11. [路线图](#11-路线图)
12. [结语](#12-结语)

**附录**
- [A. 仲裁模块技术规范](#附录-a-仲裁模块技术规范)
- [B. 核心合约参考实现](#附录-b-核心合约参考实现)
- [C. 数据可用性方案](#附录-c-数据可用性方案)
- [D. Merkle 批量结算](#附录-d-merkle-批量结算)
- [E. Gas 成本分析](#附录-e-gas-成本分析)
- [F. 术语表](#附录-f-术语表)

---

## 1. 背景与问题

### 1.1 价值转移 ≠ 服务交换

在 Alice 向 Bob 转账的交易中，区块链只保证"钱已转"。但在 API 调用、内容交付、AI 推理等场景中，买卖双方彼此不信任，结算需要验证"服务确实交付"。链上缺乏对此的通用保证。

### 1.2 声誉与中心化仲裁的局限

| 问题 | 描述 |
|------|------|
| **声誉不可执行** | 一次性交易中，高声誉无法防止"退出骗局" |
| **中心化仲裁不透明** | 规则不清晰、成本高、跨地域执行困难 |
| **链下验证困难** | 链上合约无法直接验证链下事件，需要可插拔的验证与仲裁机制 |

Web2 已经存在托管 + 仲裁的先例。例如支付宝（Alipay）等平台有争议处理与仲裁流程，说明这并不是全新的商业模式，而是现实需求。问题在于规则、证据标准与裁决完全由平台控制，并受制于平台与地域边界。TrustApp 将该流程迁移到可验证的链上结算层，使规则透明、结果可审计、跨平台可组合，并降低对单一运营方的信任依赖。

### 1.3 现有方案为何不足

#### x402 的结构性缺陷

x402 是 Coinbase 提出的 HTTP 原生支付协议，通过复用 HTTP 402 状态码实现"请求-付款-响应"的原子流程。其设计优雅但存在根本性局限：

**缺陷 1：无交付验证**

```
x402 流程：
Client → Server: GET /resource
Server → Client: 402 Payment Required + PaymentRequirements
Client → Server: GET /resource + X-PAYMENT header (签名授权)
Server: 验证签名 → 结算 → 返回资源
```

问题：Server 收到付款后，**返回什么完全由 Server 决定**。Client 无法证明：
- 资源是否完整交付
- 资源质量是否符合约定
- 计量数据（如 token 消耗）是否真实

**缺陷 2：Facilitator 中心化信任**

x402 的 `verify` 和 `settle` 依赖 facilitator 服务：

```
Server → Facilitator: /verify (检查签名有效性)
Server → Facilitator: /settle (执行链上结算)
```

风险场景：
- Facilitator 与 Provider 串谋：验证通过但不实际执行服务
- Facilitator 宕机：整个支付流程中断
- Facilitator 审查：可选择性拒绝某些交易

**缺陷 3：仅适用于即时交付**

x402 假设"付款 → 立即返回资源"，无法处理：
- 异步服务（AI 训练任务、渲染作业）
- 多阶段交付（电商物流、里程碑付款）
- 持续服务（订阅、API 配额消耗）

**缺陷 4：无争议解决路径**

若 Client 认为服务未交付或质量不达标，x402 无任何追索机制。付款签名一旦提交，资金转移不可逆。

#### ERC-8004 的软约束困境

ERC-8004 为 AI Agent 建立链上身份与声誉注册，包含 Identity Registry、Reputation Registry、Validation Registry 三层。其局限在于：

**缺陷 1：声誉不可执行**

```solidity
// ERC-8004 的 Reputation 记录
struct Feedback {
    uint256 agentId;
    address clientAddress;
    uint8 score;        // 1-100
    bytes32 dataHash;   // 反馈详情
}
```

问题：**声誉是历史统计，不是未来保证**
- 高声誉 Agent 仍可执行"退出骗局"（积累声誉 → 大额欺诈 → 跑路）
- 一次性高价值交易中，声誉的威慑力趋近于零
- 声誉损失是"软惩罚"，无经济强制力

**缺陷 2：反馈系统可被操纵**

ERC-8004 的反馈权重与支付金额关联（payment-weighted），但：
- 自我交易刷分：Agent 用多个钱包给自己好评
- 贿赂攻击：付费购买好评
- 恶意差评：竞争对手攻击

缺乏质押惩罚机制，操纵成本极低。

**缺陷 3：与结算层脱节**

ERC-8004 定位为"身份与发现层"，明确声明：

> "Payments are orthogonal to this protocol"

这意味着：
- 声誉记录与实际支付无原子绑定
- 差评不能触发退款
- 好评不能解锁托管资金

声誉系统与经济系统割裂，无法形成闭环激励。

**缺陷 4：验证层过于抽象**

ERC-8004 的 Validation Registry 支持多种验证者（TEE、ZK、委员会），但：
- 未定义验证接口标准
- 未说明验证结果如何影响结算
- 验证失败的经济后果不明确

### 1.4 TrustApp 的差异化定位

| 维度 | x402 | ERC-8004 | TrustApp |
|------|------|----------|------------|
| **核心问题** | 如何付款 | 如何发现/评价 | 如何验证交付并强制执行 |
| **信任模型** | 信任 Facilitator | 信任声誉历史 | 信任博弈机制（质押+惩罚） |
| **资金流** | 即时转移 | 无 | 托管 → 挑战期 → 结算 |
| **交付验证** | 无 | 事后反馈（软） | 收据 + 可插拔证明（硬） |
| **争议解决** | 无 | 无 | 链上仲裁 + 经济罚没 |
| **适用场景** | 即时资源访问 | Agent 发现与筛选 | 复杂服务结算 |
| **惩罚机制** | 无 | 声誉下降 | 质押罚没 + 资金扣押 |
| **可组合性** | HTTP 层 | 身份层 | 结算层（可与前两者集成） |

### 1.5 TrustApp 的核心主张

**主张 1：信任不应依赖声誉，应依赖经济博弈**

声誉是概率性威慑，质押是确定性约束。TrustApp 要求：
- Provider 必须质押才能提供高价值服务
- 作恶的经济损失 > 作恶的经济收益
- 诚实行为是纳什均衡

**主张 2：交付验证必须链上可执行**

"可验证"不等于"可执行"。TrustApp 确保：
- 验证结果直接触发资金流动
- 争议裁决自动执行（无需人工介入链下）
- 所有状态转换链上可审计

**主张 3：协议应覆盖完整服务生命周期**

x402 只管"付款"，ERC-8004 只管"发现"。TrustApp 管：
- 服务请求（ServiceRequest）
- 服务执行（链下，但有收据）
- 服务结算（Settlement）
- 服务争议（Arbitration）
- 服务惩罚（Slash）

### 1.6 设计目标

TrustApp 的目标是：

- 不修改底层共识（不做 L1/L2 改造）
- 可识别服务交易类型（与普通交易区分）
- 可验证与可仲裁（证据驱动、可处罚）
- 高可扩展与可插拔（支持多种证明与业务）
- 面向 API 计费与 SaaS 替代（链上结算、链下执行）

---

## 2. 系统总览

### 2.1 关键角色

| 角色 | 职责 |
|------|------|
| **Payer（付款方）** | 发起服务请求，承担支付责任 |
| **Provider（服务方）** | 执行服务并提交收据 |
| **Relayer（中继）** | 可选，代为提交交易 |
| **Arbitrator（仲裁者）** | 执行裁决逻辑（委员会/法院/TEE/ZK） |
| **Watcher（观察者）** | 监督、挑战或提供证据 |

### 2.2 关键合约模块

```
┌─────────────────────────────────────────────────────────────────┐
│                        TrustApp Protocol                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │  EntryPoint  │────►│   Registry   │◄────│   Stake      │    │
│  │  统一入口    │     │  模块注册    │     │   Pool       │    │
│  └──────┬───────┘     └──────────────┘     └──────┬───────┘    │
│         │                                         │             │
│         │         ┌──────────────┐                │             │
│         └────────►│  Settlement  │◄───────────────┘             │
│                   │  托管与结算  │                              │
│                   └──────┬───────┘                              │
│                          │                                      │
│                   ┌──────▼───────┐                              │
│                   │  Arbitration │                              │
│                   │  争议与裁决  │                              │
│                   └──────────────┘                              │
│                          │                                      │
│         ┌────────────────┼────────────────┐                    │
│         ▼                ▼                ▼                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │  Committee  │  │    TEE      │  │     ZK      │            │
│  │  Arbitrator │  │  Arbitrator │  │  Arbitrator │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| 模块 | 功能 |
|------|------|
| **EntryPoint** | 统一入口，识别服务类交易类型 |
| **Registry** | 模块注册、版本与策略管理 |
| **Settlement** | 托管与结算逻辑 |
| **Arbitration** | 争议管理与裁决执行 |
| **Stake & Slash** | 保证金与惩罚机制 |

### 2.3 ZIP 可插拔模块

模块可被打包为 ZIP，并通过 Registry 注册：

```
module.zip/
├── manifest.json      # 模块元信息与能力声明
├── contracts/         # 链上验证/结算适配器
├── offchain/          # 执行器/聚合器/证明生成
└── spec/              # Receipt 结构与证据要求
```

### 2.4 信用执行环境（CEE）与减法理论

TrustApp 不是新的 L1/L2，而是在 Base 之上构建的**信用执行环境 (Credit Execution Environment, CEE)**。CEE 只处理符合 TaaS 协议的交易，并为其赋予"信用状态"（可仲裁、可罚没、可结算）。

这是一种共识的"减法实践"：把与共识无关的服务验证逻辑从 L1 剥离，仅使用 Base 作为最终结算与安全锚点，从而降低系统复杂度并提升迭代速度。

---

## 3. 交易类型与数据结构

### 3.1 ServiceTx（服务交易）

服务交易通过 EIP-712 类型化数据签名，经 EntryPoint 合约执行，从而与普通交易区分。

```solidity
struct ServiceTx {
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

**交易类型（kind）**：

| Kind | 值 | 描述 |
|------|-----|------|
| PAY_DIRECT | 0x01 | 直接支付（可识别的服务支付） |
| PAY_ESCROW | 0x02 | 托管锁定 |
| SETTLE | 0x03 | 结算（带 Receipt） |
| REFUND | 0x04 | 退款 |
| ARB_OPEN | 0x10 | 发起仲裁 |
| ARB_EVIDENCE | 0x11 | 提交证据 |
| ARB_DECISION | 0x12 | 裁决提交 |

### 3.1.1 结算主键与摘要规范

为减少实现与审计歧义，协议层采用统一规则：

- **链上结算主键**：`settlementId`（唯一、不可复用，所有 settle/finalize/dispute/confirm 均以此为主键）
- **业务内容摘要**：`requestHash`（或 `requestId`）仅作为关联索引，不作为结算主键
- **收据唯一性**：`receiptId` 唯一标识 UsageReceipt，链上需防重放

### 3.2 ServiceRequest（服务请求）

```solidity
struct ServiceRequest {
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

`maxAmount` 必须不超过付款方已锁定的预付余额。默认策略是在创建请求时锁定 `maxAmount`；轻量策略允许 Provider 在执行前调用链上 `checkAndLock()` 做即时锁定。

### 3.3 UsageReceipt（服务收据）

```solidity
struct UsageReceipt {
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

### 3.3.1 ConfirmService（快速路径确认）

用于快速路径的链下确认消息：

```solidity
struct ConfirmService {
    bytes32 settlementId;  // 结算主键
    address payer;
    address provider;
    address token;
    uint256 amount;
    bytes32 receiptId;
    bytes32 requestHash;   // 可选：仅用于关联索引
    bytes32 policyId;      // 可选但强烈建议
    uint8 rating;          // 可选：服务质量评分 (0-100)
    uint64 deadline;       // 过期时间（替代 timestamp）
    uint256 nonce;         // 防止重放（链上必须消费）
}
```

ConfirmService 是对某一笔结算的**最终授权**。Payer 使用 EIP-712 签署此消息，Provider 通过 `settleWithConfirm(...)` 提交签名与收据。EntryPoint 验证签名后，Settlement 立即释放资金并**消费 nonce/confirmHash**，跳过挑战期。
链上必须记录并消费 nonce，可采用 `confirmNonce` 严格递增或 `confirmDigest` 一次性消费两种方式之一。

规范化消息体要求与 ServiceTx 同等级别的深度绑定；ConfirmService 一旦上链消费不可撤销，若需撤销必须走 dispute/arb 流程。

### 3.4 批量结算（可选）

```solidity
struct ReceiptBatch {
    bytes32 batchId;
    bytes32 merkleRoot;
    uint64  fromTime;
    uint64  toTime;
}
```

### 3.5 交易类型显性化与防伪约束

为防止垃圾交易与伪造请求，TrustApp 对服务类交易引入"显性化"约束：

| 约束 | 描述 |
|------|------|
| **EIP-712 深度绑定** | 所有 ServiceTx 必须使用固定的 Domain Separator（name=TrustAppEntryPoint、version=1、chainId 与 verifyingContract） |
| **Magic Bytes 前缀** | 在 ServiceRequest 的签名数据中强制加入 `TRUST_V1_` 前缀，避免与普通签名混淆 |
| **SDK 约束** | 官方 SDK（JS/Go/Rust）输出标准化结构与前缀，Provider 在链下只接受由 SDK 生成的请求 |
| **Sequencer 标签（展望）** | 与 Base 等 L2 合作，将符合协议格式的交易在资源池阶段打上优先标签 |

这些约束提高协议可识别性与可审计性，并为 SDK 生态建立护城河。

---

## 4. 核心流程

### 4.1 预付托管模型（唯一支持）

TrustApp 只支持预付/锁仓模式。付款方需先在 Settlement 合约中存入余额或锁定额度，服务方仅能在该额度内结算。默认策略是在创建 ServiceRequest 时锁定 `maxAmount`；轻量策略允许 Provider 在执行前链上 `checkAndLock()`，用于中频场景。若余额不足，服务必须暂停或要求充值。

```
┌─────────┐                    ┌─────────────┐                    ┌──────────┐
│  Payer  │                    │  Settlement │                    │ Provider │
└────┬────┘                    └──────┬──────┘                    └────┬─────┘
     │                                │                                │
     │  1. deposit(token, amount)     │                                │
     │───────────────────────────────►│                                │
     │                                │                                │
     │  2. Balance locked             │                                │
     │◄───────────────────────────────│                                │
     │                                │                                │
     │  3. ServiceRequest (off-chain) │                                │
     │─────────────────────────────────────────────────────────────────►
     │                                │                                │
     │                                │  4. Execute service            │
     │                                │                                │
     │                                │  5. settle(receipt)            │
     │                                │◄───────────────────────────────│
     │                                │                                │
     │  6. Challenge window           │                                │
     │                                │                                │
     │  7. Finalize                   │                                │
     │                                │───────────────────────────────►│
```

### 4.2 双路径结算模型（乐观执行）

TrustApp 采用"双路径"结算模型，平衡速度与安全性：

**路径 A：快速路径（客户端确认 - Payer 无需上链）**

这是覆盖 >99% 正常情况的"快乐路径"：

1. **链下确认**：Payer 签署链下 EIP-712 类型化数据消息：
   ```solidity
   struct ConfirmService {
       bytes32 settlementId;
       address payer;
       address provider;
       address token;
       uint256 amount;
       bytes32 receiptId;
       bytes32 requestHash;   // 可选：关联索引
       bytes32 policyId;      // 可选但强烈建议
       uint8 rating;          // 可选：服务质量评分
       uint64 deadline;
       uint256 nonce;
   }
   ```
   Payer 不需要发链上交易，仅需链下签名。

2. **签名传输**：Payer 将此数字签名传输给 Provider（链下，如 HTTP 响应头）。

3. **即时结算**：Provider 调用 `settleWithConfirm(receipt, confirm, confirmSig)`。EntryPoint 验证签名字段一致、nonce 被消费、未 disputed/未 finalized，随后立即释放资金。

4. **结果**：资金**瞬间释放**，无需等待挑战期。这减少了网络负载和成本。

**路径 B：慢速路径（证明与争议）**

如果 Payer 未响应或拒绝确认：

1. Provider 提交 `ServiceTx(kind=SETTLE)`，包含 `UsageReceipt` 和可选的交付证明（如 TEE 认证、Oracle 数据）。

2. 进入挑战期（challengeWindow），期间 Payer 或 Watcher 可发起争议。

3. 若无争议，挑战期结束后自动结算；若有争议，进入仲裁流程。

**设计优势**：

- **零摩擦体验**：快速路径提供 Web2 支付的无摩擦体验，同时保持链上可执行性
- **成本优化**：大部分交易通过链下确认完成，显著降低链上负载（附录 E 的 Gas 数据支撑了批量与快速路径的设计选择）
- **安全保证**：慢速路径保留"无须信任"的保证，作为争议时的后备机制

### 4.3 API 计费流程（典型场景）

1. Payer 预存余额或锁定额度
2. Payer 签署 ServiceRequest
3. Provider 完成 API 调用，生成 UsageReceipt
4. **快速路径**：Payer 链下签署 ConfirmService → Provider 调用 `settleWithConfirm(...)` → 瞬间结算
   **或慢速路径**：Provider 提交 ServiceTx(kind=SETTLE) → 挑战期 → 结算
5. 若有争议，进入仲裁流程

### 4.4 电商交付流程（扩展场景）

- 以物流签名、第三方见证或预言机证明作为 responseHash 的来源
- 允许多阶段交付与分批结算

### 4.5 乐观结算（Withdraw-then-Slash）

为解决 API 服务的流动性痛点，协议允许在足额质押前提下提前回款：

1. Provider 提交 UsageReceipt 后，若其在 Stake Pool 的有效质押 ≥ `minStakeMultiple * amount`（如 2x），即可即时领取 `advanceRate`（如 80%）
2. 余下资金与等值质押进入锁定区，等待 challengeWindow 结束
3. 若挑战成功，触发"惩罚性罚没"：回收已预支资金，并从质押池扣罚；若挑战失败，释放剩余资金与质押

该机制使服务方近乎实时回款，同时由质押池提供安全背书。

#### 4.5.1 乐观结算的博弈论安全分析

**模型设定**

设一次服务交易：
- 服务金额：`A`
- Provider 质押：`S`（要求 `S ≥ k * A`，k 为质押倍数）
- 预支比例：`r`（如 80%）
- 罚没比例：`p`（如 150%，即罚没 1.5 倍作恶金额）
- 挑战成功概率：`q`（取决于 Watcher 活跃度和证据质量）
- 挑战成本：`c`（Gas + 保证金机会成本）

**Provider 作恶分析**

作恶收益（收款但不提供服务）：
```
R_cheat = r * A * (1 - q)  // 预支金额 × 未被挑战的概率
```

作恶成本（被挑战成功时）：
```
C_cheat = q * (r * A + p * A)  // 回收预支 + 额外罚没
        = q * A * (r + p)
```

**诚实条件**（作恶无利可图）：
```
C_cheat > R_cheat
q * A * (r + p) > r * A * (1 - q)
q > r / (2r + p)
```

**数值示例**：
- r = 0.8（预支 80%）
- p = 1.5（罚没 150%）

```
q > 0.8 / (1.6 + 1.5) = 0.8 / 3.1 ≈ 0.258
```

即：**只要挑战成功概率 > 25.8%，Provider 作恶无利可图。**

**质押倍数下界推导**

为确保罚没可执行（Provider 有足够质押可罚）：
```
S ≥ p * A
k * A ≥ p * A
k ≥ p
```

若 p = 1.5，则 k ≥ 1.5（质押至少为服务金额的 1.5 倍）。

**建议配置**：

| 参数 | 建议值 | 说明 |
|------|--------|------|
| minStakeMultiple | 2.0 | 安全余量 |
| advanceRate | 0.8 | 预支 80% |
| slashMultiple | 1.5 | 罚没 150% |
| minChallengeProb | 0.3 | 系统设计目标 |

**Challenger 激励分析**

为确保有人愿意挑战：
```
challengerReward > challengeCost
arbBps * slashedAmount > c
arbBps * p * A > c
```

若 A = 1000 USDC, p = 1.5, c = 50 USDC：
```
arbBps > 50 / 1500 ≈ 3.3%
```

**建议**：arbBps ≥ 20%，确保挑战有足够激励。

**质押代币波动性风险**

若质押使用协议原生代币而非稳定币：
```
实际质押价值 = S * priceToken
安全条件：S * priceToken ≥ p * A（以 USDC 计）
```

**缓解措施**：
1. 强制使用稳定币质押（推荐）
2. 动态质押要求（根据价格波动调整 k）
3. 质押价值监控 + 强制补仓机制

**工程约束（必须可执行）**：
- 质押资产建议使用稳定币，或对波动资产设置折扣率（haircut）
- stake 锁定/解锁时序需与 `challengeWindow` 对齐，避免提前释放
- `clawback/slash` 的触发者、触发窗口与分配比例需在接口与实现中显式定义

---

## 5. 仲裁流程与状态机

### 5.1 ArbitrationPolicy（策略参数）

```solidity
struct ArbitrationPolicy {
    // 时间窗口
    uint64 challengeWindow;    // 挑战期
    uint64 bondWindow;         // 保证金提交期
    uint64 evidenceWindow;     // 证据提交期
    uint64 decisionWindow;     // 裁决期

    // 费用参数
    uint16 payerBondBps;       // Payer 保证金比例
    uint16 providerBondBps;    // Provider 保证金比例
    uint16 arbFeeBps;          // 仲裁费比例
    uint256 arbFeeFlat;        // 固定仲裁费

    // 裁决参数
    uint8  defaultOutcome;     // 默认结果：provider/payer/split
    uint8  arbMode;            // 仲裁模式：committee/court/TEE/ZK/oracle
    address bondToken;         // 保证金代币，0x0 = 与支付同币种
    uint256 minBond;           // 最小保证金
    uint256 maxBond;           // 最大保证金
    uint32 evidenceMask;       // 证据类型要求
}
```

**参数选择建议**：不同模块/行业的风险结构差异很大，`arbFeeBps`、`bondWindow`、`challengeWindow`、`minBond` 等应按场景配置（例如 API 计费偏短窗口与高频挑战、电商偏长窗口与更高保证金）。Policy 不是静态常量，而是可版本化、可按业务调参的策略集合。

### 5.2 状态机

```
SETTLE_PENDING
  │ challengeWindow 过期且无争议
  ▼
FINALIZED

SETTLE_PENDING ──openDispute()──► DISPUTE_OPEN
                                      │
DISPUTE_OPEN ─────────────────────────▼
                                   BONDING
                                      │
              ┌───────────────────────┴───────────────────────┐
              │                                               │
              ▼                                               ▼
    双方缴纳保证金                                    bondWindow 超时
              │                                               │
              ▼                                               ▼
          EVIDENCE                                    EXPIRED (defaultOutcome)
              │                                               │
              ▼                                               │
    evidenceWindow 到期                                       │
              │                                               │
              ▼                                               │
          DECISION                                            │
              │                                               │
    ┌─────────┴─────────┐                                    │
    ▼                   ▼                                    │
submitDecision()   decisionWindow 超时                        │
    │                   │                                    │
    ▼                   ▼                                    │
FINALIZED          EXPIRED (defaultOutcome)                   │
                        │                                    │
                        └────────────────────────────────────┘
                                      │
                                      ▼
                                  FINALIZED
```

### 5.3 仲裁流程细节

| 阶段 | 描述 |
|------|------|
| **OpenDispute** | 在挑战期内发起争议，资金锁定 |
| **Bonding** | 双方提交保证金，未提交一方默认败诉 |
| **Evidence** | 证据提交期，存哈希 + URI |
| **Decision** | 仲裁模块提交裁决 |
| **Finalize** | 执行分账、罚金、退还保证金 |

### 5.4 结算确定性时间（Finality Bounds）

#### 最大结算延迟（无争议）

```
T_finality = challengeWindow + blockConfirmations
           = 24h + ~3min (12 blocks on Base)
           ≈ 24 hours 3 minutes
```

#### 最大争议解决时间

```
T_dispute_max = challengeWindow + bondWindow + evidenceWindow + decisionWindow
              = 24h + 48h + 72h + 48h
              = 192 hours (8 days)
```

#### 统计最终性

在 n 个独立 Watcher 监控下，恶意结算被挑战的概率：

```
P(caught) = 1 - (1-p)^n
```

若单个 Watcher 检测率 p = 0.8，n = 5：

```
P(caught) = 1 - 0.2^5 = 99.97%
```

**建议**：对于高价值交易（> $10k），要求至少 3 个注册 Watcher 确认。

#### 确定性时间总结

| 场景 | 最大时间 | 说明 |
|------|----------|------|
| 正常结算（无争议） | 24h 3min | challengeWindow + 确认 |
| 争议解决 | 8 天 | 全流程最大值 |
| 超时自动裁决 | 8 天 + 7 天 = 15 天 | 含 grace period |

### 5.5 超时自动裁决（Force Finalization）

#### 问题

若所有仲裁者离线或拒绝响应，争议可能永久悬而未决。

#### 解决方案：分层超时机制

| 阶段 | 超时条件 | 自动结果 |
|------|----------|----------|
| BONDING | bondWindow 到期 | 未提交方败诉 |
| EVIDENCE | evidenceWindow 到期 | 进入 DECISION |
| DECISION | decisionWindow 到期 | defaultOutcome 生效 |
| DECISION + grace | decisionWindow + 7d | 任何人可调用 `forceFinalize()` |

#### forceFinalize() 逻辑

```solidity
function forceFinalize(bytes32 disputeId) external {
    Dispute storage d = disputes[disputeId];
    require(d.state == DisputeState.DECISION, "Invalid state");
    require(
        block.timestamp > d.decisionDeadline + GRACE_PERIOD,
        "Grace period not passed"
    );

    // 裁决逻辑
    if (d.providerEvidenceValid && !d.payerEvidenceValid) {
        // Provider 提交了有效收据签名
        _executeDecision(disputeId, Outcome.PROVIDER_WINS);
    } else if (d.payerEvidenceValid && !d.providerEvidenceValid) {
        // Payer 提交了服务失败证明
        _executeDecision(disputeId, Outcome.PAYER_WINS);
    } else {
        // 双方都无有效证据 → 视为无效争议，资金罚没入国库
        _executeDecision(disputeId, Outcome.INVALID);
    }
}
```

**无效争议（INVALID）处理**：

当双方均未提交有效证据时，不应简单 50/50 分账（这会激励恶意争议），而应：
- 争议金额的 **liquidateBps**（如 50%-100%）罚没进入协议国库或仲裁基金
- 剩余部分（如有）按比例退还
- 双方保证金全部罚没

这种设计惩罚"开启争议却不举证"的行为，避免争议机制被滥用。

**保证**：即使整个仲裁层失效，资金不会永久锁定。

---

## 6. 经济模型与激励

### 6.1 费用结构

| 费用类型 | 描述 |
|----------|------|
| **Service Fee** | 服务方收入 |
| **Protocol Fee** | 协议维护收入（默认 0.3%） |
| **Arbitration Fee** | 仲裁成本 |
| **预付锁仓余额** | 服务消费仅能发生在锁定资金之内，避免"未付款"风险 |

### 6.2 保证金与惩罚

- 付款方/服务方均需抵押保证金防止恶意挑战与拒绝交付
- 裁决结果决定保证金释放或罚没

### 6.3 诚实激励

- 无争议快速结算成本最低
- 恶意方承担更高经济惩罚

### 6.4 Stake Pool 经济分配（TaaS 之心）

当仲裁判定 Provider 违约时，罚没的质押金额记为 S。协议将 S 按比例自动分配：

```
S_payer     = S * payerBps      // 补偿买家
S_arb       = S * arbBps        // 仲裁奖励
S_treasury  = S * treasuryBps   // 协议国库
S_burn      = S * burnBps       // 销毁
```

**默认建议区间**：

| 分配对象 | 比例 | 说明 |
|----------|------|------|
| 补偿买家 | 50%-60% | 激励 Payer 挑战 |
| 仲裁奖励 | 20%-30% | 激励仲裁者维护 |
| 协议国库 | 10% | 协议可持续发展 |
| 销毁 | 10% | 通缩机制 |

该分配确保买家有动力挑战、仲裁者有动力维护、服务方不敢作恶。

### 6.5 抗 Sybil 攻击的经济分析

#### 攻击场景

攻击者创建 N 个虚假 Provider 身份，互相"交易"刷高信誉或欺诈高价值订单。

#### 防护机制：质押门槛

每个 Provider 必须质押 `minStake` 才能接受结算。

**Sybil 攻击成本**：
```
C_sybil = N × minStake + N × registrationFee + opportunity_cost
```

**攻击收益**：
```
R_sybil = 单次高价值欺诈金额 A
```

**安全条件**：
```
C_sybil > R_sybil
N × minStake > A
```

**示例计算**：

若 `minStake = 1000 USDC`，攻击 10,000 USDC 交易需要：
```
N > 10,000 / 1,000 = 10 个身份
实际成本 = 10 × 1000 = 10,000 USDC（无利可图）
```

#### 建议参数

| 参数 | 建议值 | 说明 |
|------|--------|------|
| minStake | 0.5 × avgTxAmount | 目标市场的平均交易额的一半 |
| registrationFee | 10-50 USDC | 增加身份创建成本 |
| highValueMultiple | 3x | 高价值交易要求更高质押倍数 |

#### 额外防护层

| 机制 | 描述 |
|------|------|
| **时间加权** | 新 Provider 的结算额度随时间逐步解锁 |
| **交易对手多样性** | 只与少数地址交易的 Provider 信誉权重降低 |
| **链上行为分析** | 检测批量注册、相似交易模式 |

---

## 7. 安全模型与信任假设

### 7.1 威胁模型（Threat Model）

#### 攻击者能力假设

| 攻击者类型 | 能力 | 目标 |
|------------|------|------|
| 恶意 Payer | 控制自己的私钥；可发起虚假争议 | 获得服务但不付款 |
| 恶意 Provider | 控制自己的私钥；可伪造收据 | 收款但不提供服务 |
| 恶意 Relayer | 可审查/延迟交易；不能伪造签名 | MEV 提取、拒绝服务 |
| 恶意仲裁者 | 取决于仲裁模式（见 7.2） | 偏袒裁决、贿赂 |
| 恶意 Watcher | 可发起虚假挑战 | 骚扰、拒绝服务 |
| 网络攻击者 | 可观察/延迟链上交易 | 抢跑、三明治攻击 |

#### 不在威胁模型内的攻击

- 底层链（Base）的共识攻击（假设 L1/L2 安全）
- 智能合约漏洞（假设合约经过审计）
- 密码学原语破解（假设 ECDSA、Keccak 安全）

### 7.2 信任假设（Trust Assumptions）

#### 基础假设（所有模式通用）

| 假设 | 描述 | 若违反的后果 |
|------|------|--------------|
| **TA-1: 链活性** | Base 在合理时间内出块 | 争议可能超时，默认裁决生效 |
| **TA-2: 链安全** | Base 交易不可逆（足够确认后） | 已结算资金可能被双花 |
| **TA-3: 时间同步** | 链上时间戳误差 < 15分钟 | 挑战窗口可能被绕过 |
| **TA-4: 数据可用** | 证据存储（IPFS/Arweave）可访问 | 仲裁可能因证据缺失而默认裁决 |

#### 仲裁模式特定假设

**Committee 模式**
```
TA-C1: 仲裁委员会中至少 2/3 成员诚实
TA-C2: 委员会成员无法被贿赂（贿赂成本 > 潜在收益）
TA-C3: 委员会响应时间 < decisionWindow
```

**TEE 模式**
```
TA-T1: TEE 硬件（如 Intel SGX）无侧信道漏洞
TA-T2: Remote Attestation 服务可用且诚实
TA-T3: Provider 的 TEE enclave 代码经过审计
```

**ZK 模式**
```
TA-Z1: ZK 证明系统（Groth16/PLONK）计算健全
TA-Z2: 可信设置（若需要）未被破坏
TA-Z3: 服务逻辑可表达为算术电路
```

**Oracle 模式**
```
TA-O1: Oracle 网络中至少 2/3 节点诚实
TA-O2: Oracle 数据源可靠（如 Chainlink Price Feed）
TA-O3: Oracle 响应时间 < decisionWindow
```

### 7.3 安全属性定义

#### 属性 1：Safety（安全性）

**S1 - Payer 保护**
> 若 Provider 未交付符合 ServiceRequest 规范的服务，诚实的 Payer 不会损失超过争议成本（保证金 + Gas）的资金。

形式化：
```
∀ tx ∈ ServiceTx:
  IF Provider.delivered(tx) = FALSE
  AND Payer.honest(tx) = TRUE
  THEN Payer.loss(tx) ≤ disputeCost(tx)
```

**S2 - Provider 保护**
> 若 Provider 交付了符合 ServiceRequest 规范的服务，诚实的 Provider 不会损失应得收入。

形式化：
```
∀ tx ∈ ServiceTx:
  IF Provider.delivered(tx) = TRUE
  AND Provider.honest(tx) = TRUE
  THEN Provider.revenue(tx) ≥ agreedAmount(tx) - protocolFee(tx)
```

**S3 - 无双重支付**
> 同一笔结算（`settlementId`）或收据（`receiptId`）不会被结算两次。

形式化：
```
∀ s ∈ Settlement:
  |{finalize(s) : s.settlementId}| ≤ 1
```

#### 属性 2：Liveness（活性）

**L1 - 最终结算**
> 任何进入 SETTLE_PENDING 状态的交易，将在 maxSettlementTime 内达到 FINALIZED 状态。

```
maxSettlementTime = challengeWindow + bondWindow + evidenceWindow + decisionWindow + gracePeriod
```

**L2 - 争议可发起**
> 在 challengeWindow 内，任何持有有效证据的参与方都可以发起争议。

**L3 - 资金可取回**
> 在 FINALIZED 状态后，资金在 withdrawalDelay 内可被合法所有者提取。

#### 属性 3：Fairness（公平性）

**F1 - 仲裁公正**
> 在仲裁者诚实假设下，裁决结果与客观事实一致的概率 ≥ fairnessThreshold（如 95%）。

**F2 - 激励兼容**
> 对于理性参与者，诚实行为是占优策略。

```
∀ participant ∈ {Payer, Provider, Arbitrator}:
  E[payoff(honest)] > E[payoff(dishonest)]
```

### 7.4 双花问题与防护

#### 为什么 TrustApp 不存在传统双花？

传统双花发生在**同一笔价值转移被两次确认**的情况下，通常需要独立账本或并行分叉才能实现。TrustApp 并不是 DAG 或个人链结构，而是部署在 Base 上的智能合约系统，所有资金托管在 Settlement 合约中，状态变更由 Base 共识保证原子性与全局顺序性。

```
┌─────────────────────────────────────────────────────────┐
│                    TrustApp 架构                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│   传统双花问题：                                          │
│   ┌─────┐                                                │
│   │Alice│ ──── 10 BTC ───► Bob                          │
│   │     │ ──── 10 BTC ───► Carol  (同一笔钱花两次)       │
│   └─────┘                                                │
│                                                          │
│   TrustApp 防护：                                       │
│   ┌─────┐      ┌────────────┐      ┌─────────┐          │
│   │Payer│ ───► │ Settlement │ ───► │Provider │          │
│   └─────┘      │  Contract  │      └─────────┘          │
│                │ (链上状态) │                            │
│                └────────────┘                            │
│                      │                                   │
│                      ▼                                   │
│              Base 共识保证                                │
│              单一全局状态                                │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

**关键点**：
- 资金由 Settlement 合约托管，余额变更在同一链上原子发生
- Base 提供唯一全局状态，不存在"局部一致性"或分叉账本
- ServiceTx 结算结果不可被链下并行写入覆盖

#### 双花变种场景与防护

虽然传统双花不适用，但仍存在若干变种场景需要在协议层处理。

**场景 1：收据双花（Receipt Double-Claiming）**

攻击：Provider 尝试用同一个 UsageReceipt 结算两次。

```solidity
// 防护：链上唯一性检查
mapping(bytes32 => bool) public settledReceipts;

function settle(UsageReceipt calldata receipt, ...) external {
    require(!settledReceipts[receipt.receiptId], "Receipt already settled");
    settledReceipts[receipt.receiptId] = true;
    // ... 结算逻辑
}
```

**场景 2：请求双用（Request Double-Use）**

攻击：Payer 用同一个 ServiceRequest 向多个 Provider 请求服务。

```solidity
// 防护：requestId 绑定单一 Provider
struct ServiceRequest {
    bytes32 requestId;
    address provider;  // 明确绑定
    // ...
}

// 链上验证
require(request.provider == msg.sender, "Wrong provider");
```

**场景 3：余额竞争（Balance Race Condition）**

攻击：Payer 预存 100 USDC，同时向两个 Provider 发起 100 USDC 请求。

```
时间线：
T0: Payer 余额 = 100 USDC
T1: Provider A 链下检查余额 = 100 ✓
T2: Provider B 链下检查余额 = 100 ✓
T3: Provider A 执行服务
T4: Provider B 执行服务
T5: Provider A 结算成功，余额 = 0
T6: Provider B 结算失败！（余额不足）
```

**防护方案**：

| 方案 | 描述 | 适用场景 |
|------|------|----------|
| **默认策略** | 发起请求时锁定余额，未结算则解锁 | 高价值交易 |
| **轻量策略** | Provider 在执行前调用链上 `checkAndLock()` | 中等价值 |
| **超额质押** | Provider 质押覆盖潜在损失 | 低价值高频 |

```solidity
// 默认策略：请求时锁定（可作为 Settlement 或 RequestRegistry 的一部分）
function createRequest(
    bytes32 requestId,
    address token,
    uint256 amount
) external {
    require(escrows[msg.sender][token].balance >= amount, "Insufficient");
    
    // 立即锁定，防止竞争
    escrows[msg.sender][token].balance -= amount;
    escrows[msg.sender][token].locked += amount;
    
    pendingRequests[requestId] = PendingRequest({
        payer: msg.sender,
        amount: amount,
        lockedAt: block.timestamp,
        // ...
    });
}
```

**场景 4：跨链双花（Cross-Chain Double-Spending）**

攻击：若 TrustApp 扩展到多链，Payer 在 Base 和 Arbitrum 上用同一身份双花。

**防护方案**：

| 方案 | 描述 |
|------|------|
| **独立余额** | 每条链独立托管，互不影响 |
| **跨链锁定** | 通过桥接协议同步锁定状态 |
| **主链结算** | 所有最终结算汇总到主链 |

当前版本（v0.4）仅支持 Base 单链，跨链双花不适用。未来多链扩展的初步设计方向：

1. **主链锚定模式**：所有链的结算最终汇总到 Base 主链，其他链仅作执行层
2. **独立验证模式**：每条链独立运行 TrustApp 实例，通过跨链消息协议（如 LayerZero、Chainlink CCIP）同步 Provider 质押状态
3. **共享质押池**：质押集中在主链，其他链通过轻客户端验证质押证明

具体方案将在多链扩展阶段（路线图阶段 5）详细设计。

#### 双花防护总结

| 双花类型 | 风险等级 | 防护机制 | 状态 |
|----------|----------|----------|------|
| 传统双花（链上） | 无 | Base 共识 + 单一全局状态 | 已解决 |
| 收据双花 | 低 | receiptId 唯一性检查 | 已解决 |
| 请求双用 | 低 | requestId + provider 绑定 | 已解决 |
| 余额竞争 | 中 | 默认锁定 / 可选链上检查锁定 | 已解决 |
| 跨链双花 | N/A | 当前单链，不适用 | 未来版本 |

### 7.5 安全性与风险总结

| 风险 | 防护措施 |
|------|----------|
| 伪造收据 | 通过签名校验与挑战机制防止 |
| 拒绝配合 | 保证金与默认裁决惩罚不配合方 |
| 仲裁腐败 | 可切换仲裁模块与阈值治理 |
| 重放攻击 | nonce + deadline + 链上消费（nonce 或 confirmHash） |
| 数据可用性 | 证据以内容寻址存储 |
| 付款跑路 | 仅支持预付锁仓，批量结算只能消费已锁定余额 |
| 乐观结算风险 | 依赖质押充足与挑战活跃度，需明确 minStakeMultiple 与 advanceRate 上限 |
| 仲裁者离线 | 超时自动裁决 + forceFinalize() |
| MEV 攻击 | Relayer 可能抢跑 settle 交易，建议使用私有 mempool（如 Flashbots Protect）或 Base 私有 Sequencer 通道 |
| ConfirmService 重放 | deadline 过期后签名失效，nonce 链上强制消费，未消费的过期签名不可使用 |

### 7.6 协议能力边界

TrustApp 诚实地划定其能力边界：

#### TrustApp 能解决的问题

| 场景 | 验证方式 |
|------|----------|
| Provider 收款后完全不执行服务 | 无收据 → Payer 胜诉 |
| Provider 提交虚假的计量数据 | TEE/ZK 验证 |
| 明确的交付物争议（有客观标准） | 证据比对 |

#### TrustApp 难以解决的问题

| 场景 | 原因 | 建议 |
|------|------|------|
| 服务质量的主观争议 | 无客观标准 | 需 Committee 人工裁决 |
| 无法重放的实时服务 | 无法事后验证 | 需 TEE 或信任假设 |
| 隐私敏感服务的验证 | 输入/输出不可公开 | 需 ZK 或 MPC |

#### 可验证性等级

建议为不同服务类型定义「可验证性等级」：

| 等级 | 描述 | 质押要求 |
|------|------|----------|
| Level 1 | 确定性计算（ZK 可验证） | 低（1.5x） |
| Level 2 | TEE 可证明执行 | 中（2x） |
| Level 3 | Oracle 可确认交付 | 中（2x） |
| Level 4 | 主观质量（需人工裁决） | 高（3x）+ 声誉权重 |

---

## 8. 治理与升级

### 8.1 Registry 白名单

- 模块与政策需经治理审核后注册
- 注册信息包含版本、审计报告、风险等级

### 8.2 版本化 policyId

- 历史策略不可变
- 升级通过创建新 policyId 实现
- 链上记录升级轨迹，便于审计

### 8.3 紧急暂停机制

| 触发条件 | 执行者 | 影响范围 |
|----------|--------|----------|
| 合约漏洞 | 多签委员会 | 全局暂停 |
| 模块异常 | 模块管理员 | 单模块暂停 |
| 仲裁攻击 | 治理投票 | 特定仲裁者移除 |

### 8.4 治理代币（可选）

若协议引入治理代币，建议：

| 功能 | 描述 |
|------|------|
| 参数调整投票 | 费率、窗口期、罚没比例 |
| 仲裁者选举 | Committee 成员任免 |
| 国库分配 | 协议收入使用 |
| 模块审核 | 新模块上线投票 |

---

## 9. 典型应用场景

### 9.1 API Token 计费

**场景**：AI API 按 token 消耗计费

**流程**：
1. Payer 预存 100 USDC
2. 调用 API，每次生成 UsageReceipt
3. Provider 每日批量结算
4. 挑战期内，Payer 可对任意收据发起争议

**适用仲裁模式**：TEE（API 在可信环境执行）或 ZK（计算可证明）

### 9.2 SaaS 替代

**场景**：订阅服务与功能解锁

**流程**：
1. Payer 预付月费
2. Provider 按功能访问生成收据
3. 月末批量结算
4. 若功能故障，Payer 可发起争议退款

**适用仲裁模式**：Oracle（uptime 监控）或 Committee

### 9.3 电商场景

**场景**：物流证明 + 可仲裁退款

**流程**：
1. Payer 下单，资金托管
2. Provider 发货，物流 Oracle 确认
3. 签收后结算
4. 若货不对版，进入 Committee 仲裁

**适用仲裁模式**：Oracle + Committee

### 9.4 AI 推理服务

**场景**：TEE/ZK 验证计算证明

**流程**：
1. Payer 提交推理请求
2. Provider 在 TEE 内执行，生成 Attestation
3. 链上验证 Attestation，自动结算

**适用仲裁模式**：TEE 或 ZK

---

## 10. x402 迁移路径

对于现有 x402 用户，TrustApp 提供兼容适配，允许渐进式迁移。

### 10.1 HTTP Header 扩展

**原 x402**：
```http
GET /resource HTTP/1.1
X-PAYMENT: <EIP-3009 signature>
```

**TrustApp 增强**：
```http
GET /resource HTTP/1.1
X-PAYMENT: <EIP-3009 signature>
X-TRUSTAPP-REQUEST: <ServiceRequest hash>
X-TRUSTAPP-POLICY: <policyId>
X-TRUSTAPP-RECEIPT: <previous receipt hash>  // 可选，用于链式验证
```

### 10.2 渐进式迁移阶段

```
┌─────────────────────────────────────────────────────────────────────┐
│                        迁移路径                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  阶段 0          阶段 1              阶段 2              阶段 3      │
│  ───────         ──────              ──────              ──────      │
│  纯 x402    →    x402 +          →   x402 +          →   完整       │
│                  TrustApp          TrustApp          TrustApp │
│                  Receipt             Escrow                         │
│                                                                      │
│  • 即时结算      • 增加收据          • 增加托管          • 启用仲裁 │
│  • 无保护        • 可追溯            • 挑战期保护        • 完整保护 │
│  • 信任 Provider • 部分可验证        • 资金安全          • 经济博弈 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

| 阶段 | 改动 | 保护级别 | 适用场景 |
|------|------|----------|----------|
| **阶段 0** | 纯 x402 | 无 | 信任 Provider |
| **阶段 1** | + TrustApp Receipt | 可追溯 | 需要审计轨迹 |
| **阶段 2** | + TrustApp Escrow | 资金安全 | 中等价值交易 |
| **阶段 3** | 完整 TrustApp | 完整博弈保护 | 高价值/不信任场景 |

### 10.3 协议握手序列

```
┌─────────┐                    ┌─────────┐                    ┌────────────────────────┐
│  Client │                    │  Server │                    │ TrustApp (链上合约/网络) │
└────┬────┘                    └────┬────┘                    └───────────┬────────────┘
     │                              │                                │
     │  GET /resource               │                                │
     │─────────────────────────────►│                                │
     │                              │                                │
     │  402 Payment Required        │                                │
     │  + X-TRUSTAPP-SUPPORTED    │                                │
     │◄─────────────────────────────│                                │
     │                              │                                │
     │  (If TrustApp mode)        │                                │
     │  deposit(amount)             │                                │
     │───────────────────────────────────────────────────────────────►
     │                              │                                │
     │  GET /resource               │                                │
     │  + X-PAYMENT                 │                                │
     │  + X-TRUSTAPP-REQUEST      │                                │
     │─────────────────────────────►│                                │
     │                              │                                │
     │                              │  verify request (on-chain)      │
     │                              │───────────────────────────────►│
     │                              │                                │
     │                              │  execute service               │
     │                              │                                │
     │  200 OK + Response           │                                │
     │  + X-TRUSTAPP-RECEIPT      │                                │
     │◄─────────────────────────────│                                │
     │                              │                                │
     │                              │  settle(receipt, on-chain)      │
     │                              │───────────────────────────────►│
     │                              │                                │
```

注：图中的 TrustApp 指去中心化的合约/验证网络，不是中心化的 server。

### 10.4 SDK 兼容层

```typescript
// TrustApp SDK 提供 x402 兼容层
import { TrustAppClient } from '@trustapp/sdk';

const client = new TrustAppClient({
  mode: 'x402-compat',  // 兼容模式
  fallback: 'pure-x402', // 若 Server 不支持 TrustApp，回退到纯 x402
  autoUpgrade: true,     // 自动检测并升级到 TrustApp
});

// 使用方式与 x402 相同
const response = await client.fetch('https://api.example.com/resource', {
  payment: {
    amount: '1000000', // 1 USDC
    token: 'USDC',
  },
  // TrustApp 增强选项（可选）
  trustapp: {
    escrow: true,         // 启用托管
    policy: 'default-api', // 仲裁策略
  },
});
```

### 10.5 迁移收益对比

| 维度 | 纯 x402 | TrustApp |
|------|---------|------------|
| 实现复杂度 | 低 | 中 |
| 资金安全 | 无 | 托管保护 |
| 争议解决 | 无 | 链上仲裁 |
| 适用交易额 | < $100 | 任意 |
| Provider 信誉依赖 | 高 | 低 |
| 结算延迟 | 即时 | 挑战期（可乐观结算） |

---

## 11. 路线图

| 阶段 | 里程碑 | 目标 |
|------|--------|------|
| **阶段 1** | EntryPoint + Settlement + 简化仲裁 | MVP 上线 |
| **阶段 2** | Receipt 批量结算 + Watcher 网络 | 降低 Gas，提高监督 |
| **阶段 3** | 多仲裁模式适配（TEE/ZK/Oracle） | 覆盖更多场景 |
| **阶段 4** | 生态模块市场 + SDK | 开发者生态 |
| **阶段 5** | x402 兼容层 + 跨链扩展 | 扩大采用 |

### 11.1 未来展望

以下方向将在核心协议成熟后作为可选模块探索设计。

#### 11.1.1 跨链结算与双花防护

当前版本仅支持 Base 单链。多链扩展需解决**跨链双花**问题：Payer 可能在多条链上用同一身份重复消费。

**候选方案**：

| 方案 | 机制 | 权衡 |
|------|------|------|
| **主链锚定** | 所有结算最终汇总到 Base，其他链仅作执行层 | 安全性高，但跨链延迟 |
| **共享质押池** | 质押集中在主链，其他链通过轻客户端验证 | 资本效率高，桥接依赖 |
| **独立实例 + 状态同步** | 各链独立运行，通过 LayerZero/CCIP 同步状态 | 灵活，但一致性复杂 |

关键设计约束：
- 跨链消息延迟期间的余额锁定策略
- 质押状态的跨链可验证性
- 争议仲裁的管辖权归属

#### 11.1.2 企业级信用模型

当前协议以**预付锁仓**为核心，适用于无信任基础的交易场景。但现实商业中存在大量长期合作关系（如供应链账期、企业间 net-30/net-60 结算），双方基于历史信用而非逐笔锁仓。

**分层信用模型（Tiered Credit Model）**：

| 层级 | 信用来源 | 适用场景 |
|------|----------|----------|
| Tier 0 | 全额质押/锁仓 | 新关系、一次性交易 |
| Tier 1 | 部分质押 + 历史记录 | 建立中的合作关系 |
| Tier 2 | 低质押 + 链下追索权 | 长期合作伙伴 |

核心设计原则：
- **链上覆盖高频小额风险**：质押与罚没机制保障日常交易
- **链下覆盖低频大额风险**：法律合同备案 + 仲裁追索权作为补充
- **信用额度需有担保来源**：纯声誉不可执行，必须结合部分质押或法律绑定

#### 11.1.3 隐私保护结算

当前所有结算数据链上可见。对于敏感商业场景（如企业采购、竞价服务），可探索：

| 技术 | 应用 |
|------|------|
| **ZK 收据** | 证明"服务已完成且金额正确"，不暴露具体内容 |
| **加密托管** | 结算金额加密，仅争议时解密 |
| **私有仲裁** | 证据提交与裁决过程在 TEE 或私有通道进行 |

#### 11.1.4 去中心化仲裁网络

当前仲裁依赖预配置的仲裁者。长期可探索：
- **仲裁者市场**：竞争性仲裁服务，按专业领域和历史表现匹配
- **质押驱动选择**：仲裁者需质押，错误裁决触发罚没

---

## 12. 结语

TrustApp 不是新的 L1，而是部署在现有链上的"信任协议"。它用可识别交易类型、可验证收据和可插拔仲裁机制，为 API 计费和服务经济提供一个可执行、可扩展的信任基础。

与 x402 的"如何付款"和 ERC-8004 的"如何信任"不同，TrustApp 回答的是"如何验证交付并强制执行"。三者可以互补：

- x402 作为支付触发层
- ERC-8004 作为身份发现层
- TrustApp 作为结算保障层

随着模块化验证与仲裁工具成熟，TrustApp 将成为连接链上资金与链下服务的核心桥梁。

---

## 附录 A：仲裁模块技术规范

### A.1 通用仲裁接口

所有仲裁模块必须实现以下接口：

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IArbitrator {
    /// @notice 仲裁结果枚举
    enum Outcome {
        PENDING,          // 等待裁决
        PAYER_WINS,       // Payer 胜诉，全额退款
        PROVIDER_WINS,    // Provider 胜诉，全额结算
        SPLIT,            // 分账（按比例）
        INVALID           // 无效争议，双方保证金罚没
    }

    /// @notice 裁决详情
    struct Decision {
        Outcome outcome;
        uint16 payerShareBps;     // Payer 分得比例（基点）
        uint16 providerShareBps;  // Provider 分得比例
        bytes32 reasonHash;       // 裁决理由哈希
        uint64 decidedAt;         // 裁决时间
    }

    /// @notice 提交裁决
    function submitDecision(
        bytes32 disputeId,
        Decision calldata decision,
        bytes calldata proof
    ) external;

    /// @notice 验证裁决有效性
    function verifyDecision(
        bytes32 disputeId,
        Decision calldata decision,
        bytes calldata proof
    ) external view returns (bool);

    /// @notice 获取仲裁模式
    function arbMode() external view returns (uint8);

    /// @notice 获取最小仲裁者数量
    function minArbitrators() external view returns (uint8);

    /// @notice 获取仲裁超时时间
    function decisionTimeout() external view returns (uint64);
}
```

### A.2 Committee 仲裁模式

#### 架构

```
┌─────────────────────────────────────────────────────────┐
│                    Committee Arbitrator                  │
├─────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │
│  │Arbiter 1│  │Arbiter 2│  │Arbiter 3│  │Arbiter N│    │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘    │
│       │            │            │            │          │
│       └────────────┴─────┬──────┴────────────┘          │
│                          │                              │
│                    ┌─────▼─────┐                        │
│                    │ Aggregator│ (多签/阈值签名)         │
│                    └─────┬─────┘                        │
│                          │                              │
└──────────────────────────┼──────────────────────────────┘
                           │
                    ┌──────▼──────┐
                    │ Settlement  │
                    └─────────────┘
```

#### 信任假设

| 假设 | 要求 |
|------|------|
| 诚实多数 | ≥ 2/3 仲裁者诚实 |
| 抗串谋 | 仲裁者之间无法有效协调作恶 |
| 激励对齐 | 仲裁奖励 > 贿赂收益 |

#### 适用场景

- 主观质量争议（服务质量不达标）
- 复杂业务逻辑（需要人工判断）
- 高价值交易（值得支付仲裁成本）

### A.3 TEE 仲裁模式

#### 架构

```
┌─────────────────────────────────────────────────────────┐
│                    Provider Server                       │
│  ┌───────────────────────────────────────────────────┐  │
│  │                 TEE Enclave (SGX)                 │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  │  │
│  │  │Service Logic│  │Receipt Sign │  │Attestation│ │  │
│  │  └─────────────┘  └─────────────┘  └──────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                           │
                    Remote Attestation
                           │
                    ┌──────▼──────┐
                    │ TEE Verifier│ (链上合约)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ Settlement  │
                    └─────────────┘
```

#### 证据格式

```solidity
struct TEEEvidence {
    bytes32 requestId;
    bytes32 receiptId;
    bytes32 mrenclave;        // Enclave 代码度量
    bytes32 mrsigner;         // Enclave 签名者度量
    bytes attestationReport;  // SGX Remote Attestation 报告
    bytes enclaveSignature;   // Enclave 内签名
    bytes32 inputHash;
    bytes32 outputHash;
    uint64 executionTime;
}
```

#### 信任假设

| 假设 | 要求 |
|------|------|
| TEE 硬件安全 | Intel SGX 无已知可利用漏洞 |
| Attestation 可用 | Intel IAS / DCAP 服务在线 |
| Enclave 代码正确 | mrenclave 对应的代码经过审计 |

#### 适用场景

- 确定性计算（给定输入，输出唯一）
- 隐私敏感服务（输入/输出不便公开）
- 高频低价值交易（自动验证，无需人工）

### A.4 ZK 仲裁模式

#### 架构

```
┌─────────────────────────────────────────────────────────┐
│                    Provider (Prover)                     │
│  ┌─────────────┐      ┌─────────────┐                   │
│  │Service Logic│ ───► │ ZK Prover   │                   │
│  │  (Circuit)  │      │(Groth16/PLONK)                  │
│  └─────────────┘      └──────┬──────┘                   │
└──────────────────────────────┼──────────────────────────┘
                               │ proof
                        ┌──────▼──────┐
                        │ ZK Verifier │ (链上合约)
                        └──────┬──────┘
                               │
                        ┌──────▼──────┐
                        │ Settlement  │
                        └─────────────┘
```

#### 证据格式

```solidity
struct ZKEvidence {
    bytes32 requestId;
    bytes32 receiptId;
    bytes32 inputCommitment;   // 输入的 Pedersen 承诺
    bytes32 outputCommitment;  // 输出的承诺
    uint256 computeUnits;      // 计算量
    bytes proof;               // ZK 证明
    bytes32 circuitId;         // 已注册电路的 ID
}
```

#### 适用场景

- ML 推理（证明模型正确执行）
- 数据转换（证明 ETL 正确）
- 隐私计算（输入保密但输出可验证）

### A.5 Oracle 仲裁模式

#### 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    外部数据源                                    │
│  (物流 API、IoT 传感器、第三方见证...)                           │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
       ┌──────────┐    ┌──────────┐    ┌──────────┐
       │ Oracle 1 │    │ Oracle 2 │    │ Oracle N │
       │(Chainlink)│   │  (API3)  │    │ (Custom) │
       └────┬─────┘    └────┬─────┘    └────┬─────┘
            │               │               │
            └───────────────┼───────────────┘
                           │
                    ┌──────▼──────┐
                     │   Oracle    │
                     │ Aggregator  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   Oracle    │
                    │  Arbitrator │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ Settlement  │
                    └─────────────┘
```

#### 证据格式

```solidity
struct OracleEvidence {
    bytes32 requestId;
    bytes32 receiptId;

    // Oracle 数据
    bytes32 dataHash;              // Oracle 提供数据的哈希
    bytes32 dataType;              // 类型标识符（如 "DELIVERY", "UPTIME", "PRICE"）
    bytes encodedData;             // ABI 编码的 Oracle 响应

    // Oracle 证明
    address[] oracleAddresses;     // 参与的 Oracle 节点
    bytes[] oracleSignatures;      // 每个 Oracle 的签名
    uint64[] timestamps;           // 每个 Oracle 的时间戳

    // 聚合
    uint8 minConfirmations;        // 所需最小确认数
    uint8 actualConfirmations;     // 实际收到的确认数
    bytes32 aggregatedDataHash;    // 聚合结果的哈希

    // 元数据
    uint64 validUntil;             // 数据过期时间
    bytes32 sourceId;              // 注册的数据源标识符
}
```

#### Oracle 仲裁者接口

```solidity
interface IOracleArbitrator is IArbitrator {

    /// @notice Oracle 节点状态
    enum OracleStatus {
        INACTIVE,
        ACTIVE,
        SUSPENDED,
        SLASHED
    }

    /// @notice 注册的 Oracle 节点
    struct OracleNode {
        address nodeAddress;
        bytes32 nodeId;
        uint256 stake;
        uint64 registeredAt;
        uint64 lastActiveAt;
        OracleStatus status;
        uint32 successCount;
        uint32 failureCount;
        bytes32[] supportedDataTypes;
    }

    /// @notice 数据源配置
    struct DataSource {
        bytes32 sourceId;
        string endpoint;
        bytes32[] requiredOracles;    // 所需的特定 Oracle
        uint8 minConfirmations;
        uint64 maxDataAge;            // 数据的最大年龄（秒）
        uint16 deviationThresholdBps; // Oracle 之间的最大偏差
    }

    /// @notice 注册新的 Oracle 节点
    function registerOracle(
        bytes32 nodeId,
        uint256 stakeAmount,
        bytes32[] calldata supportedDataTypes
    ) external;

    /// @notice 提交 Oracle 证明
    function submitAttestation(
        bytes32 disputeId,
        bytes32 dataHash,
        bytes calldata signature,
        uint64 timestamp
    ) external;

    /// @notice 聚合 Oracle 响应并提交裁决
    function aggregateAndDecide(
        bytes32 disputeId,
        OracleEvidence calldata evidence
    ) external;

    /// @notice 验证 Oracle 证据有效性
    function verifyOracleEvidence(
        OracleEvidence calldata evidence
    ) external view returns (bool valid, string memory reason);

    /// @notice 获取 Oracle 节点信息
    function getOracleNode(address node) external view returns (OracleNode memory);

    /// @notice 检查数据源是否已注册
    function isDataSourceRegistered(bytes32 sourceId) external view returns (bool);

    /// @notice 罚没行为不当的 Oracle
    function slashOracle(address oracle, uint256 amount, bytes32 reason) external;
}
```

#### Oracle 聚合逻辑

```solidity
contract OracleAggregator {

    /// @notice 聚合策略
    enum AggregationStrategy {
        MEDIAN,           // 使用中位数
        WEIGHTED_AVERAGE, // 质押加权平均
        THRESHOLD,        // 布尔阈值（如 2/3 同意）
        FIRST_VALID       // 第一个有效响应（用于非数值数据）
    }

    /// @notice 聚合 Oracle 响应
    function aggregate(
        bytes32[] calldata dataHashes,
        uint256[] calldata stakes,
        AggregationStrategy strategy
    ) external pure returns (bytes32 result, bool valid) {
        if (strategy == AggregationStrategy.THRESHOLD) {
            // 统计匹配的响应
            uint256 maxCount = 0;
            bytes32 mostCommon;

            for (uint i = 0; i < dataHashes.length; i++) {
                uint256 count = 0;
                for (uint j = 0; j < dataHashes.length; j++) {
                    if (dataHashes[i] == dataHashes[j]) count++;
                }
                if (count > maxCount) {
                    maxCount = count;
                    mostCommon = dataHashes[i];
                }
            }

            // 要求 2/3 同意
            valid = (maxCount * 3 >= dataHashes.length * 2);
            result = mostCommon;
        }
        // ... 其他策略
    }

    /// @notice 检查数值 Oracle 响应之间的偏差
    function checkDeviation(
        uint256[] calldata values,
        uint16 maxDeviationBps
    ) external pure returns (bool withinThreshold) {
        if (values.length < 2) return true;

        uint256 min = values[0];
        uint256 max = values[0];

        for (uint i = 1; i < values.length; i++) {
            if (values[i] < min) min = values[i];
            if (values[i] > max) max = values[i];
        }

        // 偏差 = (max - min) / min
        uint256 deviation = ((max - min) * 10000) / min;
        withinThreshold = (deviation <= maxDeviationBps);
    }
}
```

#### 信任假设

| 假设 | 要求 |
|------|------|
| 诚实多数 | ≥ 2/3 Oracle 节点诚实 |
| 数据源可靠性 | 外部 API/传感器正常工作 |
| 质押充足 | Oracle 质押覆盖潜在损失 |
| 及时性 | Oracle 在 decisionWindow 内响应 |

#### 最佳用例

- 物流交付（快递签收确认）
- IoT 传感器验证
- 外部 API 依赖（正常运行时间监控）
- 支付转换的价格源
- 地理位置验证

### A.6 仲裁模式选择指南

| 场景 | 推荐模式 | 理由 |
|------|----------|------|
| API Token 计费 | TEE 或 ZK | 计算可验证，高频自动 |
| AI 推理服务 | ZK | 模型执行可证明 |
| 电商物流 | Oracle | 依赖外部物流数据 |
| 内容创作 | Committee | 质量主观，需人工判断 |
| SaaS 功能解锁 | TEE | 访问控制可在 TEE 内执行 |
| 高价值咨询 | Committee | 复杂交付，难以自动验证 |

---

## 附录 B：核心合约参考实现

> **⚠️ 免责声明**：本附录中的合约代码仅供参考，用于说明协议设计意图。实际部署前必须经过专业安全审计，开发团队不对直接使用这些代码造成的任何损失承担责任。

### B.1 数据结构定义

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

library TrustAppTypes {
    
    enum TxKind {
        PAY_DIRECT,
        PAY_ESCROW,
        SETTLE,
        REFUND,
        ARB_OPEN,
        ARB_EVIDENCE,
        ARB_DECISION
    }
    
    enum DisputeState {
        NONE,
        SETTLE_PENDING,
        DISPUTE_OPEN,
        BONDING,
        EVIDENCE,
        DECISION,
        FINALIZED,
        EXPIRED
    }
    
    struct ServiceTx {
        uint8 kind;
        bytes32 moduleId;
        address payer;
        address provider;
        address token;
        uint256 amount;
        uint256 nonce;
        uint64 deadline;
        bytes32 requestHash;
        bytes32 policyId;
    }
    
    struct ServiceRequest {
        bytes32 requestId;
        bytes32 moduleId;
        address payer;
        address provider;
        address token;
        uint256 maxAmount;
        uint64 expiry;
        bytes32 payloadHash;
    }
    
    struct UsageReceipt {
        bytes32 requestId;
        bytes32 receiptId;
        bytes32 payloadHash;
        bytes32 responseHash;
        uint256 amount;
        uint64 startTime;
        uint64 endTime;
        uint64 nonce;
        bytes32 aux;
    }

    struct ConfirmService {
        bytes32 settlementId;
        address payer;
        address provider;
        address token;
        uint256 amount;
        bytes32 receiptId;
        bytes32 requestHash;
        bytes32 policyId;
        uint8 rating;
        uint64 deadline;
        uint256 nonce;
    }
    
    struct ArbitrationPolicy {
        uint64 challengeWindow;
        uint64 bondWindow;
        uint64 evidenceWindow;
        uint64 decisionWindow;
        uint16 payerBondBps;
        uint16 providerBondBps;
        uint16 arbFeeBps;
        uint256 arbFeeFlat;
        uint8 defaultOutcome;
        uint8 arbMode;
        address bondToken;
        uint256 minBond;
        uint256 maxBond;
        uint32 evidenceMask;
    }
    
    struct EscrowAccount {
        address token;
        uint256 balance;
        uint256 locked;
        uint256 nonce;
    }
    
    struct Dispute {
        bytes32 receiptId;
        DisputeState state;
        address challenger;
        uint256 payerBond;
        uint256 providerBond;
        uint64 openedAt;
        uint64 bondDeadline;
        uint64 evidenceDeadline;
        uint64 decisionDeadline;
        bytes32[] evidenceHashes;
    }
}
```

### B.2 EntryPoint 合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/utils/cryptography/EIP712.sol";
import "@openzeppelin/contracts/utils/cryptography/ECDSA.sol";

contract EntryPoint is EIP712 {
    
    bytes32 public constant SERVICE_TX_TYPEHASH = keccak256(
        "ServiceTx(uint8 kind,bytes32 moduleId,address payer,address provider,address token,uint256 amount,uint256 nonce,uint64 deadline,bytes32 requestHash,bytes32 policyId)"
    );

    bytes32 public constant CONFIRM_SERVICE_TYPEHASH = keccak256(
        "ConfirmService(bytes32 settlementId,address payer,address provider,address token,uint256 amount,bytes32 receiptId,bytes32 requestHash,bytes32 policyId,uint8 rating,uint64 deadline,uint256 nonce)"
    );
    
    bytes8 public constant MAGIC_PREFIX = "TRUST_V1";
    
    address public immutable settlement;
    address public immutable arbitration;
    address public immutable registry;
    
    mapping(address => uint256) public nonces;
    mapping(address => uint256) public confirmNonces;
    
    event ServiceTxExecuted(
        bytes32 indexed requestHash,
        uint8 kind,
        address indexed payer,
        address indexed provider,
        uint256 amount
    );
    
    constructor(
        address _settlement,
        address _arbitration,
        address _registry
    ) EIP712("TrustAppEntryPoint", "1") {
        settlement = _settlement;
        arbitration = _arbitration;
        registry = _registry;
    }
    
    function execute(
        TrustAppTypes.ServiceTx calldata tx_,
        bytes calldata signature
    ) external {
        require(block.timestamp <= tx_.deadline, "Expired");
        require(tx_.nonce == nonces[tx_.payer], "Invalid nonce");
        nonces[tx_.payer]++;
        
        bytes32 structHash = keccak256(abi.encode(
            SERVICE_TX_TYPEHASH,
            tx_.kind, tx_.moduleId, tx_.payer, tx_.provider,
            tx_.token, tx_.amount, tx_.nonce, tx_.deadline,
            tx_.requestHash, tx_.policyId
        ));
        bytes32 digest = _hashTypedDataV4(structHash);
        address signer = ECDSA.recover(digest, signature);
        require(signer == tx_.payer, "Invalid signature");
        
        // Route based on kind
        if (tx_.kind == uint8(TrustAppTypes.TxKind.PAY_ESCROW)) {
            ISettlement(settlement).deposit(tx_.payer, tx_.token, tx_.amount);
        } else if (tx_.kind == uint8(TrustAppTypes.TxKind.SETTLE)) {
            ISettlement(settlement).settle(
                tx_.payer, tx_.provider, tx_.token, tx_.amount, tx_.requestHash, tx_.policyId
            );
        }
        // ... other kinds
        
        emit ServiceTxExecuted(
            tx_.requestHash, tx_.kind, tx_.payer, tx_.provider, tx_.amount
        );
    }

    function settleWithConfirm(
        TrustAppTypes.UsageReceipt calldata receipt,
        TrustAppTypes.ConfirmService calldata confirm,
        bytes calldata confirmSig
    ) external {
        require(block.timestamp <= confirm.deadline, "Expired");
        require(confirm.nonce == confirmNonces[confirm.payer], "Invalid nonce");
        confirmNonces[confirm.payer]++;
        require(confirm.receiptId == receipt.receiptId, "Receipt mismatch");
        require(confirm.amount == receipt.amount, "Amount mismatch");

        bytes32 structHash = keccak256(abi.encode(
            CONFIRM_SERVICE_TYPEHASH,
            confirm.settlementId, confirm.payer, confirm.provider, confirm.token,
            confirm.amount, confirm.receiptId, confirm.requestHash, confirm.policyId,
            confirm.rating, confirm.deadline, confirm.nonce
        ));
        bytes32 digest = _hashTypedDataV4(structHash);
        address signer = ECDSA.recover(digest, confirmSig);
        require(signer == confirm.payer, "Invalid signature");

        ISettlement(settlement).settleWithConfirm(receipt, confirm);
    }
    
    function domainSeparator() external view returns (bytes32) {
        return _domainSeparatorV4();
    }
}
```

### B.3 Settlement 合约（核心逻辑）

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract Settlement is ReentrancyGuard {
    using SafeERC20 for IERC20;
    
    address public immutable entryPoint;
    address public immutable arbitration;
    address public immutable stakePool;
    
    // payer => token => EscrowAccount
    mapping(address => mapping(address => TrustAppTypes.EscrowAccount)) public escrows;
    
    // settlementId => PendingSettlement
    mapping(bytes32 => PendingSettlement) public pendingSettlements;
    
    struct PendingSettlement {
        address payer;
        address provider;
        address token;
        uint256 amount;
        bytes32 requestHash;
        uint64 settleTime;
        bool finalized;
    }
    
    uint16 public protocolFeeBps = 30; // 0.3%
    
    function deposit(address payer, address token, uint256 amount) external {
        IERC20(token).safeTransferFrom(payer, address(this), amount);
        escrows[payer][token].balance += amount;
    }
    
    function settle(
        address payer,
        address provider,
        address token,
        uint256 amount,
        bytes32 requestHash
    ) external nonReentrant returns (bytes32 settlementId) {
        require(escrows[payer][token].balance >= amount, "Insufficient");
        
        escrows[payer][token].balance -= amount;
        escrows[payer][token].locked += amount;
        
        settlementId = keccak256(abi.encode(
            payer, provider, token, amount, requestHash, block.number
        ));
        require(pendingSettlements[settlementId].payer == address(0), "Settlement exists");
        pendingSettlements[settlementId] = PendingSettlement({
            payer: payer,
            provider: provider,
            token: token,
            amount: amount,
            requestHash: requestHash,
            settleTime: uint64(block.timestamp),
            finalized: false
        });
    }
    
    function finalize(bytes32 settlementId) external nonReentrant {
        PendingSettlement storage ps = pendingSettlements[settlementId];
        require(!ps.finalized, "Already finalized");
        
        // Check challenge window passed
        // Check no active dispute
        
        uint256 fee = (ps.amount * protocolFeeBps) / 10000;
        uint256 providerAmount = ps.amount - fee;
        
        escrows[ps.payer][ps.token].locked -= ps.amount;
        
        IERC20(ps.token).safeTransfer(ps.provider, providerAmount);
        
        ps.finalized = true;
    }
    
    function advanceWithdraw(bytes32 settlementId) external nonReentrant {
        PendingSettlement storage ps = pendingSettlements[settlementId];
        require(msg.sender == ps.provider, "Only provider");
        
        // Check stake >= 2x amount
        uint256 stake = IStakePool(stakePool).getStake(ps.provider, ps.token);
        require(stake >= ps.amount * 2, "Insufficient stake");
        
        // Advance 80%
        uint256 advance = (ps.amount * 80) / 100;
        
        IStakePool(stakePool).lockStake(ps.provider, ps.token, ps.amount * 2);
        IERC20(ps.token).safeTransfer(ps.provider, advance);
    }
}
```

### B.4 完整接口定义

#### ISettlement 接口

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface ISettlement {

    /// @notice 托管账户状态
    struct EscrowAccount {
        uint256 balance;      // 可用余额
        uint256 locked;       // 为待结算锁定的金额
        uint256 nonce;        // 账户 nonce
    }

    /// @notice 待结算状态
    struct PendingSettlement {
        address payer;
        address provider;
        address token;
        uint256 amount;
        bytes32 requestHash;
        uint64 settleTime;
        uint64 challengeDeadline;
        bytes32 policyId;
        bool finalized;
        bool disputed;
    }

    // ============ 存款与提取 ============

    /// @notice 存入代币到托管账户
    function deposit(address payer, address token, uint256 amount) external;

    /// @notice 提取可用余额
    function withdraw(address token, uint256 amount) external;

    /// @notice 获取托管余额
    function getBalance(address payer, address token) external view returns (uint256 available, uint256 locked);

    // ============ 结算 ============

    /// @notice 发起结算（锁定资金，开始挑战期）
    function settle(
        address payer,
        address provider,
        address token,
        uint256 amount,
        bytes32 requestHash,
        bytes32 policyId
    ) external returns (bytes32 settlementId);

    /// @notice 快速路径结算（确认签名 + 收据）
    function settleWithConfirm(
        TrustAppTypes.UsageReceipt calldata receipt,
        TrustAppTypes.ConfirmService calldata confirm
    ) external returns (bytes32 settlementId);

    /// @notice 挑战期结束后完成结算
    function finalize(bytes32 settlementId) external;

    /// @notice 批量完成多个结算
    function batchFinalize(bytes32[] calldata settlementIds) external;

    /// @notice 获取待结算详情
    function getPendingSettlement(bytes32 settlementId) external view returns (PendingSettlement memory);

    // ============ 乐观结算 ============

    /// @notice Provider 提前提取（需要质押）
    function advanceWithdraw(bytes32 settlementId) external;

    /// @notice 挑战成功时回收已预支资金
    function clawback(bytes32 settlementId, uint256 amount) external;

    // ============ 争议集成 ============

    /// @notice 标记结算为争议中（由 Arbitration 调用）
    function markDisputed(bytes32 settlementId) external;

    /// @notice 执行仲裁裁决
    function executeDecision(
        bytes32 settlementId,
        uint16 payerShareBps,
        uint16 providerShareBps
    ) external;

    // ============ 管理 ============

    /// @notice 设置协议费用
    function setProtocolFee(uint16 feeBps) external;

    /// @notice 设置 EntryPoint 地址
    function setEntryPoint(address entryPoint) external;

    /// @notice 紧急暂停
    function pause() external;

    /// @notice 取消暂停
    function unpause() external;
}
```

#### IStakePool 接口

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IStakePool {

    /// @notice Provider 的质押状态
    struct StakeInfo {
        uint256 total;        // 总质押金额
        uint256 locked;       // 为待结算锁定的金额
        uint256 slashed;      // 累计被罚没金额
        uint64 lastStakeTime; // 最后质押修改时间
        uint64 unlockTime;    // 冷却解锁时间
    }

    // ============ 质押 ============

    /// @notice 质押代币
    function stake(address token, uint256 amount) external;

    /// @notice 请求解除质押（开始冷却期）
    function requestUnstake(address token, uint256 amount) external;

    /// @notice 冷却期结束后完成解除质押
    function unstake(address token, uint256 amount) external;

    /// @notice 获取质押信息
    function getStake(address provider, address token) external view returns (uint256 available, uint256 locked);

    /// @notice 获取完整质押信息
    function getStakeInfo(address provider, address token) external view returns (StakeInfo memory);

    // ============ 锁定 ============

    /// @notice 为乐观结算锁定质押
    function lockStake(address provider, address token, uint256 amount) external;

    /// @notice 结算完成后解锁质押
    function unlockStake(address provider, address token, uint256 amount) external;

    /// @notice 检查 Provider 是否有足够质押
    function hasSufficientStake(
        address provider,
        address token,
        uint256 requiredAmount,
        uint256 multiplier
    ) external view returns (bool);

    // ============ 罚没 ============

    /// @notice 罚没 Provider 质押（由 Arbitration 调用）
    function slash(
        address provider,
        address token,
        uint256 amount,
        bytes32 reason
    ) external returns (uint256 actualSlashed);

    /// @notice 分配罚没资金
    function distributeSlash(
        uint256 amount,
        address payer,
        address arbitrator,
        uint16 payerBps,
        uint16 arbBps,
        uint16 treasuryBps,
        uint16 burnBps
    ) external;

    // ============ 参数 ============

    /// @notice 获取最小质押要求
    function minStake(address token) external view returns (uint256);

    /// @notice 获取解除质押冷却期
    function unstakeCooldown() external view returns (uint64);

    // ============ 管理 ============

    /// @notice 设置最小质押
    function setMinStake(address token, uint256 amount) external;

    /// @notice 设置解除质押冷却期
    function setUnstakeCooldown(uint64 cooldown) external;

    /// @notice 添加授权的罚没者（如 Arbitration 合约）
    function addSlasher(address slasher) external;
}
```

#### IRegistry 接口

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IRegistry {

    /// @notice 模块注册信息
    struct ModuleInfo {
        bytes32 moduleId;
        string name;
        string version;
        address implementation;
        bytes32 specHash;
        uint8 riskTier;
        string auditReportUri;
        bool active;
        uint64 registeredAt;
    }

    /// @notice 注册新模块
    function registerModule(
        bytes32 moduleId,
        string calldata name,
        string calldata version,
        address implementation,
        bytes32 specHash,
        uint8 riskTier,
        string calldata auditReportUri
    ) external;

    /// @notice 更新模块（创建新版本）
    function updateModule(
        bytes32 moduleId,
        string calldata newVersion,
        address newImplementation,
        bytes32 newSpecHash
    ) external;

    /// @notice 停用模块
    function deactivateModule(bytes32 moduleId) external;

    /// @notice 获取模块信息
    function getModule(bytes32 moduleId) external view returns (ModuleInfo memory);

    /// @notice 检查模块是否激活
    function isModuleActive(bytes32 moduleId) external view returns (bool);

    // ============ 策略注册 ============

    /// @notice 注册仲裁策略
    function registerPolicy(
        bytes32 policyId,
        TrustAppTypes.ArbitrationPolicy calldata policy
    ) external;

    /// @notice 获取策略
    function getPolicy(bytes32 policyId) external view returns (TrustAppTypes.ArbitrationPolicy memory);

    /// @notice 检查策略是否激活
    function isPolicyActive(bytes32 policyId) external view returns (bool);

    // ============ 仲裁者注册 ============

    /// @notice 为某个模式注册仲裁者
    function registerArbitrator(
        uint8 arbMode,
        address arbitrator
    ) external;

    /// @notice 获取某个模式的仲裁者
    function getArbitrator(uint8 arbMode) external view returns (address);

    // ============ 访问控制 ============

    /// @notice 检查地址是否为授权的模块管理员
    function isModuleAdmin(bytes32 moduleId, address account) external view returns (bool);

    /// @notice 授予模块管理员角色
    function grantModuleAdmin(bytes32 moduleId, address account) external;
}
```

#### IArbitration 接口

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IArbitration {

    /// @notice 争议信息
    struct DisputeInfo {
        bytes32 disputeId;
        bytes32 settlementId;
        bytes32 receiptId;
        TrustAppTypes.DisputeState state;
        address challenger;
        address challenged;
        uint256 payerBond;
        uint256 providerBond;
        uint64 openedAt;
        uint64 bondDeadline;
        uint64 evidenceDeadline;
        uint64 decisionDeadline;
        bytes32 policyId;
        bytes32[] evidenceHashes;
    }

    // ============ 争议生命周期 ============

    /// @notice 发起争议
    function openDispute(
        bytes32 settlementId,
        bytes32 receiptId,
        bytes32 policyId,
        bytes calldata evidence
    ) external payable returns (bytes32 disputeId);

    /// @notice 提交保证金
    function submitBond(bytes32 disputeId) external payable;

    /// @notice 提交证据
    function submitEvidence(
        bytes32 disputeId,
        bytes32 evidenceHash,
        string calldata evidenceUri
    ) external;

    /// @notice 提交裁决（由仲裁者调用）
    function submitDecision(
        bytes32 disputeId,
        IArbitrator.Outcome outcome,
        uint16 payerShareBps,
        uint16 providerShareBps,
        bytes32 reasonHash
    ) external;

    /// @notice 完成争议（执行裁决）
    function finalizeDispute(bytes32 disputeId) external;

    /// @notice 强制完成（超时后）
    function forceFinalize(bytes32 disputeId) external;

    // ============ 查询 ============

    /// @notice 获取争议信息
    function getDispute(bytes32 disputeId) external view returns (DisputeInfo memory);

    /// @notice 检查争议是否可强制完成
    function canForceFinalize(bytes32 disputeId) external view returns (bool);
}
```

### B.5 部署脚本

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Script.sol";

contract DeployTrustApp is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        
        vm.startBroadcast(deployerPrivateKey);
        
        // 1. Deploy Registry
        Registry registry = new Registry();
        
        // 2. Deploy StakePool
        StakePool stakePool = new StakePool();
        
        // 3. Deploy Arbitration
        Arbitration arbitration = new Arbitration(address(stakePool));
        
        // 4. Deploy Settlement
        Settlement settlement = new Settlement(address(arbitration), address(stakePool));
        
        // 5. Deploy EntryPoint
        EntryPoint entryPoint = new EntryPoint(
            address(settlement),
            address(arbitration),
            address(registry)
        );
        
        // 6. Set dependencies
        settlement.setEntryPoint(address(entryPoint));
        
        vm.stopBroadcast();
        
        console.log("EntryPoint:", address(entryPoint));
        console.log("Settlement:", address(settlement));
        console.log("Arbitration:", address(arbitration));
    }
}
```

### B.6 ProviderRegistry 合约

ProviderRegistry 管理 Provider 的注册、能力声明和访问控制。

#### 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      ProviderRegistry                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │  Provider   │    │  Service    │    │  Capability │          │
│  │  Profiles   │    │   Types     │    │  Proofs     │          │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘          │
│         │                  │                  │                  │
│         └──────────────────┼──────────────────┘                  │
│                            │                                     │
│                     ┌──────▼──────┐                              │
│                     │   Access    │                              │
│                     │   Control   │                              │
│                     └──────┬──────┘                              │
│                            │                                     │
│         ┌──────────────────┼──────────────────┐                  │
│         ▼                  ▼                  ▼                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │  Whitelist  │    │  Blacklist  │    │  Rate Limit │          │
│  └─────────────┘    └─────────────┘    └─────────────┘          │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

#### 接口定义

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IProviderRegistry {

    /// @notice Provider 状态
    enum ProviderStatus {
        UNREGISTERED,
        PENDING,          // 等待验证
        ACTIVE,
        SUSPENDED,
        BLACKLISTED
    }

    /// @notice Provider 档案
    struct ProviderProfile {
        address providerAddress;
        bytes32 providerId;
        string name;
        string metadataUri;         // IPFS/Arweave URI，用于扩展元数据
        ProviderStatus status;
        uint64 registeredAt;
        uint64 lastActiveAt;
        uint256 totalSettled;       // 累计结算金额
        uint256 disputeCount;
        uint256 disputeLossCount;
        bytes32[] serviceTypes;     // 支持的服务类型
        bytes32[] moduleIds;        // 注册的模块
    }

    /// @notice 服务类型定义
    struct ServiceType {
        bytes32 typeId;
        string name;
        string description;
        bytes32 specHash;           // 服务规范哈希
        uint8 requiredArbMode;      // 所需的仲裁模式
        uint256 minStakeRequired;   // 该服务类型所需的最小质押
        bool active;
    }

    /// @notice 能力证明（如 TEE 证明、ZK 凭证）
    struct CapabilityProof {
        bytes32 proofId;
        bytes32 capabilityType;     // "TEE_SGX", "ZK_PROVER", "ORACLE_NODE" 等
        bytes proofData;            // 编码的证明数据
        uint64 issuedAt;
        uint64 expiresAt;
        address issuer;             // 证明签发者（如 Intel 对于 SGX）
        bool verified;
    }

    // ============ 注册 ============

    /// @notice 注册为 Provider
    function registerProvider(
        string calldata name,
        string calldata metadataUri,
        bytes32[] calldata serviceTypes
    ) external payable returns (bytes32 providerId);

    /// @notice 更新 Provider 档案
    function updateProfile(
        string calldata name,
        string calldata metadataUri
    ) external;

    /// @notice 添加支持的服务类型
    function addServiceType(bytes32 serviceTypeId) external;

    /// @notice 移除支持的服务类型
    function removeServiceType(bytes32 serviceTypeId) external;

    /// @notice 提交能力证明
    function submitCapabilityProof(
        bytes32 capabilityType,
        bytes calldata proofData,
        uint64 expiresAt
    ) external returns (bytes32 proofId);

    // ============ 查询 ============

    /// @notice 获取 Provider 档案
    function getProvider(address provider) external view returns (ProviderProfile memory);

    /// @notice 通过 ID 获取 Provider
    function getProviderById(bytes32 providerId) external view returns (ProviderProfile memory);

    /// @notice 检查 Provider 是否激活
    function isProviderActive(address provider) external view returns (bool);

    /// @notice 检查 Provider 是否支持服务类型
    function supportsServiceType(address provider, bytes32 serviceTypeId) external view returns (bool);

    /// @notice 获取 Provider 的能力证明
    function getCapabilityProofs(address provider) external view returns (CapabilityProof[] memory);

    /// @notice 验证能力证明是否有效
    function hasValidCapability(address provider, bytes32 capabilityType) external view returns (bool);

    // ============ 服务类型管理 ============

    /// @notice 注册新的服务类型
    function registerServiceType(
        bytes32 typeId,
        string calldata name,
        string calldata description,
        bytes32 specHash,
        uint8 requiredArbMode,
        uint256 minStakeRequired
    ) external;

    /// @notice 获取服务类型信息
    function getServiceType(bytes32 typeId) external view returns (ServiceType memory);

    // ============ 访问控制 ============

    /// @notice 暂停 Provider
    function suspendProvider(address provider, string calldata reason) external;

    /// @notice 重新激活被暂停的 Provider
    function reactivateProvider(address provider) external;

    /// @notice 将 Provider 加入黑名单
    function blacklistProvider(address provider, string calldata reason) external;

    /// @notice 检查 Provider 是否在黑名单中
    function isBlacklisted(address provider) external view returns (bool);

    // ============ 速率限制 ============

    /// @notice 获取 Provider 的结算限额
    function getSettlementLimit(address provider) external view returns (uint256 daily, uint256 perTx);

    /// @notice 检查 Provider 是否可以结算金额
    function canSettle(address provider, uint256 amount) external view returns (bool);

    /// @notice 记录结算（由 Settlement 合约调用）
    function recordSettlement(address provider, uint256 amount) external;

    // ============ 声誉集成 ============

    /// @notice 获取 Provider 声誉分数（0-10000）
    function getReputationScore(address provider) external view returns (uint16);

    /// @notice 记录争议结果（由 Arbitration 调用）
    function recordDisputeOutcome(address provider, bool won) external;
}
```

#### Provider 注册流程

```
┌─────────┐                    ┌──────────────────┐                    ┌───────────┐
│ Provider│                    │ ProviderRegistry │                    │ StakePool │
└────┬────┘                    └────────┬─────────┘                    └─────┬─────┘
     │                                  │                                    │
     │  1. registerProvider()           │                                    │
     │  + registration fee              │                                    │
     │─────────────────────────────────►│                                    │
     │                                  │                                    │
     │  2. Validate inputs              │                                    │
     │  Create profile (PENDING)        │                                    │
     │                                  │                                    │
     │  3. stake()                      │                                    │
     │───────────────────────────────────────────────────────────────────────►
     │                                  │                                    │
     │                                  │  4. Check stake meets minimum      │
     │                                  │◄────────────────────────────────────
     │                                  │                                    │
     │  5. submitCapabilityProof()      │                                    │
     │  (optional, for TEE/ZK)          │                                    │
     │─────────────────────────────────►│                                    │
     │                                  │                                    │
     │  6. Verify proof                 │                                    │
     │  Activate provider               │                                    │
     │                                  │                                    │
     │  7. ProviderActivated event      │                                    │
     │◄─────────────────────────────────│                                    │
     │                                  │                                    │
     │  Provider can now accept         │                                    │
     │  ServiceRequests                 │                                    │
```

#### 基于层级的限额

Provider 根据质押和声誉被分配到不同层级，决定其运营限额：

| 层级 | 最小质押 | 最大单笔 | 每日限额 | 所需声誉 |
|------|----------|----------|----------|----------|
| Starter | 100 USDC | 100 USDC | 1,000 USDC | N/A（新用户） |
| Basic | 1,000 USDC | 500 USDC | 10,000 USDC | >= 5000 |
| Standard | 5,000 USDC | 2,000 USDC | 50,000 USDC | >= 7000 |
| Premium | 25,000 USDC | 10,000 USDC | 250,000 USDC | >= 8500 |
| Enterprise | 100,000 USDC | 无限制 | 无限制 | >= 9500 |

```solidity
/// @notice 层级配置
struct TierConfig {
    uint256 minStake;
    uint256 maxPerTx;
    uint256 dailyLimit;
    uint16 minReputation;
}

/// @notice 获取 Provider 的当前层级
function getProviderTier(address provider) external view returns (uint8 tier, TierConfig memory config);
```

### B.7 Watcher 网络规范

Watcher 监控结算并可以挑战欺诈性声明。健康的 Watcher 网络对协议安全至关重要。

#### 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                       Watcher Network                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐     │
│  │ Watcher 1 │  │ Watcher 2 │  │ Watcher 3 │  │ Watcher N │     │
│  │ (Full)    │  │ (Light)   │  │ (Full)    │  │ (Light)   │     │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘     │
│        │              │              │              │            │
│        └──────────────┼──────────────┼──────────────┘            │
│                       │              │                           │
│                ┌──────▼──────┐┌──────▼──────┐                    │
│                │  Event      ││  Challenge  │                    │
│                │  Indexer    ││  Coordinator│                    │
│                └──────┬──────┘└──────┬──────┘                    │
│                       │              │                           │
│                       └──────┬───────┘                           │
│                              │                                   │
│                       ┌──────▼──────┐                            │
│                       │  Watcher    │                            │
│                       │  Registry   │                            │
│                       └─────────────┘                            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

#### Watcher 类型

| 类型 | 职责 | 质押要求 | 奖励份额 |
|------|------|----------|----------|
| **Full Watcher** | 索引所有结算，验证收据，发起挑战 | 10,000 USDC | 挑战者奖励的 70% |
| **Light Watcher** | 监控特定 Provider 或模块，向 Full Watcher 报告 | 1,000 USDC | 挑战者奖励的 20% |
| **Delegated Watcher** | 接收代币持有者的委托，分享奖励 | 可变 | 基于委托 |

#### 接口定义

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IWatcherRegistry {

    /// @notice Watcher 状态
    enum WatcherStatus {
        INACTIVE,
        ACTIVE,
        SUSPENDED,
        EXITING     // 解绑期
    }

    /// @notice Watcher 类型
    enum WatcherType {
        FULL,
        LIGHT,
        DELEGATED
    }

    /// @notice Watcher 档案
    struct WatcherProfile {
        address watcherAddress;
        bytes32 watcherId;
        WatcherType watcherType;
        WatcherStatus status;
        uint256 stake;
        uint256 delegatedStake;
        uint64 registeredAt;
        uint64 lastActiveAt;
        uint32 successfulChallenges;
        uint32 failedChallenges;
        uint256 totalRewards;
        bytes32[] watchedModules;    // 该 Watcher 监控的模块
        address[] watchedProviders;  // 要监控的特定 Provider
    }

    /// @notice 挑战记录
    struct ChallengeRecord {
        bytes32 challengeId;
        bytes32 settlementId;
        address watcher;
        uint64 timestamp;
        bool successful;
        uint256 reward;
    }

    // ============ 注册 ============

    /// @notice 注册为 Watcher
    function registerWatcher(
        WatcherType watcherType,
        bytes32[] calldata watchedModules
    ) external payable returns (bytes32 watcherId);

    /// @notice 增加质押
    function addStake(uint256 amount) external;

    /// @notice 请求解除质押（开始解绑期）
    function requestUnstake(uint256 amount) external;

    /// @notice 解绑期结束后完成解除质押
    function unstake() external;

    /// @notice 更新监控的模块
    function updateWatchedModules(bytes32[] calldata modules) external;

    /// @notice 添加要监控的特定 Provider
    function addWatchedProvider(address provider) external;

    // ============ 委托 ============

    /// @notice 将质押委托给 Watcher
    function delegate(address watcher, uint256 amount) external;

    /// @notice 取消委托
    function undelegate(address watcher, uint256 amount) external;

    /// @notice 获取委托给 Watcher 的总质押
    function getDelegatedStake(address watcher) external view returns (uint256);

    // ============ 挑战 ============

    /// @notice 报告可疑结算
    function reportSuspicious(
        bytes32 settlementId,
        bytes calldata evidence
    ) external returns (bytes32 reportId);

    /// @notice 记录挑战结果（由 Arbitration 调用）
    function recordChallengeOutcome(
        bytes32 challengeId,
        address watcher,
        bool successful,
        uint256 reward
    ) external;

    // ============ 查询 ============

    /// @notice 获取 Watcher 档案
    function getWatcher(address watcher) external view returns (WatcherProfile memory);

    /// @notice 检查 Watcher 是否激活
    function isWatcherActive(address watcher) external view returns (bool);

    /// @notice 获取模块的活跃 Watcher
    function getWatchersForModule(bytes32 moduleId) external view returns (address[] memory);

    /// @notice 获取挑战历史
    function getChallengeHistory(address watcher) external view returns (ChallengeRecord[] memory);

    // ============ 覆盖率指标 ============

    /// @notice 获取结算金额的 Watcher 覆盖率
    function getWatcherCoverage(uint256 amount) external view returns (
        uint8 watcherCount,
        uint256 totalStake,
        bool sufficientCoverage
    );

    /// @notice 结算金额所需的最小 Watcher 数量
    function requiredWatchers(uint256 amount) external view returns (uint8);

    // ============ 奖励 ============

    /// @notice 领取累计奖励
    function claimRewards() external returns (uint256);

    /// @notice 获取待领取奖励
    function pendingRewards(address watcher) external view returns (uint256);
}
```

#### Watcher 激励模型

```
挑战奖励分配：

┌─────────────────────────────────────────────────────────────────┐
│                    罚没金额 (S)                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │
│  │ Payer (50%)   │  │Challenger(20%)│  │ Protocol(30%) │        │
│  └───────────────┘  └───────┬───────┘  └───────────────┘        │
│                             │                                    │
│                             ▼                                    │
│                  ┌─────────────────────┐                         │
│                  │  Challenger Reward  │                         │
│                  │    Distribution     │                         │
│                  └─────────┬───────────┘                         │
│                            │                                     │
│           ┌────────────────┼────────────────┐                    │
│           ▼                ▼                ▼                    │
│    ┌────────────┐   ┌────────────┐   ┌────────────┐             │
│    │ Initiator  │   │ Full       │   │ Light      │             │
│    │ (40%)      │   │ Watchers   │   │ Watchers   │             │
│    │            │   │ (40%)      │   │ (20%)      │             │
│    └────────────┘   └────────────┘   └────────────┘             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### 覆盖率要求

高价值结算需要最小 Watcher 覆盖率：

| 结算金额 | 最小 Watcher 数 | 最小总质押 |
|----------|----------------|------------|
| < 1,000 USDC | 1 | 5,000 USDC |
| 1,000 - 10,000 USDC | 2 | 20,000 USDC |
| 10,000 - 100,000 USDC | 3 | 100,000 USDC |
| > 100,000 USDC | 5 | 500,000 USDC |

如果覆盖率不足，结算进入延长挑战期（2 倍正常时间）。

### B.8 事件日志定义

用于链下索引和监控的完整事件定义：

```solidity
// ============ EntryPoint 事件 ============

event ServiceTxExecuted(
    bytes32 indexed requestHash,
    uint8 indexed kind,
    address indexed payer,
    address provider,
    uint256 amount,
    bytes32 moduleId,
    bytes32 policyId
);

event NonceUsed(address indexed payer, uint256 nonce);

// ============ Settlement 事件 ============

event Deposited(
    address indexed payer,
    address indexed token,
    uint256 amount,
    uint256 newBalance
);

event Withdrawn(
    address indexed payer,
    address indexed token,
    uint256 amount,
    uint256 newBalance
);

event SettlementInitiated(
    bytes32 indexed settlementId,
    bytes32 indexed requestHash,
    address indexed payer,
    address provider,
    address token,
    uint256 amount,
    uint64 challengeDeadline
);

event SettlementFinalized(
    bytes32 indexed settlementId,
    address indexed provider,
    uint256 providerAmount,
    uint256 protocolFee
);

event AdvanceWithdrawn(
    bytes32 indexed settlementId,
    address indexed provider,
    uint256 advanceAmount,
    uint256 lockedStake
);

event Clawback(
    bytes32 indexed settlementId,
    address indexed provider,
    uint256 clawbackAmount
);

// ============ Arbitration 事件 ============

event DisputeOpened(
    bytes32 indexed disputeId,
    bytes32 indexed settlementId,
    address indexed challenger,
    address challenged,
    bytes32 policyId
);

event BondSubmitted(
    bytes32 indexed disputeId,
    address indexed party,
    uint256 amount
);

event EvidenceSubmitted(
    bytes32 indexed disputeId,
    address indexed submitter,
    bytes32 evidenceHash,
    string evidenceUri
);

event DecisionSubmitted(
    bytes32 indexed disputeId,
    IArbitrator.Outcome outcome,
    uint16 payerShareBps,
    uint16 providerShareBps,
    bytes32 reasonHash
);

event DisputeFinalized(
    bytes32 indexed disputeId,
    IArbitrator.Outcome outcome,
    uint256 payerAmount,
    uint256 providerAmount,
    uint256 slashedAmount
);

event ForceFinalized(
    bytes32 indexed disputeId,
    IArbitrator.Outcome outcome,
    address indexed caller
);

// ============ StakePool 事件 ============

event Staked(
    address indexed provider,
    address indexed token,
    uint256 amount,
    uint256 totalStake
);

event UnstakeRequested(
    address indexed provider,
    address indexed token,
    uint256 amount,
    uint64 unlockTime
);

event Unstaked(
    address indexed provider,
    address indexed token,
    uint256 amount,
    uint256 remainingStake
);

event StakeLocked(
    address indexed provider,
    address indexed token,
    uint256 amount,
    bytes32 settlementId
);

event StakeUnlocked(
    address indexed provider,
    address indexed token,
    uint256 amount,
    bytes32 settlementId
);

event Slashed(
    address indexed provider,
    address indexed token,
    uint256 amount,
    bytes32 reason
);

event SlashDistributed(
    uint256 totalAmount,
    uint256 payerAmount,
    uint256 arbAmount,
    uint256 treasuryAmount,
    uint256 burnAmount
);

// ============ ProviderRegistry 事件 ============

event ProviderRegistered(
    address indexed provider,
    bytes32 indexed providerId,
    string name
);

event ProviderActivated(
    address indexed provider,
    bytes32 indexed providerId
);

event ProviderSuspended(
    address indexed provider,
    string reason
);

event ProviderBlacklisted(
    address indexed provider,
    string reason
);

event ServiceTypeAdded(
    address indexed provider,
    bytes32 indexed serviceTypeId
);

event CapabilityProofSubmitted(
    address indexed provider,
    bytes32 indexed proofId,
    bytes32 capabilityType
);

event ReputationUpdated(
    address indexed provider,
    uint16 oldScore,
    uint16 newScore
);

// ============ WatcherRegistry 事件 ============

event WatcherRegistered(
    address indexed watcher,
    bytes32 indexed watcherId,
    IWatcherRegistry.WatcherType watcherType
);

event WatcherStakeChanged(
    address indexed watcher,
    uint256 oldStake,
    uint256 newStake
);

event SuspiciousReported(
    bytes32 indexed reportId,
    bytes32 indexed settlementId,
    address indexed watcher
);

event ChallengeRecorded(
    bytes32 indexed challengeId,
    address indexed watcher,
    bool successful,
    uint256 reward
);

event RewardsClaimed(
    address indexed watcher,
    uint256 amount
);

event DelegationChanged(
    address indexed delegator,
    address indexed watcher,
    uint256 amount,
    bool isDelegation  // true = 委托, false = 取消委托
);

// ============ Registry 事件 ============

event ModuleRegistered(
    bytes32 indexed moduleId,
    string name,
    string version,
    address implementation
);

event ModuleUpdated(
    bytes32 indexed moduleId,
    string newVersion,
    address newImplementation
);

event ModuleDeactivated(
    bytes32 indexed moduleId
);

event PolicyRegistered(
    bytes32 indexed policyId,
    uint8 arbMode
);

event ArbitratorRegistered(
    uint8 indexed arbMode,
    address arbitrator
);
```

---

## 附录 C：数据可用性方案

### C.1 证据存储架构

```
┌─────────────────────────────────────────────────────────┐
│                    Evidence Layer                        │
├─────────────────────────────────────────────────────────┤
│  ┌─────────────┐     ┌─────────────┐     ┌──────────┐  │
│  │    IPFS     │     │  Arweave    │     │ Filecoin │  │
│  │  (热存储)   │     │ (永久存储)  │     │(激励存储)│  │
│  └──────┬──────┘     └──────┬──────┘     └────┬─────┘  │
│         │                   │                  │        │
│         └───────────────────┼──────────────────┘        │
│                             │                           │
│                      ┌──────▼──────┐                    │
│                      │   DA Layer  │                    │
│                      │  Aggregator │                    │
│                      └──────┬──────┘                    │
└─────────────────────────────┼───────────────────────────┘
                              │
                       ┌──────▼──────┐
                       │  Arbitration│
                       └─────────────┘
```

### C.2 证据提交格式

```solidity
struct EvidenceRecord {
    bytes32 evidenceHash;       // 内容哈希
    string ipfsCid;             // IPFS CID
    bytes32 arweaveTxId;        // Arweave 交易ID（可选）
    bytes32 filecoinDealId;     // Filecoin Deal ID（可选）
    uint64 submittedAt;
    address submitter;
}
```

### C.3 DA 不可用处理

| 情况 | 处理方式 |
|------|----------|
| 证据不可检索 | 视为提交方举证失败，适用默认裁决 |
| 证据哈希不匹配 | 证据无效，不计入裁决 |
| Filecoin Deal 过期 | 应在争议期前续期，否则风险自担 |

---

## 附录 D：Merkle 批量结算

### D.1 批量结算架构

对于高频、低价值的交易（如 API 调用），单个链上结算成本过高。Merkle 批量结算将多个收据聚合为单个链上承诺。

```
┌─────────────────────────────────────────────────────────────────┐
│                    批量结算流程                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  链下 (Provider)                     链上                          │
│  ┌─────────────────────┐                                         │
│  │ Receipt 1           │                                         │
│  │ Receipt 2           │                                         │
│  │ Receipt 3           │     批量                                 │
│  │ ...                 │ ──────────►  ┌──────────────────┐       │
│  │ Receipt N           │              │  ReceiptBatch    │       │
│  └─────────────────────┘              │  - batchId       │       │
│           │                           │  - merkleRoot    │       │
│           ▼                           │  - totalAmount   │       │
│  ┌─────────────────────┐              │  - receiptCount  │       │
│  │   Merkle Tree       │              └────────┬─────────┘       │
│  │                     │                       │                 │
│  │       Root          │                       ▼                 │
│  │      /    \         │              挑战窗口                    │
│  │    H01    H23       │                       │                 │
│  │   /  \   /  \       │                       ▼                 │
│  │  H0  H1 H2  H3      │              ┌─────────────────┐        │
│  │  |   |  |   |       │              │ Dispute (if any)│        │
│  │  R0  R1 R2  R3      │              │ - merkleProof   │        │
│  └─────────────────────┘              │ - receiptData   │        │
│                                       └─────────────────┘        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### D.2 数据结构

```solidity
/// @notice 批量结算链上承诺
struct ReceiptBatch {
    bytes32 batchId;
    bytes32 merkleRoot;        // 收据 Merkle 树根
    address provider;
    address token;
    uint256 totalAmount;       // 所有收据金额总和
    uint32 receiptCount;       // 批次中的收据数量
    uint64 fromTime;           // 最早收据时间戳
    uint64 toTime;             // 最晚收据时间戳
    uint64 submittedAt;
    bytes32 policyId;
}

/// @notice 用于 Merkle 证明验证的单个收据
struct MerkleReceipt {
    bytes32 receiptId;
    bytes32 requestId;
    address payer;
    uint256 amount;
    uint64 timestamp;
    bytes32 payloadHash;
    bytes32 responseHash;
}

/// @notice 单个收据争议的 Merkle 证明
struct MerkleProof {
    bytes32[] proof;           // 兄弟节点哈希
    uint256 index;             // 树中叶节点索引
    MerkleReceipt receipt;     // 争议的收据
}
```

### D.3 批量结算合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";

contract BatchSettlement {

    mapping(bytes32 => ReceiptBatch) public batches;
    mapping(bytes32 => mapping(bytes32 => bool)) public disputedReceipts; // batchId => receiptId => disputed

    uint32 public constant MAX_BATCH_SIZE = 10000;
    uint64 public constant MIN_BATCH_INTERVAL = 1 hours;

    event BatchSubmitted(
        bytes32 indexed batchId,
        address indexed provider,
        bytes32 merkleRoot,
        uint256 totalAmount,
        uint32 receiptCount
    );

    event ReceiptDisputed(
        bytes32 indexed batchId,
        bytes32 indexed receiptId,
        address indexed challenger
    );

    /// @notice 提交一批收据
    function submitBatch(
        ReceiptBatch calldata batch,
        bytes calldata providerSignature
    ) external returns (bytes32 batchId) {
        require(batch.receiptCount <= MAX_BATCH_SIZE, "Batch too large");
        require(batch.toTime > batch.fromTime, "Invalid time range");

        // 验证 Provider 签名
        bytes32 batchHash = keccak256(abi.encode(batch));
        address signer = ECDSA.recover(batchHash, providerSignature);
        require(signer == batch.provider, "Invalid signature");

        batchId = keccak256(abi.encodePacked(
            batch.provider,
            batch.merkleRoot,
            batch.fromTime,
            batch.toTime
        ));

        batches[batchId] = batch;

        // 按比例锁定付款方资金
        _lockPayerFunds(batch);

        emit BatchSubmitted(
            batchId,
            batch.provider,
            batch.merkleRoot,
            batch.totalAmount,
            batch.receiptCount
        );
    }

    /// @notice 争议批次中的特定收据
    function disputeReceipt(
        bytes32 batchId,
        MerkleProof calldata proof
    ) external {
        ReceiptBatch storage batch = batches[batchId];
        require(batch.batchId != bytes32(0), "Batch not found");

        // 验证 Merkle 证明
        bytes32 leaf = keccak256(abi.encode(proof.receipt));
        require(
            MerkleProof.verify(proof.proof, batch.merkleRoot, leaf),
            "Invalid Merkle proof"
        );

        // 验证收据未被争议
        require(
            !disputedReceipts[batchId][proof.receipt.receiptId],
            "Already disputed"
        );

        disputedReceipts[batchId][proof.receipt.receiptId] = true;

        // 发起单个争议
        IArbitration(arbitration).openDispute(
            batchId,
            proof.receipt.receiptId,
            batch.policyId,
            abi.encode(proof)
        );

        emit ReceiptDisputed(batchId, proof.receipt.receiptId, msg.sender);
    }

    /// @notice 挑战期结束后完成批次
    function finalizeBatch(bytes32 batchId) external {
        ReceiptBatch storage batch = batches[batchId];
        require(batch.batchId != bytes32(0), "Batch not found");

        // 检查挑战期已过
        uint64 challengeDeadline = batch.submittedAt + IRegistry(registry)
            .getPolicy(batch.policyId).challengeWindow;
        require(block.timestamp > challengeDeadline, "Challenge window active");

        // 计算争议金额
        uint256 disputedAmount = _calculateDisputedAmount(batchId);
        uint256 settleAmount = batch.totalAmount - disputedAmount;

        // 结算无争议金额
        ISettlement(settlement).finalizeBatch(
            batch.provider,
            batch.token,
            settleAmount
        );
    }

    /// @notice 链下生成 Merkle 树（参考实现）
    function computeMerkleRoot(
        MerkleReceipt[] memory receipts
    ) external pure returns (bytes32) {
        require(receipts.length > 0, "Empty receipts");

        // 计算叶节点哈希
        bytes32[] memory leaves = new bytes32[](receipts.length);
        for (uint i = 0; i < receipts.length; i++) {
            leaves[i] = keccak256(abi.encode(receipts[i]));
        }

        // 自底向上构建树
        while (leaves.length > 1) {
            bytes32[] memory nextLevel = new bytes32[]((leaves.length + 1) / 2);
            for (uint i = 0; i < leaves.length; i += 2) {
                if (i + 1 < leaves.length) {
                    nextLevel[i / 2] = keccak256(abi.encodePacked(
                        leaves[i] < leaves[i + 1] ? leaves[i] : leaves[i + 1],
                        leaves[i] < leaves[i + 1] ? leaves[i + 1] : leaves[i]
                    ));
                } else {
                    nextLevel[i / 2] = leaves[i];
                }
            }
            leaves = nextLevel;
        }

        return leaves[0];
    }
}
```

### D.4 部分争议解决

当批次中的单个收据被争议时：

```
批次争议解决：

┌────────────────────────────────────────────────────────────────┐
│                    1000 个收据的批次                             │
│                    总计: 10,000 USDC                            │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│  │ 无争议          │  │ 争议 #42        │  │ 争议 #789      │  │
│  │ 998 个收据      │  │ 50 USDC         │  │ 30 USDC        │  │
│  │ 9,920 USDC      │  │ → 仲裁          │  │ → 仲裁         │  │
│  └────────┬────────┘  └────────┬────────┘  └───────┬────────┘  │
│           │                    │                    │           │
│           ▼                    ▼                    ▼           │
│   正常结算              等待仲裁              等待仲裁          │
│   挑战期后                                        │           │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

| 场景 | 操作 |
|------|------|
| 无争议 | 挑战期后整个批次结算 |
| 单个收据争议 | 争议金额保留；其余结算 |
| 多个收据争议 | 每个争议金额单独保留 |
| >10% 批次争议 | 延长挑战期（2 倍） |
| Provider 败诉 | 从质押池罚没 |

---

## 附录 E：Gas 成本分析

### E.1 各操作 Gas 估算

基于 Base (L2) 的 Gas 成本。估算假设典型的合约大小和存储模式。

| 操作 | Gas（估算） | 成本 @ 0.01 gwei | 备注 |
|------|------------|------------------|------|
| **Deposit** | 65,000 | ~$0.001 | ERC20 转账 + 存储 |
| **Withdraw** | 45,000 | ~$0.001 | ERC20 转账 + 存储更新 |
| **Single Settlement** | 120,000 | ~$0.002 | 完整结算流程 |
| **Batch Submit (1000 receipts)** | 150,000 | ~$0.002 | 仅 Merkle 根 |
| **Batch Finalize** | 80,000 | ~$0.001 | 无争议 |
| **Open Dispute** | 180,000 | ~$0.003 | 创建争议 + 保证金 |
| **Submit Evidence** | 95,000 | ~$0.002 | 存储证据哈希 |
| **Submit Decision** | 140,000 | ~$0.002 | 仲裁者裁决 |
| **Force Finalize** | 160,000 | ~$0.003 | 超时解决 |
| **Stake** | 70,000 | ~$0.001 | Provider 质押 |
| **Unstake Request** | 55,000 | ~$0.001 | 开始冷却期 |
| **Slash** | 180,000 | ~$0.003 | 罚没 + 分配 |
| **Register Provider** | 200,000 | ~$0.003 | 完整注册 |
| **Register Watcher** | 150,000 | ~$0.002 | Watcher 注册 |

### E.2 成本对比：单个 vs 批量结算

```
成本分析: 1000 次 API 调用 @ $0.01 每次 = $10 总计

单个结算（每次调用）:
  - 每次结算 Gas: 120,000
  - 总 Gas: 120,000 × 1000 = 120,000,000
  - 成本 @ 0.01 gwei: ~$2.00
  - 开销: 交易价值的 20%

批量结算:
  - 批次提交: 150,000
  - 批次完成: 80,000
  - 总 Gas: 230,000
  - 成本 @ 0.01 gwei: ~$0.004
  - 开销: 交易价值的 0.04%

节省: 99.8%
```

### E.3 盈亏平衡分析

何时单个结算比批量结算更有意义？

```
单个结算成本: C_single = gas_single × gasPrice
批量结算成本: C_batch = (gas_submit + gas_finalize) / N

盈亏平衡: C_single = C_batch
N = (gas_submit + gas_finalize) / gas_single
N = 230,000 / 120,000
N ≈ 2
```

**建议**: 对于 > 2 个收据，批量结算更具成本效益。

### E.4 争议成本分析

完整争议解决成本：

| 参与方 | 操作 | Gas | 成本 |
|--------|------|-----|------|
| **Challenger** | 发起争议 | 180,000 | ~$0.003 |
| **Challenger** | 提交保证金 | （已包含） | - |
| **Provider** | 提交保证金 | 65,000 | ~$0.001 |
| **Challenger** | 提交证据 | 95,000 | ~$0.002 |
| **Provider** | 提交证据 | 95,000 | ~$0.002 |
| **Arbitrator** | 提交裁决 | 140,000 | ~$0.002 |
| **System** | 执行裁决 | 160,000 | ~$0.003 |
| **总计** | | 735,000 | ~$0.013 |

**注意**: 保证金金额（非 Gas）是争议的主要成本。在 L2 上 Gas 可忽略不计。

### E.5 扩展预测

| 每日交易量 | 结算策略 | 估算每日 Gas 成本 |
|------------|----------|-------------------|
| 1,000 笔 | 单个 | ~$2.00 |
| 1,000 笔 | 批量（每小时） | ~$0.10 |
| 10,000 笔 | 单个 | ~$20.00 |
| 10,000 笔 | 批量（每小时） | ~$0.60 |
| 100,000 笔 | 单个 | ~$200.00 |
| 100,000 笔 | 批量（每小时） | ~$5.80 |
| 1,000,000 笔 | 单个 | ~$2,000.00 |
| 1,000,000 笔 | 批量（每小时） | ~$58.00 |

### E.6 优化策略

| 策略 | Gas 节省 | 权衡 |
|------|----------|------|
| **批量结算** | 95-99% | 延迟最终性 |
| **Merkle 证明** | 争议时 90%+ | 复杂性 |
| **Calldata 压缩** | 20-40% | 链下处理 |
| **存储打包** | 10-30% | 代码复杂性 |
| **EIP-4844 blobs** | 数据 80%+ | Base 支持待定 |

### E.7 L2 vs L1 成本对比

| 链 | 平均 Gas 价格 | 结算成本 | 备注 |
|------|--------------|----------|------|
| **Base** | 0.001-0.01 gwei | ~$0.002 | 主要部署目标 |
| **Arbitrum** | 0.01-0.1 gwei | ~$0.01 | 替代 L2 |
| **Optimism** | 0.001-0.01 gwei | ~$0.002 | 替代 L2 |
| **Ethereum L1** | 20-100 gwei | ~$5-25 | 不推荐 |

**建议**: 部署在 Base 上，相比 L1 成本降低 1000 倍。

---

## 附录 F：术语表

| 术语 | 定义 |
|------|------|
| **Payer** | 服务购买方，预付资金 |
| **Provider** | 服务提供方，执行服务并提交收据 |
| **ServiceTx** | 服务交易，经 EntryPoint 识别的结构化交易 |
| **UsageReceipt** | 服务收据，Provider 签署的执行证明 |
| **Challenge Window** | 挑战期，结算前等待争议的时间窗口 |
| **Bond** | 保证金，争议双方需抵押的资金 |
| **Slash** | 罚没，违规方损失的质押 |
| **CEE** | Credit Execution Environment，信用执行环境 |
| **TEE** | Trusted Execution Environment，可信执行环境 |
| **ZK** | Zero Knowledge，零知识证明 |
| **Watcher** | 观察者，监督结算并可发起挑战 |
| **Facilitator** | x402 中的结算协调者（TrustApp 中被智能合约替代） |
| **arbMode** | 仲裁模式，决定争议如何裁决 |
| **defaultOutcome** | 默认裁决结果，超时时自动适用 |
| **settlementId** | 结算主键，唯一标识一笔结算，用于所有 settle/finalize/dispute 操作 |
| **receiptId** | 收据唯一标识符，用于防止收据重放 |
| **requestHash** | ServiceRequest 的内容摘要，仅作为业务关联索引，非结算主键 |
| **ConfirmService** | 快速路径确认消息，Payer 签署后可跳过挑战期即时结算 |

---

**文档版本**: v0.4.1
**最后更新**: 2025年12月
**变更记录**:
- v0.1: 初始草案
- v0.2: 增加仲裁流程细节
- v0.3: 整合安全模型、博弈分析、x402 迁移路径、确定性时间、Sybil 防护
- v0.4: 扩展 Oracle 仲裁模式、添加 ProviderRegistry 和 Watcher 网络规范、完善接口定义、添加 Gas 成本分析和 Merkle 批量结算
- v0.4.1: 修正章节编号、补充术语表、增加 MEV/跨链/ConfirmService 安全说明、添加合约代码免责声明

---

*TrustApp Protocol - Trust as a Service for the Decentralized Service Economy*

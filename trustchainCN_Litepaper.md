# TrustChain：面向去中心化服务经济的信任即服务 (TaaS) 协议

**状态**: 草案 / 提案  
**版本**: v0.3  
**日期**: 2025年12月  
**作者**: 0xwilsonwu（联合讨论稿）  
**部署目标**: Base（EVM 兼容）

---

## 摘要

区块链擅长价值转移，但在"服务交换"层面仍存在关键缺口：链上只能证明付款已发生，却无法保证服务已交付。现有方案往往依赖声誉或中心化仲裁，无法在高价值、一次性或跨地域的服务交易中提供可靠保障。

TrustChain 提出一种不修改底层共识的**应用层信任协议**，通过标准化的交易类型、链下收据与链上仲裁，建立"可验证的服务结算"。本协议以 Base 为结算层，使用 EIP-712 类型化数据、可插拔仲裁模块与经济惩罚机制，让服务交付与资金结算在不可信环境下实现可扩展的可信协作。

### 核心创新

| 创新点 | 描述 |
|--------|------|
| **可识别的服务交易类型** | 服务类交易通过 EntryPoint 进入协议层，与普通链上转账清晰区分 |
| **标准化收据模型** | 服务调用通过 ServiceRequest 和 UsageReceipt 表示，可批量结算与可争议 |
| **可插拔仲裁框架** | 委员会、法院、TEE、ZK、预言机均可作为仲裁模块，按策略接入 |
| **经济对齐机制** | 保证金、挑战窗口与惩罚规则为诚实行为提供激励 |
| **仅预付模式** | 所有服务调用在执行前完成锁仓或余额预存，消除"付款跑路"风险 |
| **乐观结算** | 在足额质押的前提下允许服务方提前回款，通过挑战与罚没保护安全性 |

本白皮书完整描述 TrustChain 的协议架构、交易与收据格式、仲裁流程、激励模型与安全边界，并提供面向 API 计费与 SaaS 替代的落地路径。

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
- [D. 术语表](#附录-d-术语表)

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

### 1.4 TrustChain 的差异化定位

| 维度 | x402 | ERC-8004 | TrustChain |
|------|------|----------|------------|
| **核心问题** | 如何付款 | 如何发现/评价 | 如何验证交付并强制执行 |
| **信任模型** | 信任 Facilitator | 信任声誉历史 | 信任博弈机制（质押+惩罚） |
| **资金流** | 即时转移 | 无 | 托管 → 挑战期 → 结算 |
| **交付验证** | 无 | 事后反馈（软） | 收据 + 可插拔证明（硬） |
| **争议解决** | 无 | 无 | 链上仲裁 + 经济罚没 |
| **适用场景** | 即时资源访问 | Agent 发现与筛选 | 复杂服务结算 |
| **惩罚机制** | 无 | 声誉下降 | 质押罚没 + 资金扣押 |
| **可组合性** | HTTP 层 | 身份层 | 结算层（可与前两者集成） |

### 1.5 TrustChain 的核心主张

**主张 1：信任不应依赖声誉，应依赖经济博弈**

声誉是概率性威慑，质押是确定性约束。TrustChain 要求：
- Provider 必须质押才能提供高价值服务
- 作恶的经济损失 > 作恶的经济收益
- 诚实行为是纳什均衡

**主张 2：交付验证必须链上可执行**

"可验证"不等于"可执行"。TrustChain 确保：
- 验证结果直接触发资金流动
- 争议裁决自动执行（无需人工介入链下）
- 所有状态转换链上可审计

**主张 3：协议应覆盖完整服务生命周期**

x402 只管"付款"，ERC-8004 只管"发现"。TrustChain 管：
- 服务请求（ServiceRequest）
- 服务执行（链下，但有收据）
- 服务结算（Settlement）
- 服务争议（Arbitration）
- 服务惩罚（Slash）

### 1.6 设计目标

TrustChain 的目标是：

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
│                        TrustChain Protocol                       │
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

TrustChain 不是新的 L1/L2，而是在 Base 之上构建的**信用执行环境 (Credit Execution Environment, CEE)**。CEE 只处理符合 TaaS 协议的交易，并为其赋予"信用状态"（可仲裁、可罚没、可结算）。

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

`maxAmount` 必须不超过付款方已锁定的预付余额。服务方应在链下执行前检查可用余额与授权额度。

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

为防止垃圾交易与伪造请求，TrustChain 对服务类交易引入"显性化"约束：

| 约束 | 描述 |
|------|------|
| **EIP-712 深度绑定** | 所有 ServiceTx 必须使用固定的 Domain Separator（name=TrustChainEntryPoint、version=1、chainId 与 verifyingContract） |
| **Magic Bytes 前缀** | 在 ServiceRequest 的签名数据中强制加入 `TRUST_V1_` 前缀，避免与普通签名混淆 |
| **SDK 约束** | 官方 SDK（JS/Go/Rust）输出标准化结构与前缀，Provider 在链下只接受由 SDK 生成的请求 |
| **Sequencer 标签（展望）** | 与 Base 等 L2 合作，将符合协议格式的交易在资源池阶段打上优先标签 |

这些约束提高协议可识别性与可审计性，并为 SDK 生态建立护城河。

---

## 4. 核心流程

### 4.1 预付托管模型（唯一支持）

TrustChain 只支持预付/锁仓模式。付款方需先在 Settlement 合约中存入余额或锁定额度，服务方仅能在该额度内结算。若余额不足，服务必须暂停或要求充值。

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

### 4.2 API 计费流程（典型场景）

1. Payer 预存余额或锁定额度
2. Payer 签署 ServiceRequest
3. Provider 完成 API 调用，生成 UsageReceipt
4. Provider 提交 ServiceTx(kind=SETTLE)（从已锁定余额中扣除）
5. 进入挑战期（challengeWindow）
6. 无争议则结算；有争议进入仲裁

### 4.3 电商交付流程（扩展场景）

- 以物流签名、第三方见证或预言机证明作为 responseHash 的来源
- 允许多阶段交付与分批结算

### 4.4 乐观结算（Withdraw-then-Slash）

为解决 API 服务的流动性痛点，协议允许在足额质押前提下提前回款：

1. Provider 提交 UsageReceipt 后，若其在 Stake Pool 的有效质押 ≥ `minStakeMultiple * amount`（如 2x），即可即时领取 `advanceRate`（如 80%）
2. 余下资金与等值质押进入锁定区，等待 challengeWindow 结束
3. 若挑战成功，触发"惩罚性罚没"：回收已预支资金，并从质押池扣罚；若挑战失败，释放剩余资金与质押

该机制使服务方近乎实时回款，同时由质押池提供安全背书。

#### 4.4.1 乐观结算的博弈论安全分析

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
        // 否则按 defaultOutcome（通常 50/50 split）
        _executeDecision(disputeId, Outcome.SPLIT);
    }
}
```

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
> 同一笔 ServiceRequest 的资金不会被结算两次。

形式化：
```
∀ req ∈ ServiceRequest:
  |{settle(tx) : tx.requestId = req.requestId}| ≤ 1
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

#### 为什么 TrustChain 不存在传统双花？

传统双花发生在**同一笔价值转移被两次确认**的情况下，通常需要独立账本或并行分叉才能实现。TrustChain 并不是 DAG 或个人链结构，而是部署在 Base 上的智能合约系统，所有资金托管在 Settlement 合约中，状态变更由 Base 共识保证原子性与全局顺序性。

```
┌─────────────────────────────────────────────────────────┐
│                    TrustChain 架构                       │
├─────────────────────────────────────────────────────────┤
│                                                          │
│   传统双花问题：                                          │
│   ┌─────┐                                                │
│   │Alice│ ──── 10 BTC ───► Bob                          │
│   │     │ ──── 10 BTC ───► Carol  (同一笔钱花两次)       │
│   └─────┘                                                │
│                                                          │
│   TrustChain 防护：                                       │
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
| **锁定机制** | 发起请求时锁定余额，未结算则解锁 | 高价值交易 |
| **即时验证** | Provider 在执行前调用链上 `checkAndLock()` | 中等价值 |
| **超额质押** | Provider 质押覆盖潜在损失 | 低价值高频 |

```solidity
// 推荐：请求时锁定（可作为 Settlement 或 RequestRegistry 的一部分）
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

攻击：若 TrustChain 扩展到多链，Payer 在 Base 和 Arbitrum 上用同一身份双花。

**防护方案**：

| 方案 | 描述 |
|------|------|
| **独立余额** | 每条链独立托管，互不影响 |
| **跨链锁定** | 通过桥接协议同步锁定状态 |
| **主链结算** | 所有最终结算汇总到主链 |

当前版本（v0.3）仅支持 Base 单链，跨链双花不适用；未来多链扩展需专门设计。

#### 双花防护总结

| 双花类型 | 风险等级 | 防护机制 | 状态 |
|----------|----------|----------|------|
| 传统双花（链上） | 无 | Base 共识 + 单一全局状态 | 已解决 |
| 收据双花 | 低 | receiptId 唯一性检查 | 已解决 |
| 请求双用 | 低 | requestId + provider 绑定 | 已解决 |
| 余额竞争 | 中 | 请求时锁定 / 链上检查锁定 | 已解决 |
| 跨链双花 | N/A | 当前单链，不适用 | 未来版本 |

### 7.5 安全性与风险总结

| 风险 | 防护措施 |
|------|----------|
| 伪造收据 | 通过签名校验与挑战机制防止 |
| 拒绝配合 | 保证金与默认裁决惩罚不配合方 |
| 仲裁腐败 | 可切换仲裁模块与阈值治理 |
| 重放攻击 | nonce + expiry + requestId |
| 数据可用性 | 证据以内容寻址存储 |
| 付款跑路 | 仅支持预付锁仓，批量结算只能消费已锁定余额 |
| 乐观结算风险 | 依赖质押充足与挑战活跃度，需明确 minStakeMultiple 与 advanceRate 上限 |
| 仲裁者离线 | 超时自动裁决 + forceFinalize() |

### 7.6 协议能力边界

TrustChain 诚实地划定其能力边界：

#### TrustChain 能解决的问题

| 场景 | 验证方式 |
|------|----------|
| Provider 收款后完全不执行服务 | 无收据 → Payer 胜诉 |
| Provider 提交虚假的计量数据 | TEE/ZK 验证 |
| 明确的交付物争议（有客观标准） | 证据比对 |

#### TrustChain 难以解决的问题

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

对于现有 x402 用户，TrustChain 提供兼容适配，允许渐进式迁移。

### 10.1 HTTP Header 扩展

**原 x402**：
```http
GET /resource HTTP/1.1
X-PAYMENT: <EIP-3009 signature>
```

**TrustChain 增强**：
```http
GET /resource HTTP/1.1
X-PAYMENT: <EIP-3009 signature>
X-TRUSTCHAIN-REQUEST: <ServiceRequest hash>
X-TRUSTCHAIN-POLICY: <policyId>
X-TRUSTCHAIN-RECEIPT: <previous receipt hash>  // 可选，用于链式验证
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
│                  TrustChain          TrustChain          TrustChain │
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
| **阶段 1** | + TrustChain Receipt | 可追溯 | 需要审计轨迹 |
| **阶段 2** | + TrustChain Escrow | 资金安全 | 中等价值交易 |
| **阶段 3** | 完整 TrustChain | 完整博弈保护 | 高价值/不信任场景 |

### 10.3 协议握手序列

```
┌─────────┐                    ┌─────────┐                    ┌─────────────┐
│  Client │                    │  Server │                    │ TrustChain  │
└────┬────┘                    └────┬────┘                    └──────┬──────┘
     │                              │                                │
     │  GET /resource               │                                │
     │─────────────────────────────►│                                │
     │                              │                                │
     │  402 Payment Required        │                                │
     │  + X-TRUSTCHAIN-SUPPORTED    │                                │
     │◄─────────────────────────────│                                │
     │                              │                                │
     │  (If TrustChain mode)        │                                │
     │  deposit(amount)             │                                │
     │───────────────────────────────────────────────────────────────►
     │                              │                                │
     │  GET /resource               │                                │
     │  + X-PAYMENT                 │                                │
     │  + X-TRUSTCHAIN-REQUEST      │                                │
     │─────────────────────────────►│                                │
     │                              │                                │
     │                              │  verify request                │
     │                              │───────────────────────────────►│
     │                              │                                │
     │                              │  execute service               │
     │                              │                                │
     │  200 OK + Response           │                                │
     │  + X-TRUSTCHAIN-RECEIPT      │                                │
     │◄─────────────────────────────│                                │
     │                              │                                │
     │                              │  settle(receipt)               │
     │                              │───────────────────────────────►│
     │                              │                                │
```

### 10.4 SDK 兼容层

```typescript
// TrustChain SDK 提供 x402 兼容层
import { TrustChainClient } from '@trustchain/sdk';

const client = new TrustChainClient({
  mode: 'x402-compat',  // 兼容模式
  fallback: 'pure-x402', // 若 Server 不支持 TrustChain，回退到纯 x402
  autoUpgrade: true,     // 自动检测并升级到 TrustChain
});

// 使用方式与 x402 相同
const response = await client.fetch('https://api.example.com/resource', {
  payment: {
    amount: '1000000', // 1 USDC
    token: 'USDC',
  },
  // TrustChain 增强选项（可选）
  trustchain: {
    escrow: true,         // 启用托管
    policy: 'default-api', // 仲裁策略
  },
});
```

### 10.5 迁移收益对比

| 维度 | 纯 x402 | TrustChain |
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

---

## 12. 结语

TrustChain 不是新的 L1，而是部署在现有链上的"信任协议"。它用可识别交易类型、可验证收据和可插拔仲裁机制，为 API 计费和服务经济提供一个可执行、可扩展的信任基础。

与 x402 的"如何付款"和 ERC-8004 的"如何信任"不同，TrustChain 回答的是"如何验证交付并强制执行"。三者可以互补：

- x402 作为支付触发层
- ERC-8004 作为身份发现层
- TrustChain 作为结算保障层

随着模块化验证与仲裁工具成熟，TrustChain 将成为连接链上资金与链下服务的核心桥梁。

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
┌─────────────────────────────────────────────────────────┐
│                    External Data Source                  │
│  (物流 API、IoT 传感器、第三方见证...)                    │
└─────────────────────────────────────────────────────────┘
                           │
                    ┌──────▼──────┐
                    │   Oracle    │ (Chainlink / API3)
                    │   Network   │
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

#### 适用场景

- 物流交付（快递签收确认）
- IoT 服务（传感器数据验证）
- 外部 API 依赖（第三方服务状态）

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

### B.1 数据结构定义

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

library TrustChainTypes {
    
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
    
    bytes8 public constant MAGIC_PREFIX = "TRUST_V1";
    
    address public immutable settlement;
    address public immutable arbitration;
    address public immutable registry;
    
    mapping(address => uint256) public nonces;
    
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
    ) EIP712("TrustChainEntryPoint", "1") {
        settlement = _settlement;
        arbitration = _arbitration;
        registry = _registry;
    }
    
    function execute(
        TrustChainTypes.ServiceTx calldata tx_,
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
        if (tx_.kind == uint8(TrustChainTypes.TxKind.PAY_ESCROW)) {
            ISettlement(settlement).deposit(tx_.payer, tx_.token, tx_.amount);
        } else if (tx_.kind == uint8(TrustChainTypes.TxKind.SETTLE)) {
            ISettlement(settlement).settle(
                tx_.payer, tx_.provider, tx_.token, tx_.amount, tx_.requestHash
            );
        }
        // ... other kinds
        
        emit ServiceTxExecuted(
            tx_.requestHash, tx_.kind, tx_.payer, tx_.provider, tx_.amount
        );
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
    mapping(address => mapping(address => TrustChainTypes.EscrowAccount)) public escrows;
    
    // requestHash => PendingSettlement
    mapping(bytes32 => PendingSettlement) public pendingSettlements;
    
    struct PendingSettlement {
        address payer;
        address provider;
        address token;
        uint256 amount;
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
    ) external nonReentrant {
        require(escrows[payer][token].balance >= amount, "Insufficient");
        
        escrows[payer][token].balance -= amount;
        escrows[payer][token].locked += amount;
        
        pendingSettlements[requestHash] = PendingSettlement({
            payer: payer,
            provider: provider,
            token: token,
            amount: amount,
            settleTime: uint64(block.timestamp),
            finalized: false
        });
    }
    
    function finalize(bytes32 requestHash) external nonReentrant {
        PendingSettlement storage ps = pendingSettlements[requestHash];
        require(!ps.finalized, "Already finalized");
        
        // Check challenge window passed
        // Check no active dispute
        
        uint256 fee = (ps.amount * protocolFeeBps) / 10000;
        uint256 providerAmount = ps.amount - fee;
        
        escrows[ps.payer][ps.token].locked -= ps.amount;
        
        IERC20(ps.token).safeTransfer(ps.provider, providerAmount);
        
        ps.finalized = true;
    }
    
    function advanceWithdraw(bytes32 requestHash) external nonReentrant {
        PendingSettlement storage ps = pendingSettlements[requestHash];
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

### B.4 部署脚本

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Script.sol";

contract DeployTrustChain is Script {
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

## 附录 D：术语表

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
| **Facilitator** | x402 中的结算协调者（TrustChain 中被智能合约替代） |
| **arbMode** | 仲裁模式，决定争议如何裁决 |
| **defaultOutcome** | 默认裁决结果，超时时自动适用 |

---

**文档版本**: v0.3  
**最后更新**: 2025年12月  
**变更记录**:
- v0.1: 初始草案
- v0.2: 增加仲裁流程细节
- v0.3: 整合安全模型、博弈分析、x402 迁移路径、确定性时间、Sybil 防护

---

*TrustChain Protocol - Trust as a Service for the Decentralized Service Economy*

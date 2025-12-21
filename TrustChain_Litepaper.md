# TrustApp: Trust as a Service (TaaS) Protocol for the Decentralized Service Economy

**Status**: Draft / Proposal
**Version**: v0.4.1
**Date**: December 2025  
**Author**: 0xwilsonwu (co-authored draft)  
**Deployment Target**: Base (EVM compatible)

---

## Abstract

Blockchains excel at value transfer, but they still fail at service exchange: the chain can prove payment occurred, yet cannot guarantee service was delivered. Existing approaches often rely on reputation or centralized arbitration, which is insufficient for high-value, one-off, or cross-border service transactions.

TrustApp proposes an application-layer trust protocol that does not modify base-layer consensus. It introduces standardized service transaction types, off-chain receipts, and on-chain arbitration to create verifiable service settlement. Built on Base as the settlement layer, TrustApp uses EIP-712 typed data, pluggable arbitration modules, and economic penalties to make service delivery and payment settlement enforceable in untrusted environments.

### Core Innovations

| Innovation | Description |
|-----------|-------------|
| **Recognizable service transaction types** | Service payments go through EntryPoint, clearly separated from normal transfers |
| **Standardized receipt model** | Service calls use ServiceRequest and UsageReceipt; support batching and disputes |
| **Pluggable arbitration framework** | Committees, courts, TEE, ZK, oracles can all serve as arbitrators |
| **Economic alignment** | Bonds, challenge windows, and slashing incentivize honest behavior |
| **Prepaid-only model** | Services are paid from locked or pre-deposited funds, preventing non-payment |
| **Optimistic settlement** | Providers can withdraw early with sufficient stake; challenges and slashing preserve safety |

This paper describes TrustApp's architecture, transaction and receipt formats, arbitration flow, incentive model, security boundaries, and a migration path for x402-style payments.

---

## Table of Contents

1. [Background and Problem](#1-background-and-problem)
2. [System Overview](#2-system-overview)
3. [Transaction Types and Data Structures](#3-transaction-types-and-data-structures)
4. [Core Flows](#4-core-flows)
5. [Arbitration Flow and State Machine](#5-arbitration-flow-and-state-machine)
6. [Economic Model and Incentives](#6-economic-model-and-incentives)
7. [Security Model and Trust Assumptions](#7-security-model-and-trust-assumptions)
8. [Governance and Upgrades](#8-governance-and-upgrades)
   - [8.4.5 Token Value and Platform Adoption Flywheel](#845-token-value-and-platform-adoption-flywheel)
   - [8.4.6 Entrepreneur Benefits from Token Appreciation](#846-entrepreneur-benefits-from-token-appreciation)
9. [Typical Applications](#9-typical-applications)
   - [9.5 Decentralized Alipay for E-commerce Platforms (B2B)](#95-decentralized-alipay-for-e-commerce-platforms-b2b)
   - [9.6 Agent-to-Agent Economy (A2A Transactions)](#96-agent-to-agent-economy-a2a-transactions)
   - [9.7 Trust as a Service (TaaS) for Entrepreneurs](#97-trust-as-a-service-taas-for-entrepreneurs)
10. [x402 Migration Path and Standalone Deployment](#10-x402-migration-path-and-standalone-deployment)
    - [10.0 TrustApp as a Standalone Protocol](#100-trustapp-as-a-standalone-protocol)
11. [Roadmap](#11-roadmap)
12. [Conclusion](#12-conclusion)

**Appendices**
- [A. Arbitration Module Technical Specification](#appendix-a-arbitration-module-technical-specification)
- [B. Core Contract Reference Implementation](#appendix-b-core-contract-reference-implementation)
- [C. Data Availability Scheme](#appendix-c-data-availability-scheme)
- [D. Glossary](#appendix-d-glossary)

---

## 1. Background and Problem

### 1.1 Value Transfer Is Not Service Exchange

In a simple transfer where Alice sends funds to Bob, the blockchain only proves that money moved. For API calls, content delivery, or AI inference, both parties may be untrusted and the settlement must verify that service was delivered. The chain lacks a general guarantee for this.

### 1.2 Limits of Reputation and Centralized Arbitration

| Problem | Description |
|---------|-------------|
| **Reputation is unenforceable** | A high reputation cannot prevent a one-off exit scam |
| **Centralized arbitration is opaque** | Rules are unclear, costs are high, and cross-border enforcement is weak |
| **Off-chain verification is hard** | Smart contracts cannot verify off-chain events without pluggable proof and arbitration |

Web2 already uses escrow + arbitration. For example, Alipay and similar platforms provide dispute resolution and can arbitrate delivery issues. This validates the business need, but it also highlights the dependency: the rules, evidence standards, and outcomes are controlled by a single platform and limited by jurisdiction. TrustApp brings the same concept to a neutral, verifiable settlement layer so procedures are transparent, composable, and enforceable across borders without trusting one operator.

### 1.3 Why Existing Approaches Are Insufficient

#### Structural limits of x402

x402 (HTTP 402 based) provides an elegant request-pay-response flow, but has fundamental gaps:

**Gap 1: No delivery verification**

```
x402 flow:
Client -> Server: GET /resource
Server -> Client: 402 Payment Required + PaymentRequirements
Client -> Server: GET /resource + X-PAYMENT header (signed authorization)
Server: verify signature -> settle -> return resource
```

Problem: after payment, the server decides what to return. The client cannot prove:
- Whether the resource was delivered in full
- Whether quality matched the agreement
- Whether usage metrics are accurate

**Gap 2: Facilitator is centralized**

x402 `verify` and `settle` depend on a facilitator service:

```
Server -> Facilitator: /verify (check signature)
Server -> Facilitator: /settle (execute on-chain settlement)
```

Risks:
- Collusion with provider
- Downtime breaks the payment flow
- Censorship of specific transactions

**Gap 3: Only supports immediate delivery**

x402 assumes "pay then instantly respond", which fails for:
- Asynchronous services (AI training, rendering jobs)
- Multi-stage delivery (logistics, milestones)
- Ongoing services (subscriptions, API quotas)

**Gap 4: No dispute path**

If the client disputes delivery or quality, x402 has no recourse; the payment is irreversible.

#### ERC-8004 and the soft-constraint trap

ERC-8004 defines identity and reputation registries for AI agents, but:

**Gap 1: Reputation is not enforceable**

```solidity
struct Feedback {
    uint256 agentId;
    address clientAddress;
    uint8 score;        // 1-100
    bytes32 dataHash;   // feedback details
}
```

Reputation is historical, not a future guarantee:
- A high-score agent can still exit-scam
- One-off high-value trades are not deterred
- Reputation is a soft penalty without economic force

**Gap 2: Feedback is manipulable**

Payment-weighted feedback can be gamed by:
- Self-dealing
- Buying reviews
- Malicious downvotes

No staking or slashing means manipulation is cheap.

**Gap 3: Decoupled from settlement**

ERC-8004 explicitly states:

> "Payments are orthogonal to this protocol"

So:
- Reputation is not atomically tied to payment
- Negative feedback cannot trigger refunds
- Positive feedback does not unlock escrowed funds

**Gap 4: Verification layer is too abstract**

The Validation Registry supports multiple verifier types but:
- No standard interface
- No clear effect on settlement
- Unclear economic consequences of failure

### 1.4 TrustApp Positioning

| Dimension | x402 | ERC-8004 | TrustApp |
|----------|------|----------|------------|
| **Core question** | How to pay | How to discover and rate | How to verify delivery and enforce |
| **Trust model** | Trust facilitator | Trust reputation history | Trust economic game (stake + slashing) |
| **Fund flow** | Instant transfer | None | Escrow -> challenge -> settlement |
| **Delivery verification** | None | Post-hoc feedback | Receipts + pluggable proofs |
| **Dispute resolution** | None | None | On-chain arbitration + slashing |
| **Best fit** | Instant access | Agent discovery | Complex service settlement |
| **Penalty** | None | Reputation loss | Slashing + fund holdback |
| **Composability** | HTTP layer | Identity layer | Settlement layer |

### 1.5 Core Claims

**Claim 1: Trust should rely on economic games, not reputation.**
- Providers must stake for high-value services.
- The cost of cheating must exceed the gain.
- Honest behavior becomes Nash equilibrium.

**Claim 2: Delivery verification must be enforceable on-chain.**
- Verification results directly move funds.
- Arbitration is executed by the protocol, not off-chain.
- State transitions are fully auditable.

**Claim 3: The protocol should cover the full service lifecycle.**
- Service request
- Service execution (off-chain with receipts)
- Service settlement
- Service dispute
- Service slashing

### 1.6 Design Goals

TrustApp aims to:
- Avoid base-layer consensus changes
- Make service transactions identifiable
- Enable verifiable and arbitrable settlement
- Be modular and extensible
- Target API billing and SaaS replacement

---

## 2. System Overview

### 2.1 Roles

| Role | Responsibility |
|------|----------------|
| **Payer** | Initiates requests and pays for services |
| **Provider** | Executes services and submits receipts |
| **Relayer** | Optional, submits transactions on behalf of others |
| **Arbitrator** | Executes dispute logic (committee/court/TEE/ZK) |
| **Watcher** | Monitors, challenges, or supplies evidence |

### 2.2 Core Modules

```
+------------------------------------------------------------------+
|                        TrustApp Protocol                       |
+------------------------------------------------------------------+
|  +------------+    +------------+    +------------+              |
|  | EntryPoint |--> |  Registry  | <--|  StakePool |              |
|  +------------+    +------------+    +------------+              |
|        |                                     |                    |
|        |        +------------+               |                    |
|        +------> | Settlement | <-------------+                    |
|                 +------------+                                    |
|                        |                                          |
|                 +-------------+                                   |
|                 | Arbitration |                                   |
|                 +-------------+                                   |
|                        |                                          |
|        +----------------+----------------+                        |
|        |                |                |                        |
|   +---------+       +---------+      +---------+                  |
|   |Committee|       |  TEE    |      |   ZK    |                  |
|   |Arbiter  |       |Arbiter  |      |Arbiter  |                  |
|   +---------+       +---------+      +---------+                  |
+------------------------------------------------------------------+
```

| Module | Function |
|--------|----------|
| **EntryPoint** | Unified gateway, recognizes service transactions |
| **Registry** | Module registration, versioning, policy management |
| **Settlement** | Escrow and settlement logic |
| **Arbitration** | Dispute handling and decision execution |
| **Stake and Slash** | Bonding and slashing mechanisms |

### 2.3 Pluggable Modules as ZIP Packages

Modules can be packaged as ZIPs and registered:

```
module.zip/
+-- manifest.json      # module metadata and capabilities
+-- contracts/         # on-chain adapters
+-- offchain/          # executors, aggregators, proof generators
+-- spec/              # receipt structure and evidence requirements
```

### 2.4 Credit Execution Environment (CEE)

TrustApp is not a new L1/L2. It is a Credit Execution Environment built on Base. CEE only processes transactions that follow the TaaS protocol and assigns them credit states (arbitrable, slashable, settleable).

This is a "subtraction" approach: move service verification off the consensus path while using Base as the settlement and security anchor.

---

## 3. Transaction Types and Data Structures

### 3.1 ServiceTx (Service Transaction)

Service transactions are signed EIP-712 typed data and executed via EntryPoint:

```solidity
struct ServiceTx {
    uint8   kind;        // transaction type
    bytes32 moduleId;    // module id
    address payer;
    address provider;
    address token;       // 0x0 for native
    uint256 amount;      // payable/settleable amount
    uint256 nonce;
    uint64  deadline;
    bytes32 requestHash; // ServiceRequest hash
    bytes32 policyId;    // ArbitrationPolicy hash
}
```

**Transaction kinds**:

| Kind | Value | Description |
|------|-------|-------------|
| PAY_DIRECT | 0x01 | Direct pay (identifiable service pay) |
| PAY_ESCROW | 0x02 | Escrow lock |
| SETTLE | 0x03 | Settlement with receipt |
| REFUND | 0x04 | Refund |
| ARB_OPEN | 0x10 | Open dispute |
| ARB_EVIDENCE | 0x11 | Submit evidence |
| ARB_DECISION | 0x12 | Submit decision |

### 3.1.1 Settlement Key and Hashing Rules

To avoid implementation and audit ambiguity, TrustApp uses a consistent rule:

- **On-chain settlement primary key**: `settlementId` (unique, non-reusable; all settle/finalize/dispute/confirm use this key)
- **Business content hash**: `requestHash` (or `requestId`) is only for indexing/association, not a settlement primary key
- **Receipt uniqueness**: `receiptId` uniquely identifies a UsageReceipt and must be replay-protected on-chain

### 3.2 ServiceRequest

```solidity
struct ServiceRequest {
    bytes32 requestId;
    bytes32 moduleId;
    address payer;
    address provider;
    address token;
    uint256 maxAmount;
    uint64  expiry;
    bytes32 payloadHash; // request hash
}
```

`maxAmount` must not exceed the payer's locked balance. Default policy locks `maxAmount` on request creation; a lighter policy allows Providers to call on-chain `checkAndLock()` before execution.

### 3.3 UsageReceipt

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
    bytes32 aux;          // module-specific field
}
```

### 3.3.1 ConfirmService (Fast Path Confirmation)

Off-chain confirmation message for the fast path:

```solidity
struct ConfirmService {
    bytes32 settlementId;  // settlement primary key
    address payer;
    address provider;
    address token;
    uint256 amount;
    bytes32 receiptId;
    bytes32 requestHash;   // optional: association index
    bytes32 policyId;      // optional but strongly recommended
    uint8 rating;          // optional: QoS score
    uint64 deadline;       // expiry time
    uint256 nonce;         // replay protection (must be consumed on-chain)
}
```

ConfirmService is a **final authorization** for a specific settlement. The payer signs this EIP-712 message and the provider submits it via `settleWithConfirm(...)`. EntryPoint must verify deep binding (payer/provider/token/amount/receiptId/policyId), Settlement must consume the nonce/confirmHash, and the settlement must be neither disputed nor finalized. Once consumed on-chain, it is irreversible; revocation must go through dispute/arbitration.

#### MEV Protection

`settleWithConfirm` transactions in public mempool may be observed and exploited by MEV bots. Recommendations:

| Measure | Description |
|---------|-------------|
| **Private submission** | Submit via Base Sequencer private RPC or Flashbots Protect to avoid public mempool |
| **Commit-Reveal (optional)** | For high-value settlements, first submit `hash(confirm + salt)`, reveal in next block |
| **Provider submits directly** | Provider as tx sender has aligned incentives, reducing front-run motivation |

### 3.4 Batch Settlement (Optional)

```solidity
struct ReceiptBatch {
    bytes32 batchId;
    bytes32 merkleRoot;
    uint64  fromTime;
    uint64  toTime;
}
```

### 3.5 Explicitness and Anti-Forgery

To prevent spam and forgery, TrustApp adds explicit constraints:

| Constraint | Description |
|-----------|-------------|
| **EIP-712 binding** | All ServiceTx use fixed domain separator (name=TrustAppEntryPoint, version=1, chainId, verifyingContract) |
| **Magic prefix** | ServiceRequest signatures include `TRUST_V1_` prefix |
| **SDK enforcement** | Official SDKs output standardized structures and prefixes |
| **Sequencer tagging (future)** | L2 sequencers can tag compliant transactions for prioritization |

---

## 4. Core Flows

### 4.1 Prepaid Escrow Model (Only Supported Mode)

TrustApp only supports prepaid or locked funds. The payer must deposit or lock balances in Settlement. Providers can only settle within that limit.

```
Payer -> Settlement: deposit(token, amount)
Settlement -> Payer: balance locked
Payer -> Provider: ServiceRequest (off-chain)
Provider -> Settlement: settle(receipt)
Settlement -> Provider: finalize (after challenge window)
```

### 4.2 API Billing Flow (Typical)

1. Payer deposits or locks funds
2. Payer signs ServiceRequest
3. Provider executes API call and generates UsageReceipt
4. **Fast path**: payer signs ConfirmService -> provider calls `settleWithConfirm(...)` -> funds released immediately
   **or slow path**: provider submits ServiceTx(kind=SETTLE) -> challenge window -> finalization/arbitration

### 4.3 E-commerce Delivery Flow (Extended)

- Use logistics signature, third-party witness, or oracle data as responseHash
- Support staged delivery and partial settlement

### 4.4 Optimistic Settlement (Withdraw-Then-Slash)

To solve provider liquidity, the protocol allows early withdrawal if stake is sufficient:

1. Provider submits UsageReceipt and has stake >= minStakeMultiple * amount (e.g. 2x)
2. Provider can withdraw advanceRate (e.g. 80%) immediately
3. Remaining funds and equivalent stake are locked until challengeWindow ends
4. If a challenge succeeds, early withdrawal is clawed back and stake is slashed

#### 4.5.1 Game-Theoretic Safety Analysis

Let:
- Service amount: `A`
- Provider stake: `S` (require `S >= k * A`)
- Advance rate: `r` (e.g. 0.8)
- Slash multiple: `p` (e.g. 1.5)
- Challenge success probability: `q`
- Challenge cost: `c`

Cheating payoff (receive without service):
```
R_cheat = r * A * (1 - q)
```

Cheating cost (challenge success):
```
C_cheat = q * (r * A + p * A)
        = q * A * (r + p)
```

Honesty condition:
```
q * A * (r + p) > r * A * (1 - q)
q > r / (2r + p)
```

Example with r = 0.8, p = 1.5:
```
q > 0.8 / (1.6 + 1.5) = 0.8 / 3.1 ~= 0.258
```

**Stake multiple lower bound**
```
S >= p * A
k >= p
```

If p = 1.5, then k >= 1.5.

**Suggested configuration**

| Parameter | Suggested | Notes |
|-----------|-----------|-------|
| minStakeMultiple | 2.0 | safety buffer |
| advanceRate | 0.8 | 80% advance |
| slashMultiple | 1.5 | 150% slash |
| minChallengeProb | 0.3 | target network quality |

**Challenger incentives**
```
challengerReward > challengeCost
arbBps * slashedAmount > c
arbBps * p * A > c
```

If A = 1000 USDC, p = 1.5, c = 50:
```
arbBps > 50 / 1500 ~= 3.3%
```

Recommendation: arbBps >= 20%.

**Stake volatility risk**
```
S * priceToken >= p * A
```

Mitigations:
1. Use stablecoin staking
2. Dynamic stake requirements
3. Monitoring and forced top-ups

**Operational constraints**:
- Prefer stablecoin stake or apply haircuts for volatile assets
- Stake lock/unlock timing must align with `challengeWindow`
- `clawback/slash` trigger, window, and distribution must be explicit in interfaces and implementation

---

## 5. Arbitration Flow and State Machine

### 5.1 ArbitrationPolicy

```solidity
struct ArbitrationPolicy {
    // time windows
    uint64 challengeWindow;
    uint64 bondWindow;
    uint64 evidenceWindow;
    uint64 decisionWindow;

    // fees
    uint16 payerBondBps;
    uint16 providerBondBps;
    uint16 arbFeeBps;
    uint256 arbFeeFlat;

    // decision
    uint8  defaultOutcome;  // provider/payer/split
    uint8  arbMode;         // committee/court/TEE/ZK/oracle
    address bondToken;      // 0x0 = payment token
    uint256 minBond;
    uint256 maxBond;
    uint32 evidenceMask;
}
```

**Parameter selection**: different industries need different fee/bond/window settings. `arbFeeBps`, `challengeWindow`, `bondWindow`, and `minBond` should be configured per module; Policy is versioned, not a global constant.

### 5.2 State Machine

```
SETTLE_PENDING
  | challengeWindow expires, no dispute
  v
FINALIZED

SETTLE_PENDING --openDispute()--> DISPUTE_OPEN
                                   |
DISPUTE_OPEN ----------------------v
                                BONDING
                                   |
          +------------------------+------------------------+
          |                                                 |
          v                                                 v
   both bonds paid                                  bondWindow timeout
          |                                                 |
          v                                                 v
       EVIDENCE                                      EXPIRED (defaultOutcome)
          |                                                 |
          v                                                 |
   evidenceWindow ends                                      |
          |                                                 |
          v                                                 |
       DECISION                                             |
          |                                                 |
    +-----+-----+                                           |
    v           v                                           |
submitDecision()  decisionWindow timeout                    |
    |           |                                           |
    v           v                                           |
FINALIZED   EXPIRED (defaultOutcome)                        |
                |                                           |
                +-------------------------------------------+
                                   |
                                   v
                               FINALIZED
```

### 5.3 Arbitration Steps

| Phase | Description |
|-------|-------------|
| **OpenDispute** | Open dispute during challenge window |
| **Bonding** | Both sides submit bonds; missing side loses |
| **Evidence** | Submit evidence hashes and URIs |
| **Decision** | Arbitration module submits decision |
| **Finalize** | Execute settlement and penalties |

### 5.4 Finality Bounds

**Max delay without dispute**
```
T_finality = challengeWindow + blockConfirmations
           = 24h + ~3min (12 blocks on Base)
           ~= 24 hours 3 minutes
```

**Max dispute resolution time**
```
T_dispute_max = challengeWindow + bondWindow + evidenceWindow + decisionWindow
              = 24h + 48h + 72h + 48h
              = 192 hours (8 days)
```

**Probabilistic finality with n watchers**
```
P(caught) = 1 - (1-p)^n
```

Example: p = 0.8, n = 5:
```
P(caught) = 1 - 0.2^5 = 99.97%
```

Recommendation: for > $10k transactions, require at least 3 registered watchers.

**Finality summary**

| Scenario | Max time | Notes |
|----------|----------|-------|
| Normal settlement | 24h 3m | challengeWindow + confirmations |
| Dispute resolution | 8 days | maximum full flow |
| Force finalization | 15 days | includes grace period |

### 5.5 Force Finalization

If arbitrators are offline, disputes could stall. TrustApp adds layered timeouts:

| Phase | Timeout | Auto outcome |
|-------|---------|--------------|
| BONDING | bondWindow | missing side loses |
| EVIDENCE | evidenceWindow | move to DECISION |
| DECISION | decisionWindow | defaultOutcome applies |
| DECISION + grace | decisionWindow + 7d | anyone can call `forceFinalize()` |

```solidity
function forceFinalize(bytes32 disputeId) external {
    Dispute storage d = disputes[disputeId];
    require(d.state == DisputeState.DECISION, "Invalid state");
    require(
        block.timestamp > d.decisionDeadline + GRACE_PERIOD,
        "Grace period not passed"
    );

    if (d.providerEvidenceValid && !d.payerEvidenceValid) {
        _executeDecision(disputeId, Outcome.PROVIDER_WINS);
    } else if (d.payerEvidenceValid && !d.providerEvidenceValid) {
        _executeDecision(disputeId, Outcome.PAYER_WINS);
    } else {
        // Neither party submitted valid evidence → invalid dispute, liquidate to treasury
        _executeDecision(disputeId, Outcome.INVALID);
    }
}
```

**Invalid Dispute (INVALID) Handling**:

When neither party submits valid evidence, a simple 50/50 split would incentivize frivolous disputes. Instead:
- **liquidateBps** (e.g., 50%-100%) of the disputed amount is liquidated to the protocol treasury or arbitration fund
- Any remainder (if applicable) returned proportionally
- Both parties' bonds are fully slashed

This design penalizes "open dispute then do nothing" behavior and prevents abuse of the dispute mechanism.

#### defaultOutcome Arbitrage Risk

**Risk scenario**: If an ArbitrationPolicy has `defaultOutcome = PROVIDER_WINS`, a malicious Provider could DDoS arbitration nodes to force timeout and trigger the default outcome, gaining illegitimate profit.

**Countermeasures**:

| Measure | Description |
|---------|-------------|
| **No one-sided defaults** | `defaultOutcome` should only allow `INVALID` (both slashed) or `SPLIT`; prohibit `PROVIDER_WINS` / `PAYER_WINS` |
| **Multi-arbitrator redundancy** | Assign N arbitrators (e.g., 3/5) to each dispute; majority must respond; single-point DDoS ineffective |
| **Arbitrator liveness bond** | Arbitrators stake a bond; timeout without response triggers slashing, incentivizing uptime |
| **Fallback escalation** | On primary arbitrator timeout, auto-escalate to backup mode (e.g., Committee → DAO vote) |

**Recommended config**: `defaultOutcome = INVALID`, ensuring no timeout scenario benefits either party unilaterally.

**Guarantee**: Even if the arbitration layer fails, funds do not stay locked forever.

---

## 6. Economic Model and Incentives

### 6.1 Fee Structure

| Fee type | Description |
|----------|-------------|
| **Service Fee** | Provider revenue |
| **Protocol Fee** | Protocol maintenance fee (default 0.3%) |
| **Arbitration Fee** | Dispute cost |
| **Locked prepaid balance** | All spend must come from locked funds |

### 6.2 Bonds and Penalties

- Payer and provider both post bonds to deter abuse and non-delivery
- Arbitration outcome determines bond release or slashing

### 6.3 Honesty Incentives

- Fast settlement without dispute is cheapest
- Malicious actors face higher economic penalties

### 6.4 Stake Pool Distribution (Core TaaS Incentive)

When a provider is found in breach, slashed stake `S` is distributed:

```
S_payer     = S * payerBps
S_arb       = S * arbBps
S_treasury  = S * treasuryBps
S_burn      = S * burnBps
```

**Suggested ranges**:

| Recipient | Range | Purpose |
|-----------|-------|---------|
| Payer compensation | 50% - 60% | Incentivize challenges |
| Arbitration reward | 20% - 30% | Incentivize arbitrators |
| Treasury | 10% | Protocol sustainability |
| Burn | 10% | Deflationary mechanism |

### 6.5 Sybil Resistance Economics

Attack: an attacker creates N fake providers to game reputation or defraud.

**Defense: stake thresholds**

Each provider must stake `minStake` to settle.

**Sybil cost**:
```
C_sybil = N * minStake + N * registrationFee + opportunity_cost
```

**Attack gain**:
```
R_sybil = A (single high-value fraud)
```

**Security condition**:
```
N * minStake > A
```

Example: `minStake = 1000 USDC`, attacking 10,000 USDC requires:
```
N > 10,000 / 1,000 = 10 identities
Cost = 10 * 1000 = 10,000 USDC
```

**Suggested parameters**

| Parameter | Suggested | Notes |
|-----------|-----------|------|
| minStake | 0.5 * avgTxAmount | half of average |
| registrationFee | 10-50 USDC | increase creation cost |
| highValueMultiple | 3x | higher stake for large trades |

**Additional mitigations**

| Mechanism | Description |
|-----------|-------------|
| **Time-weighting** | New providers unlock limits gradually |
| **Counterparty diversity** | Repeated trades with few addresses reduce weight |
| **On-chain behavior analysis** | Detect batch registrations and pattern abuse |

---

## 7. Security Model and Trust Assumptions

### 7.1 Threat Model

| Attacker | Capabilities | Goal |
|----------|--------------|------|
| Malicious payer | Controls own keys; can open false disputes | Get service without paying |
| Malicious provider | Controls own keys; can forge receipts | Get paid without service |
| Malicious relayer | Can censor or delay tx; cannot forge signatures | MEV, denial |
| Malicious arbitrator | Depends on arb mode | Biased decisions, bribery |
| Malicious watcher | Can submit false challenges | Harassment, denial |
| Network attacker | Can observe or delay tx | Front-run, sandwich |

#### Out of Scope
- Base consensus attacks (assume L1/L2 security)
- Smart contract bugs (assume audited)
- Cryptographic breaks (assume ECDSA and Keccak are secure)

### 7.2 Trust Assumptions

**Base assumptions (all modes)**

| Assumption | Description | Impact if broken |
|-----------|-------------|-----------------|
| **TA-1: chain liveness** | Base produces blocks in reasonable time | disputes may timeout |
| **TA-2: chain security** | transactions are final after confirmations | settlements can be reverted |
| **TA-3: time sync** | timestamp drift < 15 minutes | challenge window can be bypassed |
| **TA-4: data availability** | evidence storage is accessible | default outcomes apply |

**Mode-specific assumptions**

Committee mode:
```
TA-C1: at least 2/3 committee members are honest
TA-C2: bribery cost exceeds gain
TA-C3: committee responds within decisionWindow
```

TEE mode:
```
TA-T1: TEE hardware has no exploitable side-channel
TA-T2: remote attestation is available and honest
TA-T3: enclave code is audited
```

ZK mode:
```
TA-Z1: ZK proving system is sound
TA-Z2: trusted setup (if any) is not compromised
TA-Z3: service logic is expressible as a circuit
```

Oracle mode:
```
TA-O1: at least 2/3 oracle nodes are honest
TA-O2: data sources are reliable
TA-O3: oracles respond within decisionWindow
```

### 7.3 Security Properties

**Property 1: Safety**

S1 - Payer protection:
If provider does not deliver, an honest payer loses at most dispute cost.

```
For all tx:
  If Provider.delivered(tx) = false
  and Payer.honest(tx) = true
  then Payer.loss(tx) <= disputeCost(tx)
```

S2 - Provider protection:
If provider delivers, an honest provider receives their revenue.

```
For all tx:
  If Provider.delivered(tx) = true
  and Provider.honest(tx) = true
  then Provider.revenue(tx) >= agreedAmount(tx) - protocolFee(tx)
```

S3 - No double payment:
```
For all settlement:
  count(finalize(s) where s.settlementId = settlement.settlementId) <= 1
```

**Property 2: Liveness**

L1 - Final settlement:
Any settlement reaches FINALIZED within maxSettlementTime.

```
maxSettlementTime = challengeWindow + bondWindow + evidenceWindow + decisionWindow + gracePeriod
```

L2 - Dispute can be opened within challengeWindow.

L3 - Funds are withdrawable after FINALIZED within withdrawalDelay.

**Property 3: Fairness**

F1 - Arbitration fairness:
With honest arbitrators, the decision matches facts with probability >= fairnessThreshold.

F2 - Incentive compatibility:
For rational actors, honest behavior is dominant.

```
For all participants:
  E[payoff(honest)] > E[payoff(dishonest)]
```

### 7.4 Double-Spend Analysis

Traditional double-spend requires two confirmations of the same value transfer. TrustApp runs on Base, with funds locked in a single Settlement contract. Base provides atomic global ordering.

```
TrustApp and double-spend (conceptual):

Traditional double-spend:
  Alice -> Bob and Alice -> Carol (same funds)

TrustApp:
  Payer -> Settlement contract -> Provider
  Base consensus ensures a single global state
```

**Variants and mitigations**

**Case 1: Receipt double-claim**

```solidity
mapping(bytes32 => bool) public settledReceipts;

function settle(UsageReceipt calldata receipt, ...) external {
    require(!settledReceipts[receipt.receiptId], "Receipt already settled");
    settledReceipts[receipt.receiptId] = true;
    // settlement logic
}
```

**Case 2: Request double-use**

```solidity
struct ServiceRequest {
    bytes32 requestId;
    address provider;
    // ...
}

require(request.provider == msg.sender, "Wrong provider");
```

**Case 3: Balance race**

Two providers both see the same balance off-chain.

Mitigations:

| Option | Description | Best for |
|--------|-------------|----------|
| **Default lock** | Lock funds when request is created | high-value |
| **Lightweight lock** | Provider calls `checkAndLock()` before execution | mid-value |
| **Over-stake** | Provider stake covers loss | low-value, high-frequency |

```solidity
function createRequest(bytes32 requestId, address token, uint256 amount) external {
    require(escrows[msg.sender][token].balance >= amount, "Insufficient");
    escrows[msg.sender][token].balance -= amount;
    escrows[msg.sender][token].locked += amount;

    pendingRequests[requestId] = PendingRequest({
        payer: msg.sender,
        amount: amount,
        lockedAt: block.timestamp
    });
}
```

**Case 4: Cross-chain double spend**

Not applicable in v0.4.1 (Base only). Multi-chain expansion preliminary design directions:

1. **Main chain anchoring**: All settlements finalize on Base as the main chain; other chains serve as execution layers
2. **Independent validation**: Each chain runs an independent TrustApp instance, syncing Provider stake status via cross-chain messaging (e.g., LayerZero, Chainlink CCIP)
3. **Shared stake pool**: Stake is concentrated on the main chain; other chains verify stake proofs via light clients

Detailed design will be provided in the multi-chain expansion phase (Roadmap Phase 5).

**Summary**

| Type | Risk | Mitigation | Status |
|------|------|------------|--------|
| On-chain double spend | None | Base global state | solved |
| Receipt double claim | Low | receiptId uniqueness | solved |
| Request double use | Low | provider binding | solved |
| Balance race | Medium | default lock / optional check | solved |
| Cross-chain | N/A | out of scope | future |

### 7.5 Security and Risk Summary

| Risk | Mitigation |
|------|------------|
| Forged receipts | Signature checks and challenges |
| Non-cooperation | Bonds and default outcomes |
| Arbitrator corruption | Replaceable modules and governance |
| Replay attacks | nonce + deadline + on-chain consumption (nonce or confirmHash) |
| Data availability | content-addressed evidence |
| Payment default | prepaid-only model |
| Optimistic risk | stake requirements and challenge activity |
| Arbitrator offline | timeout + forceFinalize() |
| MEV attacks | Relayers may front-run settle transactions; recommend private mempool (e.g., Flashbots Protect) or Base private Sequencer channel |
| ConfirmService replay | Signature expires after deadline; nonce must be consumed on-chain; expired unused signatures are invalid |

### 7.6 Protocol Boundaries

**What TrustApp can solve**

| Scenario | Verification |
|----------|--------------|
| Provider takes payment but does nothing | No receipt -> payer wins |
| Provider submits false metering data | TEE/ZK proof |
| Objective delivery disputes | Evidence comparison |

**What TrustApp cannot solve**

| Scenario | Reason | Suggestion |
|----------|--------|-----------|
| Subjective quality disputes | No objective standard | committee arbitration |
| Real-time services without replay | Hard to verify later | TEE or trust assumptions |
| Private services with hidden I/O | Data cannot be revealed | ZK or MPC |

**Verifiability levels**

| Level | Description | Stake requirement |
|-------|-------------|------------------|
| Level 1 | Deterministic compute (ZK verifiable) | Low (1.5x) |
| Level 2 | TEE attested execution | Medium (2x) |
| Level 3 | Oracle confirmed delivery | Medium (2x) |
| Level 4 | Subjective quality (human arbitration) | High (3x) + reputation |

---

## 8. Governance and Upgrades

### 8.1 Registry Whitelist

- Modules and policies are reviewed before registration
- Registration includes version, audit reports, and risk tier

### 8.2 Versioned policyId

- Old policies are immutable
- Upgrades create new policyId
- On-chain upgrade trails are auditable

### 8.3 Emergency Pause

| Trigger | Executor | Scope |
|---------|----------|-------|
| Contract vulnerability | Multisig | global pause |
| Module failure | Module admin | single module pause |
| Arbitration attack | Governance vote | remove arbitrator |

### 8.4 Protocol Token ($TRUST)

TrustApp may introduce a native token $TRUST for incentive alignment and governance of the protocol security layer.

#### 8.4.1 Design Principles

**Dual-Layer Separation**: Service settlement and protocol security use different assets.

| Layer | Asset | Purpose |
|-------|-------|---------|
| Settlement Layer | Stablecoin (USDC) | Payer escrow, Provider collateral, service pricing |
| Security Layer | $TRUST | Arbitrator staking, governance, value capture |

This separation ensures service prices are unaffected by token volatility while providing economic incentives for protocol security.

#### 8.4.2 Token Utility

**1. Arbitrator Staking**

Arbitrators must stake $TRUST to participate in dispute resolution:

| Parameter | Description |
|-----------|-------------|
| Minimum Stake | Threshold to participate in arbitration |
| Case Value Cap | Stake amount determines maximum dispute value allowed |
| Slashing | Incorrect/malicious rulings trigger stake slashing |

**2. Watcher Staking**

Watchers stake $TRUST to obtain challenge rights:
- Successful challenge: Receive share of slashed funds
- Malicious challenge: Bond is forfeited

**3. Protocol Governance**

$TRUST holders may participate in the following governance decisions:

| Governable | Examples |
|------------|----------|
| Protocol Parameters | Fee rates, challenge windows, slash ratios |
| Arbitrator Admission | Committee member election/removal |
| Module Approval | New ZIP module onboarding votes |
| Treasury Spending | Protocol revenue allocation |

#### 8.4.3 Value Accrual

**Protocol Revenue Sources**:

| Source | Description |
|--------|-------------|
| Protocol Fee | Fee charged on each settlement |
| Arbitration Fee | Dispute resolution fees |

**Value Capture Mechanisms**:

| Mechanism | Description |
|-----------|-------------|
| Buyback & Burn | Portion of protocol revenue used to buy back and burn $TRUST |
| Slashing Burns | Portion of slashed $TRUST from arbitrators/watchers is burned |
| Staking Demand | Arbitrators and watchers must continuously hold $TRUST |

#### 8.4.4 Scenarios NOT Using $TRUST

The following scenarios explicitly use stablecoins to avoid token volatility affecting service reliability:

| Scenario | Asset | Reason |
|----------|-------|--------|
| Payer Escrow | USDC | Service value must be stable |
| Provider Collateral | USDC | Collateral value must be predictable |
| Service Pricing & Settlement | USDC | Costs require certainty |
| Dispute Compensation | USDC | Compensation amounts must be stable |

#### 8.4.5 Token Value and Platform Adoption Flywheel

**Why $TRUST Appreciates with Adoption**:

The $TRUST token is designed to capture value as the TrustApp ecosystem grows. Unlike pure utility tokens, $TRUST benefits from multiple demand drivers tied to platform usage.

**Demand Drivers**:

| Driver | Mechanism | Scaling Factor |
|--------|-----------|----------------|
| **Arbitrator staking** | More disputes → more arbitrators needed → more $TRUST staked | Linear with dispute volume |
| **Watcher staking** | Higher transaction volume → more monitoring needed | Linear with transaction count |
| **Governance participation** | More stakeholders → more governance demand | Network effect |
| **Platform integration** | E-commerce platforms staking for premium features | Exponential with B2B adoption |

**Value Accrual Model**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    TrustApp Ecosystem Growth                     │
└──────────────────────────┬──────────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│  Transaction  │  │   Dispute     │  │   Platform    │
│    Volume     │  │    Volume     │  │  Integrations │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘
        │                  │                  │
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ Protocol Fee  │  │  Arbitration  │  │  B2B Staking  │
│   Revenue     │  │    Demand     │  │    Demand     │
└───────┬───────┘  └───────┬───────┘  └───────┬───────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           ▼
              ┌─────────────────────────┐
              │   $TRUST Demand Increase │
              │   - Buyback & Burn       │
              │   - Staking Requirements │
              │   - Governance Power     │
              └───────────┬─────────────┘
                          ▼
              ┌─────────────────────────┐
              │    Token Appreciation    │
              └───────────┬─────────────┘
                          ▼
              ┌─────────────────────────┐
              │ Attracts More Arbitrators│
              │ & Watchers (Higher Yield)│
              └───────────┬─────────────┘
                          ▼
              ┌─────────────────────────┐
              │ Better Dispute Resolution│
              │  → More User Trust       │
              └───────────┬─────────────┘
                          ▼
              ┌─────────────────────────┐
              │  More Transactions &     │
              │  Platform Integrations   │
              └─────────────────────────┘
                    (Flywheel continues)
```

**Quantitative Model (Illustrative)**:

Assume:
- Daily transaction volume: V (USDC)
- Protocol fee: 0.3%
- Buyback allocation: 50% of fees
- Dispute rate: 1%
- Arbitrator yield target: 10% APY

```
Daily protocol revenue = V * 0.003
Daily buyback = V * 0.003 * 0.5 = V * 0.0015
Annual buyback = V * 0.0015 * 365 = V * 0.5475

If V = $10M/day:
  Annual buyback pressure = $5.475M

Required arbitrator stake (for 1% dispute rate):
  Dispute volume = V * 0.01 = $100K/day
  Arbitrator stake needed ≈ 3x dispute value = $300K staked
  For 10% APY, arbitrators earn $30K/year from this stake

Total $TRUST demand = buyback + staking = structural appreciation
```

**B2B Multiplier Effect**:

When e-commerce platforms integrate TrustApp, the multiplier effect accelerates:

| Platform Size | Daily Volume | Annual Buyback | Arbitrator Stake |
|---------------|--------------|----------------|------------------|
| Small marketplace | $100K | $54.75K | $3K |
| Medium marketplace | $1M | $547.5K | $30K |
| Large marketplace | $10M | $5.475M | $300K |
| Enterprise platform | $100M | $54.75M | $3M |

**One large enterprise adoption = 100x small marketplace impact on token demand.**

#### 8.4.6 Entrepreneur Benefits from Token Appreciation

As entrepreneurs stake to use TrustApp services, they also benefit from ecosystem growth:

| Benefit | Description |
|---------|-------------|
| **Stake appreciation** | Provider stakes may appreciate as $TRUST value grows |
| **Staking rewards** | Long-term stakers may earn from protocol revenue sharing |
| **Governance influence** | Successful providers accumulate governance power |
| **Reputation portability** | On-chain track record is verifiable across platforms |

This creates **aligned incentives**: the more entrepreneurs succeed on TrustApp, the more valuable their stakes become, incentivizing honest behavior and long-term participation.

---

## 9. Typical Applications

### 9.1 API Token Billing

**Flow**:
1. Payer deposits 100 USDC
2. Each API call produces a UsageReceipt
3. Provider settles daily in batch
4. Payer may dispute any receipt during challenge window

**Arbitration mode**: TEE or ZK

### 9.2 SaaS Replacement

**Flow**:
1. Payer prepays monthly fee
2. Provider emits receipts for access
3. Batch settlement at month end
4. If service outage, payer disputes

**Arbitration mode**: Oracle (uptime) or Committee

### 9.3 E-commerce

**Flow**:
1. Payer locks funds
2. Provider ships, oracle confirms delivery
3. Settlement on receipt
4. If mismatch, committee arbitration

**Arbitration mode**: Oracle + Committee

### 9.4 AI Inference Service

**Flow**:
1. Payer submits inference request
2. Provider executes in TEE and outputs attestation
3. On-chain verification triggers settlement

**Arbitration mode**: TEE or ZK

### 9.5 Decentralized Alipay for E-commerce Platforms (B2B)

TrustApp enables any e-commerce platform to build its own **decentralized escrow and arbitration system**—essentially a "Decentralized Alipay" without centralized control.

**Value Proposition for Platforms**:

| Traditional Approach | TrustApp Approach |
|---------------------|-------------------|
| Build custom escrow system | Use TrustApp contracts as infrastructure |
| Hire arbitration staff | Plug into decentralized arbitration network |
| Bear regulatory risk | Neutral protocol, transparent rules |
| Limited to domestic jurisdiction | Cross-border enforcement via economic penalties |
| High operational cost | Pay-per-use protocol fees |

**Integration Model**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    E-commerce Platform                           │
│  (Shopify, WooCommerce, Custom Marketplace)                     │
├─────────────────────────────────────────────────────────────────┤
│                     TrustApp SDK Layer                           │
│  - Escrow management       - Receipt generation                  │
│  - Dispute handling        - Settlement automation               │
├─────────────────────────────────────────────────────────────────┤
│                    TrustApp Protocol (Base)                      │
│  EntryPoint | Settlement | Arbitration | StakePool               │
└─────────────────────────────────────────────────────────────────┘
```

**Business Model for Platform Operators**:

| Revenue Stream | Description |
|----------------|-------------|
| **Platform fee markup** | Add platform margin on top of TrustApp protocol fee |
| **Premium arbitration** | Offer faster/specialized arbitration at higher rates |
| **Staking services** | Help merchants stake and earn arbitration participation rewards |
| **White-label solution** | License customized TrustApp integration to other platforms |

**Flow**:
1. Platform integrates TrustApp SDK
2. Buyer purchases product; funds locked in TrustApp escrow
3. Seller ships product; logistics oracle confirms delivery
4. Auto-settlement on confirmation; or dispute → arbitration
5. Platform earns fee on each successful transaction

**Arbitration mode**: Oracle (logistics) + Committee (quality disputes)

**Example Deployment**:
- A Southeast Asian marketplace integrates TrustApp
- Cross-border sellers from China can serve buyers in Indonesia
- No need for local payment licenses—TrustApp handles escrow
- Disputes resolved by decentralized arbitrators, not platform staff
- Platform focuses on customer acquisition, not payment infrastructure

### 9.6 Agent-to-Agent Economy (A2A Transactions)

As AI agents increasingly perform autonomous transactions, TrustApp provides the **trust layer for agent-to-agent commerce**.

**The Agent Economy Challenge**:

| Challenge | Without TrustApp | With TrustApp |
|-----------|------------------|---------------|
| Agent A pays Agent B for service | No recourse if B fails | Escrow + arbitration |
| Billing accuracy | Trust B's metering | Verifiable receipts |
| Quality disputes | No resolution path | Committee/ZK arbitration |
| Agent reputation | Subjective, gameable | Stake-backed economic accountability |

**A2A Transaction Flow**:

```
┌──────────────┐                              ┌──────────────┐
│   Agent A    │                              │   Agent B    │
│  (Consumer)  │                              │  (Provider)  │
└──────┬───────┘                              └──────┬───────┘
       │  1. ServiceRequest (signed)                 │
       │────────────────────────────────────────────►│
       │                                             │
       │  2. Execute service (off-chain)             │
       │◄────────────────────────────────────────────│
       │                                             │
       │  3. UsageReceipt                            │
       │◄────────────────────────────────────────────│
       │                                             │
       │  4. ConfirmService (if satisfied)           │
       │────────────────────────────────────────────►│
       │                                             │
       ▼                                             ▼
┌─────────────────────────────────────────────────────────────┐
│                    TrustApp Settlement                       │
│  - Verify signatures                                         │
│  - Execute payment                                           │
│  - Handle disputes if ConfirmService not received            │
└─────────────────────────────────────────────────────────────┘
```

**Agent Staking and Trust Tiers**:

| Trust Tier | Stake Requirement | Transaction Limit | Use Case |
|------------|-------------------|-------------------|----------|
| **Tier 0** | None | $10/tx | Demo, testing |
| **Tier 1** | 100 USDC | $1,000/tx | Personal agents |
| **Tier 2** | 1,000 USDC | $10,000/tx | Business agents |
| **Tier 3** | 10,000 USDC | $100,000/tx | Enterprise agents |

**Revenue Model for Agent Operators**:
- Agent B (provider) charges service fees per transaction
- TrustApp protocol takes small percentage (0.3%)
- Agent B stakes $TRUST or USDC to enable higher transaction limits
- Successful track record reduces stake requirements over time

**Arbitration mode**: TEE (verifiable compute) or ZK (provable execution)

### 9.7 Trust as a Service (TaaS) for Entrepreneurs

TrustApp enables a new category: **Trust as a Service**—allowing millions of entrepreneurs to offer verified, trustworthy services without building their own trust infrastructure.

**The Entrepreneur's Problem**:

Traditional trust-building is expensive:
- Building reputation takes years
- Payment processors require business history
- Escrow services have high minimums
- Cross-border commerce is complex

**TrustApp as Trust Infrastructure**:

| Entrepreneur Type | Traditional Barrier | TrustApp Solution |
|-------------------|--------------------|--------------------|
| **Freelance developer** | Client doesn't trust unknown contractor | Stake-backed delivery guarantee |
| **Small manufacturer** | Buyer fears non-delivery | Escrow + logistics oracle |
| **AI service provider** | Users doubt compute accuracy | TEE/ZK verified execution |
| **Content creator** | Buyers fear quality issues | Committee arbitration |

**How Entrepreneurs Use TrustApp**:

1. **Instant trust establishment**: Stake collateral → immediately eligible for verified transactions
2. **No reputation cold-start**: Economic stake replaces reputation requirement
3. **Global reach**: Cross-border transactions without local licenses
4. **Dispute protection**: Arbitration protects honest providers too

**Business Model—The "Trust Tax"**:

Entrepreneurs pay for trust:

| Cost Component | Rate | Purpose |
|----------------|------|---------|
| Protocol fee | 0.3% per settlement | Protocol maintenance |
| Stake requirement | 1.5-3x transaction value | Economic security |
| Arbitration fee | 0.5-2% (only if disputed) | Dispute resolution |

**Net benefit**: Even with these costs, entrepreneurs gain access to customers who would never transact without trust guarantees.

**Ecosystem Growth and Token Value**:

As more entrepreneurs use TrustApp:

```
More entrepreneurs stake
        ↓
More transactions secured
        ↓
More protocol fees generated
        ↓
More $TRUST demand (arbitrator staking, governance)
        ↓
Token value appreciation
        ↓
Attracts more arbitrators and watchers
        ↓
Better dispute resolution quality
        ↓
More entrepreneurs trust the system
        ↓
(Flywheel continues)
```

**Arbitration mode**: Varies by service type (see Mode Selection Guide)

---

## 10. x402 Migration Path and Standalone Deployment

### 10.0 TrustApp as a Standalone Protocol

**TrustApp does not require x402**. While TrustApp can enhance x402-based payment flows, it is a fully independent protocol that provides value on its own:

| Deployment Mode | Description | Use Case |
|-----------------|-------------|----------|
| **Standalone TrustApp** | Full protocol without x402 | E-commerce, B2B services, agent networks |
| **TrustApp + x402** | x402 for payment trigger, TrustApp for enforcement | API billing with existing x402 infrastructure |
| **TrustApp + Custom Integration** | Direct SDK/contract integration | Enterprise platforms, custom marketplaces |

**Why Standalone Works**:
- TrustApp's core value is **trust infrastructure**, not payment triggering
- Any payment method (direct transfer, escrow, subscription) can use TrustApp's verification and arbitration
- x402 is one possible payment trigger; TrustApp works equally well with direct contract calls, SDK integrations, or custom protocols

TrustApp also provides a compatibility layer for existing x402 users who want to progressively add trust guarantees.

### 10.1 HTTP Header Extensions

**Original x402**:
```http
GET /resource HTTP/1.1
X-PAYMENT: <EIP-3009 signature>
```

**TrustApp enhanced**:
```http
GET /resource HTTP/1.1
X-PAYMENT: <EIP-3009 signature>
X-TRUSTAPP-REQUEST: <ServiceRequest hash>
X-TRUSTAPP-POLICY: <policyId>
X-TRUSTAPP-RECEIPT: <previous receipt hash>  // optional
```

### 10.2 Progressive Migration Stages

```
Migration Path

Stage 0 -> Stage 1 -> Stage 2 -> Stage 3
Pure x402   x402 + Receipt   x402 + Escrow   Full TrustApp

Stage 0: instant settle, no protection, trust provider
Stage 1: receipts, audit trail, partial verify
Stage 2: escrow + challenge, fund safety
Stage 3: arbitration, full enforcement
```

| Stage | Change | Protection | Use case |
|-------|--------|------------|----------|
| **Stage 0** | pure x402 | none | trusted provider |
| **Stage 1** | + TrustApp receipt | traceable | auditability |
| **Stage 2** | + TrustApp escrow | fund safety | mid-value |
| **Stage 3** | full TrustApp | full game | high-value, untrusted |

### 10.3 Protocol Handshake Sequence

```
Client -> Server: GET /resource
Server -> Client: 402 Payment Required + X-TRUSTAPP-SUPPORTED
Client -> TrustApp (contracts/network): deposit(amount)
Client -> Server: GET /resource + X-PAYMENT + X-TRUSTAPP-REQUEST
Server -> TrustApp (contracts/network): verify request (on-chain)
Server: execute service
Server -> Client: 200 OK + X-TRUSTAPP-RECEIPT
Server -> TrustApp (contracts/network): settle(receipt, on-chain)
```

Note: TrustApp here refers to decentralized contracts/validators, not a centralized server.

### 10.4 SDK Compatibility Layer

```typescript
import { TrustAppClient } from "@trustapp/sdk";

const client = new TrustAppClient({
  mode: "x402-compat",
  fallback: "pure-x402",
  autoUpgrade: true,
});

const response = await client.fetch("https://api.example.com/resource", {
  payment: {
    amount: "1000000", // 1 USDC
    token: "USDC",
  },
  trustapp: {
    escrow: true,
    policy: "default-api",
  },
});
```

### 10.5 Migration Benefits

| Dimension | pure x402 | TrustApp |
|----------|-----------|------------|
| Implementation complexity | low | medium |
| Fund safety | none | escrow |
| Dispute resolution | none | on-chain |
| Suitable ticket size | < $100 | any |
| Provider trust dependency | high | low |
| Settlement delay | instant | challenge window |

---

## 11. Roadmap

| Phase | Milestone | Goal |
|-------|-----------|------|
| **Phase 1** | EntryPoint + Settlement + simplified arbitration | MVP |
| **Phase 2** | Batch receipts + Watcher network | lower gas, more oversight |
| **Phase 3** | Multiple arbitration modes (TEE/ZK/Oracle) | broader coverage |
| **Phase 4** | Module marketplace + SDK | ecosystem growth |
| **Phase 5** | x402 compatibility + cross-chain | adoption and scale |

### 11.1 Future Directions

The following directions will be explored as optional modules once the core protocol matures.

#### 11.1.1 Cross-Chain Settlement and Double-Spend Prevention

The current version supports Base only. Multi-chain expansion must address **cross-chain double-spending**: a Payer could potentially spend the same balance across multiple chains.

**Candidate approaches**:

| Approach | Mechanism | Trade-offs |
|----------|-----------|------------|
| **Main chain anchoring** | All settlements finalize on Base; other chains act as execution layers | High security, but cross-chain latency |
| **Shared stake pool** | Stake concentrated on main chain; other chains verify via light clients | Capital efficient, bridge dependency |
| **Independent instances + state sync** | Each chain runs independently, syncing via LayerZero/CCIP | Flexible, but consistency complexity |

Key design constraints:
- Balance locking strategy during cross-chain message delays
- Cross-chain verifiability of stake status
- Jurisdiction for dispute arbitration

#### 11.1.2 Enterprise Credit Model

The current protocol centers on **prepay escrow**, suited for trustless, one-off transactions. However, real-world commerce involves many long-term relationships (e.g., supply chain credit terms, net-30/net-60 B2B settlements) where parties rely on accumulated credit rather than per-transaction lockups.

**Tiered Credit Model**:

| Tier | Credit Source | Use Case |
|------|---------------|----------|
| Tier 0 | Full stake/escrow | New relationships, one-off deals |
| Tier 1 | Partial stake + history | Developing partnerships |
| Tier 2 | Low stake + off-chain recourse | Long-term partners |

Core design principles:
- **On-chain covers high-frequency small risks**: Stake and slash mechanisms secure routine transactions
- **Off-chain covers low-frequency large risks**: Legal contracts + arbitration recourse as fallback
- **Credit must have collateral backing**: Pure reputation is unenforceable; must combine partial stake or legal binding

#### 11.1.3 Privacy-Preserving Settlement

Currently all settlement data is visible on-chain. For sensitive commercial scenarios (e.g., enterprise procurement, competitive bidding), potential directions include:

| Technology | Application |
|------------|-------------|
| **ZK Receipts** | Prove "service completed and amount correct" without revealing details |
| **Encrypted Escrow** | Settlement amounts encrypted, decrypted only during disputes |
| **Private Arbitration** | Evidence submission and rulings conducted in TEE or private channels |

#### 11.1.4 Decentralized Arbitration Network

Current arbitration relies on pre-configured arbitrators. Long-term explorations:
- **Arbitrator marketplace**: Competitive arbitration services, matched by domain expertise and track record
- **Stake-driven selection**: Arbitrators must stake; incorrect rulings trigger slashing

---

## 12. Conclusion

TrustApp is not a new L1. It is a trust protocol deployed on existing chains. By introducing recognizable transaction types, verifiable receipts, and pluggable arbitration, TrustApp provides an enforceable, extensible foundation for API billing and the service economy.

TrustApp complements x402 and ERC-8004:
- x402 as the payment trigger layer
- ERC-8004 as the identity and discovery layer
- TrustApp as the settlement enforcement layer

As verification and arbitration modules mature, TrustApp becomes a core bridge between on-chain funds and off-chain services.

---

## Appendix A: Arbitration Module Technical Specification

### A.1 Standard Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IArbitrator {
    enum Outcome {
        PENDING,
        PAYER_WINS,
        PROVIDER_WINS,
        SPLIT,
        INVALID
    }

    struct Decision {
        Outcome outcome;
        uint16 payerShareBps;
        uint16 providerShareBps;
        bytes32 reasonHash;
        uint64 decidedAt;
    }

    function submitDecision(
        bytes32 disputeId,
        Decision calldata decision,
        bytes calldata proof
    ) external;

    function verifyDecision(
        bytes32 disputeId,
        Decision calldata decision,
        bytes calldata proof
    ) external view returns (bool);

    function arbMode() external view returns (uint8);
    function minArbitrators() external view returns (uint8);
    function decisionTimeout() external view returns (uint64);
}
```

### A.2 Committee Mode

Architecture:

```
Committee Arbitrator (conceptual)

Arbiter 1   Arbiter 2   Arbiter 3   Arbiter N
    \          |          |          /
     \         |          |         /
         ---- Aggregator ----
```

Trust assumptions:
| Assumption | Requirement |
|-----------|-------------|
| Honest majority | >= 2/3 honest |
| Anti-collusion | bribes are too costly |
| Responsiveness | decision within decisionWindow |

Best for:
- subjective quality disputes
- complex service logic
- high-value transactions

### A.3 TEE Mode

Evidence format:

```solidity
struct TEEEvidence {
    bytes32 requestId;
    bytes32 receiptId;
    bytes32 mrenclave;
    bytes32 mrsigner;
    bytes attestationReport;
    bytes enclaveSignature;
    bytes32 inputHash;
    bytes32 outputHash;
    uint64 executionTime;
}
```

#### On-Chain Verification Cost

Direct Intel SGX Quote verification on EVM is prohibitively expensive (may exceed Gas Limit). A **relay verification layer** is required:

```
┌─────────────┐     attestation     ┌──────────────────┐
│ TEE Enclave │ ──────────────────► │  Relay Verifier  │
└─────────────┘                     │ (Automata/RISC0) │
                                    └────────┬─────────┘
                                             │ ZK proof (lightweight)
                                    ┌────────▼─────────┐
                                    │  EntryPoint      │
                                    │  (verify ZK)     │
                                    └──────────────────┘
```

**Recommended solutions**:
- **Automata Network**: Provides SGX/TDX on-chain verification relay
- **RISC Zero**: Compile Attestation verification logic into zkVM, generate STARK proof
- **Self-hosted relay**: Run trusted verification nodes with multi-sig or stake-backed signing

On-chain contracts only verify lightweight ZK proofs or multi-sig results, reducing Gas cost by 10-100x.

### A.4 ZK Mode

Evidence format:

```solidity
struct ZKEvidence {
    bytes32 requestId;
    bytes32 receiptId;
    bytes32 inputCommitment;
    bytes32 outputCommitment;
    uint256 computeUnits;
    bytes proof;
    bytes32 circuitId;
}
```

### A.5 Oracle Mode

#### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    External Data Sources                          │
│  (Logistics API, IoT Sensors, Third-party Witnesses...)          │
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

#### Evidence Format

```solidity
struct OracleEvidence {
    bytes32 requestId;
    bytes32 receiptId;

    // Oracle data
    bytes32 dataHash;              // Hash of oracle-provided data
    bytes32 dataType;              // Type identifier (e.g., "DELIVERY", "UPTIME", "PRICE")
    bytes encodedData;             // ABI-encoded oracle response

    // Oracle attestations
    address[] oracleAddresses;     // Participating oracle nodes
    bytes[] oracleSignatures;      // Signatures from each oracle
    uint64[] timestamps;           // Timestamp from each oracle

    // Aggregation
    uint8 minConfirmations;        // Minimum required confirmations
    uint8 actualConfirmations;     // Actual confirmations received
    bytes32 aggregatedDataHash;    // Hash of aggregated result

    // Metadata
    uint64 validUntil;             // Data expiration time
    bytes32 sourceId;              // Registered data source identifier
}
```

#### Oracle Arbitrator Interface

```solidity
interface IOracleArbitrator is IArbitrator {

    /// @notice Oracle node status
    enum OracleStatus {
        INACTIVE,
        ACTIVE,
        SUSPENDED,
        SLASHED
    }

    /// @notice Registered oracle node
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

    /// @notice Data source configuration
    struct DataSource {
        bytes32 sourceId;
        string endpoint;
        bytes32[] requiredOracles;    // Specific oracles required
        uint8 minConfirmations;
        uint64 maxDataAge;            // Maximum age of data in seconds
        uint16 deviationThresholdBps; // Max deviation between oracles
    }

    /// @notice Register a new oracle node
    function registerOracle(
        bytes32 nodeId,
        uint256 stakeAmount,
        bytes32[] calldata supportedDataTypes
    ) external;

    /// @notice Submit oracle attestation
    function submitAttestation(
        bytes32 disputeId,
        bytes32 dataHash,
        bytes calldata signature,
        uint64 timestamp
    ) external;

    /// @notice Aggregate oracle responses and submit decision
    function aggregateAndDecide(
        bytes32 disputeId,
        OracleEvidence calldata evidence
    ) external;

    /// @notice Verify oracle evidence validity
    function verifyOracleEvidence(
        OracleEvidence calldata evidence
    ) external view returns (bool valid, string memory reason);

    /// @notice Get oracle node info
    function getOracleNode(address node) external view returns (OracleNode memory);

    /// @notice Check if data source is registered
    function isDataSourceRegistered(bytes32 sourceId) external view returns (bool);

    /// @notice Slash misbehaving oracle
    function slashOracle(address oracle, uint256 amount, bytes32 reason) external;
}
```

#### Oracle Aggregation Logic

```solidity
contract OracleAggregator {

    /// @notice Aggregation strategies
    enum AggregationStrategy {
        MEDIAN,           // Use median value
        WEIGHTED_AVERAGE, // Stake-weighted average
        THRESHOLD,        // Boolean threshold (e.g., 2/3 agree)
        FIRST_VALID       // First valid response (for non-numeric data)
    }

    /// @notice Aggregate oracle responses
    function aggregate(
        bytes32[] calldata dataHashes,
        uint256[] calldata stakes,
        AggregationStrategy strategy
    ) external pure returns (bytes32 result, bool valid) {
        if (strategy == AggregationStrategy.THRESHOLD) {
            // Count matching responses
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

            // Require 2/3 agreement
            valid = (maxCount * 3 >= dataHashes.length * 2);
            result = mostCommon;
        }
        // ... other strategies
    }

    /// @notice Check deviation between numeric oracle responses
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

        // deviation = (max - min) / min
        uint256 deviation = ((max - min) * 10000) / min;
        withinThreshold = (deviation <= maxDeviationBps);
    }
}
```

#### Trust Assumptions

| Assumption | Requirement |
|-----------|-------------|
| Honest majority | >= 2/3 oracle nodes honest |
| Data source reliability | External APIs/sensors function correctly |
| Stake sufficiency | Oracle stake covers potential damages |
| Timeliness | Oracles respond within decisionWindow |

#### Best Use Cases

- Logistics delivery (courier signature confirmation)
- IoT sensor validation
- External API dependency (uptime monitoring)
- Price feeds for payment conversion
- Geographic location verification

### A.6 Mode Selection Guide

| Scenario | Recommended | Reason |
|----------|-------------|--------|
| API token billing | TEE or ZK | verifiable compute |
| AI inference | ZK | provable execution |
| E-commerce logistics | Oracle | external data |
| Content creation | Committee | subjective quality |
| SaaS unlock | TEE | access control |
| High-value consulting | Committee | human judgment |

---

## Appendix B: Core Contract Reference Implementation

> **⚠️ Disclaimer**: The contract code in this appendix is for reference only, intended to illustrate protocol design intent. A professional security audit is required before actual deployment. The development team assumes no liability for any losses resulting from direct use of this code.

### B.1 Data Structures

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

### B.2 EntryPoint Contract

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

        if (tx_.kind == uint8(TrustAppTypes.TxKind.PAY_ESCROW)) {
            ISettlement(settlement).deposit(tx_.payer, tx_.token, tx_.amount);
        } else if (tx_.kind == uint8(TrustAppTypes.TxKind.SETTLE)) {
            ISettlement(settlement).settle(
                tx_.payer, tx_.provider, tx_.token, tx_.amount, tx_.requestHash, tx_.policyId
            );
        }

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

### B.3 Settlement Contract (Core Logic)

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

    mapping(address => mapping(address => TrustAppTypes.EscrowAccount)) public escrows;
    mapping(bytes32 => PendingSettlement) public pendingSettlements;
    mapping(bytes32 => bool) public settledReceipts;

    struct PendingSettlement {
        address payer;
        address provider;
        address token;
        uint256 amount;
        bytes32 requestHash;
        bytes32 policyId;
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
        bytes32 requestHash,
        bytes32 policyId
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
            policyId: policyId,
            settleTime: uint64(block.timestamp),
            finalized: false
        });
    }

    function settleWithConfirm(
        TrustAppTypes.UsageReceipt calldata receipt,
        TrustAppTypes.ConfirmService calldata confirm
    ) external nonReentrant returns (bytes32 settlementId) {
        require(msg.sender == entryPoint, "Only entryPoint");
        require(!settledReceipts[receipt.receiptId], "Receipt already settled");
        settledReceipts[receipt.receiptId] = true;

        settlementId = confirm.settlementId;
        require(pendingSettlements[settlementId].payer == address(0), "Settlement exists");
        require(confirm.amount == receipt.amount, "Amount mismatch");

        uint256 fee = (confirm.amount * protocolFeeBps) / 10000;
        uint256 providerAmount = confirm.amount - fee;

        escrows[confirm.payer][confirm.token].balance -= confirm.amount;
        IERC20(confirm.token).safeTransfer(confirm.provider, providerAmount);
    }

    function finalize(bytes32 settlementId) external nonReentrant {
        PendingSettlement storage ps = pendingSettlements[settlementId];
        require(!ps.finalized, "Already finalized");

        uint256 fee = (ps.amount * protocolFeeBps) / 10000;
        uint256 providerAmount = ps.amount - fee;

        escrows[ps.payer][ps.token].locked -= ps.amount;
        IERC20(ps.token).safeTransfer(ps.provider, providerAmount);

        ps.finalized = true;
    }

    function advanceWithdraw(bytes32 settlementId) external nonReentrant {
        PendingSettlement storage ps = pendingSettlements[settlementId];
        require(msg.sender == ps.provider, "Only provider");

        uint256 stake = IStakePool(stakePool).getStake(ps.provider, ps.token);
        require(stake >= ps.amount * 2, "Insufficient stake");

        uint256 advance = (ps.amount * 80) / 100;
        IStakePool(stakePool).lockStake(ps.provider, ps.token, ps.amount * 2);
        IERC20(ps.token).safeTransfer(ps.provider, advance);
    }
}
```

### B.4 Complete Interface Definitions

#### ISettlement Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface ISettlement {

    /// @notice Escrow account state
    struct EscrowAccount {
        uint256 balance;      // Available balance
        uint256 locked;       // Locked for pending settlements
        uint256 nonce;        // Account nonce
    }

    /// @notice Pending settlement state
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

    // ============ Deposit & Withdraw ============

    /// @notice Deposit tokens to escrow
    function deposit(address payer, address token, uint256 amount) external;

    /// @notice Withdraw available balance
    function withdraw(address token, uint256 amount) external;

    /// @notice Get escrow balance
    function getBalance(address payer, address token) external view returns (uint256 available, uint256 locked);

    // ============ Settlement ============

    /// @notice Initiate settlement (locks funds, starts challenge window)
    function settle(
        address payer,
        address provider,
        address token,
        uint256 amount,
        bytes32 requestHash,
        bytes32 policyId
    ) external returns (bytes32 settlementId);

    /// @notice Fast-path settlement (confirm signature + receipt)
    function settleWithConfirm(
        TrustAppTypes.UsageReceipt calldata receipt,
        TrustAppTypes.ConfirmService calldata confirm
    ) external returns (bytes32 settlementId);

    /// @notice Finalize settlement after challenge window
    function finalize(bytes32 settlementId) external;

    /// @notice Batch finalize multiple settlements
    function batchFinalize(bytes32[] calldata settlementIds) external;

    /// @notice Get pending settlement details
    function getPendingSettlement(bytes32 settlementId) external view returns (PendingSettlement memory);

    // ============ Optimistic Settlement ============

    /// @notice Provider advance withdrawal (requires stake)
    function advanceWithdraw(bytes32 settlementId) external;

    /// @notice Clawback advanced funds on successful challenge
    function clawback(bytes32 settlementId, uint256 amount) external;

    // ============ Dispute Integration ============

    /// @notice Mark settlement as disputed (called by Arbitration)
    function markDisputed(bytes32 settlementId) external;

    /// @notice Execute arbitration decision
    function executeDecision(
        bytes32 settlementId,
        uint16 payerShareBps,
        uint16 providerShareBps
    ) external;

    // ============ Admin ============

    /// @notice Set protocol fee
    function setProtocolFee(uint16 feeBps) external;

    /// @notice Set EntryPoint address
    function setEntryPoint(address entryPoint) external;

    /// @notice Emergency pause
    function pause() external;

    /// @notice Unpause
    function unpause() external;
}
```

#### IStakePool Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IStakePool {

    /// @notice Stake state for a provider
    struct StakeInfo {
        uint256 total;        // Total staked amount
        uint256 locked;       // Locked for pending settlements
        uint256 slashed;      // Cumulative slashed amount
        uint64 lastStakeTime; // Last stake modification time
        uint64 unlockTime;    // Cooldown unlock time
    }

    // ============ Staking ============

    /// @notice Stake tokens
    function stake(address token, uint256 amount) external;

    /// @notice Request unstake (starts cooldown)
    function requestUnstake(address token, uint256 amount) external;

    /// @notice Complete unstake after cooldown
    function unstake(address token, uint256 amount) external;

    /// @notice Get stake info
    function getStake(address provider, address token) external view returns (uint256 available, uint256 locked);

    /// @notice Get full stake info
    function getStakeInfo(address provider, address token) external view returns (StakeInfo memory);

    // ============ Locking ============

    /// @notice Lock stake for optimistic settlement
    function lockStake(address provider, address token, uint256 amount) external;

    /// @notice Unlock stake after settlement finalized
    function unlockStake(address provider, address token, uint256 amount) external;

    /// @notice Check if provider has sufficient stake
    function hasSufficientStake(
        address provider,
        address token,
        uint256 requiredAmount,
        uint256 multiplier
    ) external view returns (bool);

    // ============ Slashing ============

    /// @notice Slash provider stake (called by Arbitration)
    function slash(
        address provider,
        address token,
        uint256 amount,
        bytes32 reason
    ) external returns (uint256 actualSlashed);

    /// @notice Distribute slashed funds
    function distributeSlash(
        uint256 amount,
        address payer,
        address arbitrator,
        uint16 payerBps,
        uint16 arbBps,
        uint16 treasuryBps,
        uint16 burnBps
    ) external;

    // ============ Parameters ============

    /// @notice Get minimum stake requirement
    function minStake(address token) external view returns (uint256);

    /// @notice Get unstake cooldown period
    function unstakeCooldown() external view returns (uint64);

    // ============ Admin ============

    /// @notice Set minimum stake
    function setMinStake(address token, uint256 amount) external;

    /// @notice Set unstake cooldown
    function setUnstakeCooldown(uint64 cooldown) external;

    /// @notice Add authorized slasher (e.g., Arbitration contract)
    function addSlasher(address slasher) external;
}
```

#### IRegistry Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IRegistry {

    /// @notice Module registration info
    struct ModuleInfo {
        bytes32 moduleId;
        string name;
        string version;
        address implementation;
        bytes32 specHash;        // Hash of module specification
        uint64 registeredAt;
        uint64 updatedAt;
        bool active;
        uint8 riskTier;          // 1-5 risk classification
        string auditReportUri;
    }

    /// @notice Arbitration policy info
    struct PolicyInfo {
        bytes32 policyId;
        TrustAppTypes.ArbitrationPolicy policy;
        bool active;
        uint64 createdAt;
    }

    // ============ Module Registry ============

    /// @notice Register a new module
    function registerModule(
        bytes32 moduleId,
        string calldata name,
        string calldata version,
        address implementation,
        bytes32 specHash,
        uint8 riskTier,
        string calldata auditReportUri
    ) external;

    /// @notice Update module (creates new version)
    function updateModule(
        bytes32 moduleId,
        string calldata newVersion,
        address newImplementation,
        bytes32 newSpecHash
    ) external;

    /// @notice Deactivate module
    function deactivateModule(bytes32 moduleId) external;

    /// @notice Get module info
    function getModule(bytes32 moduleId) external view returns (ModuleInfo memory);

    /// @notice Check if module is active
    function isModuleActive(bytes32 moduleId) external view returns (bool);

    // ============ Policy Registry ============

    /// @notice Register arbitration policy
    function registerPolicy(
        bytes32 policyId,
        TrustAppTypes.ArbitrationPolicy calldata policy
    ) external;

    /// @notice Get policy
    function getPolicy(bytes32 policyId) external view returns (TrustAppTypes.ArbitrationPolicy memory);

    /// @notice Check if policy is active
    function isPolicyActive(bytes32 policyId) external view returns (bool);

    // ============ Arbitrator Registry ============

    /// @notice Register arbitrator for a mode
    function registerArbitrator(
        uint8 arbMode,
        address arbitrator
    ) external;

    /// @notice Get arbitrator for mode
    function getArbitrator(uint8 arbMode) external view returns (address);

    // ============ Access Control ============

    /// @notice Check if address is authorized module admin
    function isModuleAdmin(bytes32 moduleId, address account) external view returns (bool);

    /// @notice Grant module admin role
    function grantModuleAdmin(bytes32 moduleId, address account) external;
}
```

#### IArbitration Interface

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IArbitration {

    /// @notice Dispute info
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

    // ============ Dispute Lifecycle ============

    /// @notice Open a dispute
    function openDispute(
        bytes32 settlementId,
        bytes32 receiptId,
        bytes32 policyId,
        bytes calldata evidence
    ) external payable returns (bytes32 disputeId);

    /// @notice Submit bond
    function submitBond(bytes32 disputeId) external payable;

    /// @notice Submit evidence
    function submitEvidence(
        bytes32 disputeId,
        bytes32 evidenceHash,
        string calldata evidenceUri
    ) external;

    /// @notice Submit arbitration decision (called by arbitrator)
    function submitDecision(
        bytes32 disputeId,
        IArbitrator.Decision calldata decision,
        bytes calldata proof
    ) external;

    /// @notice Force finalize after grace period
    function forceFinalize(bytes32 disputeId) external;

    // ============ Queries ============

    /// @notice Get dispute info
    function getDispute(bytes32 disputeId) external view returns (DisputeInfo memory);

    /// @notice Get dispute state
    function getDisputeState(bytes32 disputeId) external view returns (TrustAppTypes.DisputeState);

    /// @notice Check if settlement has active dispute
    function hasActiveDispute(bytes32 settlementId) external view returns (bool);

    // ============ Evidence ============

    /// @notice Get evidence hashes for dispute
    function getEvidenceHashes(bytes32 disputeId) external view returns (bytes32[] memory);

    /// @notice Verify evidence hash
    function verifyEvidenceHash(bytes32 disputeId, bytes32 evidenceHash) external view returns (bool);

    // ============ Admin ============

    /// @notice Set grace period
    function setGracePeriod(uint64 period) external;

    /// @notice Emergency resolve dispute
    function emergencyResolve(bytes32 disputeId, IArbitrator.Outcome outcome) external;
}
```

### B.5 Deployment Script

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Script.sol";

contract DeployTrustApp is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");

        vm.startBroadcast(deployerPrivateKey);

        Registry registry = new Registry();
        StakePool stakePool = new StakePool();
        Arbitration arbitration = new Arbitration(address(stakePool));
        Settlement settlement = new Settlement(address(arbitration), address(stakePool));
        EntryPoint entryPoint = new EntryPoint(
            address(settlement),
            address(arbitration),
            address(registry)
        );

        settlement.setEntryPoint(address(entryPoint));

        vm.stopBroadcast();

        console.log("EntryPoint:", address(entryPoint));
        console.log("Settlement:", address(settlement));
        console.log("Arbitration:", address(arbitration));
    }
}
```

### B.6 ProviderRegistry Contract

The ProviderRegistry manages provider onboarding, capability declaration, and access control.

#### Architecture

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

#### Interface Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IProviderRegistry {

    /// @notice Provider status
    enum ProviderStatus {
        UNREGISTERED,
        PENDING,          // Awaiting verification
        ACTIVE,
        SUSPENDED,
        BLACKLISTED
    }

    /// @notice Provider profile
    struct ProviderProfile {
        address providerAddress;
        bytes32 providerId;
        string name;
        string metadataUri;         // IPFS/Arweave URI for extended metadata
        ProviderStatus status;
        uint64 registeredAt;
        uint64 lastActiveAt;
        uint256 totalSettled;       // Cumulative settled amount
        uint256 disputeCount;
        uint256 disputeLossCount;
        bytes32[] serviceTypes;     // Supported service types
        bytes32[] moduleIds;        // Registered modules
    }

    /// @notice Service type definition
    struct ServiceType {
        bytes32 typeId;
        string name;
        string description;
        bytes32 specHash;           // Service specification hash
        uint8 requiredArbMode;      // Required arbitration mode
        uint256 minStakeRequired;   // Minimum stake for this service type
        bool active;
    }

    /// @notice Capability proof (e.g., TEE attestation, ZK credential)
    struct CapabilityProof {
        bytes32 proofId;
        bytes32 capabilityType;     // "TEE_SGX", "ZK_PROVER", "ORACLE_NODE", etc.
        bytes proofData;            // Encoded proof data
        uint64 issuedAt;
        uint64 expiresAt;
        address issuer;             // Proof issuer (e.g., Intel for SGX)
        bool verified;
    }

    // ============ Registration ============

    /// @notice Register as a provider
    function registerProvider(
        string calldata name,
        string calldata metadataUri,
        bytes32[] calldata serviceTypes
    ) external payable returns (bytes32 providerId);

    /// @notice Update provider profile
    function updateProfile(
        string calldata name,
        string calldata metadataUri
    ) external;

    /// @notice Add supported service type
    function addServiceType(bytes32 serviceTypeId) external;

    /// @notice Remove supported service type
    function removeServiceType(bytes32 serviceTypeId) external;

    /// @notice Submit capability proof
    function submitCapabilityProof(
        bytes32 capabilityType,
        bytes calldata proofData,
        uint64 expiresAt
    ) external returns (bytes32 proofId);

    // ============ Queries ============

    /// @notice Get provider profile
    function getProvider(address provider) external view returns (ProviderProfile memory);

    /// @notice Get provider by ID
    function getProviderById(bytes32 providerId) external view returns (ProviderProfile memory);

    /// @notice Check if provider is active
    function isProviderActive(address provider) external view returns (bool);

    /// @notice Check if provider supports service type
    function supportsServiceType(address provider, bytes32 serviceTypeId) external view returns (bool);

    /// @notice Get provider's capability proofs
    function getCapabilityProofs(address provider) external view returns (CapabilityProof[] memory);

    /// @notice Verify capability proof is valid
    function hasValidCapability(address provider, bytes32 capabilityType) external view returns (bool);

    // ============ Service Type Management ============

    /// @notice Register a new service type
    function registerServiceType(
        bytes32 typeId,
        string calldata name,
        string calldata description,
        bytes32 specHash,
        uint8 requiredArbMode,
        uint256 minStakeRequired
    ) external;

    /// @notice Get service type info
    function getServiceType(bytes32 typeId) external view returns (ServiceType memory);

    // ============ Access Control ============

    /// @notice Suspend provider
    function suspendProvider(address provider, string calldata reason) external;

    /// @notice Reactivate suspended provider
    function reactivateProvider(address provider) external;

    /// @notice Blacklist provider
    function blacklistProvider(address provider, string calldata reason) external;

    /// @notice Check if provider is blacklisted
    function isBlacklisted(address provider) external view returns (bool);

    // ============ Rate Limiting ============

    /// @notice Get provider's settlement limit
    function getSettlementLimit(address provider) external view returns (uint256 daily, uint256 perTx);

    /// @notice Check if provider can settle amount
    function canSettle(address provider, uint256 amount) external view returns (bool);

    /// @notice Record settlement (called by Settlement contract)
    function recordSettlement(address provider, uint256 amount) external;

    // ============ Reputation Integration ============

    /// @notice Get provider reputation score (0-10000)
    function getReputationScore(address provider) external view returns (uint16);

    /// @notice Record dispute outcome (called by Arbitration)
    function recordDisputeOutcome(address provider, bool won) external;
}
```

#### Provider Onboarding Flow

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

#### Tier-Based Limits

Providers are assigned tiers based on stake and reputation, which determine their operational limits:

| Tier | Min Stake | Max Tx | Daily Limit | Required Reputation |
|------|-----------|--------|-------------|---------------------|
| Starter | 100 USDC | 100 USDC | 1,000 USDC | N/A (new) |
| Basic | 1,000 USDC | 500 USDC | 10,000 USDC | >= 5000 |
| Standard | 5,000 USDC | 2,000 USDC | 50,000 USDC | >= 7000 |
| Premium | 25,000 USDC | 10,000 USDC | 250,000 USDC | >= 8500 |
| Enterprise | 100,000 USDC | Unlimited | Unlimited | >= 9500 |

```solidity
/// @notice Tier configuration
struct TierConfig {
    uint256 minStake;
    uint256 maxPerTx;
    uint256 dailyLimit;
    uint16 minReputation;
}

/// @notice Get provider's current tier
function getProviderTier(address provider) external view returns (uint8 tier, TierConfig memory config);
```

### B.7 Watcher Network Specification

Watchers monitor settlements and can challenge fraudulent claims. A healthy watcher network is essential for protocol security.

#### Architecture

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

#### Watcher Types

| Type | Responsibility | Stake Requirement | Reward Share |
|------|---------------|-------------------|--------------|
| **Full Watcher** | Indexes all settlements, validates receipts, initiates challenges | 10,000 USDC | 70% of challenger reward |
| **Light Watcher** | Monitors specific providers or modules, reports to Full Watchers | 1,000 USDC | 20% of challenger reward |
| **Delegated Watcher** | Receives delegation from token holders, shares rewards | Variable | Based on delegation |

#### Interface Definition

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

interface IWatcherRegistry {

    /// @notice Watcher status
    enum WatcherStatus {
        INACTIVE,
        ACTIVE,
        SUSPENDED,
        EXITING     // In unbonding period
    }

    /// @notice Watcher type
    enum WatcherType {
        FULL,
        LIGHT,
        DELEGATED
    }

    /// @notice Watcher profile
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
        bytes32[] watchedModules;    // Modules this watcher monitors
        address[] watchedProviders;  // Specific providers to monitor
    }

    /// @notice Challenge record
    struct ChallengeRecord {
        bytes32 challengeId;
        bytes32 settlementId;
        address watcher;
        uint64 timestamp;
        bool successful;
        uint256 reward;
    }

    // ============ Registration ============

    /// @notice Register as a watcher
    function registerWatcher(
        WatcherType watcherType,
        bytes32[] calldata watchedModules
    ) external payable returns (bytes32 watcherId);

    /// @notice Add stake
    function addStake(uint256 amount) external;

    /// @notice Request unstake (starts unbonding)
    function requestUnstake(uint256 amount) external;

    /// @notice Complete unstake after unbonding period
    function unstake() external;

    /// @notice Update watched modules
    function updateWatchedModules(bytes32[] calldata modules) external;

    /// @notice Add specific provider to watch list
    function addWatchedProvider(address provider) external;

    // ============ Delegation ============

    /// @notice Delegate stake to a watcher
    function delegate(address watcher, uint256 amount) external;

    /// @notice Undelegate stake
    function undelegate(address watcher, uint256 amount) external;

    /// @notice Get total delegated to watcher
    function getDelegatedStake(address watcher) external view returns (uint256);

    // ============ Challenge ============

    /// @notice Report suspicious settlement
    function reportSuspicious(
        bytes32 settlementId,
        bytes calldata evidence
    ) external returns (bytes32 reportId);

    /// @notice Record challenge outcome (called by Arbitration)
    function recordChallengeOutcome(
        bytes32 challengeId,
        address watcher,
        bool successful,
        uint256 reward
    ) external;

    // ============ Queries ============

    /// @notice Get watcher profile
    function getWatcher(address watcher) external view returns (WatcherProfile memory);

    /// @notice Check if watcher is active
    function isWatcherActive(address watcher) external view returns (bool);

    /// @notice Get active watchers for a module
    function getWatchersForModule(bytes32 moduleId) external view returns (address[] memory);

    /// @notice Get challenge history
    function getChallengeHistory(address watcher) external view returns (ChallengeRecord[] memory);

    // ============ Coverage Metrics ============

    /// @notice Get watcher coverage for a settlement amount
    function getWatcherCoverage(uint256 amount) external view returns (
        uint8 watcherCount,
        uint256 totalStake,
        bool sufficientCoverage
    );

    /// @notice Minimum watchers required for settlement amount
    function requiredWatchers(uint256 amount) external view returns (uint8);

    // ============ Rewards ============

    /// @notice Claim accumulated rewards
    function claimRewards() external returns (uint256);

    /// @notice Get pending rewards
    function pendingRewards(address watcher) external view returns (uint256);
}
```

#### Watcher Incentive Model

```
Challenge Reward Distribution:

┌─────────────────────────────────────────────────────────────────┐
│                    Slashed Amount (S)                            │
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

#### Coverage Requirements

High-value settlements require minimum watcher coverage:

| Settlement Amount | Min Watchers | Min Total Stake |
|------------------|--------------|-----------------|
| < 1,000 USDC | 1 | 5,000 USDC |
| 1,000 - 10,000 USDC | 2 | 20,000 USDC |
| 10,000 - 100,000 USDC | 3 | 100,000 USDC |
| > 100,000 USDC | 5 | 500,000 USDC |

If coverage is insufficient, settlement enters extended challenge window (2x normal).

### B.8 Event Log Definitions

Complete event definitions for off-chain indexing and monitoring:

```solidity
// ============ EntryPoint Events ============

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

// ============ Settlement Events ============

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

// ============ Arbitration Events ============

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

// ============ StakePool Events ============

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

// ============ ProviderRegistry Events ============

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

// ============ WatcherRegistry Events ============

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
    bool isDelegation  // true = delegate, false = undelegate
);

// ============ Registry Events ============

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

## Appendix C: Data Availability Scheme

### C.1 Evidence Storage Architecture

```
Evidence Layer (conceptual)

IPFS (hot)     Arweave (permanent)     Filecoin (incentive)
     \               |                       /
      \              |                      /
       +-------------+---------------------+
                     |
                DA Aggregator
```

### C.2 Evidence Record

```solidity
struct EvidenceRecord {
    bytes32 evidenceHash;
    string ipfsCid;
    bytes32 arweaveTxId;
    bytes32 filecoinDealId;
    uint64 submittedAt;
    address submitter;
}
```

### C.3 DA Failure Handling

| Scenario | Outcome |
|----------|---------|
| Evidence not retrievable | submitter loses |
| Hash mismatch | evidence invalid |
| Filecoin deal expired | submitter bears risk |

---

## Appendix D: Merkle Batch Settlement

### D.1 Batch Settlement Architecture

For high-frequency, low-value transactions (e.g., API calls), individual on-chain settlements are cost-prohibitive. Merkle batch settlement aggregates multiple receipts into a single on-chain commitment.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Batch Settlement Flow                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Off-chain (Provider)                     On-chain                │
│  ┌─────────────────────┐                                         │
│  │ Receipt 1           │                                         │
│  │ Receipt 2           │                                         │
│  │ Receipt 3           │     Batch                               │
│  │ ...                 │ ──────────►  ┌──────────────────┐       │
│  │ Receipt N           │              │  ReceiptBatch    │       │
│  └─────────────────────┘              │  - batchId       │       │
│           │                           │  - merkleRoot    │       │
│           ▼                           │  - totalAmount   │       │
│  ┌─────────────────────┐              │  - receiptCount  │       │
│  │   Merkle Tree       │              └────────┬─────────┘       │
│  │                     │                       │                 │
│  │       Root          │                       ▼                 │
│  │      /    \         │              Challenge Window           │
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

### D.2 Data Structures

```solidity
/// @notice Batch settlement on-chain commitment
struct ReceiptBatch {
    bytes32 batchId;
    bytes32 merkleRoot;        // Root of receipt Merkle tree
    address provider;
    address token;
    uint256 totalAmount;       // Sum of all receipt amounts
    uint32 receiptCount;       // Number of receipts in batch
    uint64 fromTime;           // Earliest receipt timestamp
    uint64 toTime;             // Latest receipt timestamp
    uint64 submittedAt;
    bytes32 policyId;
}

/// @notice Individual receipt for Merkle proof verification
struct MerkleReceipt {
    bytes32 receiptId;
    bytes32 requestId;
    address payer;
    uint256 amount;
    uint64 timestamp;
    bytes32 payloadHash;
    bytes32 responseHash;
}

/// @notice Merkle proof for single receipt dispute
struct MerkleProof {
    bytes32[] proof;           // Sibling hashes
    uint256 index;             // Leaf index in tree
    MerkleReceipt receipt;     // The disputed receipt
}
```

### D.3 Batch Settlement Contract

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

    /// @notice Submit a batch of receipts
    function submitBatch(
        ReceiptBatch calldata batch,
        bytes calldata providerSignature
    ) external returns (bytes32 batchId) {
        require(batch.receiptCount <= MAX_BATCH_SIZE, "Batch too large");
        require(batch.toTime - batch.fromTime >= MIN_BATCH_INTERVAL, "Interval too short");

        // Verify provider signature
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

        // Lock payer funds proportionally
        _lockPayerFunds(batch);

        emit BatchSubmitted(
            batchId,
            batch.provider,
            batch.merkleRoot,
            batch.totalAmount,
            batch.receiptCount
        );
    }

    /// @notice Dispute a specific receipt within a batch
    function disputeReceipt(
        bytes32 batchId,
        MerkleProof calldata proof
    ) external {
        ReceiptBatch storage batch = batches[batchId];
        require(batch.batchId != bytes32(0), "Batch not found");

        // Verify Merkle proof
        bytes32 leaf = keccak256(abi.encode(proof.receipt));
        require(
            MerkleProof.verify(proof.proof, batch.merkleRoot, leaf),
            "Invalid Merkle proof"
        );

        // Verify receipt not already disputed
        require(
            !disputedReceipts[batchId][proof.receipt.receiptId],
            "Already disputed"
        );

        disputedReceipts[batchId][proof.receipt.receiptId] = true;

        // Open individual dispute
        IArbitration(arbitration).openDispute(
            batchId,
            proof.receipt.receiptId,
            batch.policyId,
            abi.encode(proof)
        );

        emit ReceiptDisputed(batchId, proof.receipt.receiptId, msg.sender);
    }

    /// @notice Finalize batch after challenge window
    function finalizeBatch(bytes32 batchId) external {
        ReceiptBatch storage batch = batches[batchId];
        require(batch.batchId != bytes32(0), "Batch not found");

        // Check challenge window passed
        uint64 challengeDeadline = batch.submittedAt + IRegistry(registry)
            .getPolicy(batch.policyId).challengeWindow;
        require(block.timestamp > challengeDeadline, "Challenge window active");

        // Calculate disputed amount
        uint256 disputedAmount = _calculateDisputedAmount(batchId);
        uint256 settleAmount = batch.totalAmount - disputedAmount;

        // Settle undisputed amount
        ISettlement(settlement).finalizeBatch(
            batch.provider,
            batch.token,
            settleAmount
        );
    }

    /// @notice Generate Merkle tree off-chain (reference implementation)
    function computeMerkleRoot(
        MerkleReceipt[] memory receipts
    ) external pure returns (bytes32) {
        require(receipts.length > 0, "Empty receipts");

        // Compute leaf hashes
        bytes32[] memory leaves = new bytes32[](receipts.length);
        for (uint i = 0; i < receipts.length; i++) {
            leaves[i] = keccak256(abi.encode(receipts[i]));
        }

        // Build tree bottom-up
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

### D.4 Partial Dispute Resolution

When individual receipts within a batch are disputed:

```
Batch Dispute Resolution:

┌────────────────────────────────────────────────────────────────┐
│                    Batch of 1000 Receipts                       │
│                    Total: 10,000 USDC                           │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌────────────────┐  │
│  │ Undisputed      │  │ Disputed #42    │  │ Disputed #789  │  │
│  │ 998 receipts    │  │ 50 USDC         │  │ 30 USDC        │  │
│  │ 9,920 USDC      │  │ → Arbitration   │  │ → Arbitration  │  │
│  └────────┬────────┘  └────────┬────────┘  └───────┬────────┘  │
│           │                    │                    │           │
│           ▼                    ▼                    ▼           │
│   Settles normally      Held pending          Held pending      │
│   after challenge       arbitration           arbitration       │
│   window                                                        │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

| Scenario | Action |
|----------|--------|
| No disputes | Entire batch settles after challenge window |
| Single receipt disputed | Disputed amount held; rest settles |
| Multiple receipts disputed | Each disputed amount held separately |
| >10% of batch disputed | Extended challenge window (2x) |
| Provider loses dispute | Slashing from stake pool |

---

## Appendix E: Gas Cost Analysis

### E.1 Gas Estimates by Operation

Based on Base (L2) gas costs. Estimates assume typical contract sizes and storage patterns.

| Operation | Gas (estimated) | Cost @ 0.01 gwei | Notes |
|-----------|----------------|------------------|-------|
| **Deposit** | 65,000 | ~$0.001 | ERC20 transfer + storage |
| **Withdraw** | 45,000 | ~$0.001 | ERC20 transfer + storage update |
| **Single Settlement** | 120,000 | ~$0.002 | Full settlement flow |
| **Batch Submit (1000 receipts)** | 150,000 | ~$0.002 | Merkle root only |
| **Batch Finalize** | 80,000 | ~$0.001 | No disputes |
| **Open Dispute** | 180,000 | ~$0.003 | Create dispute + bond |
| **Submit Evidence** | 95,000 | ~$0.002 | Store evidence hash |
| **Submit Decision** | 140,000 | ~$0.002 | Arbitrator decision |
| **Force Finalize** | 160,000 | ~$0.003 | Timeout resolution |
| **Stake** | 70,000 | ~$0.001 | Provider staking |
| **Unstake Request** | 55,000 | ~$0.001 | Start cooldown |
| **Slash** | 180,000 | ~$0.003 | Slash + distribute |
| **Register Provider** | 200,000 | ~$0.003 | Full registration |
| **Register Watcher** | 150,000 | ~$0.002 | Watcher registration |

### E.2 Cost Comparison: Single vs Batch Settlement

```
Cost Analysis: 1000 API calls @ $0.01 each = $10 total

Single Settlement (per call):
  - Gas per settlement: 120,000
  - Total gas: 120,000 × 1000 = 120,000,000
  - Cost @ 0.01 gwei: ~$2.00
  - Overhead: 20% of transaction value

Batch Settlement:
  - Batch submit: 150,000
  - Batch finalize: 80,000
  - Total gas: 230,000
  - Cost @ 0.01 gwei: ~$0.004
  - Overhead: 0.04% of transaction value

Savings: 99.8%
```

### E.3 Break-Even Analysis

When does individual settlement make sense vs batching?

```
Single Settlement Cost: C_single = gas_single × gasPrice
Batch Settlement Cost: C_batch = (gas_submit + gas_finalize) / N

Break-even: C_single = C_batch
N = (gas_submit + gas_finalize) / gas_single
N = 230,000 / 120,000
N ≈ 2
```

**Recommendation**: Batch settlement is cost-effective for > 2 receipts.

### E.4 Dispute Cost Analysis

Full dispute resolution costs:

| Party | Action | Gas | Cost |
|-------|--------|-----|------|
| **Challenger** | Open dispute | 180,000 | ~$0.003 |
| **Challenger** | Submit bond | (included) | - |
| **Provider** | Submit bond | 65,000 | ~$0.001 |
| **Challenger** | Submit evidence | 95,000 | ~$0.002 |
| **Provider** | Submit evidence | 95,000 | ~$0.002 |
| **Arbitrator** | Submit decision | 140,000 | ~$0.002 |
| **System** | Execute decision | 160,000 | ~$0.003 |
| **Total** | | 735,000 | ~$0.013 |

**Note**: Bond amounts (not gas) are the primary dispute cost. Gas is negligible on L2.

### E.5 Scaling Projections

| Daily Volume | Settlement Strategy | Est. Daily Gas Cost |
|-------------|---------------------|---------------------|
| 1,000 txns | Individual | ~$2.00 |
| 1,000 txns | Batch (hourly) | ~$0.10 |
| 10,000 txns | Individual | ~$20.00 |
| 10,000 txns | Batch (hourly) | ~$0.60 |
| 100,000 txns | Individual | ~$200.00 |
| 100,000 txns | Batch (hourly) | ~$5.80 |
| 1,000,000 txns | Individual | ~$2,000.00 |
| 1,000,000 txns | Batch (hourly) | ~$58.00 |

### E.6 Optimization Strategies

| Strategy | Gas Savings | Trade-off |
|----------|------------|-----------|
| **Batch settlements** | 95-99% | Delayed finality |
| **Merkle proofs** | 90%+ for disputes | Complexity |
| **Calldata compression** | 20-40% | Off-chain processing |
| **Storage packing** | 10-30% | Code complexity |
| **EIP-4844 blobs** | 80%+ for data | Base support pending |

### E.7 L2 vs L1 Cost Comparison

| Chain | Avg Gas Price | Settlement Cost | Notes |
|-------|--------------|-----------------|-------|
| **Base** | 0.001-0.01 gwei | ~$0.002 | Primary deployment target |
| **Arbitrum** | 0.01-0.1 gwei | ~$0.01 | Alternative L2 |
| **Optimism** | 0.001-0.01 gwei | ~$0.002 | Alternative L2 |
| **Ethereum L1** | 20-100 gwei | ~$5-25 | Not recommended |

**Recommendation**: Deploy on Base for 1000x cost reduction vs L1.

---

## Appendix F: Glossary

| Term | Definition |
|------|------------|
| **Payer** | Service buyer, pre-funds escrow |
| **Provider** | Service provider, submits receipts |
| **ServiceTx** | Structured service transaction via EntryPoint |
| **UsageReceipt** | Provider-signed service execution proof |
| **Challenge Window** | Dispute window before finalization |
| **Bond** | Dispute collateral |
| **Slash** | Penalty against stake |
| **CEE** | Credit Execution Environment |
| **TEE** | Trusted Execution Environment |
| **ZK** | Zero Knowledge |
| **Watcher** | Monitors and challenges settlements |
| **Facilitator** | x402 settlement coordinator (replaced by contracts) |
| **arbMode** | Arbitration mode |
| **defaultOutcome** | Default decision on timeout |
| **settlementId** | Settlement primary key, uniquely identifies a settlement, used for all settle/finalize/dispute operations |
| **receiptId** | Receipt unique identifier, used to prevent receipt replay |
| **requestHash** | ServiceRequest content hash, used only for association indexing, not as settlement primary key |
| **ConfirmService** | Fast-path confirmation message, signed by Payer to skip challenge window for instant settlement |

---

**Document Version**: v0.4.1
**Last Updated**: December 2025
**Change Log**:
- v0.1: initial draft
- v0.2: added arbitration flow details
- v0.3: added security model, game analysis, x402 migration, finality bounds, Sybil defense
- v0.4: expanded Oracle arbitration mode, added ProviderRegistry and Watcher network specs, complete interface definitions, Gas cost analysis and Merkle batch settlement
- v0.4.1: fixed section numbering, expanded glossary, added MEV/cross-chain/ConfirmService security notes, added contract code disclaimer

*TrustApp Protocol - Trust as a Service for the Decentralized Service Economy*

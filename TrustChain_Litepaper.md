# TrustChain: Trust as a Service (TaaS) Protocol for the Decentralized Service Economy

**Status**: Draft / Proposal  
**Version**: v0.3  
**Date**: December 2025  
**Author**: 0xwilsonwu (co-authored draft)  
**Deployment Target**: Base (EVM compatible)

---

## Abstract

Blockchains excel at value transfer, but they still fail at service exchange: the chain can prove payment occurred, yet cannot guarantee service was delivered. Existing approaches often rely on reputation or centralized arbitration, which is insufficient for high-value, one-off, or cross-border service transactions.

TrustChain proposes an application-layer trust protocol that does not modify base-layer consensus. It introduces standardized service transaction types, off-chain receipts, and on-chain arbitration to create verifiable service settlement. Built on Base as the settlement layer, TrustChain uses EIP-712 typed data, pluggable arbitration modules, and economic penalties to make service delivery and payment settlement enforceable in untrusted environments.

### Core Innovations

| Innovation | Description |
|-----------|-------------|
| **Recognizable service transaction types** | Service payments go through EntryPoint, clearly separated from normal transfers |
| **Standardized receipt model** | Service calls use ServiceRequest and UsageReceipt; support batching and disputes |
| **Pluggable arbitration framework** | Committees, courts, TEE, ZK, oracles can all serve as arbitrators |
| **Economic alignment** | Bonds, challenge windows, and slashing incentivize honest behavior |
| **Prepaid-only model** | Services are paid from locked or pre-deposited funds, preventing non-payment |
| **Optimistic settlement** | Providers can withdraw early with sufficient stake; challenges and slashing preserve safety |

This paper describes TrustChain's architecture, transaction and receipt formats, arbitration flow, incentive model, security boundaries, and a migration path for x402-style payments.

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
9. [Typical Applications](#9-typical-applications)
10. [x402 Migration Path](#10-x402-migration-path)
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

### 1.4 TrustChain Positioning

| Dimension | x402 | ERC-8004 | TrustChain |
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

TrustChain aims to:
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
|                        TrustChain Protocol                       |
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

TrustChain is not a new L1/L2. It is a Credit Execution Environment built on Base. CEE only processes transactions that follow the TaaS protocol and assigns them credit states (arbitrable, slashable, settleable).

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

`maxAmount` must not exceed the payer's locked balance. Providers should check remaining balance and authorization before service execution.

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

To prevent spam and forgery, TrustChain adds explicit constraints:

| Constraint | Description |
|-----------|-------------|
| **EIP-712 binding** | All ServiceTx use fixed domain separator (name=TrustChainEntryPoint, version=1, chainId, verifyingContract) |
| **Magic prefix** | ServiceRequest signatures include `TRUST_V1_` prefix |
| **SDK enforcement** | Official SDKs output standardized structures and prefixes |
| **Sequencer tagging (future)** | L2 sequencers can tag compliant transactions for prioritization |

---

## 4. Core Flows

### 4.1 Prepaid Escrow Model (Only Supported Mode)

TrustChain only supports prepaid or locked funds. The payer must deposit or lock balances in Settlement. Providers can only settle within that limit.

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
4. Provider submits ServiceTx(kind=SETTLE)
5. Challenge window begins
6. If undisputed, settlement finalizes; otherwise arbitration

### 4.3 E-commerce Delivery Flow (Extended)

- Use logistics signature, third-party witness, or oracle data as responseHash
- Support staged delivery and partial settlement

### 4.4 Optimistic Settlement (Withdraw-Then-Slash)

To solve provider liquidity, the protocol allows early withdrawal if stake is sufficient:

1. Provider submits UsageReceipt and has stake >= minStakeMultiple * amount (e.g. 2x)
2. Provider can withdraw advanceRate (e.g. 80%) immediately
3. Remaining funds and equivalent stake are locked until challengeWindow ends
4. If a challenge succeeds, early withdrawal is clawed back and stake is slashed

#### 4.4.1 Game-Theoretic Safety Analysis

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

If arbitrators are offline, disputes could stall. TrustChain adds layered timeouts:

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
        _executeDecision(disputeId, Outcome.SPLIT);
    }
}
```

Guarantee: even if the arbitration layer fails, funds do not stay locked forever.

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
For all request:
  count(settle(tx) where tx.requestId = request.requestId) <= 1
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

Traditional double-spend requires two confirmations of the same value transfer. TrustChain runs on Base, with funds locked in a single Settlement contract. Base provides atomic global ordering.

```
TrustChain and double-spend (conceptual):

Traditional double-spend:
  Alice -> Bob and Alice -> Carol (same funds)

TrustChain:
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
| **Lock on request** | Lock funds when request is created | high-value |
| **On-chain check** | Provider calls `checkAndLock()` before execution | mid-value |
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

Not applicable in v0.3 (Base only). Multi-chain expansion will require dedicated designs.

**Summary**

| Type | Risk | Mitigation | Status |
|------|------|------------|--------|
| On-chain double spend | None | Base global state | solved |
| Receipt double claim | Low | receiptId uniqueness | solved |
| Request double use | Low | provider binding | solved |
| Balance race | Medium | lock or check | solved |
| Cross-chain | N/A | out of scope | future |

### 7.5 Security and Risk Summary

| Risk | Mitigation |
|------|------------|
| Forged receipts | Signature checks and challenges |
| Non-cooperation | Bonds and default outcomes |
| Arbitrator corruption | Replaceable modules and governance |
| Replay attacks | nonce + expiry + requestId |
| Data availability | content-addressed evidence |
| Payment default | prepaid-only model |
| Optimistic risk | stake requirements and challenge activity |
| Arbitrator offline | timeout + forceFinalize() |

### 7.6 Protocol Boundaries

**What TrustChain can solve**

| Scenario | Verification |
|----------|--------------|
| Provider takes payment but does nothing | No receipt -> payer wins |
| Provider submits false metering data | TEE/ZK proof |
| Objective delivery disputes | Evidence comparison |

**What TrustChain cannot solve**

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

### 8.4 Governance Token (Optional)

If a token is introduced:

| Function | Description |
|----------|-------------|
| Parameter votes | fee rates, windows, slash multiple |
| Arbitrator elections | committee selection |
| Treasury allocation | protocol income usage |
| Module approvals | approve new modules |

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

---

## 10. x402 Migration Path

TrustChain provides a compatibility layer for existing x402 users.

### 10.1 HTTP Header Extensions

**Original x402**:
```http
GET /resource HTTP/1.1
X-PAYMENT: <EIP-3009 signature>
```

**TrustChain enhanced**:
```http
GET /resource HTTP/1.1
X-PAYMENT: <EIP-3009 signature>
X-TRUSTCHAIN-REQUEST: <ServiceRequest hash>
X-TRUSTCHAIN-POLICY: <policyId>
X-TRUSTCHAIN-RECEIPT: <previous receipt hash>  // optional
```

### 10.2 Progressive Migration Stages

```
Migration Path

Stage 0 -> Stage 1 -> Stage 2 -> Stage 3
Pure x402   x402 + Receipt   x402 + Escrow   Full TrustChain

Stage 0: instant settle, no protection, trust provider
Stage 1: receipts, audit trail, partial verify
Stage 2: escrow + challenge, fund safety
Stage 3: arbitration, full enforcement
```

| Stage | Change | Protection | Use case |
|-------|--------|------------|----------|
| **Stage 0** | pure x402 | none | trusted provider |
| **Stage 1** | + TrustChain receipt | traceable | auditability |
| **Stage 2** | + TrustChain escrow | fund safety | mid-value |
| **Stage 3** | full TrustChain | full game | high-value, untrusted |

### 10.3 Protocol Handshake Sequence

```
Client -> Server: GET /resource
Server -> Client: 402 Payment Required + X-TRUSTCHAIN-SUPPORTED
Client -> TrustChain: deposit(amount)
Client -> Server: GET /resource + X-PAYMENT + X-TRUSTCHAIN-REQUEST
Server -> TrustChain: verify request
Server: execute service
Server -> Client: 200 OK + X-TRUSTCHAIN-RECEIPT
Server -> TrustChain: settle(receipt)
```

### 10.4 SDK Compatibility Layer

```typescript
import { TrustChainClient } from "@trustchain/sdk";

const client = new TrustChainClient({
  mode: "x402-compat",
  fallback: "pure-x402",
  autoUpgrade: true,
});

const response = await client.fetch("https://api.example.com/resource", {
  payment: {
    amount: "1000000", // 1 USDC
    token: "USDC",
  },
  trustchain: {
    escrow: true,
    policy: "default-api",
  },
});
```

### 10.5 Migration Benefits

| Dimension | pure x402 | TrustChain |
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

---

## 12. Conclusion

TrustChain is not a new L1. It is a trust protocol deployed on existing chains. By introducing recognizable transaction types, verifiable receipts, and pluggable arbitration, TrustChain provides an enforceable, extensible foundation for API billing and the service economy.

TrustChain complements x402 and ERC-8004:
- x402 as the payment trigger layer
- ERC-8004 as the identity and discovery layer
- TrustChain as the settlement enforcement layer

As verification and arbitration modules mature, TrustChain becomes a core bridge between on-chain funds and off-chain services.

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

Best for:
- logistics delivery
- IoT sensor validation
- external API dependency

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

### B.1 Data Structures

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

        if (tx_.kind == uint8(TrustChainTypes.TxKind.PAY_ESCROW)) {
            ISettlement(settlement).deposit(tx_.payer, tx_.token, tx_.amount);
        } else if (tx_.kind == uint8(TrustChainTypes.TxKind.SETTLE)) {
            ISettlement(settlement).settle(
                tx_.payer, tx_.provider, tx_.token, tx_.amount, tx_.requestHash
            );
        }

        emit ServiceTxExecuted(
            tx_.requestHash, tx_.kind, tx_.payer, tx_.provider, tx_.amount
        );
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

    mapping(address => mapping(address => TrustChainTypes.EscrowAccount)) public escrows;
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

        uint256 fee = (ps.amount * protocolFeeBps) / 10000;
        uint256 providerAmount = ps.amount - fee;

        escrows[ps.payer][ps.token].locked -= ps.amount;
        IERC20(ps.token).safeTransfer(ps.provider, providerAmount);

        ps.finalized = true;
    }

    function advanceWithdraw(bytes32 requestHash) external nonReentrant {
        PendingSettlement storage ps = pendingSettlements[requestHash];
        require(msg.sender == ps.provider, "Only provider");

        uint256 stake = IStakePool(stakePool).getStake(ps.provider, ps.token);
        require(stake >= ps.amount * 2, "Insufficient stake");

        uint256 advance = (ps.amount * 80) / 100;
        IStakePool(stakePool).lockStake(ps.provider, ps.token, ps.amount * 2);
        IERC20(ps.token).safeTransfer(ps.provider, advance);
    }
}
```

### B.4 Deployment Script

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "forge-std/Script.sol";

contract DeployTrustChain is Script {
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

## Appendix D: Glossary

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

---

**Document Version**: v0.3  
**Last Updated**: December 2025  
**Change Log**:
- v0.1: initial draft
- v0.2: added arbitration flow details
- v0.3: added security model, game analysis, x402 migration, finality bounds, Sybil defense

*TrustChain Protocol - Trust as a Service for the Decentralized Service Economy*

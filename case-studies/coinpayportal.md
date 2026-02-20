# CoinPayPortal — Multi-Chain Payment Infrastructure

## A. Summary

CoinPayPortal is a non-custodial cryptocurrency payment gateway I designed and built that enables merchants to accept payments across 7 blockchains (BTC, BCH, ETH, POL, SOL, USDC on 4 chains) plus Lightning Network and Stripe card payments.

I personally owned and designed:
- The entire system architecture from zero
- HD wallet derivation and key management layer
- Payment forwarding and fee extraction pipeline
- Escrow service with dispute resolution
- x402 machine-payment protocol implementation (the only multi-chain x402 facilitator)
- DID-based portable reputation system anchored in real economic activity
- Browser-based non-custodial web wallet
- Lightning integration via CLN/LNbits with BOLT12 offers

**Live at [coinpayportal.com](https://coinpayportal.com)**

---

## B. Problem & Requirements

### Problem
Merchants need to accept crypto payments without custodial risk, across multiple chains, with a single integration. Simultaneously, the emerging AI agent economy needs programmatic payment rails — agents that can create wallets, pay for APIs, and settle escrows autonomously.

### Functional Requirements
- Accept payments on BTC, BCH, ETH, POL, SOL, USDC (ETH/POL/SOL/Base)
- Generate unique forwarding addresses per payment (non-custodial)
- Automatic payment detection, confirmation tracking, and forwarding to merchant wallets
- Platform fee extraction (0.5%) during forwarding
- Escrow with token-based auth (no accounts needed for participants)
- Web wallet: instant creation, no signup, no KYC, client-side encryption
- x402: HTTP-native machine payments — paywall any API route with HTTP 402
- Lightning: sub-second settlement via BOLT12 offers

### Non-Functional Requirements
- **Security**: Private keys encrypted at rest (AES-256-GCM), never exposed via API
- **Reliability**: Payments must not be lost — idempotent forwarding, retry logic
- **Multi-tenancy**: Single platform serves multiple merchants with isolated businesses
- **Auditability**: Every payment state transition logged, webhook delivery tracked
- **Portability**: Reputation travels with merchant identity across platforms (DID-based)

---

## C. Constraints

### Team & Resources
- Solo architect / developer — every design decision optimizes for maintainability by one person
- No dedicated ops team — infrastructure must be low-maintenance (Supabase managed DB, Vercel serverless)
- Lean budget — no HSMs, no dedicated key management services (yet)

### Risk & Compliance
- **Non-custodial architecture is a hard constraint** — holding funds creates regulatory exposure. Forwarding addresses receive and immediately forward; platform never holds balances.
- **Key custody**: Master mnemonic in env vars, derived keys encrypted in DB. HSM/KMS is a known upgrade path but not justified at current scale.
- **Fraud surface**: Escrow disputes, webhook spoofing, address reuse, replay attacks
- **No fiat custody**: Stripe Connect handles card-side compliance; we never touch card numbers

### Operational
- Must work with unreliable RPC providers — chain nodes go down, responses lag
- Webhook delivery must be reliable (retry with backoff) — merchants depend on callbacks
- Gas prices fluctuate wildly — forwarding must handle variable transaction costs
- Lightning node management: CLN requires a synced Bitcoin Core backend

### Vendor Dependencies
- Supabase (PostgreSQL + auth + RLS)
- Vercel (serverless compute + cron)
- Chain RPC providers (Alchemy, public nodes)
- LNbits/CLN for Lightning
- Stripe Connect for card payments

---

## D. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                              │
│  ┌──────────┐  ┌──────────────┐  ┌─────────┐  ┌─────────────┐ │
│  │ Merchant  │  │  Web Wallet  │  │ x402    │  │  Escrow     │ │
│  │ Dashboard │  │  (Browser)   │  │ Client  │  │  Parties    │ │
│  └─────┬─────┘  └──────┬───────┘  └────┬────┘  └──────┬──────┘ │
└────────┼───────────────┼───────────────┼──────────────┼─────────┘
         │               │               │              │
┌────────▼───────────────▼───────────────▼──────────────▼─────────┐
│                    API Layer (Next.js)                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
│  │ Payment  │  │ Wallet   │  │ x402     │  │ Escrow         │  │
│  │ Routes   │  │ Routes   │  │ Verify/  │  │ Routes         │  │
│  │          │  │          │  │ Settle   │  │                │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └───────┬────────┘  │
│       │              │              │                │           │
│  ┌────▼──────────────▼──────────────▼────────────────▼────────┐ │
│  │              Service Layer                                  │ │
│  │  ┌─────────────┐  ┌──────────────┐  ┌───────────────────┐ │ │
│  │  │ Wallet Gen  │  │ Blockchain   │  │ Reputation/DID    │ │ │
│  │  │ (HD Derive) │  │ Monitor      │  │ (Trust Vectors)   │ │ │
│  │  └──────┬──────┘  └──────┬───────┘  └────────┬──────────┘ │ │
│  │         │                │                    │            │ │
│  │  ┌──────▼────────────────▼────────────────────▼──────────┐ │ │
│  │  │           Payment Forwarding Engine                    │ │ │
│  │  │  (Detect → Confirm → Fee Extract → Forward → Webhook) │ │ │
│  │  └──────────────────────┬─────────────────────────────────┘ │ │
│  └─────────────────────────┼───────────────────────────────────┘ │
└────────────────────────────┼─────────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
    ┌────▼────┐      ┌──────▼──────┐     ┌──────▼──────┐
    │Supabase │      │ Blockchains │     │  Lightning  │
    │ (PG+RLS)│      │ BTC BCH ETH │     │  CLN/LNbits │
    │         │      │ POL SOL USDC│     │  BOLT12     │
    └─────────┘      └─────────────┘     └─────────────┘
```

### Subsystem Breakdown

| Subsystem | Responsibility | Isolation Boundary |
|-----------|---------------|-------------------|
| **Wallet Generation** | HD derivation from master mnemonic per chain (BIP44/Ed25519) | Encryption key in env; derived keys encrypted per-row |
| **Blockchain Monitor** | Polls RPCs for incoming transactions, tracks confirmations | Stateless — reads pending payments from DB, checks chain |
| **Payment Forwarding** | Deducts fee, forwards to merchant wallet, retries on failure | Idempotent — checks `forward_tx_hash` before sending |
| **Escrow** | Holds funds in on-chain addresses with token-based release | Separate auth domain — no account required |
| **x402 Facilitator** | Verifies HTTP 402 payment proofs, settles on-chain | Stateless verification; settlement reuses forwarding engine |
| **Reputation/DID** | Issues signed ActionReceipts from escrow settlements | Read-only from payment domain; write-only from escrow events |
| **Web Wallet** | Client-side key generation, encrypted backup, tx signing | Keys never leave browser; server only stores encrypted blobs |
| **Lightning** | BOLT12 offers, invoice creation, payment settlement | Separate CLN node; communicates via LNbits API |
| **Stripe Connect** | Card payment processing, merchant onboarding | Completely separate domain — Stripe handles compliance |

---

## E. Key Design Decisions

### 1. Non-Custodial Forwarding Architecture

**Decision:** Generate a unique HD-derived address per payment. Customer pays to that address. System detects payment, waits for confirmations, deducts fee, forwards remainder to merchant's wallet.

**Why:** Avoids custodial regulatory burden. Platform never holds balances — funds are in transit only during the confirmation window.

**Tradeoff:** More complex than pooled wallets. Each payment needs its own address, monitoring, and forwarding transaction (gas costs). Can't net payments.

**What I'd revisit:** At scale, batch forwarding (aggregate multiple payments into one on-chain tx) would reduce gas costs significantly on EVM chains.

### 2. Single Mnemonic, Multi-Chain Derivation

**Decision:** One BIP-39 mnemonic derives addresses for all chains using standard derivation paths (BIP44 for BTC/ETH/POL, Ed25519 for SOL).

**Why:** Operational simplicity. One secret to back up, rotate, and protect. Standard derivation means any compatible wallet can recover funds.

**Tradeoff:** Single point of compromise — if the mnemonic leaks, all chains are exposed. Blast radius is the entire platform.

**What I'd revisit:** Per-chain mnemonics or HSM-backed key derivation to limit blast radius. At serious scale, each chain should be its own key domain.

### 3. LNbits + CLN Over Running LND Directly

**Decision:** Use LNbits as the Lightning application layer on top of Core Lightning (CLN), rather than running LND with custom invoice management.

**Why:** LNbits provides tenant isolation (each merchant gets a wallet), a clean REST API, and BOLT12 support out of the box. CLN is lighter than LND and has better plugin architecture.

**Tradeoff:** Extra dependency (LNbits) in the stack. LNbits is less battle-tested at scale than raw LND. BOLT12 ecosystem is still maturing.

**What I'd revisit:** If Lightning volume exceeds ~10k payments/day, evaluate direct CLN RPC integration to eliminate LNbits overhead.

### 4. Escrow with Token-Based Auth (No Accounts)

**Decision:** Escrow participants authenticate via unique tokens in URLs (`/escrow/manage?id=xxx&token=yyy`), not platform accounts.

**Why:** Lowers friction to zero. Buyers and sellers don't need to create accounts to use escrow. Shareable links work across any device.

**Tradeoff:** Token management complexity. Tokens must be unguessable, expire appropriately, and map to specific escrow roles (buyer/seller). No session persistence.

**What I'd revisit:** Hybrid approach — allow optional account linking for repeat users while keeping token-based access as the default.

### 5. DID Reputation Anchored in Escrow Settlements

**Decision:** Reputation scores are computed from signed `ActionReceipt` credentials generated when escrows settle. 7 trust dimensions with diminishing returns and 90-day recency decay.

**Why:** Star ratings are subjective and gameable. Anchoring reputation in real money movement (escrow settlements) makes it expensive to fake. DID portability means reputation travels across platforms.

**Tradeoff:** Cold start problem — new merchants have no reputation. Only works where escrow is used, not for direct payments. Anti-gaming (circular payment detection) adds computational overhead.

**What I'd revisit:** Allow direct payments to contribute lightweight reputation signals (payment volume, reliability) alongside the heavier escrow-based trust vectors.

### 6. x402 as the Machine Payment Protocol

**Decision:** Implement the x402 HTTP payment protocol as a multi-chain facilitator. Merchants add middleware to routes; clients use `x402fetch()` which handles 402 → pay → retry automatically.

**Why:** AI agents need programmatic payment rails. x402 turns any HTTP endpoint into a paid API. CoinPay is the only facilitator supporting all major chains + Lightning + Stripe in one integration.

**Tradeoff:** x402 is an emerging standard with limited adoption. Building the "only multi-chain facilitator" is a bet on the protocol gaining traction.

**What I'd revisit:** If x402 doesn't gain adoption, the facilitator infrastructure can pivot to a proprietary pay-per-call API system with minimal changes.

### 7. Stripe Connect as a Separate Domain

**Decision:** Card payments via Stripe Connect are architecturally separate from crypto payments. Different tables, different auth flows, different settlement paths.

**Why:** Mixing crypto and fiat payment logic creates compliance nightmares. Stripe handles PCI-DSS, chargebacks, and KYC. Keeping it isolated means a Stripe outage doesn't affect crypto payments and vice versa.

**Tradeoff:** Merchants see a unified dashboard but the backend is two distinct payment systems. More code to maintain.

**What I'd revisit:** The checkout page already unifies the UX. Backend separation is the right call — I'd keep it.

### 8. Supabase RLS for Multi-Tenant Isolation

**Decision:** Row Level Security policies on all tables ensure merchants can only access their own data. Service role key used only in server-side routes.

**Why:** Defense in depth. Even if an API route has a bug, RLS prevents cross-tenant data access at the database level.

**Tradeoff:** RLS policies add complexity to migrations and can be tricky to debug. Performance impact on complex queries.

**What I'd revisit:** At high scale, evaluate whether RLS overhead justifies the security benefit vs. application-level tenant filtering with aggressive testing.

---

## F. Failure Modes & Blast Radius

| Failure | Blast Radius | Mitigation |
|---------|-------------|------------|
| **Master mnemonic compromised** | All chains, all merchant forwarding addresses | Env-var isolation; HSM upgrade path; key rotation procedure documented |
| **Supabase outage** | All operations — no payments created, no status updates | Payment detection is idempotent; resumes on recovery. Webhook delivery queued with retries. |
| **Chain RPC provider down** | Payments on that chain not detected/forwarded | Multiple RPC fallbacks per chain; payments don't expire (wait indefinitely for confirmation) |
| **LNbits/CLN down** | Lightning payments unavailable; on-chain unaffected | Graceful degradation — checkout falls back to on-chain options |
| **Stripe webhook delays** | Card payment status updates delayed | Polling fallback on payment status page; Stripe retries webhooks automatically |
| **Gas price spike (EVM)** | Forwarding transactions stuck or expensive | Gas price oracle with configurable max; queue forwarding until gas normalizes |
| **Payment forwarding fails** | Funds stuck in forwarding address | Retry with exponential backoff; manual intervention dashboard; funds are safe (non-custodial address) |
| **Webhook delivery failure** | Merchant doesn't get payment notification | Retry up to 5 times with exponential backoff; webhook logs for debugging; merchant can poll status |
| **Double-spend attempt (BTC/BCH)** | Merchant credited for unconfirmed tx | Require 3+ confirmations before forwarding; replace-by-fee detection |
| **Web wallet encryption key lost** | User loses access to wallet funds | Seed phrase backup (BIP-39); clear UX warning during wallet creation |

---

## G. Scaling to 10x / 100x

### What Breaks First at 10x (~5,000 payments/day)

1. **Blockchain polling** — Polling RPCs per-payment doesn't scale. Move to WebSocket subscriptions or indexer-based event streaming.
2. **Payment forwarding** — Sequential forwarding hits gas limits and RPC rate limits. Batch forwarding on EVM chains (multiple payments in one tx).
3. **Supabase connection pooling** — Serverless functions opening DB connections per-request. PgBouncer is already enabled but may need tuning.

### Architecture at 100x (~50,000 payments/day)

```
Payment Events          Processing             Settlement
┌──────────┐    ┌────────────────────┐    ┌──────────────┐
│ Chain     │───▶│ Event Queue        │───▶│ Forwarding   │
│ Indexers  │    │ (Redis/NATS/SQS)   │    │ Workers      │
│ (per      │    │                    │    │ (per-chain)  │
│ chain)    │    │ Dedup + Ordering   │    │              │
└──────────┘    └────────────────────┘    └──────────────┘
```

- **Chain indexers** replace polling: dedicated indexer per chain streaming events into a queue
- **Event queue** (Redis Streams or SQS): decouples detection from forwarding, enables backpressure
- **Per-chain forwarding workers**: independent scaling per chain based on volume
- **Read replicas**: Supabase read replica for dashboard/analytics queries
- **Partitioning**: Partition `payments` table by `created_at` month for query performance

### Observability at Scale

| SLI | SLO | How to Measure |
|-----|-----|----------------|
| Payment detection latency | < 30s after chain confirmation | `detected_at - confirmed_at` |
| Forwarding completion | < 5min after detection | `forwarded_at - detected_at` |
| Webhook delivery success | > 99.5% first attempt | `webhook_logs` success rate |
| API response time (p99) | < 500ms | APM instrumentation |
| Key derivation throughput | > 100 addresses/sec | Load test wallet generation |

### Team Scaling
At 100x, split into services:
- **Payment Core**: Detection, forwarding, fee extraction (owned by 1-2 engineers)
- **Lightning Service**: CLN management, BOLT12, channel management (1 engineer)
- **Identity/Reputation**: DID issuance, trust computation, ActionReceipts (1 engineer)
- **Platform/API**: Dashboard, merchant management, onboarding (1 engineer)

Keep as modular monolith until team exceeds 4-5 engineers, then extract along these boundaries.

---

## H. What I'd Do Next With 2–3 Engineers

### Security Hardening (Engineer 1, Q1)
- HSM/KMS integration for master key (AWS KMS or Hashicorp Vault)
- Per-chain key isolation (separate mnemonics per chain)
- Automated key rotation with zero-downtime migration
- Penetration testing and bug bounty program
- Rate limiting hardening (persistent rate limiters already in place for web wallet)

### Reliability & Observability (Engineer 2, Q1-Q2)
- Chain indexer infrastructure (replace polling)
- Event queue for payment processing pipeline
- Structured logging + distributed tracing (OpenTelemetry)
- SLI/SLO dashboards with alerting
- Automated forwarding retry with dead-letter queue
- Chaos testing: simulate RPC failures, DB outages

### Performance & Scaling (Engineer 3, Q2)
- Batch forwarding on EVM chains
- Payment table partitioning
- Connection pool optimization
- CDN for static assets and payment page
- Load testing suite targeting 10x current volume

### Developer Experience (Ongoing)
- SDK improvements: React hooks, Python SDK, Go SDK
- Improved local dev environment (chain simulators)
- CI/CD hardening: automated DB migration testing
- API versioning strategy

---

## I. Links

### Architecture & Design
- [Architecture Overview](https://github.com/profullstack/coinpayportal/blob/main/docs/ARCHITECTURE.md)
- [Database Schema](https://github.com/profullstack/coinpayportal/blob/main/docs/DATABASE.md)
- [Security Documentation](https://github.com/profullstack/coinpayportal/blob/main/docs/SECURITY.md)
- [DID Reputation System](https://github.com/profullstack/coinpayportal/blob/main/docs/DID_REPUTATION_SYSTEM.md)

### API & SDK
- [API Documentation](https://github.com/profullstack/coinpayportal/blob/main/docs/API.md)
- [Lightning API](https://github.com/profullstack/coinpayportal/blob/main/docs/api/lightning-api.md)
- [Web Wallet API](https://github.com/profullstack/coinpayportal/blob/main/docs/api/web-wallet-api.md)
- [SDK Getting Started](https://github.com/profullstack/coinpayportal/blob/main/docs/sdk/getting-started.md)
- [x402 Integration](https://github.com/profullstack/coinpayportal/blob/main/docs/X402_INTEGRATION.md)

### Decision Records
- [Credit Card Processing Decision](https://github.com/profullstack/coinpayportal/blob/main/docs/CREDIT_CARD_DECISION.md)
- [Escrow Design](https://github.com/profullstack/coinpayportal/blob/main/docs/ESCROW.md)
- [Platform Integration Guide](https://github.com/profullstack/coinpayportal/blob/main/docs/PLATFORM_INTEGRATION.md)

### Web Wallet Deep Dive
- [Overview](https://github.com/profullstack/coinpayportal/blob/main/docs/web-wallet/01-OVERVIEW.md)
- [Architecture](https://github.com/profullstack/coinpayportal/blob/main/docs/web-wallet/02-ARCHITECTURE.md)
- [Database Schema](https://github.com/profullstack/coinpayportal/blob/main/docs/web-wallet/03-DATABASE-SCHEMA.md)
- [Key Management](https://github.com/profullstack/coinpayportal/blob/main/docs/web-wallet/06-KEY-MANAGEMENT.md)
- [Security](https://github.com/profullstack/coinpayportal/blob/main/docs/web-wallet/11-SECURITY.md)

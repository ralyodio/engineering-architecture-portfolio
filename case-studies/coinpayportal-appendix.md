# CoinPayPortal — Technical Appendix

## Diagram Index

| Diagram | Location | Description |
|---------|----------|-------------|
| System Architecture (Mermaid) | [ARCHITECTURE.md](https://github.com/profullstack/coinpayportal/blob/main/docs/ARCHITECTURE.md) | High-level component diagram |
| Payment Flow (Sequence) | [ARCHITECTURE.md](https://github.com/profullstack/coinpayportal/blob/main/docs/ARCHITECTURE.md) | Customer → Chain → Forward → Webhook |
| Entity Relationship | [DATABASE.md](https://github.com/profullstack/coinpayportal/blob/main/docs/DATABASE.md) | Full ER diagram (Mermaid) |
| Deployment Architecture | [ARCHITECTURE.md](https://github.com/profullstack/coinpayportal/blob/main/docs/ARCHITECTURE.md) | Vercel + Supabase + RPC topology |
| DID Reputation Flow | [DID_REPUTATION_SYSTEM.md](https://github.com/profullstack/coinpayportal/blob/main/docs/DID_REPUTATION_SYSTEM.md) | Escrow → ActionReceipt → Trust Vector |
| Web Wallet Architecture | [web-wallet/02-ARCHITECTURE.md](https://github.com/profullstack/coinpayportal/blob/main/docs/web-wallet/02-ARCHITECTURE.md) | Client-side encryption & key flow |
| Web Wallet Key Management | [web-wallet/06-KEY-MANAGEMENT.md](https://github.com/profullstack/coinpayportal/blob/main/docs/web-wallet/06-KEY-MANAGEMENT.md) | HD derivation & encryption details |

## Technology Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| Frontend | Next.js 14 (App Router), TypeScript, TailwindCSS | SSR + serverless API routes in one codebase |
| Database | Supabase (PostgreSQL + RLS) | Managed PG with row-level security for multi-tenancy |
| Auth | Supabase Auth + JWT | Built-in, works with RLS policies |
| Crypto | bitcoinjs-lib, ethers.js, @solana/web3.js | Standard chain libraries |
| Key Derivation | @scure/bip32, bip39 | HD wallet (BIP44) from single mnemonic |
| Encryption | Node.js crypto (AES-256-GCM) | Private key encryption at rest |
| Lightning | CLN + LNbits | BOLT12 support, tenant isolation, REST API |
| Card Payments | Stripe Connect | PCI compliance handled by Stripe |
| Hosting | Vercel (serverless) | Auto-scaling, zero-config deploys |
| Testing | Vitest + Testing Library + Playwright | Unit + integration + E2E |

## Database Stats (Production, Feb 2026)

- ~525 payment addresses generated
- ~41 business wallets
- ~779 web wallets created
- ~27 web wallet transactions
- Multi-chain address distribution across BTC, BCH, ETH, POL, SOL, USDC variants

## Migration History

48 migrations covering:
- Core payment tables and RLS policies
- Escrow system with recurring support
- Web wallet with client-side encryption
- DID reputation with ActionReceipts
- x402 facilitator tables
- Stripe Connect integration
- Security hardening (RLS on all tables, search path fixes)

## Repo Structure

```
coinpayportal/
├── src/
│   ├── app/              # Next.js App Router pages + API routes
│   ├── lib/
│   │   ├── wallets/      # HD wallet generation & encryption
│   │   ├── blockchain/   # Chain-specific RPC interactions
│   │   ├── escrow/       # Escrow service layer
│   │   ├── lightning/    # LNbits integration
│   │   ├── reputation/   # DID + trust vector computation
│   │   ├── web-wallet/   # Browser wallet service
│   │   └── x402/         # x402 facilitator
│   └── components/       # React components
├── supabase/
│   └── migrations/       # 48 idempotent SQL migrations
├── docs/                 # Architecture, security, API docs
└── sdk/                  # Client SDK for merchants
```

# Engineering Architecture Portfolio — Anthony Ettinger

I design and build production payment systems, media infrastructure, and AI-native SaaS platforms. 17+ products shipped as an indie maker / solo architect.

This repo contains curated architecture case studies — problem, constraints, decisions, tradeoffs, and scaling paths.

## Case Studies

### [CoinPayPortal — Multi-Chain Payment Infrastructure](case-studies/coinpayportal.md)

Non-custodial crypto payment gateway supporting 7 blockchains, Lightning Network (BOLT12), escrow, a browser-based web wallet, x402 machine-payment protocol, DID-based portable reputation, and Stripe Connect for card payments.

**Domains:** Payment processing · Key management · Multi-chain orchestration · Identity/reputation · Escrow · Machine payments

- [Full case study](case-studies/coinpayportal.md)
- [Technical appendix & diagram index](case-studies/coinpayportal-appendix.md)
- [Implementation source (Profullstack org)](https://github.com/profullstack/coinpayportal)

### [BitTorrented — High-Volume Media Platform with Profile Isolation](case-studies/bittorrented.md)

Netflix-style multi-profile streaming platform with 6.5M+ indexed torrents, IPTV integration, podcast/radio aggregation, on-demand transcoding, and strict per-profile data isolation.

**Domains:** Content indexing at scale · Profile isolation · Media streaming · On-demand transcoding · Real-time search · DHT crawling

---

## Engineering Principles

- **Design around failure domains and blast radius** — every subsystem degrades independently
- **Separate payment, identity, and state domains** — never couple money movement with user management
- **Prefer simple abstractions that scale over brittle one-offs** — HD wallet derivation > per-chain key generation
- **Make tradeoffs explicit** — document what you chose, what you rejected, and why
- **Instrument first** — logs/metrics/traces before "optimizing"
- **Non-custodial by default** — if you can avoid holding funds, do

---

## About

**Anthony Ettinger** · [GitHub](https://github.com/profullstack) · [LinkedIn](https://linkedin.com/in/aettinger) · anthony@chovy.com

20+ years building web infrastructure. Currently focused on payments, identity, and AI-agent-native platforms.

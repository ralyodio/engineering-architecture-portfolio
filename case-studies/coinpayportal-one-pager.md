# CoinPayPortal — Interview One-Pager

Non-custodial multi-chain payment gateway I designed and built solo. 7 blockchains + Lightning + Stripe card payments. Handles merchant onboarding, payment detection, forwarding with fee extraction, escrow, a browser-based web wallet, x402 machine payments for AI agents, and a DID-based portable reputation system. Production at [coinpayportal.com](https://coinpayportal.com).

## Hardest Tradeoffs

1. **Single mnemonic vs per-chain keys** — Chose single mnemonic for operational simplicity. Accepted the blast radius tradeoff with a documented HSM upgrade path.
2. **Non-custodial forwarding vs pooled wallets** — Forwarding adds gas costs and per-payment complexity but eliminates custodial regulatory burden entirely.
3. **LNbits + CLN vs raw LND** — Gained tenant isolation and BOLT12 support out of the box. Accepted an extra dependency and less battle-tested stack.
4. **Token-based escrow auth vs account-required** — Zero-friction UX for escrow parties at the cost of token management complexity and no session persistence.
5. **Supabase RLS vs application-level tenant filtering** — Defense-in-depth at the DB level. Accepted migration complexity and query performance overhead.

## Failure & Scaling Thinking

1. **Mnemonic compromise** is the highest-severity failure — mitigated by env-var isolation today, HSM planned.
2. **RPC polling doesn't scale** past ~5k payments/day — would replace with chain indexers + event queue.
3. **Sequential forwarding** hits gas limits on EVM — batch forwarding is the 10x solution.
4. **Supabase is a single point** — read replicas for analytics, payment table partitioning by month.
5. **Lightning depends on CLN sync** — graceful fallback to on-chain; channels need liquidity management at scale.

## What I'd Do Next

1. **HSM/KMS for key management** — eliminate the single-mnemonic blast radius
2. **Event-driven payment pipeline** — chain indexers → message queue → forwarding workers (per-chain)
3. **Observability** — OpenTelemetry tracing, SLI/SLO dashboards, automated alerting on forwarding failures

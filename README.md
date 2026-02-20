# Anthony Ettinger

## AI Infrastructure & Payments Systems Architect

I design and build production systems in payments, identity, and AI-native SaaS platforms. The case studies below document real systems I've designed, deployed, and operate in production. No hype. No buzzwords. No startup fluff.

---

## Selected Architecture Case Studies

### 1. Multi-Chain Payment Infrastructure

**System: [CoinPayPortal](https://coinpayportal.com)**

#### System Overview

- **Wallet abstraction layer** spanning ETH, SOL, BTC, BCH, DOGE, XRP, ADA, BNB, and stablecoins (USDT, USDC across multiple chains)
- **HD wallet derivation** — system-owned wallets generate unique per-payment addresses from a single mnemonic, enabling deterministic address generation and commission splitting
- **Lightning Network integration** via LNbits for instant BTC micropayments
- **Stripe Connect multi-tenant architecture** — parallel fiat payment rail with per-merchant onboarding
- **Escrow separation model** — isolated escrow wallets with time-locked release conditions, separate from merchant payment flow
- **DID-based identity layer** — merchant authentication via cryptographic key pairs, not passwords

#### Core Engineering Decisions

| Decision | Tradeoff |
|---|---|
| Namespace isolation for `stripe_*` tables | Prevents coupling between crypto and fiat payment domains. Adds schema complexity but eliminates cross-domain migration risk. |
| System-owned wallets over merchant-hosted | CoinPay holds custody during payment window. Enables commission extraction (0.5–1%) at the protocol level. Increases trust surface but simplifies merchant onboarding. |
| Payment domain separated from identity domain | Merchant auth, wallet management, and payment processing are independent subsystems. Any one can be replaced without cascading changes. |
| Per-payment derived addresses | Each payment gets a unique address derived from HD wallet index. Eliminates address reuse, simplifies reconciliation, increases on-chain privacy. |
| Encrypted private key storage | Payment address private keys are AES-encrypted at rest. Decryption only occurs during settlement sweep. Reduces blast radius of DB compromise. |

#### Fraud Surface Reduction

- No merchant-supplied addresses in the payment flow — all funds route through system wallets first
- Commission is deducted before merchant settlement, not after
- Webhook signature verification with replay prevention (nonce + timestamp window)
- Rate limiting on wallet creation (Supabase-backed, survives deploys)

#### Scaling Considerations

- **Queue-based transaction processing** — payment status checks and settlement sweeps are decoupled from request/response cycle
- **Horizontal DB partition strategy** — `payment_addresses` and `business_wallets` are partitioned by cryptocurrency, enabling per-chain scaling
- **Failure domain isolation** — Lightning failures don't affect on-chain payments; Stripe failures don't affect crypto
- **Observability** — structured logging with request IDs across webhook processing, payment creation, and settlement

#### If Scaling to 10x or 100x

**What breaks first:**
- `system_wallet_indexes` becomes a write bottleneck — single row per cryptocurrency with atomic increment. At high concurrency, this serializes all payment creation for a given chain.
- Settlement sweep jobs are currently sequential per chain. At 100x volume, sweep latency exceeds block confirmation windows.

**Redesign for team scaling:**
- Extract payment processing into a standalone service with its own DB. The current monolith couples UI, API, and payment logic.
- Separate the wallet derivation service from the payment orchestration service. Different security boundaries, different deployment cadences.

**Where coupling exists:**
- Merchant auth and payment creation share a Supabase client instance. A Supabase outage blocks both.
- Webhook processing and payment status checks share the same DB connection pool.

**Where tech debt accumulates:**
- `as any` casts on Supabase insert calls where generated types lag behind migrations.
- Commission rate logic is duplicated between `system-wallet.ts` and the fee calculation module.

**Where monitoring must improve:**
- No alerting on settlement sweep failures — a stuck sweep silently holds merchant funds.
- No metrics on HD wallet index growth rate — running out of derivation space is a silent failure.
- Webhook processing latency is logged but not tracked as a histogram.

---

### 2. High-Volume Metadata Ingestion System

**System: [BitTorrented](https://bittorrented.com)**

#### System Overview

- **6.5M+ torrent metadata records** ingested from DHT network via BitMagnet crawler
- **Dual-source search** — user-submitted torrents (PostgreSQL) and DHT torrents (1.8M+ rows) unified through a single search interface
- **TMDB content enrichment** — automated matching of torrent names to movie/TV metadata with poster art, cast, and genre data
- **Netflix-style profile system** — per-profile favorites, watch history, podcast subscriptions, and collections with strict data isolation
- **IPTV integration** — M3U playlist parsing, EPG guide data, and proxy streaming with HLS support

#### Core Engineering Decisions

| Decision | Tradeoff |
|---|---|
| Longest-word-first search strategy | Two-phase search: GIN index scan on most selective word, then ILIKE filter for remaining terms. 10x faster than naive `ILIKE '%query%'` on 1.8M rows. Trades query complexity for consistent sub-second results. |
| Profile ID on all user-scoped tables | Every content interaction (favorites, history, comments, votes) is keyed by `profile_id`, not `user_id`. Enables Netflix-style family sharing with zero data leakage between profiles. Required migrating 10+ tables and all API routes. |
| No profile fallback — ever | `getActiveProfileId()` returns null if no profile cookie exists. Forces explicit profile selection. Eliminated an entire class of "wrong user sees wrong data" bugs. |
| Denormalized TMDB metadata on watchlist items | Watchlist items store title, poster, overview directly rather than joining to a TMDB cache table. Trades storage for read speed and eliminates TMDB API as a runtime dependency. |
| Service-role Supabase client for all server operations | Bypasses RLS entirely. Simplifies query logic but requires application-level authorization checks on every route. |

#### Query Optimization Under Constraints

- **8-core / 15GB RAM server** — no room for in-memory caches or materialized views
- **FTS GIN index** (`idx_torrents_name_fts`) on `to_tsvector('simple', name)` for DHT torrents — `simple` config because torrent names aren't natural language
- **Trigram GIN index** (`idx_torrents_name_trgm`) as fallback for substring matching
- **Statement timeout** set to 30s on search function to prevent runaway queries from blocking the connection pool
- **Storage growth modeling**: 9GB at 5M records → projecting 18GB at 10M. Currently growing at ~100K records/day.

#### Profile System Architecture

```
account (auth.users)
  └── profiles[] (max 10 per account)
        ├── favorites (torrents, IPTV channels, files)
        ├── collections
        ├── watch history
        ├── podcast subscriptions + listen progress
        ├── radio favorites
        ├── watchlists + items
        └── comments + votes

account-level (shared across profiles):
  ├── IPTV playlists/providers
  ├── subscription/billing
  └── family plan management
```

- Middleware enforces profile selection: authenticated users without `x-profile-id` cookie are redirected to `/select-profile`
- "Set as default" is per-device (localStorage), not DB-level — different devices can have different default profiles
- Logout clears `localStorage.clear()` + both auth and profile cookies

#### If Scaling to 10x or 100x

**What breaks first:**
- DHT search at 18M+ rows. The two-phase search strategy (GIN index → ILIKE filter) breaks down when the GIN index returns too many candidates for the longest word. Common words like "the" or "2024" would return millions of candidates.
- `bt_torrent_comments` and `bt_torrent_votes` with profile-scoped queries. Currently no composite index on `(torrent_id, profile_id)` — each query does an index scan + filter.

**Redesign for team scaling:**
- Extract the search function into a dedicated search service (Meilisearch or Typesense). PostgreSQL full-text search was the right call at 1M rows, wrong call at 50M.
- Separate the ingestion pipeline (BitMagnet) from the serving database. Currently they share a Postgres instance — ingestion write load directly impacts search latency.

**Where coupling exists:**
- BitMagnet writes directly to the same Postgres instance that serves API queries. A large ingestion batch causes query timeouts.
- Profile enforcement is split between middleware (page redirects) and individual API routes (400 responses). No centralized auth/profile gate.

**Where tech debt accumulates:**
- `as any` casts on 10+ tables where Supabase generated types don't include `profile_id` columns yet.
- Dead `getUserIdFromRequest` helper functions remain in several route files after the user_id → profile_id migration.
- Comment user display shows `user@example.com` placeholder — actual profile name lookup not yet wired.

**Where monitoring must improve:**
- No metrics on search latency percentiles — can't tell when DHT search is degrading.
- No alerting on BitMagnet ingestion rate drops — crawler failures are silent.
- Profile-scoped query performance is untracked — no way to detect if one profile's data volume causes slowdowns.

---

### 3. P2P WebRTC Video Call Infrastructure

**System: [PairUX](https://pairux.com)**

#### System Overview

- **Peer-to-peer screen sharing with simultaneous remote control** — host and viewer can control the same screen at the same time, with host priority
- **WebRTC direct P2P connections** — media never touches servers. DTLS-SRTP encrypted by default.
- **Tiered scaling architecture** — P2P for 1:1/small groups, SFU (Selective Forwarding Unit) for 10–100+ viewers, cascaded multi-region SFUs for 500+
- **Supabase Realtime as signaling plane** — WebRTC offer/answer/ICE exchange without a custom signaling server
- **Self-hosted coturn TURN server** — relay fallback for restrictive NATs, fully controlled infrastructure
- **Cross-platform distribution** — Electron desktop app (macOS/Windows/Linux) + PWA browser viewer. Published via Homebrew, WinGet, APT, AUR, Scoop, Chocolatey, RPM, Nix, Gentoo overlay, and AppImage.

#### Core Engineering Decisions

| Decision | Tradeoff |
|---|---|
| Media plane / control plane separation | Media routing (P2P or SFU) and session orchestration (signaling, state, roles) are independent subsystems. They scale differently — media is bandwidth-heavy, control is consistency-heavy. Coupling them means you can't scale one without the other. |
| Client-side topology decision (MVP) | Host chooses P2P or SFU at session start. No server-side orchestrator. Eliminates a service dependency at the cost of global optimization. Server-side Session Focus service is the planned upgrade path. |
| Supabase Realtime for signaling | Eliminates custom WebSocket server. Provides auth integration, presence, and broadcast for free. Tradeoff: dependent on Supabase's realtime infrastructure for connection establishment. |
| nut.js for input injection | Cross-platform mouse/keyboard injection on the host. Required for remote control — no browser API exists for injecting OS-level input. Requires Accessibility permissions on macOS. |
| Self-hosted TURN over managed | Full control over relay infrastructure. Only used when P2P fails (~10-15% of connections due to symmetric NATs). Predictable cost vs per-minute billing. |
| Three-role model: host / controller / viewer | Host shares screen and grants control (1 per session). Controllers can view and send input (max 3). Viewers are receive-only (25 P2P / 100+ SFU). Clear permission boundaries prevent "everyone sends everything" chaos. |

#### Network Architecture

```
Host Desktop ◄──── WebRTC P2P (direct) ────► Viewer Browser
     │                                              │
     ├── ICE candidates via STUN ──────────────────┤
     │                                              │
     └── Relay via self-hosted TURN ───────────────┘
              (only when P2P fails)

Signaling (offer/answer/ICE exchange):
     Host ◄──► Supabase Realtime ◄──► Viewer
              (no media passes through)
```

#### Security Model

- **Transport encryption**: DTLS-SRTP on all WebRTC connections (default, not optional)
- **Zero server media**: Supabase handles auth and signaling only — screen data never touches backend
- **Explicit consent**: Host must approve every control request. Emergency revoke via `Ctrl+Shift+Escape`
- **E2EE upgrade path**: Insertable Streams API for true end-to-end encryption where SFU cannot decrypt. Currently transport-only; E2EE planned for enterprise/compliance use cases.

#### Scaling Architecture

| Scale | Architecture | Capacity | Monthly Cost |
|---|---|---|---|
| **Tier 1: P2P** | Direct connections, Supabase signaling, single TURN | 1 host + 25 viewers per session, ~50 concurrent sessions | ~$50–150 |
| **Tier 2: SFU** | LiveKit or mediasoup, single region | 1 host + 100+ viewers per session, ~200 concurrent sessions | ~$200–600 |
| **Tier 3: Multi-region** | Regional SFU pools, Session Focus service, bridge cascading | Global <100ms latency, automatic failover | ~$1,000–3,000 |
| **Tier 4: Enterprise** | Cascaded SFUs, E2EE, auto-scaling, 5+ regions | 1,000+ viewers per session, compliance-ready | $5,000+ |

Each tier has explicit migration triggers — you don't move up until metrics prove you need to.

#### If Scaling to 10x or 100x

**What breaks first:**
- **P2P connection limit.** At 25+ viewers, each viewer has a direct connection to the host. Host upload bandwidth becomes the bottleneck — a 4Mbps stream × 25 peers = 100Mbps upload. Most residential connections can't sustain this. SFU is mandatory at this point.
- **Supabase Realtime channel limits.** Each session is a Realtime channel. At 1,000+ concurrent sessions, channel count and message throughput hit Supabase plan limits. Need dedicated signaling or Supabase enterprise tier.
- **Single TURN server.** At high concurrent relay usage, a single coturn instance saturates its network interface. TURN traffic is full media bitrate — one relayed 4Mbps session costs 8Mbps (in + out).

**Redesign for team scaling:**
- Extract Session Focus service — server-side orchestrator that assigns SFUs, manages topology upgrades (P2P → SFU), and handles failover. Currently the host decides topology, which doesn't work when the server needs global load visibility.
- Separate the Electron desktop app and the web viewer into independent release cycles. Currently monorepo'd with Turborepo — fine for 1-2 developers, creates merge conflicts at 5+.

**Where coupling exists:**
- Signaling and auth share the same Supabase instance. A Supabase outage blocks new connections but doesn't affect established P2P sessions (by design).
- SFU selection is client-side. The host picks the SFU endpoint at session creation — no server-side load balancing or region-aware routing.

**Where tech debt accumulates:**
- No automated integration tests for the WebRTC connection flow. Unit tests mock the RTCPeerConnection — real ICE negotiation is only tested manually.
- TURN server configuration is manual. No infrastructure-as-code, no automated certificate rotation.
- Desktop app auto-updater is platform-specific with separate code paths for macOS, Windows, and Linux.

**Where monitoring must improve:**
- No metrics on ICE candidate gathering time — can't detect when TURN fallback rate increases (indicating network infrastructure issues).
- No alerting on SFU CPU/bandwidth utilization — capacity planning is manual.
- No per-session quality metrics (frame rate, resolution, latency percentiles) — can't diagnose individual user complaints.

---

## Engineering Philosophy

- **Design for failure domains first.** Every subsystem should be able to fail without cascading. Payment failures don't break identity. Search failures don't break playback.
- **Separate payment, identity, and state layers.** These change at different rates, have different security requirements, and serve different stakeholders. Coupling them is a future rewrite.
- **Prefer extensible abstractions over brittle integrations.** A wallet abstraction layer that supports 14 chains is more valuable than 14 chain-specific implementations. But the abstraction must earn its complexity.
- **Model scaling paths before optimizing prematurely.** Know what breaks at 10x before spending cycles on optimization at 1x. Document the plan, not the premature implementation.
- **Optimize for long-term maintainability over short-term velocity.** Function names don't lie (`getActiveProfileId`, not `getCurrentProfileIdWithFallback` when there is no fallback). Dead code gets deleted. Migrations get written.

---

## Implementation

Full source code and deployment infrastructure available under the [Profullstack](https://github.com/profullstack) organization.

- [CoinPayPortal](https://github.com/profullstack/coinpayportal) — Payment infrastructure
- [BitTorrented / Media Streamer](https://github.com/profullstack/media-streamer) — Content platform
- [PairUX](https://github.com/profullstack/pairux.com) — P2P screen sharing and remote control

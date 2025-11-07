# ONION Private RPC — Tor-style Mixnet

> **Purpose:** A privacy-first Solana RPC mixnet inspired by Tor — users can run **relay nodes** (full relays) or **light nodes** (egress relays), stake the native SPL token **$ONION**, and earn fees from RPC users who pay tiny per-transaction fees. This document is the canonical plan for building the protocol end-to-end: architecture, node designs, staking/tokenomics, smart contract sketches, reward distribution, privacy guarantees, security, deployment, and an implementation roadmap.

---

## Executive summary

ONION is a decentralized privacy RPC network for Solana that mixes and forwards transaction payloads to break linkability between origin and broadcast. It will combine:

- A **mixnet** of volunteer-run relays (full relays and light relays), inspired by Tor onion routing (multi-hop with layered encryption).
- A **tokenized incentive layer** (SPL token `$ONION`) where staking mints the right to operate relays or to receive rewards.
- **RPC proxy services** for end-users and dApps who want privacy without running nodes.
- Fee market: RPC users pay a tiny fee per relayed tx; fees are collected by the protocol and distributed to stakers/operators.

Design goals: open-source, permissionless node operation, low barrier to run (light nodes), token-aligned incentives, privacy-by-default for users who opt-in, and upgradeable from simple mixer to stronger onion routing and zk-based proofs.

---

## High-level architecture

```
[Wallets/Clients] -> (1) Entry Guard (light node or public API)
                    -> (2) Mixnet: multi-hop relays (Full Relays) -> (3) Egress Relays -> Upstream RPC / Broadcast

Staking / Governance <-> Onchain SPL token and Staking Contract
Metrics / Reputation -> Postgres/Supabase (signed proofs)
Dashboard / API Keys -> Frontend
```

Components:
- **Entry Nodes (Guard/Light Nodes):** Public endpoints that accept signed tx blobs from clients and initialize onion layers.
- **Mixnet Relays (Full Relays):** Volunteer-run nodes that forward encrypted packets, strip onion layers, apply delays, and pass to the next hop.
- **Egress Relays:** Final hop that broadcasts to Solana via upstream RPC; these can re-broadcast using rotating relayer wallets to break traceability.
- **Staking Contract (SPL-based):** Manages stakes, operator registration, and reward distribution.
- **Fee Escrow / Rewards Contract:** Collects RPC fees and periodically distributes to stakers and relays according to stake + reputation.
- **Discovery / Directory Service:** A signed list of active relays (published on-chain or via IPFS) with capabilities, reputation, and operator stake.
- **Client SDK / Wallet plugin:** Creates onion-wrapped payloads and selects entry nodes; provides UX for paying fees in native SOL or $ONION.

---

## Node types & roles

1. **Entry (Guard/Light) Node**
   - Accepts client connections (HTTP/WebSocket) and acts as first hop.
   - Does not need heavy compute — can be hosted on small VPS.
   - Optionally subsidized by small stake or run by third parties (CDN-clouds).

2. **Full Relay Node**
   - Forwards onion-layered packets between relays.
   - Keeps minimal ephemeral state; must be online/low-latency.
   - Earns share of fees when used in a path proportional to uptime, stake, and reputation.

3. **Egress Relay / Broadcaster**
   - Final hop — decrypts final layer and submits the signed tx blob (or re-signed payload) to Solana via upstream RPC.
   - May hold rotating broadcaster wallets to further obfuscate IP/wallet linkage.

4. **Light Relay (optional)**
   - Minimal forwarding service for users with limited compute; less reward but easier operation.

Node operator registration: operators stake $ONION to register capability level (e.g., to be chosen as full relay vs light relay). Operators run a daemon exposing a gRPC/WebSocket API for relay-to-relay encrypted tunnels.

---

## Privacy model

- **Onion routing:** Client constructs layered encryption: EgressPublicKey{ FullRelayPubKey{ EntryPublicKey{payload}}}. Each relay peels one layer.
- **Multi-hop & batching:** Use configurable path length (default 3 hops) and batching windows (e.g., 200–2000 ms jitter + occasional batch windows) to reduce timing correlation.
- **Decoys / padding:** Optionally send dummy traffic and chaff to widen anonymity sets.
- **No custody of private keys:** Clients sign transactions locally; relays only carry signed blobs unless user delegates re-signing (explicit flow).
- **Minimal logging:** Relays do not persist mapping of inbound->outbound after forwarding; directory maintains public metrics only.

Threat model: defends against network-level correlators and naive blockchain-level analysis by breaking timing and IP linkage. Strong global adversary (who can observe large portion of relays) remains a risk — mitigations include increasing the number of relays, decoy traffic, and eventual zk/cryptographic improvements.

---

## Tokenomics & staking design

**Token:** `ONION` (SPL), total supply 1B.

**Staking rules:**
- Operators stake X ONION to register a relay slot. Example: 1.000.000 $ONION per full-relay slot, 100.000k ONION per light-relay slot.
- A single operator can run multiple slots by staking multiples of X.
- Stakes can be slashed for proven misbehavior (e.g., persistent censorship or double forwarding proofs) — governance parameterized.

**Fee model:**
- RPC users pay a tiny fee per relayed transaction: e.g., 0.00005 SOL or an equivalent $ONION micro-fee.
- Fees are credited into the Fee Escrow contract and are distributed every epoch (e.g., daily) to stake-holders and relays.

**Reward split (example):**
- 70% distributed to stakers (pro-rata by stake and uptime)
- 25% to relay operators (pro-rata by slot usage & reputation)
- 5% protocol treasury (for grants / dev)

**Access tiers:**
- Free-tier public entry nodes (rate-limited)
- Staked premium nodes: addresses that stake above threshold get priority routing or lower fees
- API-key gating: users can stake or buy API keys for higher throughput

**On-chain contracts:**
- `StakingContract`: stake, withdraw, epoch accounting, slashing hooks
- `FeeEscrow`: receive fees (SOL or ONION), epoch distribution logic (ideally simple; heavy accounting can be offloaded to off-chain reconciler with on-chain settlement proofs)
- `OperatorRegistry`: mapping of operator pubkey -> stake slots -> metadata (signed operator descriptor)

---

## Economic assumptions & sizing

- Suppose 100k tx/mo via ONION, fee 0.00005 SOL (≈5 SOL total). — revenue modest early, grows with adoption.
- Node operators can run on cheap VPS; major costs are bandwidth and stability. Keep load low per node.
- Bootstrap: protocol funds initial relays from treasury to ensure decent anonymity set.

---

## Directory & discovery

- Use a signed directory list for nodes (rotate via epoch). Directory can be stored on IPFS and a small on-chain pointer can anchor it for anti-tampering.
- Directory publishes: node pubkey, IP/endpoint, capabilities, stake-slots, uptime score, last-seen, signature of operator.
- Clients fetch directory and select randomized paths weighted by reputation / stake.

---

## Client SDK & UX

- Client SDK (JS/TS) builds onion packet and path selection logic.
- For wallets: phantom plugin or dApp integration; sign tx locally, then wrap signed blob in onion payload and submit to entry node.
- Fee payment: include fee field in request and pay in SOL or ONION via meta-transaction or an on-chain payment channel.
- Account linking: for premium gating, users sign a JWT proving wallet ownership, obtain API key from onboarding server.

---

## Security & attack mitigation

- **DoS protection:** rate limits, stake-based access, and optionally micro-fees for entry nodes.
- **Malicious relays:** reputation, operator slashing, and fallback path selection avoid over-reliance on single relays.
- **Privacy leakage:** minimize timing leakage by batching + jitter + decoys; provide knobs for increased privacy vs latency.
- **Key management:** relayer broadcaster keys encrypted with HSM or cloud KMS. Recommend operators use hardware-backed keys.

---

## Implementation Roadmap

**Phase 0 — Planning & repo** 
- Finalize tokenomics & staking params
- Create monorepo: `proxy/`, `relay-daemon/`, `worker/`, `client-sdk/`, `dashboard/`, `contracts/`

**Phase 1 — MVP (Weeks 1–4)**
- `proxy` + worker + Redis queue (stateless batching) — simple mixing (what we already prototyped)
- `client-sdk` for signing and sending signed tx blobs
- Egress broadcaster with rotating wallets
- Simple operator registry (off-chain) + directory
- Local devnet tests

**Phase 2 — Mixnet & staking**
- Build `relay-daemon` for encrypted peer-to-peer forwarding (libp2p or custom TLS tunnels)
- Implement staking contract & operator registration
- Integrate FeeEscrow and basic distribution flow (off-chain accounting + on-chain settle)
- Publish node discovery + directory on IPFS/on-chain pointer

**Phase 3 — Hardening & public alpha**
- Reputation system, slashing hooks, monitoring & dashboards
- Wallet plugin integrations (Phantom extension or dApp middleware)
- Open alpha, incentivized testnet run, bug bounties

**Phase 4 — Production & upgrades**
- Decoy traffic, dynamic path length, upgraded cryptography (ntor-like handshake), optional zk proofs for relay accounting
- Governance DAO, grants, relayer marketplace

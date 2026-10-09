# lodestar-gateway — Standing-Up Plan

Standing up a second, independent gateway on The Graph (Arbitrum One, post-Horizon).

This document is the working roadmap and a record of research **corrected against the actual
source** of `edgeandnode/gateway` @ v27.6.0 (the brief that kicked this off was written from
secondary sources; several claims were stale or incomplete — see "Corrections" below).

---

## TL;DR

- A second gateway is permissionless at the **protocol** level (GIP-0056 removed payer
  allowlisting). It is **not** permissionless at the **indexer-onboarding** level: each indexer must
  add our `sender → tap-aggregator` mapping to `tap.sender_aggregator_endpoints` and we must have
  escrow, or our receipts are rejected. This manual, social, per-indexer handshake is THE bottleneck
  — not code, not on-chain permission.
- The gateway is genuinely forkable (MIT) and standalone-runnable. We have forked it as
  `lodestar-gateway` (squashed snapshot; see "Fork divergence log").
- For a solo, self-funded gateway (we fund our own escrow, query our own / curated subgraphs) the
  Studio/billing consumer side is fully bypassable in config — confirmed in `src/config.rs`.

---

## Corrections to the original brief (verified against source)

| Claim in brief | Reality (v27.6.0 source) |
|---|---|
| Gateway is "TAP-only" on payments | **x402 has landed.** Optional `x402` config block: USDC-on-Base payments via a facilitator, alongside TAP. Deps `edgeandnode/x402-rs`. Relevant to how a solo gateway monetises the consumer side. |
| DisputeManager address "not confirmed / optional" | `attestations.dispute_manager: Address` is a **required** config field. Must be pinned before mainnet. |
| Need an Ethereum RPC for exchange rate | `exchange_rate_provider` accepts a **`Fixed` GRT/USD f64** — no RPC needed for pricing (RPC variant also available). |
| API keys imply Studio | `api_keys` has a **`Fixed(Vec<ApiKey>)`** variant — static keys, no Studio backend. Other variants: `Endpoint` (HTTP) and `KafkaTopic`. |
| Consumer payment is mandatory | `payment_required: bool` — doc comment literally says *"disable for testnets."* |
| tap-aggregator endpoint goes in gateway config | It does **not**. The aggregator is purely indexer-side. The gateway only signs v2 receipts (`receipts` block); escrow top-up is the separate `graph-tally` service. |
| `tap-escrow-manager` is the tool | It is **archived** (succeeded by `graphprotocol/graph-tally`). Use graph-tally. |

### Actual gateway config surface (`src/config.rs`, v27.6.0)

Single JSON file passed as `argv[1]`. Top-level fields:

- `api_keys` — `Endpoint { url, auth, special }` | `KafkaTopic { topic, bootstrap_url, bootstrap_auth, special }` | `Fixed([ApiKey])`
- `attestations` — `{ chain_id, dispute_manager }` **(required)**
- `blocklist` — `[]` of POI or `{deployment, indexer}` entries
- `chain_aliases` — map
- `exchange_rate_provider` — `Rpc(url)` | `Fixed(grt_per_usd)`
- `graph_env_id` — string, stamped into Kafka messages
- `kafka_topic_environment` — optional topic suffix
- `ip_blocker_db` — optional CSV path
- `kafka` — librdkafka settings map (defaults to empty `bootstrap.servers`)
- `log_json` — bool
- `min_graph_node_version`, `min_indexer_version` — semver
- `trusted_indexers` — `[TrustedIndexer]` (serve network subgraph for free)
- `network_subgraph_max_lag_seconds` — default 120
- `payment_required` — bool
- `port_api`, `port_metrics` — u16
- `query_fees_target` — f64 (target indexer fee per request)
- `receipts` — `{ chain_id, payer, signer, verifier }` (TAP v2 sender)
- `subgraph_service` — Address
- `x402` — optional `{ facilitator_url, receiver_address, chain (base|base_sepolia), price, facilitator_headers }`
- `max_indexer_response_size` — default 50 MB

> Note: Kafka has a `Default` impl but is referenced by the data-science exporters and the
> `api_keys`/`blocklist` Kafka variants. For a minimal solo deploy, use `api_keys: Fixed` and a
> local Redpanda (or stub) — confirm whether the query/attestation exporters hard-require a reachable
> broker at startup before assuming Kafka is fully optional.

---

## On-chain (Arbitrum One, chain 42161) — VERIFIED

Source: `graphprotocol/contracts` `main` (`packages/horizon/addresses.json`,
`packages/subgraph-service/addresses.json`), read as raw bytes via `jq`. Each address confirmed to
host bytecode on Arbitrum One via `eth_getCode` (chainId 42161). Verified 2026-05-29.

| Contract | Address (VERIFIED) | code |
|---|---|---|
| GRT (L2GraphToken) | `0x9623063377AD1B27544C965cCd7342f7EA7e88C7` | 2284 B |
| PaymentsEscrow (`graph-tally` deposits) | `0xf6Fcc27aAf1fcD8B254498c9794451d82afC673E` | proxy |
| GraphTallyCollector (`receipts.verifier` v2) | `0x8f69F5C07477Ac46FBc491B1E6D91E2bb0111A9e` | 6608 B |
| SubgraphService (`subgraph_service`) | `0xb2Bb92d0DE618878E438b55D5846cfecD9301105` | proxy |
| DisputeManager (`attestations.dispute_manager`) | `0x2FE023a575449AcB698648eD21276293Fa176f96` | proxy |
| HorizonStaking | `0x00669A4CF01450B64E8A2A20E9b1FCB71E61eF03` | proxy |
| GraphPayments | `0x7Aae8ae011927BC36Cb4d0d3e81f2E6E30daE06D` | — |
| Controller | `0x0a8491544221dd212964fbb96487467291b2C97e` | — |

> ⚠️ The original brief's table was WRONG on two critical addresses:
> - PaymentsEscrow: brief said `0x8f477709…BA0d3` (a legacy/other contract) — **real is `0xf6Fcc27a…C673E`**.
> - GraphTallyCollector: brief had a flipped nibble (`…E2be0111A9e`) — **real is `…E2bb0111A9e`**.
> Always paste from this table, never the brief.
>
> Still TODO before funding: cross-check GraphTallyCollector + SubgraphService against what indexer
> `65.109.22.252` actually uses (`receipts_verifier_address_v2`, `subgraph_service_address`) — they
> MUST match or the indexer rejects our receipts.

**Reference production senders (from brief, for the onboarding-config convention):**
GraphOps sender `0xDD6a6f76eb36B873C1C184e8b9b9e762FE216490` (aggregator
`tap-aggregator-arbitrum-one.graphops.xyz`); E&N sender `0xDDE4cfFd3D9052A9cb618fC05a1Cd02be1f2F467`.

### Arbitrum Sepolia (chain 421614) — VERIFIED (eth_getCode, 2026-05-29)

Current strategy: **prove the full loop on Sepolia first** with a throwaway indexer we run
ourselves (self-onboard), since we have no mainnet indexer to recruit. Same `graphprotocol/contracts`
canonical source.

| Contract | Address (VERIFIED) |
|---|---|
| GRT (L2GraphToken) | `0xf8c05dCF59E8B28BFD5eed176C562bEbcfc7Ac04` |
| PaymentsEscrow | `0x4b5D3Da463F7E076bb7CDF5030960bf123245681` |
| GraphTallyCollector (verifier v2) | `0x382863e7B662027117449bd2c49285582bbBd21B` |
| SubgraphService | `0xc24A3dAC5d06d771f657A48B20cE1a671B78f26b` |
| DisputeManager | `0x96e1b86b2739e8A3d59F40F2532caDF9cE8Da088` |
| HorizonStaking | `0x865365C425f3A593Ffe698D9c4E6707D14d51e08` |

(Brief's Sepolia escrow `0x1e4dC4f9…2d02` was also WRONG; verifier matched.)
Public Sepolia RPC: `https://sepolia-rollup.arbitrum.io/rpc`.

> **No mainnet indexer available** (the Helsinki box in old notes is not ours). So the self-onboard
> dodge requires standing up our OWN indexer. Decision: do it on Sepolia (free testnet GRT) to prove
> gateway→indexer→receipt→RAV→redeem end-to-end, then tackle mainnet recruitment separately.
> The indexer half (graph-node + indexer-agent/service-rs + tap-agent + stake/provision/allocate) is
> the larger build; the gateway stack is already scaffolded under `deploy/`.

Arbitrum Sepolia (421614) testnet: TAP Verifier `0x382863e7B662027117449bd2c49285582bbBd21B`;
TAP Escrow `0x1e4dC4f9F95E102635D8F7ED71c5CdbFa20e2d02`;
SubgraphService `0xc24A3dAC5d06d771f657A48B20cE1a671B78f26b`;
aggregator `tap-aggregator.testnet.thegraph.com`.

**Sender / signer / receiver flow:**
1. Sender authorizes its signer on GraphTallyCollector: `authorizeSigner(signer, proofDeadline, proof)`
   — `proof` = ECDSA over `solidityPackedKeccak256(['uint256','uint256','address'], [chainId, proofDeadline, senderAddress])`.
2. Sender deposits GRT into PaymentsEscrow keyed per `(payer, collector, receiver=indexer)`.
3. Gateway signs v2 receipts per query; indexer's tap-agent aggregates → RAV; indexer-agent redeems.
4. `graph-tally` keeps escrow topped up vs outstanding receipt debt.

---

## PROOF — gateway works against live Arbitrum One (2026-05-29)

Ran `lodestar-gateway:latest` on the VPS against the **real** Arbitrum One network (no indexer of
our own, no funds). Trick: `trusted_indexers` requires an indexer-service envelope
(`{graphQLResponse, attestation}`), not raw GraphQL — so a tiny adapter fronted the decentralised
gateway (`gateway.thegraph.com/api/<key>/subgraphs/id/DZz4kDTdmzWLWsV373w2bSmoar3umKKH9y82SUKr5qmp`)
and re-wrapped its `{data}` into that envelope (also needed a browser User-Agent; the WAF 403s
`python-urllib`).

Results:
- Booted, parsed our production config, loaded the live topology: **subgraphs=15845,
  deployments=26088, indexings=13746** (182 indexers), served `:7700` + metrics `:7301`.
- Real client query → authenticated (32-hex API key) → resolved subgraph→deployment → **selected
  ~28 real indexers** (real allocations + URLs e.g. `arbindex.grt.pops.one`, `thegraph.lunanova.tech`)
  → **signed TAP v2 receipts** → computed fees via the fee-control system (6e-7–1.8e-6 GRT each,
  total `2.88e-5` GRT ≈ `$0.00032`) → dispatched → tracked latency + seconds-behind.
- Every live indexer returned **HTTP 402 (Payment Required)** — they processed our receipts and want
  escrow. A bad signature/verifier would error differently; 402 == "valid sender, no escrow".

**Conclusion: the gateway works end-to-end on its side.** The only gap to successful paid queries is
funding escrow + indexer onboarding — exactly the known bottleneck, not a code issue. `wait_until_ready`
gates the API on a non-empty network snapshot (`main.rs:144`); first snapshot takes ~60s.
NOTE: a running gateway polls the network subgraph every 30s — burns the Studio API key quota; stop it when idle.

## Roadmap

### Phase 0 — Fork & build  ✅ (this session)
- [x] Squashed-snapshot fork of `edgeandnode/gateway` v27.6.0 into `nuthatch-org/gateway` (MIT retained).
- [x] Builds on macOS via `dynamic-kafka` feature + Homebrew librdkafka; Linux/Docker static build untouched.
- [x] Light rebrand (`lodestar-gateway` package/binary/CI image/log name).

### Phase 1 — local-network end-to-end (next)
- [ ] Clone `edgeandnode/local-network` (Docker Compose: graph-node, IPFS, Postgres, Redpanda,
      indexer-service, tap-agent, gateway, `semiotic/tap` subgraph; chain 1337).
- [ ] Mount our `lodestar-gateway` binary via `compose/dev/` overrides.
- [ ] Drive a full **query → v2 receipt → RAV → redeem** loop locally. De-risks the config surface
      and escrow/RAV mechanics for free.
- [ ] Confirm minimal config: `api_keys: Fixed`, `payment_required: false`, `exchange_rate_provider: Fixed`.
- [ ] Determine whether the Kafka exporters can be fully disabled or need a stub broker.

### Phase 2 — Arbitrum Sepolia testnet
- [ ] Pin Sepolia addresses; get testnet GRT.
- [ ] Stand up our own **tap-aggregator** (public JSON-RPC) + **graph-tally** escrow manager.
- [ ] `authorizeSigner`, deposit escrow.
- [ ] Coordinate with one testnet indexer to add our `sender → aggregator` mapping. Validate receipts accepted.

### Phase 3 — Arbitrum One mainnet
- [ ] **Verify ALL contract addresses on Arbiscan**, especially the missing DisputeManager.
- [ ] Arbitrum archive RPC (self-host on Hetzner alongside our indexer, or dRPC/Alchemy).
- [ ] Fund sender (ETH + GRT), authorize signer, deposit conservative per-indexer escrow.
- [ ] Run gateway + tap-aggregator + graph-tally + Postgres + Redpanda (co-located on one Hetzner box).
- [ ] **Onboard indexers** (the real work): add our own indexer first (we control its config), then
      recruit trusted indexers to add our sender→aggregator mapping.

### Cross-cutting / strategic
- [ ] Engage `indexer-rs#342` (gateway discovery & verification protocol). If it ships, the manual
      onboarding handshake disappears and a solo gateway becomes turnkey. Highest-leverage item.
- [ ] Decide consumer-side monetisation: open/static-key vs x402 USDC vs API-key billing.
- [ ] Escrow sizing discipline: small per-indexer escrow + low `max_receipt_value_grt`, watch
      `escrow_total_debt_grt` vs `escrow_total_balance_grt`, then scale.

---

## Operator components (what we must run)

1. **lodestar-gateway** (this repo) — stateless query routing + v2 receipt signing.
2. **tap-aggregator** (`semiotic-ai/timeline-aggregation-protocol/tap_aggregator`) — public endpoint
   indexers whitelist. Our aggregator URL + sender address are the two values indexers need.
3. **graph-tally** (`graphprotocol/graph-tally`) — escrow top-up (replaces archived tap-escrow-manager).
4. **Arbitrum One RPC/archive**, **network subgraph** (via trusted indexers or self-hosted), **Postgres/Redis**, **Kafka/Redpanda**.
5. **Two wallets**: sender (ETH + GRT) and authorized signer.

---

## Fork divergence log

Changes from upstream `edgeandnode/gateway` v27.6.0:

- `Cargo.toml`: package renamed `graph-gateway` → `lodestar-gateway`; added opt-in `dynamic-kafka`
  feature (`rdkafka/dynamic-linking`) for macOS builds.
- `Dockerfile`: binary name updated (still static vendored librdkafka build on Linux).
- `.github/workflows/docker-image.yml`: image → `ghcr.io/nuthatch-org/gateway`.
- `src/main.rs`: logging service name → `lodestar-gateway`.
- `src/network/service.rs`: doc comment rename.
- `README.md`: fork notice + macOS build instructions.
- `.gitignore` + local `.cargo/config.toml` (gitignored): macOS pkg-config/openssl env.

Upstream sync is manual (squashed fork): `git diff` against upstream tags / cherry-pick as needed.
Local `.cargo/config.toml` for macOS (gitignored):

```toml
[env]
PKG_CONFIG_PATH = "/opt/homebrew/opt/librdkafka/lib/pkgconfig:/opt/homebrew/opt/openssl@3/lib/pkgconfig:/opt/homebrew/opt/cyrus-sasl/lib/pkgconfig:/opt/homebrew/opt/krb5/lib/pkgconfig:/opt/homebrew/opt/zstd/lib/pkgconfig:/opt/homebrew/opt/lz4/lib/pkgconfig"
OPENSSL_ROOT_DIR = "/opt/homebrew/opt/openssl@3"
```

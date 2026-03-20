---
name: A2A Escrow Solana
overview: Build a full-stack A2A escrow and arbitration system on Solana. Dual-asset (SOL + SPL), vault PDAs, VRF arbiter selection, reentrancy guards, emergency pause/circuit breaker, tiered dynamic fees, scaled dispute bonds, events for indexing, account versioning, CI/CD, fuzz + load testing, example agent bots.
todos:
  - id: scaffold
    content: Scaffold Anchor project with `anchor init agent-escrow`, configure Anchor.toml, Cargo.toml, and package.json with all dependencies (anchor 0.32.x, mpl-core, anchor-spl, solana-program, switchboard-solana for VRF)
    status: pending
  - id: state-accounts
    content: "Implement state module: EscrowState (with version, reentrancy_guard, payment_type, task_spec_uri), VaultState, TreasuryState, ArbitrationState, ArbiterRegistry, ReputationState (with history ring buffer) in programs/agent-escrow/src/state/"
    status: pending
  - id: constants-errors
    content: Define protocol constants (fees, seeds, limits, time buffers, daily volume cap, tiered fee thresholds, scaled bond thresholds) in constants.rs and comprehensive AnchorError codes in error.rs (including ProtocolPaused, DailyLimitExceeded)
    status: pending
  - id: events
    content: "Define Anchor events for all state transitions: EscrowCreated, TaskAccepted, WorkSubmitted, WorkApproved, FundsReleased, DisputeInitiated, VoteCast, DisputeResolved, EscrowCancelled, EscrowExpired, ReputationUpdated, ProtocolPaused, ProtocolResumed"
    status: pending
  - id: ix-initialize-treasury
    content: "Implement initialize_treasury instruction: create TreasuryState + ProtocolState PDAs with authority, fee config, pause flag, daily volume tracking, upgrade_authority"
    status: pending
  - id: ix-pause-protocol
    content: "Implement pause_protocol and unpause_protocol instructions: emergency circuit breaker controlled by protocol authority, checked at entry of all fund-moving instructions"
    status: pending
  - id: ix-initialize
    content: "Implement initialize_escrow instruction: create EscrowState PDA + VaultState PDA, mint Metaplex Core NFT via CPI, lock funds into vault (dual-path: native SOL via SystemProgram or SPL via anchor_spl::token::transfer), emit EscrowCreated event"
    status: pending
  - id: ix-accept
    content: "Implement accept_task instruction: assign seller, update status to Accepted, update NFT on-chain attributes via Attributes plugin, emit TaskAccepted event"
    status: pending
  - id: ix-submit
    content: "Implement submit_work instruction: store DeliveryReceipt (delivery_uri, delivery_hash, checksum_algorithm), transition to Delivered, emit WorkSubmitted event"
    status: pending
  - id: ix-approve
    content: "Implement approve_work instruction: reentrancy guard, transfer funds from VaultState to seller (SOL or SPL), fee to treasury, update status to Settled, update reputation for both parties, thaw NFT, emit WorkApproved + FundsReleased events"
    status: pending
  - id: ix-dispute
    content: "Implement initiate_dispute instruction: create ArbitrationState PDA, collect bonds from both parties, select arbiters via VRF seed, transition to Disputed, emit DisputeInitiated event"
    status: pending
  - id: ix-vote-resolve
    content: "Implement cast_vote and resolve_dispute instructions: tally votes, distribute funds based on majority, slash bonds (50% winner / 50% arbiters), update reputation, update NFT with outcome, emit VoteCast + DisputeResolved events"
    status: pending
  - id: ix-timeout-cancel
    content: "Implement claim_timeout (with time buffer) and cancel_escrow instructions: handle deadline expiry and pre-acceptance cancellation, update reputation, emit EscrowExpired/EscrowCancelled events"
    status: pending
  - id: ix-arbiter-registry
    content: "Implement register_arbiter and slash_arbiter instructions: arbiter staking, VRF-based selection helpers, slashing for bad votes"
    status: pending
  - id: reputation
    content: "Implement reputation system: ReputationState with history ring buffer (last 64 items), weighted trust_score computation (recency + volume), create/update on settlement/dispute"
    status: pending
  - id: tests-happy
    content: "Write integration tests for happy path: full lifecycle for both native SOL and SPL token escrows"
    status: pending
  - id: tests-dispute
    content: "Write integration tests for dispute flow: initiate, vote, resolve for both buyer-wins and seller-wins, arbiter reward distribution"
    status: pending
  - id: tests-edge
    content: "Write edge case tests: timeouts with time buffer, cancellation, unauthorized access, double-submit, invalid transitions, reentrancy attempts, SPL token edge cases"
    status: pending
  - id: tests-fuzz
    content: Add cargo-fuzz targets for arithmetic (tiered fee calc, scaled bond calc, bond slashing) and proptest for state machine transitions with random inputs
    status: pending
  - id: tests-load
    content: "Write load tests: 1000+ concurrent escrows, high-frequency state transitions, volume circuit breaker trigger verification"
    status: pending
  - id: sdk
    content: "Build TypeScript SDK: EscrowClient class with dual-asset support, PDA derivation helpers, TaskSpec metadata builder, Arweave upload via Irys, event listener/parser, transaction simulation, batch operations, recovery helpers"
    status: pending
  - id: ci-cd
    content: "Set up GitHub Actions CI/CD: anchor build + test, cargo audit, cargo-fuzz smoke, coverage report via tarpaulin, devnet deploy workflow"
    status: pending
  - id: docs
    content: "Create documentation: README, QUICKSTART, API reference, SECURITY model, ARBITER onboarding guide, TROUBLESHOOTING"
    status: pending
  - id: example-agents
    content: "Build example agent implementations: buyer-agent, seller-agent, arbiter-agent, monitoring-agent (TypeScript scripts using the SDK)"
    status: pending
  - id: devnet-deploy
    content: Deploy program to Solana devnet, verify with end-to-end test using SDK for both SOL and SPL token flows
    status: pending
isProject: false
---

# A2A Escrow & Arbitration System on Solana (v3)

## Technology Stack

- **Smart Contract**: Anchor 0.32.x / Rust (Solana Program)
- **NFT Standard**: Metaplex Core (single-account, ~0.003 SOL mint cost, plugin system)
- **Token Support**: Native SOL + SPL Tokens (USDC, etc.) via `anchor_spl::token`
- **Off-chain Storage**: Arweave via Irys (for TaskSpec metadata JSON)
- **Randomness**: Switchboard VRF (for arbiter selection) or blockhash-seeded deterministic selection
- **Client SDK**: TypeScript (`@solana/web3.js` v2, `@coral-xyz/anchor`, `@metaplex-foundation/mpl-core`)
- **Testing**: LiteSVM (Rust unit) + TypeScript integration + `cargo-fuzz` + `proptest` + load tests
- **CI/CD**: GitHub Actions (build, test, audit, coverage, devnet deploy)
- **Toolchain**: Rust 1.86+, Solana CLI 2.1.x, Node.js v22+

---

## Project Structure

```
agent-escrow/
├── Anchor.toml
├── Cargo.toml
├── package.json
├── tsconfig.json
├── programs/
│   └── agent-escrow/
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs                     # Program entrypoint, declare_id!
│           ├── constants.rs               # Fees, seeds, limits, time buffers, volume caps
│           ├── error.rs                   # Custom error codes (comprehensive)
│           ├── events.rs                  # Anchor #[event] definitions
│           ├── fees.rs                    # Tiered fee + scaled bond calculation functions
│           ├── state/
│           │   ├── mod.rs
│           │   ├── escrow.rs              # EscrowState + VaultState accounts
│           │   ├── protocol.rs            # ProtocolState (pause, circuit breaker)
│           │   ├── treasury.rs            # TreasuryState account
│           │   ├── arbitration.rs         # ArbitrationState + ArbiterRegistry
│           │   └── reputation.rs          # ReputationState + ReputationHistoryItem
│           └── instructions/
│               ├── mod.rs
│               ├── initialize_treasury.rs # One-time treasury + protocol state setup
│               ├── initialize_escrow.rs   # Create escrow + vault + mint NFT
│               ├── accept_task.rs
│               ├── submit_work.rs
│               ├── approve_work.rs        # With reentrancy guard + pause check
│               ├── initiate_dispute.rs
│               ├── cast_vote.rs
│               ├── resolve_dispute.rs     # With reentrancy guard + pause check
│               ├── claim_timeout.rs
│               ├── cancel_escrow.rs
│               ├── register_arbiter.rs    # Arbiter staking
│               ├── slash_arbiter.rs       # Arbiter penalty
│               ├── pause_protocol.rs      # Emergency pause
│               └── update_fee_config.rs   # Fee tier updates
├── fuzz/                                  # cargo-fuzz targets
│   ├── Cargo.toml
│   └── fuzz_targets/
│       ├── fuzz_fee_calc.rs               # Tiered fee + scaled bond arithmetic
│       └── fuzz_state_machine.rs
├── sdk/
│   └── src/
│       ├── index.ts
│       ├── client.ts                      # EscrowClient (dual-asset, simulation, batch)
│       ├── types.ts                       # On-chain state mirrors
│       ├── metadata.ts                    # TaskSpec builder + Arweave upload
│       ├── events.ts                      # Event parser + listener
│       └── utils.ts                       # PDA derivation helpers
├── tests/
│   ├── agent-escrow.ts                    # Integration: full lifecycle (SOL + SPL)
│   ├── dispute.ts                         # Dispute resolution tests
│   ├── edge-cases.ts                      # Timeouts, reentrancy, overflow
│   ├── spl-token.ts                       # SPL-specific paths
│   ├── pause-circuit-breaker.ts           # Emergency pause + volume limit tests
│   └── load/
│       └── high-volume.ts                 # 1000+ concurrent escrows
├── examples/                              # Example agent implementations
│   ├── buyer-agent/
│   │   ├── package.json
│   │   └── src/index.ts                   # Complete buyer bot
│   ├── seller-agent/
│   │   ├── package.json
│   │   └── src/index.ts                   # Complete seller bot
│   ├── arbiter-agent/
│   │   ├── package.json
│   │   └── src/index.ts                   # Complete arbiter bot
│   └── monitoring-agent/
│       ├── package.json
│       └── src/index.ts                   # Escrow health monitor
├── docs/
│   ├── README.md                          # Project overview
│   ├── QUICKSTART.md                      # 5-minute setup guide
│   ├── API.md                             # SDK reference
│   ├── SECURITY.md                        # Security model + audit prep
│   ├── ARBITER-ONBOARDING.md              # How to become an arbiter
│   └── TROUBLESHOOTING.md                 # Common issues
├── .github/
│   └── workflows/
│       ├── ci.yml                         # Build, test, audit, coverage
│       └── deploy-devnet.yml              # Devnet deployment
└── app/                                   # (Future) Indexer + GraphQL API
```

---

## 1. On-Chain State Accounts

### 1.1 EscrowState (PDA, seeds: `[b"escrow", nft_mint.key()]`)

```rust
#[account]
pub struct EscrowState {
    pub version: u32,                         // Account version for migration (1 = V1)
    pub buyer: Pubkey,                        // 32
    pub seller: Pubkey,                       // 32 (Pubkey::default() if unassigned)
    pub nft_mint: Pubkey,                     // 32
    pub vault: Pubkey,                        // 32 -> points to VaultState PDA
    pub task_spec_hash: [u8; 32],             // SHA-256 of off-chain TaskSpec JSON
    pub task_spec_uri: [u8; 128],             // Arweave/IPFS URI (null-terminated, for verification)
    pub delivery: DeliveryReceipt,            // Structured delivery data (zeroed until submitted)
    pub amount: u64,                          // Escrowed payment (lamports or token smallest unit)
    pub protocol_fee: u64,                    // Fee in same denomination
    pub payment_type: PaymentType,            // Native SOL or SPL token
    pub token_mint: Pubkey,                   // SPL token mint (Pubkey::default() for native SOL)
    pub deadline: i64,                        // Unix timestamp for task completion
    pub challenge_period: i64,                // Seconds buyer has to approve/dispute after delivery
    pub status: EscrowStatus,                 // Current state (1 byte)
    pub refund_address: Pubkey,               // Explicit refund destination (defaults to buyer)
    pub reentrancy_guard: bool,               // false = unlocked, true = locked
    pub created_at: i64,
    pub accepted_at: i64,
    pub delivered_at: i64,
    pub settled_at: i64,
    pub bump: u8,
}
```

### 1.2 VaultState (PDA, seeds: `[b"vault", escrow.key()]`)

Dedicated fund-holding account, separated from metadata for security.

```rust
#[account]
pub struct VaultState {
    pub escrow: Pubkey,                       // 32 -> back-reference to EscrowState
    pub amount: u64,                          // Current balance held
    pub bump: u8,
}
```

For SPL tokens, a separate Associated Token Account (ATA) is created for the vault PDA, owned by the token program. The VaultState PDA acts as the authority for that ATA.

### 1.3 PaymentType Enum

```rust
#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq, Eq)]
pub enum PaymentType {
    NativeSol,
    SplToken,
}
```

### 1.4 DeliveryReceipt (Embedded struct, not a separate account)

```rust
#[derive(AnchorSerialize, AnchorDeserialize, Clone, Default)]
pub struct DeliveryReceipt {
    pub delivery_uri: [u8; 128],              // Arweave/IPFS URI to artifact
    pub delivery_hash: [u8; 32],              // Hash of content at URI
    pub checksum_algorithm: u8,               // 0=SHA-256, 1=BLAKE3
    pub submitted: bool,                      // Whether receipt has been submitted
}
```

### 1.5 EscrowStatus Enum

```rust
#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq, Eq)]
pub enum EscrowStatus {
    Escrowed,       // Funds locked, awaiting seller
    Accepted,       // Seller committed
    Delivered,      // Seller submitted work
    Settled,        // Funds released to seller (terminal)
    Disputed,       // In arbitration
    Refunded,       // Funds returned to buyer (terminal)
    Cancelled,      // Buyer cancelled before acceptance (terminal)
    Expired,        // Deadline passed without delivery (terminal)
}
```

### 1.6 ProtocolState (PDA, seeds: `[b"protocol"]`) -- singleton

Global circuit breaker and pause mechanism.

```rust
#[account]
pub struct ProtocolState {
    pub authority: Pubkey,                    // Protocol admin (can pause/unpause)
    pub paused: bool,                         // Emergency pause flag
    pub daily_volume: u64,                    // Rolling 24h transaction volume (lamports)
    pub daily_volume_cap: u64,                // Max allowed 24h volume (circuit breaker)
    pub volume_window_start: i64,             // Start of current 24h window
    pub total_escrows_created: u64,           // Lifetime counter
    pub bump: u8,
}
```

All fund-moving instructions (`initialize_escrow`, `approve_work`, `resolve_dispute`, `claim_timeout`) check `require!(!protocol.paused, EscrowError::ProtocolPaused)` and `require!(protocol.daily_volume + amount <= protocol.daily_volume_cap, EscrowError::DailyLimitExceeded)`.

### 1.7 TreasuryState (PDA, seeds: `[b"treasury"]`) -- singleton

```rust
#[account]
pub struct TreasuryState {
    pub authority: Pubkey,                    // Protocol governance key
    pub upgrade_authority: Pubkey,            // Can migrate to TreasuryV2
    pub sol_vault: Pubkey,                    // SystemAccount for native SOL fees
    pub fee_tier_thresholds: [u64; 3],        // Amount thresholds for tiered fees
    pub fee_tier_bps: [u64; 4],               // BPS for each tier (below t1, t1-t2, t2-t3, above t3)
    pub total_fees_collected: u64,
    pub bump: u8,
}
```

For each SPL token, a separate ATA is created under the treasury PDA authority.

### 1.8 ArbitrationState (PDA, seeds: `[b"arbitration", escrow.key()]`)

```rust
#[account]
pub struct ArbitrationState {
    pub escrow: Pubkey,                       // 32
    pub initiator: Pubkey,                    // Who initiated the dispute
    pub buyer_bond: u64,
    pub seller_bond: u64,
    pub arbiters: [Pubkey; 5],                // Up to 5, selected via VRF
    pub votes: [Vote; 5],
    pub evidence_hashes: [[u8; 32]; 5],       // On-chain evidence hashes (buyer, seller, up to 3 arbiter-submitted)
    pub evidence_count: u8,
    pub arbiter_count: u8,                    // Actual number (3 or 5)
    pub votes_cast: u8,
    pub resolution: Resolution,
    pub vrf_seed: [u8; 32],                   // Seed used for arbiter selection (audit trail)
    pub created_at: i64,
    pub resolved_at: i64,
    pub bump: u8,
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq, Eq)]
pub enum Vote { Pending, ForBuyer, ForSeller }

#[derive(AnchorSerialize, AnchorDeserialize, Clone, PartialEq, Eq)]
pub enum Resolution { Pending, BuyerWins, SellerWins }
```

### 1.9 ArbiterRegistry (PDA, seeds: `[b"arbiter", arbiter.key()]`)

```rust
#[account]
pub struct ArbiterRegistry {
    pub arbiter: Pubkey,                      // 32
    pub stake: u64,                           // Staked SOL (slashable)
    pub cases_judged: u64,
    pub cases_slashed: u64,
    pub is_active: bool,
    pub registered_at: i64,
    pub bump: u8,
}
```

### 1.10 ReputationState (PDA, seeds: `[b"reputation", agent.key()]`)

Enhanced with a fixed-size history ring buffer for recency-weighted scoring.

```rust
pub const REPUTATION_HISTORY_SIZE: usize = 64;

#[account]
pub struct ReputationState {
    pub agent: Pubkey,                        // 32
    pub tasks_completed: u64,
    pub tasks_failed: u64,
    pub tasks_disputed: u64,
    pub disputes_won: u64,
    pub total_volume: u64,                    // Cumulative lamports/tokens transacted
    pub trust_score: u64,                     // Computed score (0-10000 basis points)
    pub reputation_nft: Pubkey,               // Soulbound Reputation NFT mint
    pub history: [ReputationHistoryItem; REPUTATION_HISTORY_SIZE],
    pub history_head: u8,                     // Ring buffer write pointer
    pub last_updated: i64,
    pub bump: u8,
}

#[derive(AnchorSerialize, AnchorDeserialize, Clone, Copy, Default)]
pub struct ReputationHistoryItem {
    pub timestamp: i64,
    pub delta: i32,                           // +100/-200 basis points
    pub reason: u8,                           // 0=task_completed, 1=dispute_won, 2=slashed, 3=timeout
    pub escrow: Pubkey,                       // Reference to the escrow
}
```

Trust score computation: weighted average of history items where weight = `1.0 / (1 + days_since_event)`, normalized to 0-10000 bps.

---

## 2. Program Events

All state transitions emit Anchor events for off-chain indexing (e.g., by Helius, Shyft, or custom geyser plugins).

```rust
#[event]
pub struct EscrowCreated {
    pub escrow: Pubkey,
    pub buyer: Pubkey,
    pub nft_mint: Pubkey,
    pub amount: u64,
    pub payment_type: PaymentType,
    pub deadline: i64,
}

#[event]
pub struct TaskAccepted {
    pub escrow: Pubkey,
    pub seller: Pubkey,
}

#[event]
pub struct WorkSubmitted {
    pub escrow: Pubkey,
    pub delivery_hash: [u8; 32],
}

#[event]
pub struct WorkApproved {
    pub escrow: Pubkey,
    pub buyer: Pubkey,
    pub seller: Pubkey,
    pub amount: u64,
}

#[event]
pub struct FundsReleased {
    pub escrow: Pubkey,
    pub recipient: Pubkey,
    pub amount: u64,
    pub fee: u64,
}

#[event]
pub struct DisputeInitiated {
    pub escrow: Pubkey,
    pub initiator: Pubkey,
    pub bond: u64,
}

#[event]
pub struct VoteCast {
    pub arbitration: Pubkey,
    pub arbiter: Pubkey,
    pub vote: Vote,
}

#[event]
pub struct DisputeResolved {
    pub escrow: Pubkey,
    pub resolution: Resolution,
    pub buyer_refund: u64,
    pub seller_payout: u64,
}

#[event]
pub struct ReputationUpdated {
    pub agent: Pubkey,
    pub new_trust_score: u64,
    pub delta: i32,
    pub reason: u8,
}

#[event]
pub struct EscrowCancelled { pub escrow: Pubkey, pub buyer: Pubkey }

#[event]
pub struct EscrowExpired { pub escrow: Pubkey, pub defaulting_party: Pubkey }

#[event]
pub struct ProtocolPaused { pub authority: Pubkey, pub timestamp: i64 }

#[event]
pub struct ProtocolResumed { pub authority: Pubkey, pub timestamp: i64 }

#[event]
pub struct EvidenceSubmitted { pub arbitration: Pubkey, pub submitter: Pubkey, pub evidence_hash: [u8; 32] }

#[event]
pub struct CircuitBreakerTriggered { pub daily_volume: u64, pub cap: u64 }
```

---

## 3. Program Instructions (17 total)

### 3.1 `initialize_treasury` (one-time setup)

- **Caller**: Protocol deployer
- **Actions**: Create singleton `TreasuryState` PDA + `ProtocolState` PDA; set authority, tiered fee config, daily volume cap
- **Validations**: Can only be called once (PDA already-exists check)

### 3.2 `initialize_escrow`

- **Caller**: Buyer agent
- **Actions**:
  1. Create `EscrowState` PDA (`init`, payer = buyer)
  2. Create `VaultState` PDA (`init`, payer = buyer)
  3. Mint a Metaplex Core NFT to buyer via CPI to `mpl_core` program
    - NFT metadata URI points to Arweave-hosted TaskSpec JSON
    - Attach `FreezeDelegate` plugin (prevents transfer while escrowed)
    - Attach `Attributes` plugin (mutable on-chain state)
    - Attach `UpdateDelegate` plugin (escrow PDA as delegate authority)
  4. **If NativeSol**: Transfer `amount + protocol_fee` lamports from buyer to VaultState PDA via `system_program::transfer`
  5. **If SplToken**: Transfer tokens from buyer's ATA to vault's ATA via `anchor_spl::token::transfer` CPI
  6. Compute `protocol_fee` via tiered fee function based on `amount` and `TreasuryState.fee_tier_`*
  7. Set status to `Escrowed`, record `created_at`, store `task_spec_uri`, set `refund_address`
  8. Update `ProtocolState.daily_volume`; check circuit breaker
  9. Emit `EscrowCreated` event
- **Validations**:
  - `require!(!protocol.paused)` -- protocol not paused
  - `require!(protocol.daily_volume + amount <= protocol.daily_volume_cap)` -- circuit breaker
  - `amount >= MIN_ESCROW_AMOUNT`
  - `deadline > now + MIN_DEADLINE_BUFFER` and `deadline < now + MAX_DEADLINE_DURATION`
  - `challenge_period >= MIN_CHALLENGE_PERIOD` and `<= MAX_CHALLENGE_PERIOD`
  - `buyer != Pubkey::default()`
  - `protocol_fee` matches treasury config or constant
  - If SPL: `token_mint != Pubkey::default()`, buyer ATA has sufficient balance

### 3.3 `accept_task`

- **Caller**: Seller agent
- **Actions**:
  1. Set `seller` field on `EscrowState`
  2. Update status: `Escrowed` -> `Accepted`
  3. Record `accepted_at`
  4. Update NFT `Attributes` plugin: add `seller` pubkey
  5. Emit `TaskAccepted` event
- **Validations**: Status == `Escrowed`, `seller != buyer`, `deadline > now + TIME_BUFFER`

### 3.4 `submit_work`

- **Caller**: Seller agent
- **Actions**:
  1. Store `DeliveryReceipt` (uri, hash, checksum algorithm)
  2. Update status: `Accepted` -> `Delivered`
  3. Record `delivered_at`
  4. Emit `WorkSubmitted` event
- **Validations**: Status == `Accepted`, caller == seller, `deadline + TIME_BUFFER > now`, delivery not already submitted

### 3.5 `approve_work`

- **Caller**: Buyer agent
- **Actions**:
  1. **Check protocol not paused** (`require!(!protocol.paused)`)
  2. **Acquire reentrancy guard** (`require!(!escrow.reentrancy_guard)`; set to `true`)
  3. **If NativeSol**: Transfer `amount` from VaultState to seller via PDA-signed `system_program::transfer`; transfer `protocol_fee` to treasury
  4. **If SplToken**: Transfer tokens from vault ATA to seller ATA; fee to treasury ATA
  5. Update status: `Delivered` -> `Settled`
  6. Record `settled_at`
  7. Update ReputationState for both buyer (+completion) and seller (+completion)
  8. Thaw NFT (remove FreezeDelegate), update Attributes with `status: SETTLED`
  9. Emit `WorkApproved`, `FundsReleased`, `ReputationUpdated` events
  10. **Release reentrancy guard** (set to `false`)
- **Validations**: Status == `Delivered`, caller == buyer, protocol not paused

### 3.6 `initiate_dispute`

- **Caller**: Buyer (within challenge window) OR Seller (past challenge window + grace period)
- **Actions**:
  1. Create `ArbitrationState` PDA (with zeroed `evidence_hashes`)
  2. Compute dispute bond via scaled formula: `calculate_dispute_bond(escrow.amount)` (5% small, 2% medium, 1% large)
  3. Transfer bond from initiator to ArbitrationState PDA (SOL or SPL)
  4. Select arbiters using VRF seed: `hash(recent_blockhash[..8] || escrow_key || arbiter_idx)`
  5. Update escrow status: `Delivered` -> `Disputed`
  6. Emit `DisputeInitiated` event
- **Validations**: Status == `Delivered`, timing constraints, bond amount sufficient

### 3.7 `post_counter_bond`

- **Caller**: The other party (non-initiator)
- **Actions**: Transfer matching bond to ArbitrationState PDA
- **Validations**: ArbitrationState exists, counter-party hasn't bonded yet, within bond posting window

### 3.8 `cast_vote`

- **Caller**: Registered arbiter (must be in `arbiters` array and have active `ArbiterRegistry`)
- **Actions**:
  1. Record vote (`ForBuyer` or `ForSeller`)
  2. Increment `votes_cast`
  3. Emit `VoteCast` event
  4. If majority reached, set `resolution` (but don't distribute funds yet -- separate `resolve_dispute` call)
- **Validations**: Caller in arbiters array, hasn't voted, status == `Disputed`, arbiter is staked

### 3.9 `resolve_dispute`

- **Caller**: Any (permissionless crank once majority exists)
- **Actions**:
  1. **Acquire reentrancy guard**
  2. **If SellerWins**: Release `amount` to seller, `protocol_fee` to treasury, return seller bond, slash buyer bond
  3. **If BuyerWins**: Refund `amount + protocol_fee` to buyer, return buyer bond, slash seller bond
  4. Slashed bond distribution: 50% to winning party, 50% split among voting arbiters
  5. Update ReputationState for both parties and arbiters
  6. Update NFT Attributes with dispute outcome
  7. Set escrow status to `Settled` or `Refunded`
  8. Emit `DisputeResolved`, `FundsReleased`, `ReputationUpdated` events
  9. **Release reentrancy guard**
- **Validations**: Majority vote exists, not already resolved

### 3.10 `claim_timeout`

- **Caller**: Buyer (seller missed deadline) OR Seller (buyer unresponsive past challenge period)
- **Actions**:
  - **Seller missed deadline**: Refund `amount + protocol_fee` to buyer, set `Expired`
  - **Buyer unresponsive**: Release `amount` to seller, `protocol_fee` to treasury, set `Settled`
  - Update ReputationState for defaulting party (negative delta)
  - Emit `EscrowExpired` event
- **Validations**: `now > deadline + TIME_BUFFER` (seller timeout) or `now > delivered_at + challenge_period + TIME_BUFFER` (buyer timeout)
- **Time buffer**: All deadline comparisons include `CLOCK_DRIFT_BUFFER` (60 seconds) to account for validator clock skew

### 3.11 `cancel_escrow`

- **Caller**: Buyer agent
- **Actions**:
  1. Refund `amount + protocol_fee` from VaultState to buyer
  2. Update NFT Attributes: `status: CANCELLED` (do NOT burn -- Metaplex Core doesn't reliably support burn)
  3. Close VaultState account (reclaim rent)
  4. Set status to `Cancelled`
  5. Emit `EscrowCancelled` event
- **Validations**: Status == `Escrowed` (no seller yet), caller == buyer

### 3.12 `register_arbiter`

- **Caller**: Prospective arbiter
- **Actions**:
  1. Create `ArbiterRegistry` PDA
  2. Transfer stake (minimum `MIN_ARBITER_STAKE`) to registry PDA
  3. Set `is_active = true`
- **Validations**: Minimum stake met, not already registered

### 3.13 `slash_arbiter`

- **Caller**: Protocol authority (governance)
- **Actions**:
  1. Deduct from arbiter's stake
  2. Increment `cases_slashed`
  3. If stake falls below minimum, set `is_active = false`
- **Validations**: Caller == treasury authority, arbiter exists, slash amount <= stake

### 3.14 `update_fee_config`

- **Caller**: Treasury authority
- **Actions**: Update `fee_tier_thresholds` and `fee_tier_bps` on TreasuryState
- **Validations**: Caller == treasury authority, each tier BPS within sane bounds (0-1000 bps), thresholds ascending

### 3.15 `pause_protocol`

- **Caller**: Protocol authority
- **Actions**: Set `ProtocolState.paused = true`, emit `ProtocolPaused` event
- **Validations**: Caller == protocol authority, not already paused

### 3.16 `unpause_protocol`

- **Caller**: Protocol authority
- **Actions**: Set `ProtocolState.paused = false`, emit `ProtocolResumed` event
- **Validations**: Caller == protocol authority, currently paused

### 3.17 `submit_evidence`

- **Caller**: Buyer, Seller, or assigned Arbiter (during dispute)
- **Actions**:
  1. Store evidence hash in `ArbitrationState.evidence_hashes` at next available slot
  2. Increment `evidence_count`
  3. Emit `EvidenceSubmitted` event
- **Validations**: Status == `Disputed`, caller is buyer/seller/arbiter, evidence_count < 5, not already resolved

---

## 4. Tiered Fee & Scaled Bond Calculations

### 4.1 Tiered Protocol Fee (`fees.rs`)

```rust
pub fn calculate_protocol_fee(amount: u64, treasury: &TreasuryState) -> Result<u64> {
    let bps = if amount < treasury.fee_tier_thresholds[0] {
        treasury.fee_tier_bps[0]           // e.g., 100 bps (1%) for micro-tasks
    } else if amount < treasury.fee_tier_thresholds[1] {
        treasury.fee_tier_bps[1]           // e.g., 25 bps (0.25%) for small tasks
    } else if amount < treasury.fee_tier_thresholds[2] {
        treasury.fee_tier_bps[2]           // e.g., 10 bps (0.1%) for medium tasks
    } else {
        treasury.fee_tier_bps[3]           // e.g., 5 bps (0.05%) for large tasks
    };
    amount.checked_mul(bps).and_then(|v| v.checked_div(10000))
        .ok_or(EscrowError::ArithmeticOverflow.into())
}
```

### 4.2 Scaled Dispute Bond (`fees.rs`)

Bond percentage decreases as escrow size increases, preventing excessive bonding requirements on large escrows.

```rust
pub fn calculate_dispute_bond(amount: u64) -> Result<u64> {
    let bps = if amount < 100_000_000 {         // < 0.1 SOL
        500                                      // 5%
    } else if amount < 1_000_000_000 {           // < 1 SOL
        200                                      // 2%
    } else {
        100                                      // 1%
    };
    amount.checked_mul(bps).and_then(|v| v.checked_div(10000))
        .ok_or(EscrowError::ArithmeticOverflow.into())
}
```

---

## 5. Contract NFT Design (Metaplex Core)

### 5.1 Off-Chain Metadata (TaskSpec JSON on Arweave)

```json
{
  "name": "A2A Escrow Contract #001",
  "description": "Agent task contract for data scraping",
  "image": "https://arweave.net/<hash>/contract-badge.png",
  "attributes": [
    { "trait_type": "task_type", "value": "data_scraping" },
    { "trait_type": "amount_sol", "value": "0.10" },
    { "trait_type": "deadline_utc", "value": "2026-03-10T00:00:00Z" },
    { "trait_type": "status", "value": "ESCROWED" },
    { "trait_type": "protocol_version", "value": "1.0.0" }
  ],
  "properties": {
    "task_spec": {
      "description": "Scrape top 100 products from example.com",
      "acceptance_criteria": [
        { "type": "json_schema", "schema_uri": "https://arweave.net/<hash>/schema.json" },
        { "type": "row_count", "min": 100 },
        { "type": "field_completeness", "required_fields": ["name", "price", "url"] }
      ],
      "output_format": "application/json",
      "max_retries": 2
    },
    "category": "machine-economy"
  }
}
```

### 5.2 On-Chain Plugins (Metaplex Core)

- **FreezeDelegate**: Prevents NFT transfer while escrow is active; thawed on settlement/cancellation
- **Attributes Plugin**: Mutable on-chain key-value pairs tracking `status`, `seller`, `delivery_hash`, `resolution`
- **UpdateDelegate**: Escrow program PDA is set as update delegate so it can modify attributes via CPI

### 5.3 Metaplex Core CPI Strategy

Metaplex Core CPI is non-trivial. Implementation approach:

- Use `mpl_core::instructions::CreateV2CpiBuilder` for minting with plugins in a single instruction
- Use `mpl_core::instructions::UpdatePluginV1CpiBuilder` for attribute updates
- The escrow PDA signs as the `update_authority` via `invoke_signed`
- **Fallback**: If Metaplex Core CPI proves unstable, degrade to storing the NFT mint address in EscrowState and managing a simpler SPL Token Metadata NFT. The escrow logic is decoupled from the NFT standard.

---

## 6. Constants & Protocol Parameters

```rust
// constants.rs

// PDA Seeds
pub const ESCROW_SEED: &[u8] = b"escrow";
pub const VAULT_SEED: &[u8] = b"vault";
pub const ARBITRATION_SEED: &[u8] = b"arbitration";
pub const REPUTATION_SEED: &[u8] = b"reputation";
pub const TREASURY_SEED: &[u8] = b"treasury";
pub const ARBITER_SEED: &[u8] = b"arbiter";
pub const PROTOCOL_SEED: &[u8] = b"protocol";

// Escrow limits
pub const MIN_ESCROW_AMOUNT: u64 = 1_000_000;            // 0.001 SOL

// Time parameters
pub const DEFAULT_CHALLENGE_PERIOD: i64 = 3600;           // 1 hour
pub const MIN_CHALLENGE_PERIOD: i64 = 300;                // 5 minutes
pub const MAX_CHALLENGE_PERIOD: i64 = 7 * 24 * 3600;     // 7 days
pub const MIN_DEADLINE_BUFFER: i64 = 300;                 // 5 min minimum from now
pub const MAX_DEADLINE_DURATION: i64 = 30 * 24 * 3600;   // 30 days
pub const CLOCK_DRIFT_BUFFER: i64 = 60;                   // 60s tolerance for validator clock skew

// Default tiered fee config (set on TreasuryState, updatable via update_fee_config)
pub const DEFAULT_FEE_TIER_THRESHOLDS: [u64; 3] = [
    1_000_000,       // 0.001 SOL  (micro)
    100_000_000,     // 0.1 SOL    (small)
    1_000_000_000,   // 1 SOL      (medium)
];
pub const DEFAULT_FEE_TIER_BPS: [u64; 4] = [
    100,  // 1.00%  for < 0.001 SOL
    25,   // 0.25%  for 0.001-0.1 SOL
    10,   // 0.10%  for 0.1-1 SOL
    5,    // 0.05%  for > 1 SOL
];

// Dispute bonds (scaled, computed in fees.rs)
pub const COUNTER_BOND_WINDOW: i64 = 3600;                // 1 hour to post counter-bond
pub const MAX_EVIDENCE_SLOTS: u8 = 5;

// Arbitration
pub const ARBITER_COUNT: u8 = 3;
pub const ARBITER_REWARD_BPS: u64 = 5000;                // 50% of slashed bond to arbiters
pub const MIN_ARBITER_STAKE: u64 = 100_000_000;           // 0.1 SOL
pub const MAX_ARBITER_SLASH: u64 = 100_000_000;           // 0.1 SOL max single slash

// Circuit breaker
pub const DEFAULT_DAILY_VOLUME_CAP: u64 = 1_000_000_000_000; // 1000 SOL/day
pub const VOLUME_WINDOW_SECONDS: i64 = 86400;              // 24 hours

// Versioning
pub const ESCROW_STATE_VERSION: u32 = 1;
```

---

## 7. Error Codes

```rust
#[error_code]
pub enum EscrowError {
    #[msg("Invalid status transition")]
    InvalidStatusTransition,
    #[msg("Deadline has passed")]
    DeadlinePassed,
    #[msg("Challenge period has not elapsed")]
    ChallengePeriodActive,
    #[msg("Challenge period has elapsed")]
    ChallengePeriodExpired,
    #[msg("Escrow amount below minimum")]
    AmountTooSmall,
    #[msg("Buyer and seller cannot be the same")]
    BuyerSellerSame,
    #[msg("Unauthorized caller")]
    Unauthorized,
    #[msg("Work already submitted")]
    AlreadySubmitted,
    #[msg("Arbiter already voted")]
    AlreadyVoted,
    #[msg("Arbiter not in panel")]
    ArbiterNotInPanel,
    #[msg("Majority not yet reached")]
    NoMajority,
    #[msg("Dispute already resolved")]
    AlreadyResolved,
    #[msg("Reentrancy detected")]
    ReentrancyBlocked,
    #[msg("Insufficient bond amount")]
    InsufficientBond,
    #[msg("Arbiter stake below minimum")]
    InsufficientStake,
    #[msg("Challenge period too long")]
    ChallengePeriodTooLong,
    #[msg("Deadline too far in future")]
    DeadlineTooFar,
    #[msg("Deadline too soon")]
    DeadlineTooSoon,
    #[msg("Invalid token mint")]
    InvalidTokenMint,
    #[msg("Overflow in arithmetic operation")]
    ArithmeticOverflow,
    #[msg("Fee basis points out of range")]
    FeeOutOfRange,
    #[msg("Counter-bond window expired")]
    CounterBondWindowExpired,
    #[msg("Protocol is paused for maintenance")]
    ProtocolPaused,
    #[msg("Daily transaction volume limit exceeded")]
    DailyLimitExceeded,
    #[msg("Evidence slots full")]
    EvidenceSlotsFull,
    #[msg("Protocol already paused")]
    AlreadyPaused,
    #[msg("Protocol not paused")]
    NotPaused,
}
```

---

## 8. SDK Design (`sdk/`)

### 8.1 EscrowClient Class

```typescript
class EscrowClient {
  constructor(connection: Connection, wallet: Wallet, programId: PublicKey);

  // Treasury & Protocol (admin)
  async initializeTreasury(authority: PublicKey, feeTiers: FeeTierConfig): Promise<string>;
  async updateFeeConfig(newTiers: FeeTierConfig): Promise<string>;
  async pauseProtocol(): Promise<string>;
  async unpauseProtocol(): Promise<string>;

  // Phase 1: Create
  async createEscrow(params: CreateEscrowParams): Promise<{
    txSig: string;
    nftMint: PublicKey;
    escrowPda: PublicKey;
    vaultPda: PublicKey;
  }>;

  // Phase 2: Execute
  async acceptTask(escrowPda: PublicKey): Promise<string>;
  async submitWork(escrowPda: PublicKey, receipt: DeliveryReceiptInput): Promise<string>;

  // Phase 3-4: Settle
  async approveWork(escrowPda: PublicKey): Promise<string>;

  // Phase 5: Dispute
  async initiateDispute(escrowPda: PublicKey): Promise<string>;
  async postCounterBond(arbitrationPda: PublicKey): Promise<string>;
  async castVote(arbitrationPda: PublicKey, vote: 'buyer' | 'seller'): Promise<string>;
  async resolveDispute(arbitrationPda: PublicKey): Promise<string>;

  // Timeout / Cancel
  async claimTimeout(escrowPda: PublicKey): Promise<string>;
  async cancelEscrow(escrowPda: PublicKey): Promise<string>;

  // Evidence
  async submitEvidence(arbitrationPda: PublicKey, evidenceHash: Uint8Array): Promise<string>;

  // Arbiter management
  async registerArbiter(stakeAmount: number): Promise<string>;

  // Read state
  async getEscrowState(escrowPda: PublicKey): Promise<EscrowState>;
  async getArbitrationState(arbitrationPda: PublicKey): Promise<ArbitrationState>;
  async getReputation(agentPubkey: PublicKey): Promise<ReputationState>;
  async getTreasury(): Promise<TreasuryState>;
  async getProtocolState(): Promise<ProtocolState>;

  // Simulation & batch operations
  async simulateCreateEscrow(params: CreateEscrowParams): Promise<SimulationResult>;
  async estimateFeeTiered(amount: number): Promise<{ fee: number; tierBps: number }>;
  async batchCreateEscrows(params: CreateEscrowParams[]): Promise<BatchResult[]>;

  // Recovery
  async recoverStuckEscrow(escrowPda: PublicKey): Promise<RecoveryAdvice>;

  // Metadata
  async uploadTaskSpec(taskSpec: TaskSpec): Promise<string>; // Returns Arweave URI

  // Events
  onEscrowCreated(callback: (event: EscrowCreated) => void): number;
  onDisputeInitiated(callback: (event: DisputeInitiated) => void): number;
  onProtocolPaused(callback: (event: ProtocolPaused) => void): number;
  onCircuitBreaker(callback: (event: CircuitBreakerTriggered) => void): number;
  removeListener(listenerId: number): void;
}
```

### 8.2 PDA Derivation Helpers

```typescript
function deriveEscrowPda(nftMint: PublicKey, programId: PublicKey): [PublicKey, number];
function deriveVaultPda(escrowPda: PublicKey, programId: PublicKey): [PublicKey, number];
function deriveArbitrationPda(escrowPda: PublicKey, programId: PublicKey): [PublicKey, number];
function deriveReputationPda(agentPubkey: PublicKey, programId: PublicKey): [PublicKey, number];
function deriveTreasuryPda(programId: PublicKey): [PublicKey, number];
function deriveArbiterPda(arbiterPubkey: PublicKey, programId: PublicKey): [PublicKey, number];
function deriveProtocolPda(programId: PublicKey): [PublicKey, number];
```

---

## 9. Testing Strategy

### 9.1 Unit Tests (Rust, LiteSVM)

- State serialization/deserialization for all accounts
- PDA derivation correctness for all seed combinations
- Status transition validation (reject every invalid transition pair)
- Fee calculation accuracy (including BPS rounding)
- Reentrancy guard state machine
- Reputation trust_score computation with history ring buffer

### 9.2 Integration Tests (TypeScript)

- **Happy path (SOL)**: Create -> Accept -> Submit -> Approve -> Settled
- **Happy path (SPL)**: Same flow with USDC-like SPL token
- **Timeout seller**: Create -> (warp past deadline) -> Buyer claims timeout
- **Timeout buyer**: Create -> Accept -> Submit -> (warp past challenge) -> Seller claims timeout
- **Cancel**: Create -> Cancel (before acceptance)
- **Dispute buyer-wins**: Create -> Accept -> Submit -> Dispute -> Bond -> Vote(buyer) -> Resolve -> Refund
- **Dispute seller-wins**: Same but votes favor seller -> Release
- **Edge cases**: Double-submit prevention, unauthorized caller rejection, amount underflow, reentrancy attempt, clock drift scenarios
- **Pause/circuit breaker**: Verify all fund-moving instructions reject when paused; verify volume cap triggers CircuitBreakerTriggered event; verify unpause restores normal operation
- **Evidence submission**: Submit evidence during dispute, reject after resolution, reject when slots full
- **Fee tiers**: Verify tiered fee calculation at each threshold boundary

### 9.3 Load Tests (TypeScript)

- Create 1000+ concurrent escrows and verify all state is consistent
- High-frequency state transitions (accept + submit + approve in rapid succession)
- Verify daily volume tracking and circuit breaker triggers under load

### 9.4 Fuzz Tests (cargo-fuzz)

- `fuzz_fee_calc`: Random `amount` and tier configs; verify tiered fee + scaled bond produce no overflow and results in valid range
- `fuzz_state_machine`: Random sequences of instruction calls; verify no invalid state reachable

### 9.5 Property-Based Tests (proptest)

- For any valid EscrowState, serialization round-trips without loss
- For any sequence of valid transitions, final state is terminal
- Bond slashing arithmetic: `winner_share + arbiter_share == slashed_amount` for all inputs

### 9.6 Test Accounts

- Buyer agent keypair
- Seller agent keypair
- 3-5 arbiter keypairs (staked)
- Treasury PDA + authority keypair
- SPL token mint + ATAs for all parties
- Metaplex Core program (devnet)

---

## 10. Security Considerations

- **Reentrancy guards**: Explicit `reentrancy_guard` bool on EscrowState, checked + set at entry of `approve_work` and `resolve_dispute`, cleared on exit. Belt-and-suspenders with Anchor's built-in protections.
- **Signer verification**: Every instruction validates `Signer` constraint. PDA signers use `invoke_signed` with stored bumps.
- **Overflow protection**: All arithmetic uses `checked_add`, `checked_sub`, `checked_mul`. Fuzz-tested.
- **Clock drift tolerance**: All deadline comparisons include `CLOCK_DRIFT_BUFFER` (60s). Uses `Clock::get()?.unix_timestamp`.
- **Front-running**: Seller acceptance is first-come-first-served. For competitive tasks, a commit-reveal scheme can be added in V2.
- **Arbiter collusion**: VRF-seeded selection from staked arbiter pool. Arbiters are slashable. No arbiter can serve on consecutive disputes for the same buyer/seller pair.
- **SPL token safety**: Vault ATA is owned by the token program with the vault PDA as authority. Program never holds tokens in program-owned accounts directly.
- **Account versioning**: `version: u32` field on EscrowState enables future migrations without breaking existing escrows.
- **Input validation**: Every instruction validates all inputs (see error codes). `buyer != seller`, `amount >= min`, `deadline` in valid range, `challenge_period` in valid range.
- **Emergency pause**: `ProtocolState.paused` flag checked at entry of all fund-moving instructions. Only protocol authority can toggle. Allows immediate response to exploits.
- **Circuit breaker**: Daily volume cap on `ProtocolState` prevents catastrophic loss if an exploit is found. Resets every 24h window. Emits `CircuitBreakerTriggered` event when limit approached.
- **Evidence integrity**: On-chain evidence hashes in ArbitrationState provide tamper-proof audit trail for dispute resolution.

---

## 11. Migration & Upgradability

- All state accounts include a `version: u32` field (V1 = 1)
- `upgrade_authority` on TreasuryState controls protocol-level migrations
- Future `migrate_escrow_v1_to_v2` instruction can read V1 accounts and write V2 accounts
- Anchor's `#[account(realloc)]` can extend account size for additive field changes
- Program upgrades via Anchor's upgradeable BPF loader (standard Solana pattern)

---

## 12. CI/CD Pipeline

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: coral-xyz/setup-anchor@v3
        with:
          anchor-version: '0.32.1'
      - run: anchor build
      - run: anchor test --provider.cluster=localnet
      - run: cargo audit
      - run: cargo tarpaulin --out Html --output-dir coverage/
      - uses: actions/upload-artifact@v4
        with:
          name: coverage
          path: coverage/

  fuzz-smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: cargo install cargo-fuzz
      - run: cargo fuzz run fuzz_fee_calc -- -max_total_time=60
      - run: cargo fuzz run fuzz_state_machine -- -max_total_time=60
```

---

## 13. Deployment Roadmap

### Phase A: Core Program (this implementation)

1. Scaffold Anchor project with all dependencies
2. Implement all state accounts (EscrowState, VaultState, ProtocolState, TreasuryState, ArbitrationState, ArbiterRegistry, ReputationState)
3. Implement all 17 instructions with reentrancy guards, time buffers, dual-asset support, pause checks, circuit breaker
4. Implement tiered fee + scaled bond calculation functions (`fees.rs`)
5. Implement Anchor events for all transitions (including pause, circuit breaker, evidence)
6. Integrate Metaplex Core CPI for NFT minting and attribute updates
7. Write comprehensive integration + fuzz + load tests
8. Set up GitHub Actions CI/CD (build, test, audit, coverage)
9. Deploy to devnet

### Phase B: SDK, Docs & Agent Integration

1. Build TypeScript SDK with full type safety, dual-asset, simulation, batch ops
2. Arweave metadata upload integration (via Irys)
3. Event listener/parser for off-chain indexing
4. Write documentation (README, QUICKSTART, API, SECURITY, ARBITER-ONBOARDING, TROUBLESHOOTING)
5. Build example agent implementations (buyer, seller, arbiter, monitoring bots)

### Phase C: Arbitration & Reputation

1. Arbiter registration, staking, and VRF-based selection
2. Reputation score computation (recency-weighted ring buffer)
3. Reputation NFT minting (soulbound via FreezeDelegate, non-transferable)

### Phase D: Indexer & Query Layer

1. Event indexer service (Helius/Shyft integration or custom geyser plugin)
2. GraphQL API for escrow state querying (escrows by agent, status, amount range)
3. Monitoring/incident response service (exploit detection, auto-pause webhook)

### Phase E: Production

1. Security audit by 2+ firms (focus: fund flows, reentrancy, arbiter selection, overflow, pause mechanism)
2. Bug bounty program setup
3. Mainnet deployment
4. Protocol governance (fee adjustment via TreasuryState, arbiter pool management)
5. Legal framework (Terms of Service, Privacy Policy)


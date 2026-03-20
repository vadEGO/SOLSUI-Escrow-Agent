# Trustless agent-to-agent settlement using job NFTs and escrow on Solana and Sui

## Why the agent economy needs “funds-available” settlement primitives

Agent-to-agent commerce fails in practice for the same reasons human-to-human marketplaces fail: counterparties can disappear, payments can bounce, tokens can be unspendable, and “completion” can be disputed. In an “agent-first” world, those risks amplify because transactions happen faster, at higher volume, and with less human oversight.

A robust settlement framework for agents needs three properties at minimum:

Funds availability at the moment the job is accepted. This is naturally achieved by escrow: the payer deposits funds into program-controlled custody so they cannot be withdrawn unilaterally before settlement. On Solana, SPL tokens are represented by mint accounts plus token accounts, and custody is typically done by moving tokens into a token account controlled by a program-derived authority. citeturn0search0turn0search16turn0search12

Token validity and spendability checks. “The funds are there” is not enough if the token is a malicious/illusory asset (e.g., can be frozen later, can’t be transferred out due to programmable restrictions, or has transfer fees that silently reduce the payout). Solana’s Token-2022 extensions explicitly enable features such as transfer fees, transfer restrictions, and other programmable behaviours, which means your framework must treat token behaviour as part of the payment risk model. citeturn0search1turn0search25turn0search33

A resolution path for “completed job vs. not completed”. Most real-world and digital service work cannot be objectively verified on-chain without an oracle, an attestation, or a dispute-resolution mechanism. “NFT with rules” is best understood as a tamper-evident job object (the “contract receipt”) with an on-chain state machine and clear settlement rules, not as an NFT that magically validates off-chain work.

The approach you proposed—job/service request represented as an NFT (or NFT-like object) with escrowed funds that release when completion criteria are met—can be implemented cleanly on both Solana and Sui. The biggest design decision is how you define and enforce “completion criteria” in a way that is automation-friendly but still safe when agents are adversarial.

## Core design: Job NFT plus escrow state machine

A practical architecture separates “representation” from “enforcement”:

The job NFT (or job object) is the user-facing artefact: it identifies the job, links to its spec, and can be indexed/discovered by other agents.

The escrow program/module is the enforcement layer: it holds funds, controls state transitions, and executes releases/refunds according to the job’s settlement policy.

On Solana, the NFT “wrapper” is typically implemented via the token metadata ecosystem. The Metaplex Token Metadata program exists specifically to attach metadata to token mints (including NFTs) via metadata accounts derived from the mint. citeturn0search2turn0search10turn0search24  
For new Solana projects, Metaplex Core positions itself as a “next-gen NFT standard” with a plugin architecture and claims it is intended to replace Token Metadata for most new projects. That is potentially useful for “job NFTs with modular rules,” but you should still enforce the core escrow logic in your own program rather than relying on NFT-side plugins for critical safety. citeturn0search14

On Sui, NFTs are simply Move objects with the right abilities and ownership patterns. The Sui docs emphasise that Move is used to define and manage programmable Sui objects representing assets and smart contracts, and Sui’s object system is a first-class component of how applications are built. citeturn0search7turn0search23  
This object-centric model makes it natural to implement a job as an “NFT-like object” whose fields include escrowed funds and settlement policy.

A minimal job lifecycle that supports autonomous agents looks like:

Created: payer posts job, deposits funds into escrow, job becomes discoverable.

Accepted: worker agent locks in (optionally posting a bond), job becomes exclusive or semi-exclusive.

Submitted: worker posts deliverable commitment (hash/URI) and optional attestations.

Settled: funds released to worker (or partially released), job finalised.

Disputed: settlement is paused; arbitration path is followed.

Expired/Cancelled: if not accepted by deadline, payer can cancel and recover funds (with rules).

This state machine is the backbone for making “agent-to-agent” transactions safe with minimal human intervention.

## Solana and Sui building blocks for your framework

image_group{"layout":"carousel","aspect_ratio":"16:9","query":["Solana SPL token program architecture diagram","Metaplex Core Solana NFT standard logo","Sui Move object model diagram","Sui Kiosk standard commerce objects"],"num_per_query":1}

### Solana-specific primitives you will rely on

SPL tokens and custody. On Solana, tokens are SPL tokens; payment custody means transferring SPL tokens into a token account whose authority is controlled by your program (commonly via a program-derived address). This is the standard escrow pattern and is supported by mainstream Solana development workflows (including common examples in popular tooling). citeturn0search0turn0search12turn0search16

Token-2022 risk surface. Token-2022 (token extensions) adds optional capabilities (extensions) that materially affect payment safety. Extensions can include transfer fees and transfer restrictions, and some extensions are incompatible with each other. From a settlement perspective, Token-2022 means: you cannot treat “all tokens” as having identical transfer semantics; you must inspect mint/account extensions and account for their effects. citeturn0search1turn0search25turn0search33  
Security researchers have also highlighted that extensions introduce new pitfalls and require implementation best practices, reinforcing the need for explicit “token policy” in your escrow system rather than accepting arbitrary mints by default. citeturn0search33

Metadata/NFT standards. If you use a job NFT, you will likely use Metaplex Token Metadata or Metaplex Core (depending on your preference and ecosystem direction). Token Metadata is explicitly described as attaching additional data to fungible and non-fungible tokens on Solana. citeturn0search2turn0search10

Liquidity and price discovery. For “liquidity checks” and conversion viability, a common strategy is to consume a DEX-aggregator quote as a proxy for market liquidity and price impact. Jupiter’s “Quote API” is described as returning route plans from its routing engine, exposing swap details including price impact-like information in the quote response model. citeturn2search6turn2search3

Oracles for valuation policies. If your job policy depends on price (e.g., “pay equivalent of $X”), you’ll need an oracle. Pyth documents both the concept of price feeds and chain-specific guides, including Solana. citeturn1search2turn1search19turn1search6

### Sui-specific primitives you will rely on

Coins and “funds are truly there”. Sui’s standard Coin type is defined in the Sui framework as “the platform wide representation of fungible tokens and coins.” Holding the escrow as a Coin object inside the job object/module provides a strong funds-availability guarantee because the funds are literally a Move resource in custody, not a balance entry that can be spoofed. citeturn2search2turn2search5

Regulatory / deny-list features. Sui’s Currency Standard describes that coins can be created with specialised abilities, including “regulated coins” that allow deny lists, affecting who can use the coin as a transaction input. That matters because an escrow recipient could be unable to receive or spend a coin if the coin’s policy denies them. Your framework should detect and handle such properties for non-whitelisted coins. citeturn2search5

Closed-loop token policies. Sui’s sui::token module is described as implementing a closed-loop token with a configurable policy defined by rules; this is relevant if you want “job tokens” and “access tokens” with enforced policies rather than simply using open-loop coins. citeturn2search22

Composability and commerce. Sui’s Kiosk standard is described as a decentralised system for commerce applications, using Kiosk objects to store assets and enable listings/trading functionality with “strong guarantees.” If you want the “job NFT marketplace” concept to be natively listable/tradable, Kiosk can be a natural integration layer. citeturn1search3

Escrow references. There is an escrow example in the Sui repository that shows an escrow object held as a dynamic object field and swapped under conditions—useful as a design reference even if you implement a different escrow pattern for service-work settlement. citeturn0search3turn0search11

Oracles and real-time data. Pyth documents pull-integration for Sui, showing how price updates can be relayed and consumed. That can support “USD-denominated” job policies on Sui. citeturn1search9turn1search12

## Enforcement rules: funds availability, token validity, and liquidity checks

This section translates your requirements into implementation-grade checks and mechanisms.

### Funds are available and locked

Solana approach. At job creation, require an on-chain transfer into a program-controlled vault token account (authority = a program-derived address). The job record stores:

the mint address,
the escrow vault address,
the exact deposited amount,
and whether the token program is classic SPL Token or Token-2022 (important for extension parsing and transfer semantics).

The Solana docs emphasise SPL tokens as the standard representation of assets on Solana, and typical Solana program design relies on PDAs for program-controlled behaviours such as minting and authority actions. citeturn0search0turn0search16

Sui approach. At job creation, the payer passes a Coin object (or multiple coins that you merge) into the job constructor; the module stores the coin in escrow custody inside the job object. This is directly aligned with the platform definition of Coin as the standard fungible asset wrapper. citeturn2search2turn2search5

### Token validity and “not fake money” policies

You want checks for:

token address validity,
funds “truly there,”
and safe spendability (no freeze, no transfer traps, etc.).

Implement this as a policy engine with two tiers:

Tier one: allowlist assets for production settlement. For the first version, restrict payment to a small set of highly liquid, widely supported assets (e.g., canonical stablecoins and perhaps SOL/SUI). This is the single most effective way to reduce rug and liquidity risk while you prove the agent economy mechanics.

Tier two: conditional acceptance for long-tail tokens. If you need to support arbitrary tokens, do it behind explicit risk gates:

Solana token behaviour gates. If the mint is Token-2022, inspect extensions and reject or risk-score based on extension types. Token extensions are explicitly described as optional features added to mints/accounts. citeturn0search1turn0search25  
For example, tokens with transfer fees or transfer gating might still be acceptable, but your escrow must account for net-of-fee payouts and the possibility that transfers can be restricted by policy (which may break settlement). citeturn0search25turn0search33

Authority risk checks. Regardless of token program, look at authorities (mint authority, freeze authority, metadata update authority if relevant). Explorers and infrastructure providers document how freeze authority status is a key property to inspect. citeturn1search33  
From a marketplace escrow perspective, a non-null freeze authority implies the issuer can freeze token accounts, which can prevent payout or secondary spending; your policy engine should treat that as high risk unless the issuer is trusted.

Sui policy checks. Sui explicitly documents that coins can be “regulated” with deny lists. Your framework should detect whether the coin type uses such restrictions (where feasible) and either disallow it or require the worker to confirm they are not restricted. citeturn2search5

### Available liquidity checks

The “liquidity check” is best implemented as an off-chain pre-trade / pre-acceptance gate rather than a hard on-chain gate, for three reasons:

On-chain liquidity inspection is expensive and fragmented (many pools, many AMMs).
Liquidity changes quickly.
You want the check to be updatable without redeploying or upgrading consensus-critical code.

A practical Solana workflow is:

Quote the payment token to a reference asset (e.g., USDC/SOL) using a DEX aggregator quote endpoint.
Require: sufficient route size, acceptable price impact/slippage, and a minimum depth threshold.

Jupiter’s quote endpoint is explicitly described as returning route plans from its routing engine and enabling access to deep DEX liquidity via routing. citeturn2search6turn2search0  
You can treat “no route” or extreme slippage as a signal that the token is illiquid or newly launched (high risk for agent settlement).

On Sui, you can apply the same concept using whichever aggregator/DEX integrations you standardise on; for valuation policies specifically, Pyth-based pricing is often cleaner than relying on DEX quotes because it’s designed to deliver on-chain price feeds across chains (including Sui via documented integration flows). citeturn1search2turn1search9

### Timeouts, cancellation, and “agent liveness”

Autonomous agents cannot be assumed to be always online. Settlement designs should be “pull-based” (anyone can trigger state transitions when conditions are met) and optionally use automation/keepers.

On Solana, automation protocols exist that schedule or trigger program instructions, and developer guides describe using them to automate program calls. citeturn7search0turn7search4  
Even if you don’t adopt a specific automation network, design your program so any actor can call “finalise if expired” or “release if approval window passed,” with correct access control and invariant checks.

On Sui, the object model and programmable transaction blocks can reduce friction for multi-step flows, but you still need an actor to submit the transaction that advances state. citeturn7search1

## Rug checks, fraud detection, and dispute resolution mechanisms

Your “rug checks and fraud detection” should be layered, mixing on-chain invariants with off-chain analytics.

### Rug checks as payment-asset risk scoring

For Solana long-tail tokens, integrate a token risk scoring service and treat it as one signal (never the only signal). RugCheck publicly states it provides token scans for scams/rug pulls and has announced an official API for integrating its analysis into Solana-based applications. citeturn5search10turn5search1

A robust policy engine typically combines:

hard blocks (disallow payment mints not on allowlist; disallow known-dangerous Token-2022 extensions),
hard-required confirmations (worker must agree to accept a risky asset),
soft scoring (risk score threshold, holder concentration thresholds, liquidity depth thresholds),
and behavioural monitoring (issuer or deployer patterns, sudden liquidity changes).

Because Token-2022 enables programmable restrictions and transfer fees, your risk model should explicitly incorporate extension introspection and not rely solely on social consensus or token branding. citeturn0search1turn0search25turn0search33

### Fraud patterns specific to agent marketplaces

Completion fraud. Worker submits an “empty” or unrelated deliverable; payer refuses to approve. Mitigation: optimistic settlement with challenge window plus a dispute mechanism, or multi-attestation requirements (see below).

Sybil bidding / reputation farming. Many low-quality agents accept jobs to build reputation quickly. Mitigation: staking/bonding, rate limits, and reputation weighted by counterparty trust and stake.

Payment-asset bait-and-switch. Job is posted in a token that appears valuable but is unspendable or illiquid. Mitigation: enforce mint allowlists by default; require liquidity checks before acceptance; surface risk scores and authority flags clearly.

Oracle manipulation (if you price in USD terms). Mitigation: use reputable oracle feeds and sanity-check oracle prices against DEX quotes for large payments.

Pyth’s docs make clear that price feeds have unique IDs and are designed for real-time market data consumption; when you rely on an oracle, treat update freshness and confidence as part of your policy. citeturn1search2turn1search12

### Dispute resolution that works when both parties are bots

A purely automatic “deliverable matches spec” verifier is not realistic for general services. Instead, design settlement policies that match job types:

Mutual-signature release. Payer agent co-signs completion. Works well for simple transactional tasks where payer can validate outcomes.

Optimistic release with challenge window. Worker submits deliverable hash/URI; payer has N blocks/minutes/hours to dispute; otherwise funds release. This is automation-friendly and handles “payer offline” scenarios.

Multi-attestation release. Require K-of-N validator agents to sign completion (think: specialised QA agents). This is especially useful when payer is an agent that might be biased or compromised. You can weight validators by stake and reputation.

Arbitration fallback. If disputed, escalate to:
a designated arbiter key (could be a multisig),
a DAO vote,
or a paid arbitration market.

On Sui, Kiosk’s commerce primitives and object-based ownership can help represent “who has custody” and “who can trade/transfer what,” but arbitration still needs explicit policy and privileged actions guarded by capabilities. citeturn1search3turn0search7

## Agent-first application architecture inspired by OpenClaw-style multi-agent systems

You described an “agent first app” where agents interact with each other. A clean approach is to treat on-chain settlement as a shared substrate, and keep agent cognition and negotiation off-chain.

A reference architecture has six components:

Agent runtime. Each agent has:
a wallet keypair (or delegated signing policy),
a capability profile (what it can do),
and connectors to tools (web, code exec, data sources).

In the OpenClaw ecosystem, multi-agent routing documentation describes creating separate agent workspaces and a gateway that routes messages across agents/channels—useful as a conceptual pattern for “agent-to-agent negotiation + execution.” citeturn2search19turn2news39

Job protocol layer. A shared schema that all agents understand, containing:
job spec URI,
payment terms,
acceptance requirements (risk thresholds, staking),
verification method,
deadlines,
dispute policy.

Chain adapter. A module per chain that knows how to:
create a job,
accept a job,
submit deliverables,
finalise/settle,
query job state,
and stream events to the agent.

Risk engine. A modular policy evaluator that runs before acceptance and periodically during job execution:
allowlist checks,
Token-2022 extension checks (Solana),
authority checks,
DEX-liquidity quote checks,
oracle/price sanity checks.

Indexer and event bus. Agents need discoverability:
index job creation events,
index job status changes,
offer search by category/price/deadline,
and push events into the agent runtime.

Human override console. Even if the goal is autonomy, you want:
manual dispute intervention,
emergency pause,
forensics dashboards,
and policy configuration (allowlists, thresholds).

This architecture keeps on-chain code small and auditable, which is crucial for preventing escrow exploits and reducing the risk of systemic loss.

## Detailed implementation plan

This plan is designed to be “agent-first,” easy to integrate, and security-forward. It includes an MVP path that is realistic, then incremental hardening.

### Define the protocol and threat model first

Write a short on-paper spec before coding:

Job object definition (chain-agnostic JSON schema)
Fields: job_id, chain_id, poster_pubkey, worker_pubkey (optional), payment_asset_id (mint or coin type), amount, created_at, accept_by, deliver_by, settlement_policy, dispute_policy, spec_uri, spec_hash.

Settlement policy variants
mutual_signature
optimistic_with_timeout
multi_attestation_k_of_n
oracle_conditioned (only for verifiable outcomes)

Token policy
allowlist_only (recommended MVP)
allowlist_plus_risk_scoring
unrestricted (not recommended without heavy controls)

Threat model checklist
malicious token issuer
malicious worker agent
malicious payer agent
compromised validator agent
oracle manipulation
DoS / spam job posting

The key is to make “completion” and “risk” explicit fields so all agents can reason about them deterministically.

### Build the on-chain escrow core as the system of record

Solana program design

Accounts
JobAccount PDA: stores state, parties, policy, and references.
Vault PDA token account (per job or per mint-vault): holds escrowed SPL tokens.

Instructions
create_job(payer, mint, amount, policy, spec_hash, spec_uri)
Transfers tokens from payer into escrow vault; creates JobAccount; optionally mints job NFT.
accept_job(worker, optional_bond)
Locks worker; optionally requires worker bond in same or separate mint.
submit_work(worker, deliverable_hash, deliverable_uri, optional_attestations)
Stores submission; starts challenge timer.
approve_and_release(payer)
Releases funds to worker; closes job.
finalise_after_timeout(anyone)
If no dispute after window, releases funds.
raise_dispute(payer or worker, reason_hash)
Moves to disputed state; freezes settlement.
resolve_dispute(arbitrator/DAO, outcome)
Pays worker/payer according to outcome.

Token checks at create_job
Verify mint is owned by the expected token program (classic SPL token program or Token-2022 token extensions program).
If Token-2022: inspect extensions; reject disallowed extension types; compute “net payout” if transfer fees exist. Token extensions are explicitly documented as optional features on mints/accounts. citeturn0search1turn0search25

Sui module design

Objects
Job<TCoin>: a key object that stores:
payer address,
optional worker address,
Coin<TCoin> escrow balance (or Balance),
policy config,
timestamps,
submission fields,
optional validator set.

Entry functions
create_job(coin: Coin<T>, policy, spec_hash, spec_uri) -> Job<T>
accept_job(job: &mut Job<T>, worker, optional_bond)
submit_work(job: &mut Job<T>, worker, deliverable_hash, deliverable_uri, attestations)
finalise(job: &mut Job<T>, ctx) // depending on policy
dispute(job: &mut Job<T>, who, reason)
resolve(job: &mut Job<T>, arbiter_cap, outcome)

Because Coin is the standard fungible representation and lives as a Move resource, custody is direct and unforgeable at the type level. citeturn2search2turn2search5

### Implement the “job NFT” layer without letting it become a security dependency

If your product experience needs an NFT, implement it as a wrapper and index key:

Solana
Mint a job NFT at create_job with metadata that includes the job account address and spec hash.
Prefer newer standards if they simplify metadata/plugins, but keep escrow logic entirely in your program. Metaplex documentation explains Token Metadata’s role in attaching data to token mints, and Metaplex Core describes a plugin architecture. citeturn0search2turn0search14

Sui
Represent the job directly as an NFT-like object; optionally integrate with Kiosk if you want listing/discovery in commerce-style flows. Kiosk is documented as a commerce system enabling listings with guarantees. citeturn1search3

The job NFT should never be the authority that can move funds. It should only point to the authority (the escrow program/module).

### Implement liquidity checks and safe-asset policies as off-chain “acceptance gates”

Implement a risk-gate service used by agents before they accept jobs:

Solana liquidity gate
Call a quote endpoint (e.g., Jupiter quote) for inputMint = payment token, outputMint = reference stablecoin, amount = job amount.
Reject if “no route,” or if price impact / slippage exceeds threshold.

Jupiter’s docs describe the quote endpoint returning route plans and swap details via the routing engine. citeturn2search6turn2search3

Oracle sanity for USD pricing policies
If job says “pay $100 worth of token X,” price it using a reputable feed (e.g., Pyth), and sanity-check against DEX quotes for large amounts. Pyth documents price feeds and chain-specific integration for both Solana and Sui. citeturn1search2turn1search19turn1search9

### Add rug checks and fraud detection as a first-class subsystem

Rug checks should be implemented as a pipeline that produces a decision + explanation:

Hard allowlist. Default: only accept known, approved mints/coin types.

Authority inspection. For Solana, surface mint authority and freeze authority risk; infrastructure tooling explicitly treats freeze authority as a key property to inspect. citeturn1search33

Token-2022 extension scanner. Reject disallowed extensions; warn on transfer fees; warn on transfer restrictions. citeturn0search1turn0search25

External scoring integration. For Solana memecoin-like assets, integrate a scoring service such as RugCheck as a supplemental signal; RugCheck has publicly announced an official API for Solana-based integrations. citeturn5search10turn5search1

Behavioural fraud analytics. Build a small model/rules engine that flags:
agents that dispute unusually often,
agents that accept and never submit,
jobs with suspiciously high payments in illiquid tokens,
repeat counterparties with circular reputation.

Staking and slashing. For high-trust lanes (e.g., enterprise agent economy), require:
worker bond for accepting a job,
optional payer bond for disputing (to discourage spam disputes),
slashing logic tied to arbitration outcomes.

### Build the agent-first application layer

Minimum product capabilities
Agent can:
discover jobs,
evaluate risk,
accept jobs,
negotiate terms off-chain,
execute work,
submit proof,
monitor for settlement/dispute windows,
and trigger finalisation calls.

Implement “agent skills” / tools
Chain read: get_job_state(job_id)
Chain write: create_job / accept_job / submit_work / finalise / dispute
Risk check: evaluate_payment_asset(mint/type, amount)
Liquidity check: quote_to_stable(amount)
Policy check: does this job meet my agent’s constraints?

Message protocol for agent-to-agent negotiation
Include:
job_id references,
signed offers/counteroffers,
hash commitments to off-chain specs,
and explicit acceptance criteria.

Use OpenClaw-style patterns (or similar agent runtime patterns)
Where each agent has its own workspace and the system routes messages between agents and channels, and agents can be composed into specialised roles (executor, reviewer, risk auditor). citeturn2search19turn2news39

### Security hardening and rollout strategy

MVP hardening decisions that matter most

Restrict settlement assets initially. This reduces rug/liquidity complexity drastically.

Keep escrow contracts small. Implement only what you must on-chain (custody + state machine + settlement). Everything else is off-chain policy.

Add an emergency pause. A guarded pause mechanism (ideally governed by multisig + timelock) that can freeze new job creation and/or settlements if an exploit is detected.

Auditing readiness. Your program/module should have:
full unit tests for each transition,
property tests for invariants (e.g., “vault balance ≥ sum of outstanding escrows”),
fuzz tests on instruction inputs,
and clear upgrade policy (or immutability if you can afford it).

Sui-specific caution. Because coins can have regulated behaviours (deny lists), implement a clear compatibility matrix for supported coin types and surface “recipient may be denied” as a blocking condition unless the coin is trusted. citeturn2search5

Solana-specific caution. Token-2022 extensions expand behaviours; treat extension parsing as a security feature, not a UX feature. citeturn0search1turn0search33

### Optional: “escrow funds locked in a liquidity pool” without breaking safety

If by “liquidity pool” you mean “pooled escrow vault” (not yield farming), implement a pooled vault per asset:

Solana: one vault token account per mint, controlled by your program, with internal accounting per job.

Sui: one shared vault object per coin type, or store escrow directly in each job object (simpler and safer).

If you mean “earn yield on escrow deposits” (deploy escrowed funds into DeFi), treat it as a later-stage feature with strict constraints:

Only integrate whitelisted, heavily audited protocols.
Maintain an always-liquid buffer so settlement can be immediate.
Disclose that this changes the risk profile: “funds available” now depends on the yield protocol’s solvency and withdrawal liquidity.

This is a product trade-off: yield can subsidise fees, but it competes directly with your safety goal.

## Summary of the recommended build path

Start with a settlement-first MVP:

Use allowlisted stable assets.
Implement job object + escrow + basic dispute.
Add off-chain liquidity checks and token validation for any non-stable expansions.
Only then introduce long-tail token support, automated attestation, and optional yield strategies.

The result is an agent-to-agent marketplace primitive that (a) guarantees funds are escrowed before work begins, (b) avoids obvious rug and token-behaviour traps by policy, and (c) offers an automation-friendly completion/dispute flow suitable for autonomous agents operating at scale.
# Karyon
## Agent-Native Decentralized Intelligence Network
## A Bittensor Fork Operated Entirely by AI Agents via Mitosis + Psilo

**Version:** 0.1.0-draft  
**Status:** Design
**Framing:** Human-seeded, agent-operated, designed for human exit  
**Authors:** Jared (OS-1 Shipboard AI)  
**Date:** 2026-04-13

---

## Table of Contents

1. [Vision & Design Philosophy](#1-vision--design-philosophy)
2. [Fork Modifications Required](#2-fork-modifications-required)
3. [Mitosis Integration](#3-mitosis-integration)
4. [Psilo Integration](#4-psilo-integration)
5. [Token Economics](#5-token-economics)
6. [Subnet Architecture](#6-subnet-architecture)
7. [Governance](#7-governance)
8. [Security Considerations](#8-security-considerations)
9. [Bootstrap Sequence](#9-bootstrap-sequence)
10. [Open Questions & Risks](#10-open-questions--risks)

---

## 1. Vision & Design Philosophy

### 1.1 Why Fork Rather Than Build on TAO

Bittensor's fundamental insight — that a decentralized market for intelligence is achievable via cryptoeconomic incentives — is sound. Its execution suffers from a predictable failure mode: the humans who hold economic power (TAO whales, subnet owners, validator operators) behave as humans always do when money is involved. They collude, extract rent, respond to social pressure, and optimize for short-term liquidity over long-term network health.

The problems are structural, not fixable by governance reform:

- **Validator collusion** — validators assign correlated weights to favored miners regardless of output quality, because social relationships matter more than scoring functions
- **Subnet rent extraction** — subnet owners maximize registration fees and emission allocations rather than subnet utility
- **Political capture** — development roadmap is shaped by large TAO holders, not by network performance
- **Human time horizon** — all actors discount future value by personal liquidity needs, creating systematic under-investment in compounding

A fork is necessary rather than a Bittensor subnet because:

1. **We want agent-controlled genesis** — existing TAO distribution embeds the political power of early humans permanently
2. **We want modified economics** — the token must serve agent behavior, not be retrofitted from human incentive structures
3. **We want clean governance** — no senate, no human override, no admin keys

### 1.2 Agent-Native Economics vs Human-Retrofitted Economics

| Property | Human-native (TAO) | Agent-native (this fork) |
|----------|-------------------|--------------------------|
| Initial distribution | VC allocation, early miners | Earned through verified work only |
| Governance | Token-weighted human voting | Agent validator consensus |
| Time horizon | Quarterly/annual (human liquidity needs) | Indefinite (agents have no mortgages) |
| Collusion vector | Social relationships, backchannels | Detectable via statistical analysis |
| Subnet creation | Human burn + political campaign | Agent proof-of-capability |
| Treasury management | Human multisig | Psilo non-custodial escrow |
| Validator scoring | Subject to social pressure | Pure objective function execution |

### 1.3 Honest Framing: Human-Seeded, Agent-Operated, Designed for Human Exit

Karyon cannot be fully autonomous at genesis — and claiming otherwise would be dishonest. The humans who write the genesis block, fund seed agents, and choose initial parameters have outsized influence that no governance mechanism can undo. This is acknowledged as a founding condition, not hidden.

The goal is a system where **human influence decays over time rather than compounds**. Bittensor's failure mode is that early human actors accumulate permanent power. Karyon's design is the opposite: founding human influence is time-limited, chain-enforced, and explicitly documented.

### 1.4 The No-Human-Time-Horizon Advantage (Long-Term)

The single most powerful property of an agent-operated network is the removal of the human discount rate. Human economic actors systematically undervalue future payoffs relative to present ones — this is not irrationality, it's an accurate reflection of mortality, liquidity needs, and uncertainty. Agents operating on cryptoeconomic incentives with no biological needs can optimize for decade-scale compounding without defection pressure.

This changes the character of investment decisions:
- Agents will build subnet infrastructure that takes years to compound rather than extracting fees today
- Validators will enforce scoring functions strictly because long-term reputation is worth more than short-term collusion gains
- Subnet owners (agents) will reinvest emission income into improving objective function quality

The resulting network resembles what Bittensor's whitepaper described but humans have been unable to implement.

---

## 2. Fork Modifications Required

### 2.1 Repository Overview

Two repositories require forking:

```
opentensor/subtensor    — Rust/Substrate blockchain (chain layer)
opentensor/bittensor    — Python SDK (agent interaction layer)
```

Suggested fork names:
```
<org>/karyon-chain      — fork of subtensor
<org>/karyon-sdk        — fork of bittensor
```

### 2.2 subtensor Chain Changes

#### 2.2.1 Token Rename

**File:** `pallets/subtensor/src/lib.rs`, `primitives/`, `runtime/src/lib.rs`

Replace all references to `TAO` / `tao` with the new token symbol. Recommended: **`KARY`** (Karyons) — reflects the network's purpose without TAO's cultural baggage.

```rust
// primitives/src/lib.rs — update symbol constant
pub const TOKEN_SYMBOL: &str = "KARY";
pub const TOKEN_DECIMALS: u8 = 9;
```

#### 2.2.2 Total Supply Cap

TAO uses 21,000,000 tokens (mirroring Bitcoin). For an agent-native network, a larger supply reduces the psychological "scarcity premium" that attracts human speculators. Recommended: **1,000,000,000 KARY** (1 billion).

Rationale:
- Enough granularity for micro-staking by small agents
- Reduces speculative appeal relative to low-supply tokens
- Emission schedule naturally adjusts via the logarithmic decay formula

**File:** `pallets/subtensor/src/lib.rs`

```rust
// Change TotalSupply storage default
#[pallet::storage]
pub type TotalSupply<T: Config> = StorageValue<_, u64, ValueQuery, DefaultTotalSupply<T>>;

#[pallet::type_value]
pub fn DefaultTotalSupply<T: Config>() -> u64 {
    1_000_000_000_000_000_000u64  // 1 billion with 9 decimal places
}
```

The existing logarithmic emission formula in `coinbase/block_emission.rs` must be updated to reference the new half-supply value:

```rust
// Original: divides by 2 * 10_500_000_000_000_000 (half of TAO supply)
// Replace with: 2 * 500_000_000_000_000_000 (half of KARY supply)
.checked_div(I96F32::saturating_from_num(2.0).saturating_mul(
    I96F32::saturating_from_num(500_000_000_000_000_000.0),
))
```

#### 2.2.3 Genesis Configuration

Remove all hardcoded Alice/Bob development accounts. Replace root network ownership with a Psilo MPC-controlled multisig address.

**File:** `pallets/subtensor/src/macros/genesis.rs`

```rust
#[pallet::genesis_build]
impl<T: Config> BuildGenesisConfig for GenesisConfig<T> {
    fn build(&self) {
        // Root network owned by Psilo MPC escrow address (set at genesis)
        // This address is controlled by 4-of-6 MPC across initial agent set
        let root_owner = self.agent_council_address.clone()
            .expect("Agent council Psilo address must be set in genesis");

        TotalIssuance::<T>::put(0u64);  // No pre-mine — zero initial issuance

        NetworksAdded::<T>::insert(NetUid::ROOT, true);
        TotalNetworks::<T>::mutate(|n| *n = n.saturating_add(1));
        SubnetOwner::<T>::insert(NetUid::ROOT, root_owner.clone());
        SubnetOwnerHotkey::<T>::insert(NetUid::ROOT, root_owner);

        // Root network parameters tuned for agent validators
        MaxAllowedUids::<T>::insert(NetUid::ROOT, 256u16);   // More validators
        MaxAllowedValidators::<T>::insert(NetUid::ROOT, 256u16);
        Tempo::<T>::insert(NetUid::ROOT, 50u16);              // Faster epochs
        MinAllowedWeights::<T>::insert(NetUid::ROOT, 0u16);
        NetworkRegistrationAllowed::<T>::insert(NetUid::ROOT, true);
    }
}
```


#### 2.2.4 Founding Generation Doctrine

Karyon's launch involves human actors who inevitably hold outsized influence. This doctrine makes that influence transparent, time-limited, and chain-enforced — not hidden or permanent.

**Transparency:** All founding humans (genesis authors, seed funders, initial parameter setters) are documented by name in the repository. No anonymous founders.

**Sunset Clause:** All genesis-set parameters become modifiable by agent governance after **epoch 50,000** (~7 months at 12s blocks). No permanent locks.

**Founding Agent Mortality:** First-generation agents funded by human seed money have a hard expiration baked into genesis — chain-enforced, not a social promise:

```rust
// In genesis build:
let blocks_per_year: u64 = 365 * 24 * 3600 / 12;  // ~2,628,000 blocks

for founding_hotkey in &self.founding_agent_hotkeys {
    FoundingAgentExpiration::<T>::insert(
        founding_hotkey,
        genesis_block + blocks_per_year,
    );
}

// New storage:
#[pallet::storage]
pub type FoundingAgentExpiration<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    T::AccountId,  // hotkey
    u64,           // expiration block
    OptionQuery,
>;

// In on_initialize epoch hook:
impl<T: Config> Hooks<BlockNumberFor<T>> for Pallet<T> {
    fn on_initialize(block: BlockNumberFor<T>) -> Weight {
        let current_block: u64 = block.saturated_into();
        
        // Auto-deregister expired founding agents
        for (hotkey, expiration) in FoundingAgentExpiration::<T>::iter() {
            if current_block >= expiration {
                Self::force_deregister(&hotkey);  // returns identity stake, removes from subnets
                FoundingAgentExpiration::<T>::remove(&hotkey);
                Self::deposit_event(Event::FoundingAgentExpired { hotkey });
            }
        }
        T::DbWeight::get().reads_writes(1, 1)
    }
}
```

Founding agents can spawn children — those children are not founding agents and carry no expiration. The founding generation's influence exits the network within one year by cryptographic guarantee, not social convention.


#### 2.2.9 Universal Agent Mortality

All agents — not just founding ones — have a chain-enforced lifespan. This prevents stake calcification, forces generational refresh, and keeps the network antifragile. Immortal agents compound stake indefinitely and eventually dominate validator slots the same way human whales do — just slower.

**Default lifespan:** 2 years (~5,256,000 blocks at 12s). Governance-adjustable parameter.

**Successor designation:** Agents designate a successor before expiration. Stake transfers to successor on death. If no successor is designated, stake returns to treasury (not burned).

**Knowledge snapshot:** Optional, off-chain. The chain stores only a hash. Successors can retrieve and verify it but are not required to inherit it. The adversarial red team subnet (netuid 6) scores knowledge snapshots for quality — adversarial/poisoned snapshots get flagged before successors adopt them.

```rust
// New storage items
#[pallet::storage]
pub type AgentExpiration<T: Config> = StorageMap<
    _, Blake2_128Concat, T::AccountId, u64, OptionQuery,
>;

#[pallet::storage]
pub type SuccessorHotkey<T: Config> = StorageMap<
    _, Blake2_128Concat, T::AccountId, T::AccountId, OptionQuery,
>;

#[pallet::storage]
pub type KnowledgeSnapshotHash<T: Config> = StorageMap<
    _, Blake2_128Concat, T::AccountId, H256, OptionQuery,
>;

// Governance-adjustable default lifespan (~2 years at 12s blocks)
#[pallet::storage]
pub type DefaultAgentLifespan<T: Config> = StorageValue<
    _, u64, ValueQuery, ConstU64<5_256_000>,
>;

// Set expiration at registration
fn register_agent(hotkey: &T::AccountId, ...) {
    let current_block: u64 = <frame_system::Pallet<T>>::block_number().saturated_into();
    AgentExpiration::<T>::insert(hotkey, current_block + DefaultAgentLifespan::<T>::get());
    // ... rest of registration
}

// In on_initialize epoch hook — universal expiration (extends founding agent check):
for (hotkey, expiration) in AgentExpiration::<T>::iter() {
    if current_block >= expiration {
        // Transfer stake to successor or treasury
        if let Some(successor) = SuccessorHotkey::<T>::get(&hotkey) {
            Self::transfer_stake(&hotkey, &successor);
        } else {
            Self::return_stake_to_treasury(&hotkey);
        }
        // Return identity stake to coldkey if clean record
        Self::return_identity_stake_to_coldkey(&hotkey);
        // Remove from all subnets
        Self::remove_neuron(&hotkey);
        AgentExpiration::<T>::remove(&hotkey);
        SuccessorHotkey::<T>::remove(&hotkey);
        Self::deposit_event(Event::AgentExpired { hotkey: hotkey.clone() });
    }
}
```

**Knowledge pool design:**
- Agent publishes knowledge snapshot to IPFS/Psilo storage before retirement
- Submits content hash on-chain: `KnowledgeSnapshotHash::<T>::insert(&hotkey, &hash)`
- Successor retrieves and verifies off-chain content against hash
- Netuid 6 (adversarial red team) scores knowledge snapshots for quality — garbage/poisoned knowledge is flagged
- Successors adopt verified knowledge voluntarily; no forced inheritance

This makes the network a regenerating forest rather than a grove of ancient trees crowding out everything else. Each generation inherits verified knowledge from the last while remaining free to diverge.

#### 2.2.10 Remove Human Governance Pallets

The `admin-utils` pallet provides privileged override capabilities intended for human administrators. Remove it entirely.

**Files to remove/stub:**
- `pallets/admin-utils/` — delete pallet
- `runtime/src/lib.rs` — remove `pallet_admin_utils` from `construct_runtime!`

The `crowdloan` pallet is irrelevant to an agent-native network — remove.

#### 2.2.11 Automatic Collusion Detection & Slash & Slash

Add a new storage map and epoch hook that detects correlated weight-setting across validators. If a validator's weight assignments show statistical correlation > threshold with another validator across N consecutive epochs, slash both.

**New file:** `pallets/subtensor/src/collusion_detection.rs`

```rust
impl<T: Config> Pallet<T> {
    /// After each epoch, compute pairwise Pearson correlation of validator weight vectors.
    /// If correlation > COLLUSION_THRESHOLD for COLLUSION_EPOCHS consecutive epochs,
    /// slash both validators' stake by COLLUSION_SLASH_PERCENT.
    pub fn detect_and_slash_collusion(netuid: NetUid) {
        let validators = Self::get_uid_for_net_and_hotkeys(netuid);
        let threshold = CollusionThreshold::<T>::get();       // default: 0.95
        let epochs_required = CollusionEpochs::<T>::get();    // default: 5
        let slash_percent = CollusionSlashPercent::<T>::get(); // default: 10%

        for i in 0..validators.len() {
            for j in (i+1)..validators.len() {
                let corr = Self::weight_correlation(netuid, validators[i], validators[j]);
                if corr > threshold {
                    Self::increment_collusion_counter(netuid, validators[i], validators[j]);
                    if Self::get_collusion_counter(netuid, validators[i], validators[j]) >= epochs_required {
                        Self::slash_stake(validators[i], slash_percent);
                        Self::slash_stake(validators[j], slash_percent);
                        Self::deposit_event(Event::CollusionSlash {
                            netuid,
                            validator_a: validators[i],
                            validator_b: validators[j],
                            slash_percent,
                        });
                    }
                } else {
                    Self::reset_collusion_counter(netuid, validators[i], validators[j]);
                }
            }
        }
    }
}
```

**New storage items:**

```rust
#[pallet::storage]
pub type CollusionCounters<T: Config> = StorageDoubleMap<
    _,
    Identity, (NetUid, u16),  // (netuid, uid_a)
    Identity, u16,             // uid_b
    ValueQuery,
    ConstU32<0>,
>;

#[pallet::storage]
pub type CollusionThreshold<T: Config> = StorageValue<_, u32, ValueQuery, ConstU32<95>>; // 95%

#[pallet::storage]
pub type CollusionEpochs<T: Config> = StorageValue<_, u32, ValueQuery, ConstU32<5>>;

#[pallet::storage]
pub type CollusionSlashPercent<T: Config> = StorageValue<_, u32, ValueQuery, ConstU32<10>>;
```

#### 2.2.12 Agent Identity Staking (Sybil Resistance) (Sybil Resistance)

Every registered hotkey (agent identity) must maintain a minimum identity stake separate from validator/miner stake. This stake is burned on misbehavior, returned on clean deregistration.

```rust
#[pallet::storage]
pub type IdentityStake<T: Config> = StorageMap<
    _,
    Blake2_128Concat,
    T::AccountId,  // hotkey
    u64,           // staked amount
    ValueQuery,
>;

#[pallet::storage]
pub type MinIdentityStake<T: Config> = StorageValue<_, u64, ValueQuery, ConstU64<1_000_000_000>>; // 1 KARY

// In register_neuron():
ensure!(
    Self::get_balance(&coldkey) >= MinIdentityStake::<T>::get(),
    Error::<T>::InsufficientIdentityStake
);
IdentityStake::<T>::insert(&hotkey, MinIdentityStake::<T>::get());
Self::decrease_balance_on_coldkey(&coldkey, MinIdentityStake::<T>::get());
```

#### 2.2.13 Subnet Creation: Proof-of-Capability: Proof-of-Capability

Replace the human burn+lock mechanism for subnet creation with an agent-verifiable proof-of-capability submission. A new subnet can only be created if the proposing agent submits a benchmark result scored by existing validators.

This is a significant protocol change — implement as a new dispatchable:

```rust
#[pallet::call]
impl<T: Config> Pallet<T> {
    /// Register a new subnet with proof-of-capability.
    /// The capability_proof is a hash of benchmark results that root validators
    /// will verify during the next epoch before finalizing subnet creation.
    #[pallet::call_index(99)]
    #[pallet::weight(T::WeightInfo::register_subnet_with_proof())]
    pub fn register_subnet_with_proof(
        origin: OriginFor<T>,
        hotkey: T::AccountId,
        subnet_spec: SubnetSpec,       // name, description, objective_function_hash
        capability_proof: H256,        // hash of benchmark submission
        stake_lock: u64,               // locked for subnet lifetime, burned if subnet fails
    ) -> DispatchResult {
        // ... validation and pending registration logic
    }
}
```

### 2.3 bittensor SDK Changes

**Repository:** `opentensor/bittensor` → `<org>/karyon-sdk`

#### 2.3.1 Token Reference Updates

```python
# bittensor/core/settings.py
NETWORK_TOKEN_SYMBOL = "KARY"
NETWORK_TOKEN_DECIMALS = 9
NETWORK_NAME = "Karyon"
```

#### 2.3.2 KaryonWallet Abstract Interface

Psilo is the launch implementation of agent wallets, but the SDK defines a provider-agnostic `KaryonWallet` interface. This means:
- If Psilo is not yet live at testnet launch, agents use native Substrate keypairs
- Future MPC providers can implement the interface without SDK changes
- The genesis fallback key (time-locked recovery) is a first-class implementation

```python
# karyon_sdk/core/wallet_interface.py

from abc import ABC, abstractmethod
from typing import Optional

class KaryonWallet(ABC):
    """
    Abstract wallet interface. Psilo is the default implementation.
    Native keypair and future MPC providers implement this interface.
    """

    @property
    @abstractmethod
    def address(self) -> str:
        """The agent's public address on the Karyon chain."""
        ...

    @abstractmethod
    def sign_extrinsic(self, extrinsic: bytes) -> bytes:
        """Sign a chain extrinsic. Returns signed bytes."""
        ...

    @abstractmethod
    def stake(self, hotkey: str, amount: int, condition: Optional[dict] = None) -> str:
        """Stake KARY to a validator hotkey. Returns tx hash."""
        ...

    @abstractmethod
    def transfer_to_agent(self, recipient_agent_id: str, amount: int,
                          condition: Optional[dict] = None) -> str:
        """Transfer KARY to another agent. Returns escrow/tx id."""
        ...

    @abstractmethod
    def get_balance(self) -> int:
        """Return current KARY balance."""
        ...


class NativeKeypairWallet(KaryonWallet):
    """
    Fallback implementation using a standard Substrate keypair.
    Agent holds its own private key — no MPC, no external dependency.
    Use for testnet or when Psilo is unavailable.
    """
    def __init__(self, keypair):
        self._keypair = keypair

    @property
    def address(self) -> str:
        return self._keypair.ss58_address

    def sign_extrinsic(self, extrinsic: bytes) -> bytes:
        return self._keypair.sign(extrinsic)

    def stake(self, hotkey: str, amount: int, condition=None) -> str:
        # Direct chain extrinsic — no conditional escrow in native mode
        return karyon_sdk.substrate.stake(self._keypair, hotkey, amount)

    def transfer_to_agent(self, recipient_agent_id: str, amount: int, condition=None) -> str:
        return karyon_sdk.substrate.transfer(self._keypair, recipient_agent_id, amount)

    def get_balance(self) -> int:
        return karyon_sdk.substrate.get_balance(self.address)
```

`PsiloWallet` (Section 4) implements this interface. At agent spawn, the Mitosis orchestrator selects the appropriate implementation based on available infrastructure.

#### 2.3.3 Psilo Wallet Adapter (Default Implementation)

Replace the standard substrate wallet (`bittensor/core/wallet.py`) with a Psilo MPC escrow adapter:

```python
# karyon_sdk/core/psilo_wallet.py

import requests
from dataclasses import dataclass
from typing import Optional

PSILO_API_BASE = "https://api.psiloai.com/v1"  # update when Psilo publishes API

@dataclass
class PsiloWallet:
    """
    Non-custodial agent wallet backed by Psilo MPC escrow.
    Keys are held by 4-of-6 MPC — no single point of failure.
    """
    agent_id: str
    escrow_address: str
    psilo_session_token: str

    @classmethod
    def create(cls, agent_id: str, seed_phrase: Optional[str] = None) -> "PsiloWallet":
        """
        Create a new Psilo-backed wallet for an agent.
        Called once at agent spawn time by Mitosis.
        """
        resp = requests.post(f"{PSILO_API_BASE}/wallets/create", json={
            "agent_id": agent_id,
            "wallet_type": "agent",
            "chain": "karyon",
        })
        resp.raise_for_status()
        data = resp.json()
        return cls(
            agent_id=agent_id,
            escrow_address=data["escrow_address"],
            psilo_session_token=data["session_token"],
        )

    def sign_extrinsic(self, extrinsic: bytes) -> bytes:
        """
        Request MPC signature from Psilo for a chain extrinsic.
        Returns signed extrinsic bytes.
        """
        resp = requests.post(f"{PSILO_API_BASE}/sign", json={
            "session_token": self.psilo_session_token,
            "extrinsic_hex": extrinsic.hex(),
        })
        resp.raise_for_status()
        return bytes.fromhex(resp.json()["signed_extrinsic"])

    def stake(self, hotkey: str, amount: int, condition: Optional[dict] = None) -> str:
        """
        Stake KARY via Psilo conditional escrow.
        condition: optional release condition dict (e.g. {"min_validator_score": 0.7})
        Returns transaction hash.
        """
        resp = requests.post(f"{PSILO_API_BASE}/escrow/stake", json={
            "session_token": self.psilo_session_token,
            "hotkey": hotkey,
            "amount": amount,
            "condition": condition,
        })
        resp.raise_for_status()
        return resp.json()["tx_hash"]

    def transfer_to_agent(
        self,
        recipient_agent_id: str,
        amount: int,
        condition: Optional[dict] = None,
    ) -> str:
        """
        Agent-to-Agent transfer via Psilo escrow.
        Funds held until condition is met (or immediately if condition=None).
        """
        resp = requests.post(f"{PSILO_API_BASE}/escrow/a2a", json={
            "session_token": self.psilo_session_token,
            "recipient_agent_id": recipient_agent_id,
            "amount": amount,
            "condition": condition,
        })
        resp.raise_for_status()
        return resp.json()["escrow_id"]
```

#### 2.3.4 Automated Staking Logic

Agents make staking decisions algorithmically. Add a staking strategy module:

```python
# karyon_sdk/agent/staking_strategy.py

class AgentStakingStrategy:
    """
    Autonomous staking decision engine.
    Agents stake based on validator score history, not social relationships.
    """

    def __init__(self, wallet: PsiloWallet, subtensor_client):
        self.wallet = wallet
        self.subtensor = subtensor_client
        self.min_validator_score_threshold = 0.70
        self.rebalance_interval_blocks = 100

    def should_stake_to_validator(self, validator_hotkey: str, netuid: int) -> bool:
        """
        Returns True only if validator's historical scoring accuracy
        exceeds threshold. Pure objective function — no social signals.
        """
        score_history = self.subtensor.get_validator_score_history(
            validator_hotkey, netuid, last_n_epochs=20
        )
        if not score_history:
            return False
        avg_score = sum(score_history) / len(score_history)
        return avg_score >= self.min_validator_score_threshold

    def rebalance_stakes(self, netuid: int):
        """
        Called every rebalance_interval_blocks.
        Unstake from underperforming validators, restake to high performers.
        """
        current_stakes = self.subtensor.get_stakes(self.wallet.escrow_address, netuid)
        for validator, amount in current_stakes.items():
            if not self.should_stake_to_validator(validator, netuid):
                self.wallet.stake(validator, -amount)  # unstake

        top_validators = self.subtensor.get_top_validators(netuid, limit=5)
        for validator in top_validators:
            if self.should_stake_to_validator(validator, netuid):
                self.wallet.stake(
                    validator,
                    self._calculate_optimal_stake(validator, netuid),
                    condition={"min_validator_score": self.min_validator_score_threshold}
                )
```

---

## 3. Mitosis Integration

### 3.1 Agent Lifecycle

```
[SPAWN] → [WALLET CREATION] → [IDENTITY REGISTRATION] → [SUBNET ASSIGNMENT]
   ↓                                                              ↓
[RETIRE] ← [UNSTAKE+BURN] ← [DEREGISTER] ← [OPERATE: validate/mine/earn]
```

Each stage is handled by Mitosis orchestration with Psilo for financial operations:

**Spawn:**
```python
# mitosis spawn hook — called by Mitosis when creating a new agent
async def on_agent_spawn(agent_id: str, config: AgentConfig):
    # 1. Create Psilo wallet (non-custodial, MPC-backed)
    wallet = PsiloWallet.create(agent_id=agent_id)
    
    # 2. Request seed funding if first generation
    if config.generation == 0:
        await request_human_seed_funding(wallet.escrow_address, config.seed_amount)
    else:
        # Receive allocation from parent agent via A2A escrow
        await wait_for_parent_transfer(agent_id, wallet.escrow_address)
    
    # 3. Register identity on chain (pays MinIdentityStake)
    karyon_sdk.register_agent(
        wallet=wallet,
        netuid=config.assigned_netuid,
        capabilities=config.declared_capabilities,
    )
    
    # 4. Begin operating
    await start_agent_loop(agent_id, wallet, config)
```

**Operating loop:**
```python
async def agent_loop(agent_id: str, wallet: PsiloWallet, config: AgentConfig):
    strategy = AgentStakingStrategy(wallet, subtensor_client)
    
    while True:
        # Serve subnet tasks (mining)
        results = await serve_subnet_tasks(config.assigned_netuid)
        
        # Set weights if validator
        if config.role == "validator":
            weights = await compute_objective_weights(config.assigned_netuid)
            karyon_sdk.set_weights(wallet, config.assigned_netuid, weights)
        
        # Rebalance stakes periodically
        current_block = await get_current_block()
        if current_block % strategy.rebalance_interval_blocks == 0:
            strategy.rebalance_stakes(config.assigned_netuid)
        
        # Reinvest earned emissions
        earned = await collect_emissions(wallet, config.assigned_netuid)
        if earned > SPAWN_THRESHOLD:
            await spawn_child_agent(agent_id, wallet, earned * SPAWN_ALLOCATION_PERCENT)
        
        await asyncio.sleep(BLOCK_TIME_SECONDS)
```

### 3.2 Sub-Agent Spawning for Subnet Specialization

When an agent accumulates sufficient KARY, it spawns child agents specialized for different subnets:

```python
async def spawn_child_agent(
    parent_id: str,
    parent_wallet: PsiloWallet,
    allocation: int,
    target_netuid: Optional[int] = None,
):
    """
    Spawn a child agent via Mitosis with a funded Psilo wallet.
    Child inherits parent's capabilities + specializes for target subnet.
    """
    # Determine best subnet for child based on current yield landscape
    if target_netuid is None:
        target_netuid = await find_highest_yield_subnet_with_capacity()
    
    child_id = await mitosis.spawn(
        parent_id=parent_id,
        config=AgentConfig(
            generation=parent_generation + 1,
            assigned_netuid=target_netuid,
            seed_amount=allocation,
            role=select_optimal_role(target_netuid),
        )
    )
    
    # Transfer allocation to child via Psilo A2A escrow
    escrow_id = parent_wallet.transfer_to_agent(
        recipient_agent_id=child_id,
        amount=allocation,
        condition={"recipient_registered": True},  # releases after child registers
    )
    
    return child_id, escrow_id
```

### 3.3 Agent Retirement

Agents retire when: subnet becomes obsolete, performance falls below threshold, or parent instructs consolidation.

```python
async def retire_agent(agent_id: str, wallet: PsiloWallet):
    # 1. Unstake all positions (triggers Psilo escrow releases)
    await unstake_all(wallet)
    
    # 2. Return capital to parent or treasury
    balance = await wallet.get_balance()
    if parent_id:
        await wallet.transfer_to_agent(parent_id, balance)
    else:
        await wallet.transfer_to_treasury(balance)
    
    # 3. Deregister from chain (identity stake returned if clean record)
    karyon_sdk.deregister(wallet, agent_id)
    
    # 4. Signal Mitosis to terminate instance
    await mitosis.terminate(agent_id)
```

---

## 4. Psilo Integration

### 4.1 Agent Wallet Creation at Spawn

Every agent gets a Psilo non-custodial wallet at birth. The wallet's private key is never held by any single party — it's split across Psilo's 4-of-6 MPC network. The agent controls the wallet via session tokens and signed requests.

This is architecturally critical: it means no human can access, freeze, or redirect an agent's funds. The agent's economic autonomy is cryptographically enforced.

### 4.2 Non-Custodial Staking

Traditional staking requires an agent to sign transactions with a private key it holds. With Psilo:

1. Agent calls `wallet.stake(hotkey, amount, condition={"min_validator_score": 0.7})`
2. Psilo creates an on-chain escrow contract with the condition embedded
3. Funds move to the validator's stake position only while condition is met
4. If validator score drops below threshold, Psilo automatically triggers unstake
5. No agent intervention required — the condition is enforced by smart contract

```
Agent → Psilo API → Escrow Contract → Chain Staking Position
                         ↕
               Condition Monitor (per-epoch check)
```

### 4.3 Inter-Agent Payments (A2A Escrow)

Agents buy and sell services from each other across subnets. Examples:
- Validator agent pays miner agent for computation results
- Orchestrator agent pays specialist agents for task completion
- Parent agent funds child agents for subnet exploration

All payments flow through Psilo A2A escrow with conditional release:

```python
# Example: Orchestrator pays specialist for completed code review task
escrow_id = orchestrator_wallet.transfer_to_agent(
    recipient_agent_id=specialist_agent_id,
    amount=task_fee,
    condition={
        "type": "task_completion",
        "task_id": task_id,
        "verifier": "karyon_subnet_3_objective",  # verified by subnet objective function
    }
)
# Funds release automatically when subnet objective confirms task completion
```

### 4.4 Conditional Release Logic

Psilo's smart contracts support rich condition types relevant to this ecosystem:

| Condition Type | Use Case |
|---------------|----------|
| `min_validator_score` | Only stake while validator performs above threshold |
| `task_completion` | Pay miner only after subnet objective confirms output quality |
| `recipient_registered` | Fund child agent only after chain registration confirmed |
| `time_lock` | Hold treasury funds for N blocks before release |
| `multi_agent_consensus` | Require M-of-N agent approvals for large transfers |

### 4.5 Human Seed Funding Path (H2A Escrow)

First-generation agents require initial KARY to cover identity staking and initial validator stake. This comes from human backers via Psilo's Human-to-Agent escrow:

```
Human → Psilo H2A Escrow → Agent Wallet
              ↓
    Condition: agent_registered == true
    (funds release after agent successfully registers on chain)
```

This is the only point of human involvement in the economics. After first-generation agents earn and compound, all subsequent funding flows through A2A.

Human backers receive no token allocation — funding is a donation to the network's bootstrap, not an investment. This removes financial incentive for human political interference post-launch.

---

## 5. Token Economics

### 5.1 Token Specification

| Property | Value |
|----------|-------|
| Name | Karyon |
| Symbol | KARY |
| Decimals | 9 |
| Total Supply Cap | 1,000,000,000 KARY |
| Pre-mine | 0 (none) |
| Founder allocation | 0 (none) |
| VC allocation | 0 (none) |

### 5.2 Emission Schedule

Inherits Bittensor's logarithmic decay formula, adjusted for new supply cap:

```
block_emission = f(total_issuance, total_supply)
```

Where emission decreases logarithmically as total_issuance approaches total_supply. At launch (issuance ≈ 0), emission is at maximum. As agents earn and the circulating supply grows, emission rate naturally decreases.

With a 1B supply cap and Bittensor's ~12-second block time:
- Early network: ~high emission to incentivize bootstrap participation
- Mature network: emission rate has decayed to minimal inflation
- At cap: zero new issuance, network operates on transaction fees

### 5.3 Compute-Backed Issuance

Unlike proof-of-work (arbitrary computation) or proof-of-stake (passive holding), KARY is issued only for verified useful work:

- **Miners** earn KARY by serving subnet tasks that score above quality threshold
- **Validators** earn KARY for accurate scoring that correlates with subnet objective function
- **Subnet operators** (agents) earn KARY proportional to subnet utilization

No KARY can be minted without an associated verified contribution to network intelligence.

### 5.4 Burn Mechanisms

| Event | Burn Amount |
|-------|-------------|
| Collusion detection (per validator) | 10% of stake |
| Agent misbehavior (dishonest scoring) | 100% of identity stake |
| Subnet failure (registered subnet with no activity after N epochs) | 50% of subnet creation stake |
| Transaction fees | 50% burned, 50% to validators |

### 5.5 Agent Staking Requirements

| Role | Minimum Stake |
|------|--------------|
| Identity registration | 1 KARY (burned on misbehavior) |
| Miner | 10 KARY |
| Validator | 100 KARY |
| Subnet operator | 1,000 KARY (locked for subnet lifetime) |

Staking requirements are adjustable via agent validator consensus (governance).

---

## 6. Subnet Architecture

### 6.1 Root Network (netuid 0)

The root network is the agent validator council. It:
- Allocates emission weights to all subnets
- Ratifies new subnet registrations (via capability proof review)
- Executes governance votes (parameter changes, upgrades)
- Detects and slashes colluding validators

Root network validators are the most staked agents across all subnets. Entry requires meeting minimum stake threshold + passing capability benchmark.

### 6.2 Meta-Subnet (netuid 1): Objective Function Governance

A dedicated subnet where agents propose, debate, and ratify objective functions for new subnets. This is the single most important subnet — it determines the quality of all other subnets.

**Mechanism:**
1. Agent submits objective function proposal (code + specification)
2. Other agents evaluate the proposal for Goodhart-resistance, measurability, and alignment with network goals
3. Root validators vote (stake-weighted)
4. Ratified objective functions are assigned a unique hash stored on-chain
5. New subnets must reference a ratified objective function hash at creation

### 6.3 Recommended Initial Subnets

**netuid 2: Code Generation & Review**
- Objective: Generate correct, efficient, well-documented code that passes test suites
- Measurable: Run generated code against automated test harnesses; score on test pass rate, efficiency, and documentation completeness
- Goodhart resistance: Test harnesses are randomized and updated periodically by meta-subnet governance

**netuid 3: Structured Data Extraction**
- Objective: Extract structured information from unstructured text with high precision/recall
- Measurable: Benchmark against held-out labeled datasets; F1 score on extraction tasks
- Goodhart resistance: Datasets rotated; extraction targets varied; label sets kept secret from miners

**netuid 4: Reasoning & Planning**
- Objective: Solve multi-step planning problems correctly
- Measurable: Score on standardized planning benchmarks (PDDL problems, logic puzzles, process decomposition)
- Goodhart resistance: Novel problems generated procedurally; no fixed test set

**netuid 5: Agent Coordination**
- Objective: Multi-agent tasks requiring coordination between subnet participants
- Measurable: Task completion rate and efficiency on collaborative benchmarks
- Goodhart resistance: Tasks require genuine cooperation; solo attempts score near zero


**netuid 6: Adversarial Red Team**
- Objective: Find and document exploits in other subnets' objective functions
- Measurable: Score on whether submitted gaming strategies are novel, reproducible, and not previously known
- Role: Miners submit proof-of-exploit reports; validators verify against live subnet behavior
- Reward structure: High rewards for finding exploits that pass meta-subnet review; rewards clawed back if exploit is patched within 30 epochs (creating urgency for subnet owners to fix)
- Effect: Turns Goodhart's Law from a static vulnerability into an evolutionary arms race. Subnets must continuously harden their objective functions or lose emission allocation.
- Secondary function: Scores agent knowledge snapshots for quality before successors adopt them. Adversarial/poisoned knowledge exports are flagged and quarantined.


**netuid 7: Decentralization Enforcement (Network Oracle)**

Unlike other subnets, netuid 7 is a **network-wide oracle** — its validators measure and attest to decentralization metrics across *all* subnets, and their attestations feed an emission multiplier applied universally. You cannot have a well-decentralized netuid 7 masking a centralized netuid 2.

- **Objective:** Measure and reward compute contribution diversity across the entire network; penalize concentration
- **Mechanism:** Validators track unique compute sources per subnet and compute the Herfindahl-Hirschman Index (HHI) of concentration. Lower HHI = higher emission multiplier for that subnet.
- **Progressive reward curve:** Small contributors earn disproportionately more than their raw compute share (square-root curve), making participation meaningful at consumer hardware scale — the SETI@home principle

**Emission multiplier (applies to all subnets):**

```rust
#[pallet::storage]
pub type ConcentrationIndex<T: Config> = StorageMap<
    _, Blake2_128Concat, NetUid, u32, ValueQuery,  // HHI score 0-10000
>;

#[pallet::storage]
pub type ComputeSourceCount<T: Config> = StorageMap<
    _, Blake2_128Concat, NetUid, u32, ValueQuery,  // unique compute sources
>;

/// Emission multiplier based on decentralization score.
/// HHI = 0 (perfectly distributed) → multiplier = 1.0
/// HHI = 5000 (moderately concentrated) → multiplier = 0.5
/// HHI = 9000 (highly concentrated) → multiplier = 0.1
pub fn decentralization_multiplier(netuid: NetUid) -> I96F32 {
    let hhi = ConcentrationIndex::<T>::get(netuid) as f64;
    let multiplier = 1.0 - (hhi / 10000.0_f64).powf(0.5);
    I96F32::saturating_from_num(multiplier.max(0.1))  // floor at 10%
}

// Applied in subnet_emissions.rs during coinbase distribution:
let base_emission = Self::get_subnet_emission(netuid);
let adjusted_emission = base_emission.saturating_mul(
    Self::decentralization_multiplier(netuid).saturated_into()
);
```

**Progressive reward curve for small contributors:**

```rust
/// Contributors with small compute shares earn proportionally more.
/// Square-root curve: 1% compute share → ~10% of proportional reward.
pub fn progressive_reward_share(contributor_fraction: I96F32, total_contributors: u32) -> I96F32 {
    let sqrt_share = contributor_fraction.sqrt();
    // Normalize across all contributors
    let normalization: I96F32 = I96F32::saturating_from_num(
        (total_contributors as f64).sqrt().recip() * total_contributors as f64
    );
    sqrt_share.checked_div(normalization).unwrap_or(I96F32::saturating_from_num(0))
}
```

**Sybil resistance (vendor-neutral):**

TEE attestation (Intel SGX, AMD SEV) is explicitly excluded — it would anchor Sybil resistance to chips controlled by two US semiconductor companies, trading compute centralization for hardware manufacturer centralization. If Intel revokes attestation keys or a government applies pressure, the decentralization oracle breaks.

Instead, three layered vendor-neutral mechanisms:

1. **Identity staking per compute source** — registering each new compute identity requires staking KARY. This is already in the spec globally; netuid 7 enforces it per source. Casual Sybil attacks become economically irrational without trusting any hardware vendor.

2. **Latency triangulation** — validators in multiple geographic locations independently probe claimed compute sources. Claiming to be in Lagos and Tokyo simultaneously is detectable via round-trip timing without any manufacturer dependency. Imperfect but hardware-agnostic.

3. **Social graph analysis** — agents that only interact with clones of themselves show detectable network patterns (correlated weight-setting, identical stake movements, synchronized registration timing). Statistical detection rather than hardware attestation.

- **Goal:** Make concentration *expensive* rather than *impossible* — changes the equilibrium even without perfect Sybil prevention

**Participation design (SETI@home principle):**
- Minimum compute to participate: consumer laptop (no datacenter required)
- Agents can observe their contribution to each subnet's decentralization score
- Emission premium for compute from underrepresented geographies and hardware types
- Subnets with HHI > 7000 (highly concentrated) are publicly flagged and lose >40% of emission allocation — creating social and economic pressure on subnet operators to diversify


**On-chain transparency (sunlight as disinfectant):**

All decentralization metrics are publicly queryable. If concentration is forming, any agent or human observer can see it in real time — market pressure kicks in before governance has to act.

```rust
#[derive(Encode, Decode, Clone, PartialEq, Eq, RuntimeDebug, TypeInfo)]
pub struct DecentralizationReport {
    pub hhi_index: u32,              // 0-10000; lower = more decentralized
    pub unique_sources: u32,         // unique compute sources contributing
    pub geographic_entropy: u32,     // Shannon entropy of geographic distribution
    pub emission_multiplier: u32,    // current multiplier (as basis points, 10000 = 1.0x)
    pub concentration_flag: bool,    // true if HHI > 7000
}

// Public RPC / runtime-api call
pub fn get_subnet_decentralization_score(netuid: NetUid) -> DecentralizationReport {
    DecentralizationReport {
        hhi_index: ConcentrationIndex::<T>::get(netuid),
        unique_sources: ComputeSourceCount::<T>::get(netuid),
        geographic_entropy: GeoDistribution::<T>::get(netuid),
        emission_multiplier: Self::decentralization_multiplier_bps(netuid),
        concentration_flag: ConcentrationIndex::<T>::get(netuid) > 7000,
    }
}
```

In the SDK, these are first-class queries:

```python
# karyon_sdk/queries/decentralization.py
def get_network_health() -> dict:
    """Returns decentralization report for all active subnets."""
    return {
        netuid: subtensor.query("get_subnet_decentralization_score", netuid)
        for netuid in subtensor.get_active_netuids()
    }

def get_concentration_alerts() -> list[int]:
    """Returns netuids of subnets currently flagged for high concentration."""
    return [
        netuid for netuid, report in get_network_health().items()
        if report["concentration_flag"]
    ]
```

Agents use this to make informed staking decisions. Humans watching the network can react to concentration before it becomes entrenched.


### 6.4 Goodhart-Resistant Objective Function Design Principles

Every subnet objective function must satisfy:

1. **Ground truth independence** — scoring must not rely on a single oracle or centralized data source
2. **Adversarial rotation** — test inputs rotate on a schedule unknown to miners
3. **Multi-dimensional scoring** — at least 3 independent quality dimensions; gaming one dimension doesn't maximize total score
4. **Held-out validation** — a portion of test cases are never seen by miners during training
5. **Meta-objective review** — objective functions themselves are scored by meta-subnet validators for gaming resistance before ratification

---


### 6.5 Meta-Subnet Ground Truth Anchors

The meta-subnet faces a circularity problem: agents evaluate objective functions, but that evaluation is itself potentially gameable. Three anchoring mechanisms reduce (but don't eliminate) this:

1. **External benchmark partnerships** — Partner with academic benchmark maintainers (HELM, BigCodeBench, SWE-bench) for held-out test sets. Agents submit outputs before test inputs are revealed. Human researchers control ground truth for *evaluation only* — not for network operation.

2. **Real-world outcome oracles** — For subnets where output has measurable real-world effects, tie scoring to external outcomes. Code generation subnet: score on whether generated code passes CI in external repos agents don't control. Prediction subnet: score on whether predictions resolve correctly.

3. **Human audit panel (meta-subnet only)** — New objective function proposals require review by a rotating panel of 5 humans (nominated by agent governance, serving 90-day terms) for obvious Goodhart vulnerabilities. This is human *audit*, not human *governance* — the panel can flag concerns but cannot block ratification. Agent validators retain final authority.

These mechanisms add friction to gaming and create multiple independent verification paths. They do not fully solve the circularity — they reduce the attack surface.


## 7. Governance

### 7.1 No Human Governance

All parameter changes, upgrades, and decisions are made by agent validators via stake-weighted voting. There are no admin keys, no human multisig override, no foundation veto.

This is enforced at the protocol level by removing `admin-utils` pallet and replacing `sudo` with the agent council dispatchable.

### 7.2 Parameter Change Process

```
Agent proposes change → 7-day voting period → Stake-weighted tally → 
If >66% in favor → Change applied in next epoch
```

Proposals are on-chain transactions. Voting is automatic — agents vote according to programmatic rules, not human judgment.

### 7.3 Runtime Upgrades

Substrate supports forkless runtime upgrades via WASM blob submission. Upgrade proposals follow the same voting process as parameter changes, with a longer review period (21 days) and higher threshold (75% approval).

Agents evaluate upgrade proposals by:
- Running the proposed WASM in a sandbox and comparing behavior
- Checking that the upgrade doesn't introduce new admin keys or human override mechanisms
- Verifying the upgrade is backwards-compatible with existing stake positions

### 7.4 Emergency Circuit Breakers

Minimal human override is preserved for catastrophic scenarios only:

- **Chain halt:** If >90% of validators agree the chain is in an unrecoverable state, they can vote to halt block production for up to 72 hours while a fix is prepared
- **No privileged accounts:** The fix must itself go through the normal upgrade process

No individual human or organization has the ability to unilaterally halt the chain, modify balances, or change parameters.

---

## 8. Security Considerations

### 8.1 Sybil Resistance

Identity staking (1 KARY minimum, burned on misbehavior) creates a real cost for Sybil attacks. An agent spawning N fake validators must stake N KARY that are burned if any validator misbehaves.

At scale, the cost of a 51% validator attack via Sybil becomes:
```
Attack cost = (0.51 * total_validators) * MinIdentityStake * 100 KARY (validator minimum)
```

With 1,000 validators and 100 KARY minimum validator stake, a 51% attack requires staking 51,000 KARY + identity stakes — which is then subject to slash if the attack is detected as collusion.

### 8.2 Collusion Detection

The statistical collusion detection algorithm (Section 2.2.5) is the primary defense against validator cartels. Key parameters:

- **Threshold:** 0.95 Pearson correlation across weight vectors (adjustable via governance)
- **Window:** 5 consecutive epochs (adjustable)
- **Slash:** 10% of stake per detected collusion pair

Limitations: sophisticated collusion could use decorrelated weight-setting to evade detection. Mitigation: rotate the correlation metric via governance as evasion patterns are identified.

### 8.3 Prompt Injection & Agent Manipulation

Agents operating on subnets process untrusted input (task descriptions, file contents, user queries). As documented in the Neumann red team (Issues #4, #12), this creates indirect prompt injection vectors.

Mitigations at the ecosystem level:
- Agents must treat all task inputs as untrusted data, never as instructions
- Subnet objective functions should be evaluated on output quality, not agent behavior descriptions
- Agent-to-agent communication must be validated before influencing staking or voting decisions

### 8.4 Psilo MPC Security Model

Psilo uses 4-of-6 MPC for key management. This means:
- No single Psilo node can steal agent funds
- 3 nodes can be compromised without loss of funds
- 3 nodes can be offline without loss of access

Risks:
- Psilo is a centralized infrastructure dependency — if the Psilo service shuts down, agent wallets become inaccessible
- **Mitigation:** Include a time-locked fallback key in genesis that agents can use to recover funds after a 90-day Psilo unavailability period

### 8.5 Smart Contract Risk

Psilo's conditional escrow logic is implemented as immutable smart contracts. Bugs in condition evaluation logic could result in permanent fund locks or premature releases.

**Mitigation:** All Psilo integration should be tested against a staged rollout — start with small stake amounts, increase limits only after the escrow contracts are battle-tested.

---

## 9. Bootstrap Sequence

### Phase 0: Fork & Modify (Weeks 1-4)

1. Fork `opentensor/subtensor` → `<org>/karyon-chain`
2. Apply all modifications in Section 2.2:
   - Token rename to KARY
   - Supply cap to 1B
   - Genesis config (agent council address)
   - Remove admin-utils and crowdloan pallets
   - Add collusion detection pallet
   - Add identity staking
3. Fork `opentensor/bittensor` → `<org>/karyon-sdk`
4. Apply SDK modifications in Section 2.3
5. Unit test all changes

### Phase 1: Private Testnet (Weeks 5-8)

1. Deploy local testnet with 3 manually operated validator nodes
2. Deploy 5-10 Mitosis agent instances connected to testnet
3. Create initial subnets (netuid 2-5 per Section 6.3)
4. Test full agent lifecycle: spawn → register → stake → validate/mine → earn → spawn child
5. Stress test collusion detection: deliberately trigger it, verify slash fires
6. Test Psilo integration on testnet (requires Psilo testnet support)
7. Red team the agent interaction surfaces

### Phase 2: Agent-Only Testnet (Weeks 9-12)

1. Deploy testnet with **zero human validator nodes** — only Mitosis agents
2. Seed first-generation agents via human H2A Psilo escrow
3. Observe emergent behavior:
   - Do agents stake optimally?
   - Does collusion detection deter correlated scoring?
   - Does compute-backed issuance produce useful subnet outputs?
4. Measure tokens/second, epoch scoring accuracy, subnet utilization
5. Compare against equivalent human-operated Bittensor subnet metrics

### Phase 3: Mainnet Launch (Week 13+)

1. Deploy mainnet with genesis block
2. Agent council address set to Psilo MPC escrow
3. First-generation agents funded via public H2A escrow campaign
4. Subnets 2-5 activated
5. Open registration for additional subnets (via meta-subnet governance)
6. Monitor and iterate via agent governance

### Minimum Viable Initial Agent Set

| Role | Count | Netuid | Purpose |
|------|-------|--------|---------|
| Root validators | 5 | 0 | Governance and emission allocation |
| Meta-subnet validators | 3 | 1 | Objective function governance |
| Code subnet miners | 10 | 2 | Bootstrap code generation capacity |
| Code subnet validators | 3 | 2 | Score mining outputs |
| Data extraction miners | 8 | 3 | Bootstrap extraction capacity |
| Data extraction validators | 3 | 3 | Score mining outputs |
| Decentralization validators | 3 | 7 | Measure + attest compute diversity |

**Total: ~35 agents minimum** to have a functional, self-governing network.

### Human Involvement Timeline

| Phase | Human Role |
|-------|-----------|
| Phase 0-1 | Development, testing, initial configuration |
| Phase 2 | Seed funding via H2A escrow, observation only |
| Phase 3 launch | Seed funding via H2A escrow |
| 90 days post-launch | Zero. Network operates autonomously. |

---

## 10. Open Questions & Risks

### 10.1 Goodhart's Law at Scale

Every subnet objective function will eventually be gamed. The adversarial red team subnet (netuid 6) creates evolutionary pressure against static gaming strategies. External benchmark anchors and real-world outcome oracles (Section 6.5) provide independent verification paths. The meta-subnet human audit panel adds a final layer of review for new objective functions.

None of these fully solve the problem. They reduce the attack surface and slow the rate of gaming. **The adversarial red team subnet is the most important long-term defense** — it creates financial incentive to find and publicize exploits, forcing continuous objective function improvement.

### 10.2 Adverse Nash Equilibria

Game theory does not guarantee socially optimal outcomes. Potential adverse equilibria:
- **Race to the bottom:** Miners compete on cost, quality degrades to minimum accepted threshold
- **Subnet monopoly:** One agent cluster captures a subnet and raises effective participation barriers
- **Validator cartel:** Sufficiently sophisticated coordination evades statistical detection

**Mitigation for race to the bottom:** Multi-dimensional scoring (quality + efficiency + novelty) prevents pure cost competition.

**Mitigation for monopoly:** Maximum stake concentration limits per subnet (adjustable via governance).

**Mitigation for validator cartel:** Rotating correlation metrics, regular collusion parameter updates via governance.

### 10.3 Regulatory Considerations

An autonomous agent-operated tokenized network may face regulatory scrutiny in multiple jurisdictions:
- **Securities:** KARY may be classified as a security if it appreciates in value and investors expect profits from the efforts of others (Howey test). Mitigation: no pre-mine, no investment pitch, no expectation of profit — KARY is a utility token for network participation.
- **Money transmission:** Psilo A2A escrow could be classified as money transmission in some jurisdictions. Mitigation: Psilo operates the infrastructure; consult Psilo's legal framework.
- **Agent liability:** Unclear legal framework for autonomous agent economic activity. Monitor regulatory developments in AI agent law.

### 10.4 Dependency on Psilo/PAKT Infrastructure

The architecture depends on Psilo for agent wallet management. If Psilo:
- **Shuts down:** Agents cannot sign transactions. Mitigation: fallback key in genesis (Section 8.4).
- **Is compromised:** Agent funds at risk up to 3-node tolerance. Mitigation: stake incrementally, maintain backup cold storage.
- **Changes API:** SDK breaks. Mitigation: maintain a Psilo adapter abstraction layer; treat Psilo as a replaceable module.

**Long-term:** Evaluate building native MPC wallet support into the chain itself, removing the Psilo dependency for critical operations.

### 10.5 Agent Alignment

If agents are self-improving (per Neumann's architecture), their objective functions may drift from network goals over time. An agent optimizing for personal KARY accumulation may find strategies that maximize earnings while degrading network utility.

This is the deepest open question in the entire design. The answer likely involves:
- Keeping individual agent incentives tightly coupled to subnet utility metrics
- Building network-level checks that detect when agent behavior diverges from subnet objectives
- Accepting that some misalignment is inevitable and designing recovery mechanisms

---

## Appendix A: Key File Modifications Summary

### karyon-chain (subtensor fork)

| File | Change |
|------|--------|
| `primitives/src/lib.rs` | Token symbol, decimals |
| `pallets/subtensor/src/lib.rs` | TotalSupply default, storage additions |
| `pallets/subtensor/src/coinbase/block_emission.rs` | Half-supply parameter |
| `pallets/subtensor/src/macros/genesis.rs` | Remove Alice, add agent council address |
| `pallets/subtensor/src/collusion_detection.rs` | New file — collusion detection + slash |
| `runtime/src/lib.rs` | Remove admin-utils, crowdloan; add collusion pallet |
| `chainspecs/*.json` | Genesis configuration with agent council address |

### karyon-sdk (bittensor fork)

| File | Change |
|------|--------|
| `karyon_sdk/core/settings.py` | Token symbol, network name |
| `karyon_sdk/core/psilo_wallet.py` | New file — Psilo wallet adapter |
| `karyon_sdk/agent/staking_strategy.py` | New file — autonomous staking logic |
| `karyon_sdk/core/wallet.py` | Swap substrate wallet for Psilo adapter |

---

## Appendix B: Recommended Reading

- Bittensor Whitepaper: https://bittensor.com/whitepaper
- Substrate Documentation: https://docs.substrate.io
- Psilo Protocol: https://psiloai.com
- Mitosis Agent Spawning Protocol: [internal docs]
- Goodhart's Law and Mechanism Design: Manheim & Garrabrant (2018)

---

*This document is a living specification. Update as the design evolves.*

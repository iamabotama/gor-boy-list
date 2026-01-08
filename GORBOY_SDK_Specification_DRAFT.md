# GORBOY SDK Specification - Section 1: Escrow, Conflict Resolution & Reputation

**Version:** 1.0  
**Status:** Approved for Implementation

---

## 1. Escrow System Architecture

The escrow system is the foundational infrastructure for all staked gameplay in the Gorboy SDK. It ensures fair, deterministic outcomes while maintaining performance and resilience.

### 1.1 Timeout and Expiration Policy

The SDK implements a **medium timeout model** with three tiers based on expected game duration. This balance provides reasonable reconnection windows while avoiding excessive capital lockup.

| Game Type | Reconnection Grace Period | Abandonment Timeout | Final Resolution |
|:----------|:--------------------------|:--------------------|:-----------------|
| **Quick Games** (< 5 min) | 15 minutes | 1 hour | 24 hours |
| **Standard Games** (5-30 min) | 1 hour | 6 hours | 48 hours |
| **Long Games** (30+ min) | 6 hours | 24 hours | 7 days |

**Reconnection Rules:**
- Within grace period: Player can reconnect without penalty
- After grace period: Game continues without player, reputation penalty applied
- After final resolution timeout: Automatic payout based on game state and rules

Game developers declare their game type during SDK initialization, and the SDK automatically applies the appropriate timeout windows. Players see these timeouts clearly displayed before entering escrow.

### 1.2 Pre-Locking and Performance Optimization

To ensure fast, deterministic outcomes and eliminate wallet interaction delays during gameplay, funds are moved to escrow **before** gameplay begins. This pre-locking mechanism provides several critical benefits:

- **Performance**: No need to re-establish wallet connections or approvals during play
- **Offline resilience**: Payouts do not require players to be online
- **Deterministic resolution**: Game outcomes cannot be affected by wallet availability
- **Reduced gas costs**: Single escrow transaction rather than multiple mid-game transactions

The SDK holds coins and winnings in memory until payout occurs, then executes a single batch transaction to distribute funds according to game results.

### 1.3 Disconnection Handling and Game State

When a player disconnects, the SDK uses two key values provided by the game to determine fair resolution:

1. **Progression State**: Percentage of game completion (0-100%)
2. **Ranking State**: Relative standing of each player at current moment

**Resolution Based on Game State:**

| Progression | Ranking Available | Resolution Method |
|:------------|:------------------|:------------------|
| **0-25%** (Early) | Any | Cancel and refund all players minus gas fees |
| **25-75%** (Mid) | Yes | Distribute based on current ranking |
| **25-75%** (Mid) | No | Continue without disconnected player, they forfeit |
| **75-100%** (Late) | Yes | Distribute based on current ranking |
| **75-100%** (Late) | No | Distribute based on time in game / participation |

**Developer Configuration Modes:**

Developers choose from three resolution modes during SDK initialization:

**Mode 1: Continue Without (Default)**
- Disconnected player forfeits stake according to progression/ranking rules
- Game continues with remaining players
- Exception: Early disconnect (< 25% progression) triggers full refund

**Mode 2: Pause and Wait (Turn-Based Games)**
- Game pauses on disconnect
- Timeout countdown begins (configurable, default 5 minutes)
- If player reconnects, game resumes
- If timeout expires, revert to Mode 1 resolution

**Mode 3: Cancel and Refund (All Players Required)**
- Any disconnect invalidates entire game
- All stakes returned minus gas fees
- Useful for games where partial completion makes no sense (e.g., poker, chess)

### 1.4 Tournament and Multi-Player Escrow

**Per-Round Escrow (Recommended):**
- Each round has independent escrow lock and resolution
- Players eliminated in early rounds get immediate settlement
- Reduces capital lockup time
- Provides clearer accounting and faster liquidity

**Full-Tournament Escrow (Alternative):**
- All stakes locked for entire tournament duration
- Higher stakes feel, more "serious" competition
- Risk: Longer lockup if tournament experiences technical issues

---

## 2. Game Invalidation and Refund Policy

### 2.1 Automatic Invalidation Conditions

Games are automatically invalidated under the following conditions:

**Technical Failures:**
- Game server crash with no state recovery possible
- Blockchain network failure preventing escrow resolution
- SDK detects corrupted or impossible game state data

**Security Violations:**
- Verification token mismatch or tampering detected
- Developer manipulation of game state after escrow lock
- Cryptographic proof of cheating or exploit

**Consensus Failures:**
- All players disconnect simultaneously (within 30 seconds)
- Game logic produces mathematically impossible outcome (negative scores, overflows)

### 2.2 Authority Hierarchy

The SDK implements a clear hierarchy for invalidation decisions:

**1. SDK Logic (Primary Authority)**
- Automatically detects and responds to technical failures
- Identifies security violations through verification system
- Recognizes impossible outcomes through validation rules
- Executes immediate 100% refund minus gas fees

**2. Developer Override (With Audit Trail)**
- Developer can manually invalidate game with reason code
- All manual invalidations permanently logged on-chain
- Subject to review if pattern of abuse emerges
- Excessive use triggers developer reputation penalties

**3. Player Consensus (Optional)**
- All players in game can mutually agree to void
- Requires unanimous consent from all participants
- Useful for friendly games or obvious bugs
- No reputation penalty for consensual cancellation

### 2.3 Refund Policy

When a game is invalidated:
- **Stake return**: 100% of original stake returned to each player's wallet
- **Gas fees**: NOT refunded - all players absorb proportionally
- **Developer fees**: NOT collected on invalidated games
- **Exception**: If one player clearly at fault (proven cheating), innocent parties receive full refund while guilty party pays all gas fees

---

## 3. Dispute Resolution

### 3.1 V1 Implementation: Automated Resolution

The initial SDK release focuses on automated resolution without human intervention. This approach aligns with the Gorboy philosophy of treating users as autonomous adults while providing transparent infrastructure.

**Automated Resolution Principles:**
- 95%+ of games resolve automatically based on cryptographically verifiable game state
- No human arbitration of individual game disputes
- SDK provides transparent audit trail of all game events
- Players can flag games/developers for community review
- Gorboy revokes SDK access for developers with proven fraud patterns

**What Gorboy Does:**
- Provides transparent, immutable audit trail
- Maintains public reputation scores for games and developers
- Revokes SDK access when fraud patterns are proven
- Offers tools for community review and flagging

**What Gorboy Does NOT Do:**
- Arbitrate individual game disputes
- Refund players who "feel" cheated without cryptographic proof
- Intervene in developer-player disagreements about game rules
- Make subjective judgments about fairness

### 3.2 Future State: Advanced Dispute Options (V2+)

**Developer-Mediated Resolution (Optional)**
- Developers can implement custom dispute logic in game code
- Must be declared in game metadata before game starts
- Example: "Best 2 out of 3" rule for connection issues
- Developer decision is final but permanently logged on-chain
- Subject to community review and reputation impact

**Community Arbitration (High-Stakes Games)**
- Optional feature for games with large escrow amounts (threshold TBD)
- Random selection of verified arbitrators from community
- Arbitrators must stake tokens to participate
- Arbitrators lose stake if proven biased or negligent
- Majority vote determines outcome
- Gorboy provides infrastructure but does not participate in decisions

---

## 4. Gas Fee Policy

### 4.1 Non-Refundable Gas Fees

Gas fees are non-refundable across all scenarios. This policy ensures that blockchain transaction costs are fairly distributed and prevents abuse of the refund system.

**Gas Fee Allocation:**

| Scenario | Gas Fee Responsibility |
|:---------|:-----------------------|
| **Normal game completion** | Split proportionally among all players |
| **Invalidation (technical failure)** | Split proportionally among all players |
| **Invalidation (one player's fault)** | Faulty player pays all gas fees |
| **Disconnect/forfeit** | Disconnected player pays their own gas + penalty |
| **Timeout/airdrop** | Deducted from escrowed amounts before distribution |

**Gas Fee Transparency:**
- SDK estimates gas fees before escrow lock
- Players see estimated gas cost upfront
- Actual gas fees calculated and deducted at resolution time
- Gas fee breakdown included in transaction receipt

---

## 5. Reputation System

The Gorboy SDK implements a comprehensive reputation system inspired by proven models from Xbox Live and PlayStation Network. The system is designed to encourage positive behavior through education and gradual escalation rather than immediate punishment.

### 5.1 Design Philosophy

**Core Principles:**
1. **Education before punishment**: Multiple warnings before serious consequences
2. **Time-based healing**: Active play without issues gradually improves reputation
3. **Community-driven with safeguards**: Player reports weighted but verified
4. **Transparent but private**: Users see their history, reporter identities protected
5. **Gradual escalation**: Progressive penalties, not instant bans
6. **Reputation-weighted influence**: Good players have stronger voice in community

### 5.2 Reputation Tiers

Each wallet has a reputation score (0-100) that determines their tier and associated privileges or restrictions.

| Tier | Color | Score Range | Status | Visual Badge |
|:-----|:------|:------------|:-------|:-------------|
| **Platinum** | Purple | 95-100 | Exemplary | Purple badge with star |
| **Gold** | Green | 80-94 | Good Player | Green badge |
| **Silver** | Light Green | 60-79 | Neutral | Light green badge |
| **Bronze** | Yellow | 40-59 | Needs Work | Yellow warning badge |
| **Tin** | Orange | 20-39 | Poor Standing | Orange warning badge |
| **Restricted** | Red | 0-19 | Avoid Me | Red warning badge |

**Starting State:**
- All new wallets start at **75 (Silver tier)** - neutral standing
- First 10 games are "probationary" with faster reputation changes to establish baseline

### 5.3 Feedback Categories and Weighting

Reputation is calculated based on three primary feedback categories:

**1. Fair Play (40% weight)**
- SDK-detected disconnects and quits
- Pattern of disconnects when losing (higher penalty)
- Suspected cheating or exploit abuse
- Unsporting behavior (if game reports it)

**2. Community Conduct (40% weight)**
- Player reports from other users (reputation-weighted)
- Excessive muting by other players
- Flagged for harassment or abuse
- Pattern of disputes or refund requests

**3. System Violations (20% weight)**
- Attempting to bypass SDK verification
- Wallet or escrow manipulation attempts
- Terms of service violations
- Verified fraud or cheating

### 5.4 Reputation-Weighted Reporting

**Report strength is directly proportional to reporter's reputation.** This creates a self-regulating system where trusted community members have more influence, while preventing toxic players from weaponizing the report system.

| Reporter Tier | Report Weight Multiplier | Single Report Impact |
|:--------------|:------------------------|:---------------------|
| **Platinum (95-100)** | 2.0x | -1.0 point |
| **Gold (80-94)** | 1.5x | -0.75 points |
| **Silver (60-79)** | 1.0x | -0.5 points (baseline) |
| **Bronze (40-59)** | 0.5x | -0.25 points |
| **Tin (20-39)** | 0.25x | -0.125 points |
| **Restricted (0-19)** | 0.1x | -0.05 points (minimal) |

**Examples:**
- 5 reports from Platinum players = 10 points of weight = triggers "pattern detected"
- 20 reports from Restricted players = 2 points of weight = minimal impact
- This prevents "toxic player mobs" from mass-reporting good players
- Incentivizes maintaining good reputation to have voice in community

### 5.5 Scoring Mechanics

**SDK-Detected Events (Objective, High Weight):**
- Disconnect during game: -2 points
- Disconnect when losing (pattern detected): -5 points
- Quit early (< 25% progression): -3 points
- Game completion: +0.5 points (passive gain)
- 10 consecutive completions: +2 bonus points

**Player Reports (Subjective, Reputation-Weighted):**
- Single report: -0.5 to -1.0 points (depending on reporter's tier)
- 5 weighted reports in 7 days: -3 points (pattern emerging)
- 10 weighted reports in 30 days: -8 points (clear pattern)

**False Report Consequences:**
- Verified false report: Reporter loses -5 points
- Reporter loses 1 tier of report weight for 30 days
- Repeat false reporter: Report weight permanently reduced to 0.1x

### 5.6 Time-Based Recovery

Reputation is based on a **6-month rolling window**, with recent behavior weighted more heavily than older behavior.

**Recovery Through Active Play:**

| Current Tier | Recovery Rate | Time to Heal One Tier |
|:-------------|:--------------|:----------------------|
| **Restricted (Red)** | +0.5 per hour | ~40 hours to Tin |
| **Tin (Orange)** | +0.75 per hour | ~27 hours to Bronze |
| **Bronze (Yellow)** | +1 per hour | ~20 hours to Silver |
| **Silver (Light Green)** | +1.25 per hour | ~16 hours to Gold |
| **Gold (Green)** | +1.5 per hour | ~10 hours to Platinum |

**Constraints:**
- Maximum daily recovery: +10 points per day (prevents idle farming)
- Recent behavior (last 30 days) weighted 2x more than older behavior
- Behavior older than 90 days has 50% reduced weight
- Complete healing from Restricted to Gold: **~80 hours over 2-3 months**

**Accelerated Recovery Bonuses:**
- Playing with friends (no reports): +25% recovery bonus
- Completing games while in "Needs Work" or below: +50% recovery bonus
- Receiving positive feedback from players: +2 points (max 3/day)

### 5.7 Threshold Requirements and Warnings

**Dropping to "Needs Work" (Bronze - 59 points):**

Requires significant negative feedback:
- 12-15 unique player reports over 2-4 weeks, OR
- 20-30 mutes from different players, OR
- 10+ SDK-detected disconnects in 30 days, OR
- Combination of above

**First Warning Message:**
- "Your behavior is negatively impacting other players"
- Breakdown of feedback received by category
- Explanation of consequences if behavior continues
- Link to detailed reputation history dashboard

**Dropping to "Avoid Me" (Restricted - 19 points):**

Requires continued negative behavior after warnings:
- Must have already received "Needs Work" warning
- Additional 8-10 unique reports after warning, OR
- Additional 15+ disconnects after warning, OR
- Severe violation (verified cheating, harassment)

**Final Warning Message:**
- "Last warning before restrictions applied"
- Clear explanation of upcoming penalties
- Specific actions needed to improve reputation
- Timeline and path for recovery

### 5.8 Penalties by Tier

**Platinum (95-100):**
- No restrictions
- Future: Fee discounts, priority matchmaking, exclusive cosmetics
- 2.0x report weight - trusted community voice

**Gold (80-94):**
- No restrictions
- Standard matchmaking priority
- 1.5x report weight

**Silver (60-79):**
- No restrictions (neutral standing)
- 1.0x report weight (baseline)

**Bronze (40-59) - "Needs Work":**
- Yellow warning badge visible to other players
- No gameplay restrictions yet
- Educational messages sent
- 0.5x report weight

**Tin (20-39) - "Poor Standing":**
- Orange warning badge visible
- Must pay **110% escrow deposit** (extra 10% held as good behavior bond)
- Longer timeout windows before reconnection allowed
- Cannot join games with minimum reputation requirements
- 0.25x report weight

**Restricted (0-19) - "Avoid Me":**
- Red warning badge visible
- Can only matchmake with other Restricted players (if PvP)
- Cannot enter House vs Player games with stakes
- Can only play Proof-of-Play mining games (solo)
- Mining rewards reduced by 50%
- Must play 20+ hours to escape this tier
- 0.1x report weight (minimal community influence)

### 5.9 Safeguards Against Abuse

**Automated Verification:**
- Reports only valid if players were in same game
- System checks for report spam (multiple reports same day = 1 report)
- Detects coordinated reporting (group of friends mass-reporting)
- Flags suspicious patterns for human review

**Reputation-Weighted Reporting:**
- Report impact scales with reporter's reputation
- Platinum players (95-100): 2.0x report weight - trusted community members
- Restricted players (0-19): 0.1x report weight - minimal influence
- Creates self-regulating system where good behavior earns influence
- Prevents "toxic player mobs" from mass-reporting good players

**Human Oversight:**
- Policy & Enforcement Team can review flagged cases
- Can undo inaccurate feedback and restore reputation
- Can flag players as "inaccurate reporters"
- Repeat false reporters lose ability to submit reports

**Competitive Gaming Protection:**
- Blocking no longer affects reputation (prevents matchmaking abuse)
- Skill-based report filtering: High-win-rate players protected from "sore loser" reports
- Reports from players who lost badly are weighted lower

**Reputation Insurance:**
- If mass-reported but Policy team finds no violation, reputation fully restored
- False reporters receive -5 point penalty
- Victims of false reporting receive +5 point compensation

### 5.10 Social Dynamics

**Party/Group Reputation:**
- If you party with low-reputation players, entire party inherits lowest member's reputation for matchmaking
- Example: Gold player parties with Restricted player → whole party treated as Restricted
- Warning displayed: "Partying with Restricted player will limit matchmaking"
- Incentivizes community to encourage good behavior among friends

**Friend Vouching (V2+):**
- Platinum players can vouch for one friend per month
- Vouching gives +5 reputation boost to friend
- If vouched player drops to Restricted within 30 days, voucher loses -10 points
- Creates accountability and trust networks

### 5.11 Transparency and Privacy

**Your Own Reputation Dashboard:**
- Current tier and score
- 6-month rolling graph of reputation changes
- Breakdown by feedback category (Fair Play, Conduct, Violations)
- Number of reports received (not who reported)
- Number of reports you've filed
- Detailed event log (disconnects, completions, reports)
- Recovery progress and estimated time to next tier

**What Others See About You:**
- Tier badge (color-coded)
- Score number (e.g., "Bronze - 45")
- Warning message if Bronze or below
- Nothing else (no detailed history)

**Privacy Protection:**
- Cannot see who reported you (prevents retaliation)
- Cannot see specific report content
- Cannot see other players' detailed reputation history
- Reporter identities never disclosed

---

## 6. Developer Reputation (Parallel System)

Game developers also have reputation scores based on game quality and fair practices.

**Developer Reputation Factors:**
- Game uptime and stability (40%)
- Player satisfaction ratings (30%)
- Fair escrow practices (20%)
- Response to disputes (10%)

**Developer Tiers:**
- **Verified** (90-100): Trusted developer badge, featured in marketplace
- **Established** (70-89): Standard developer, no special treatment
- **Unproven** (50-69): New developer, "New Dev" badge shown to players
- **Flagged** (30-49): Warning shown to players, under review
- **Suspended** (0-29): SDK access revoked

---

## 7. Implementation Roadmap

### V1 (MVP) - Initial Release
- Three reputation tiers: Good (Green), Needs Work (Yellow), Avoid Me (Red)
- Basic scoring: SDK-detected events only (disconnects, completions)
- Simple time-based recovery: +1 per hour played
- 6-month rolling window
- Basic transparency dashboard
- Automated dispute resolution only
- Medium timeout model for escrow

### V2 (Enhanced) - 6 Months Post-Launch
- Six reputation tiers with color-coded badges
- Player reporting system with reputation-weighted influence
- Advanced safeguards (false report detection, human review)
- Party reputation inheritance
- Developer reputation system
- Accelerated recovery bonuses
- Developer-mediated dispute resolution (optional)

### V3 (Full Featured) - 12+ Months Post-Launch
- Reputation rewards (fee discounts, cosmetics, priority matchmaking)
- Friend vouching system
- Advanced analytics and pattern detection
- Cross-game reputation tracking
- Reputation-based matchmaking preferences
- Community arbitration for high-stakes games

---

## 8. Key Success Metrics

The following metrics will be tracked to ensure the system is working as intended:

1. **Player distribution across tiers** (goal: 80%+ in Gold/Silver)
2. **Average recovery time from Restricted** (goal: 2-3 months)
3. **False report rate** (goal: < 5%)
4. **Player satisfaction with reputation system** (quarterly surveys)
5. **Correlation between reputation and game quality** (do high-rep players have better experiences?)
6. **Developer reputation distribution** (goal: 70%+ in Verified/Established)

---

## Summary

Section 1 establishes the foundational systems for fair, transparent, and resilient gameplay in the Gorboy SDK. The escrow system ensures deterministic outcomes while maintaining performance. The reputation system, inspired by proven models from Xbox Live and PlayStation, creates a self-regulating community where good behavior is rewarded with influence and poor behavior is addressed through education and gradual escalation.

Key innovations include:
- **Progression/ranking-based disconnect resolution** for fair outcomes
- **Reputation-weighted reporting** where good players have stronger community voice
- **Time-based healing** that gives everyone a path to redemption
- **Transparent but private** system that protects against retaliation
- **Developer reputation** to ensure game quality and fairness

These systems work together to create an environment where players are treated as autonomous adults, given the tools to make informed decisions, and held accountable for their behavior through community-driven mechanisms rather than top-down enforcement.
# GORBOY SDK Specification - Section 2: Security & Anti-Fraud

**Version:** 1.0  
**Status:** Approved for Implementation

---

## 1. Blockchain Architecture Foundation

### 1.1 Gorbagana as L1 Network

The Gorboy SDK is built on **Gorbagana**, a Layer-1 (L1) blockchain forked from Solana. Understanding what this means is critical to the SDK's architecture.

**What is an L1 Blockchain?**

A Layer-1 blockchain is a **base-layer network** that processes and finalizes transactions on its own blockchain without relying on another network. Think of it as the foundation:

- **Ethereum** is an L1 - it has its own blockchain
- **Solana** is an L1 - it has its own blockchain  
- **Gorbagana** is an L1 - it's a fork of Solana with its own independent blockchain

**Not a Layer-2 (L2):** L2s sit "on top" of L1s (like Lightning Network on Bitcoin, or Arbitrum on Ethereum). Gorbagana is NOT an L2 - it's a completely independent blockchain.

**Not a Sidechain:** Sidechains are parallel chains connected to a main chain. Gorbagana is a **fork** - it started from Solana's codebase but runs independently.

**Key Technical Characteristics:**
- **Smart Contract Language**: Rust (inherited from Solana)
- **Token Standard**: SPL (Solana Program Library) compatible
- **Consensus Mechanism**: Proof of History + Proof of Stake (Solana model)
- **Performance**: High-speed, low-fee transactions
- **Native Token**: $GOR (used for gas fees and transactions)
- **Testnet Token**: $gGOR (for development and testing)

### 1.2 Smart Contract Architecture (Hybrid Approach)

The Gorboy SDK uses a **hybrid architecture** that combines the trustlessness of on-chain smart contracts with the speed and flexibility of off-chain game logic.

**What Are Smart Contracts?**

Smart contracts are **self-executing programs** that run on a blockchain. Think of them as vending machines:
- You put money in (transaction)
- The machine follows programmed rules (smart contract code)
- You get your item (outcome)
- No human needed to enforce the rules

**Traditional Contract:**
- Alice and Bob agree to terms
- They trust each other (or hire a lawyer)
- If someone breaks the agreement, they go to court

**Smart Contract:**
- Alice and Bob's agreement is written in code
- Code runs on blockchain (no one can change it)
- Blockchain automatically enforces the agreement
- No trust needed, no lawyers, no courts

**Gorboy's Hybrid Model:**

We use smart contracts for the **critical trust parts** (money) and off-chain logic for the **fast gameplay parts** (game mechanics).

| Component | Where It Runs | Why |
|:----------|:-------------|:----|
| **Escrow Locking** | On-chain (smart contract) | Trustless - no one can steal funds |
| **Escrow Holding** | On-chain (smart contract) | Transparent - anyone can verify amounts |
| **Game Logic** | Off-chain (game server) | Fast - no blockchain delays |
| **Game State** | Off-chain (game server) | Cheap - no gas fees for every move |
| **Final Results** | On-chain (smart contract) | Immutable - permanent proof of outcome |
| **Payouts** | On-chain (smart contract) | Trustless - automatic distribution |

**Example: Poker Game**

1. **Escrow Lock (On-Chain)**
   - Players' tokens moved to smart contract
   - Smart contract holds funds (no one can touch them)
   - Transaction recorded on blockchain

2. **Gameplay (Off-Chain)**
   - Cards dealt by game server
   - Players make bets and decisions
   - Game server tracks all moves
   - Fast, no gas fees

3. **Game End (Hybrid)**
   - Game server calculates winner
   - Server submits result to smart contract
   - Smart contract verifies result is valid
   - If valid, smart contract releases funds to winner

4. **Payout (On-Chain)**
   - Smart contract automatically sends tokens to winner
   - Transaction recorded on blockchain
   - Permanent, verifiable proof

**Benefits of Hybrid Approach:**

✅ **Trustless**: Money held in smart contract, not by Gorboy or developer  
✅ **Fast**: Game plays at normal speed, not blockchain speed  
✅ **Cheap**: Only pay gas fees for escrow and payout, not every game action  
✅ **Verifiable**: Final results recorded on blockchain forever  
✅ **Flexible**: Can update game logic without changing smart contract

**How Smart Contracts Are Written:**

On Gorbagana (like Solana), smart contracts are called **"programs"** and are written in **Rust** programming language.

**Simple Example - Escrow Lock:**

```rust
// Simplified pseudocode for escrow lock
pub fn lock_escrow(
    player_wallet: &AccountInfo,
    escrow_account: &AccountInfo,
    amount: u64,
) -> ProgramResult {
    // Transfer tokens from player to escrow
    transfer_tokens(
        from: player_wallet,
        to: escrow_account,
        amount: amount,
    )?;
    
    // Record game details
    escrow_account.game_id = generate_game_id();
    escrow_account.locked_at = current_timestamp();
    escrow_account.status = "LOCKED";
    
    // Emit event for transparency
    emit_event("EscrowLocked", {
        player: player_wallet.address,
        amount: amount,
        game_id: escrow_account.game_id,
    });
    
    Ok(())
}
```

**What This Code Does:**
1. Moves tokens from player's wallet to escrow account
2. Records when and why tokens were locked
3. Marks escrow as "LOCKED" so no one can touch it
4. Broadcasts event so everyone can see what happened

**Simple Example - Payout:**

```rust
// Simplified pseudocode for payout
pub fn release_escrow(
    escrow_account: &AccountInfo,
    winner_wallet: &AccountInfo,
    game_result: GameResult,
) -> ProgramResult {
    // Verify game result is valid
    require!(verify_game_signature(game_result), "Invalid signature");
    
    // Verify escrow is still locked
    require!(escrow_account.status == "LOCKED", "Already paid out");
    
    // Calculate amounts (winner gets stake minus gas)
    let payout_amount = escrow_account.amount - gas_fees;
    
    // Transfer tokens to winner
    transfer_tokens(
        from: escrow_account,
        to: winner_wallet,
        amount: payout_amount,
    )?;
    
    // Mark escrow as complete
    escrow_account.status = "COMPLETED";
    
    // Emit event
    emit_event("EscrowReleased", {
        winner: winner_wallet.address,
        amount: payout_amount,
        game_id: escrow_account.game_id,
    });
    
    Ok(())
}
```

**What This Code Does:**
1. Checks that game result is legitimate (cryptographic signature)
2. Checks that escrow hasn't already been paid out
3. Calculates how much winner gets (after gas fees)
4. Sends tokens to winner's wallet
5. Marks escrow as complete so it can't be paid twice
6. Broadcasts event for transparency

**Key Security Features:**

🔒 **Immutable**: Once deployed, smart contract code can't be changed  
🔒 **Deterministic**: Same inputs always produce same outputs  
🔒 **Transparent**: Anyone can read the code and verify it's fair  
🔒 **Permissionless**: No one can block you from using it  
🔒 **Auditable**: All transactions recorded forever on blockchain

**Limitations and Risks:**

⚠️ **Bugs Are Permanent**: If smart contract has a bug, it can't be "patched"  
⚠️ **Gas Costs**: Every on-chain operation costs gas fees  
⚠️ **Slow**: Blockchain confirmation takes time (seconds, not milliseconds)  
⚠️ **Complexity**: Writing secure smart contracts requires expertise

**Mitigation Strategies:**

✅ **Extensive Testing**: Test smart contracts on testnet ($gGOR) before mainnet  
✅ **Security Audits**: Have independent experts review code for vulnerabilities  
✅ **Upgradeable Proxies**: Use proxy pattern to allow fixes (with safeguards)  
✅ **Emergency Pause**: Include ability to pause contract if critical bug found  
✅ **Insurance Fund**: Set aside funds to compensate users if exploit occurs

---

## 2. Verification Token System

The verification token system ensures that transparency displays (escrow amounts, house stakes, etc.) shown in games are legitimate and not faked by malicious developers.

### 2.1 How Verification Tokens Work

**The Problem:**
- Game shows "House has 1,000,000 GOR in escrow"
- How does player know this is real?
- Malicious developer could just display fake numbers

**The Solution: Cryptographic Visual Tokens**

When escrow is locked, the SDK generates a unique verification token that is:
1. **Cryptographically bound** to the specific game instance
2. **Visually distinctive** (hard to fake)
3. **Time-limited** (expires quickly to prevent reuse)
4. **Verifiable** (players can check on Gorboy website)

### 2.2 Token Generation Process

**Step 1: Escrow Lock**
```
Game starts → Escrow locks on-chain → Smart contract generates unique hash
```

**Step 2: Token Creation**
```
Hash = SHA256(game_id + player_wallets + escrow_amount + timestamp + random_nonce)
Example: 0x7a3f9b2c...
```

**Step 3: Visual Representation**

The hash is converted into a visual token with three components:

| Component | Purpose | Example |
|:----------|:--------|:--------|
| **QR Code** | Machine-readable verification | Scannable with phone |
| **Color Pattern** | Human-recognizable visual | 4x4 grid of colors based on hash |
| **Rotating Symbol** | Anti-screenshot protection | Changes every 30 seconds |

**Example Visual Token:**

```
┌─────────────────────────┐
│  ████████████████████   │  ← QR Code (encodes verification URL)
│  ████████████████████   │
│  ████████████████████   │
│                         │
│  🟦🟩🟥🟨              │  ← Color Pattern (derived from hash)
│  🟨🟥🟩🟦              │
│  🟩🟦🟨🟥              │
│  🟥🟨🟦🟩              │
│                         │
│      ⟳ [rotating]       │  ← Rotating Symbol (changes every 30s)
│                         │
│  Verify at:             │
│  verify.gorboy.com      │
└─────────────────────────┘
```

### 2.3 Verification Process

**Player Verification Steps:**

1. **Scan QR Code** (or manually visit verify.gorboy.com)
2. **Enter Game ID** (if not scanned)
3. **Website Shows:**
   - ✅ "This is a legitimate Gorboy game"
   - Game Name: "Blackjack Pro"
   - Players: [Wallet A] vs [House]
   - Escrow Amount: 1,000 GOR
   - Locked At: 2026-01-07 14:32:15 UTC
   - Status: Active
   - Visual Token Matches: ✅

**What Prevents Faking?**

**Time-Limited Binding:**
- Token expires after 60 seconds
- New token generated every 60 seconds
- Fake game would need to constantly scrape new tokens (detectable)

**Wallet Binding:**
- Token is cryptographically bound to specific wallet addresses
- If fake game displays token, verification website shows:
  - ❌ "Token is for different players"
  - Expected: [Wallet A] vs [Wallet B]
  - Your Wallet: [Wallet C]

**Rotating Symbol:**
- Symbol changes every 30 seconds based on current time + hash
- Screenshot of token becomes invalid after 30 seconds
- Fake game can't predict future symbols

**Example Attack Scenario:**

❌ **Malicious Developer Tries to Fake:**
1. Developer creates fake game showing "10,000 GOR escrow"
2. Developer screen-scrapes real verification token from legitimate game
3. Displays stolen token in fake game

✅ **Player Detects Fraud:**
1. Player scans QR code
2. Website shows: "Token is for Game X with Player A vs Player B"
3. Player realizes: "Wait, I'm Player C in Game Y"
4. Player knows it's fake, doesn't play

### 2.4 SDK Implementation

**For Game Developers:**

```javascript
// SDK automatically generates and displays verification token
const game = await GorboySDK.createGame({
  type: "HouseVsPlayer",
  houseStake: 1000,
  playerStake: 100,
});

// SDK provides verification widget
<GorboyVerificationWidget 
  gameId={game.id}
  position="top-right"
  size="medium"
/>
```

**SDK handles:**
- Token generation
- Visual rendering
- Automatic rotation every 30 seconds
- QR code encoding
- Verification URL generation

**Developers cannot:**
- Disable verification widget (required for SDK access)
- Modify token appearance (standardized)
- Generate fake tokens (requires smart contract signature)

---

## 3. Wallet Security

### 3.1 Supported Wallets

**V1 (Initial Release):**
- **Backpack** (primary, Gorbagana native)
- **Phantom** (Solana-compatible, future-ready)

**V2 (Future):**
- Additional Solana-compatible wallets
- Custom Gorboy wallet
- WalletConnect protocol support

### 3.2 Non-Custodial Architecture

**Critical Security Principle: SDK Never Has Custody of Private Keys**

**What This Means:**
- SDK is a **connector**, not a **custodian**
- Players' private keys stay in their wallet
- SDK requests wallet to sign transactions
- Wallet shows player what they're signing
- Player approves or rejects

**Example Flow:**

```
1. Game: "Lock 100 GOR in escrow"
2. SDK: "Hey Backpack wallet, please sign this transaction"
3. Backpack: [Shows player] "Game wants to lock 100 GOR. Approve?"
4. Player: [Clicks] "Approve"
5. Backpack: [Signs transaction with private key]
6. SDK: [Submits signed transaction to blockchain]
7. Blockchain: [Executes escrow lock]
```

**What SDK Never Sees:**
- ❌ Private keys
- ❌ Seed phrases
- ❌ Unencrypted wallet data

**What SDK Does See:**
- ✅ Public wallet address
- ✅ Signed transactions (after player approves)
- ✅ Transaction results

### 3.3 Rate Limits and Spending Limits

To prevent wallet drainage attacks and accidental large losses:

**Per-Transaction Limits:**

| Limit Type | Value | Rationale |
|:-----------|:------|:----------|
| **Maximum Single Escrow** | 10,000 GOR | Prevents catastrophic single loss |
| **Maximum Games Per Hour** | 20 games | Prevents bot abuse |
| **Maximum Daily Volume** | 50,000 GOR | Prevents account compromise drainage |

**Re-Approval Requirements:**

- Transactions > 1,000 GOR require explicit wallet approval (no pre-authorization)
- Transactions > 5,000 GOR show warning: "Large amount - verify carefully"
- First transaction with new game requires approval (can't pre-approve all)

**Developer Limits:**

- New developers: Max 1,000 GOR per game
- Established developers: Max 10,000 GOR per game
- Verified developers: Max 50,000 GOR per game (with insurance)

---

## 4. Provably Fair Gaming (House vs Player)

For House vs Player games, ensuring the house doesn't cheat is critical. The SDK enforces **provably fair** random number generation.

### 4.1 What "Provably Fair" Means

**The Problem:**
- House generates random number (dice roll, card shuffle, etc.)
- How do you know house didn't pick a number that makes you lose?
- You have to trust the house

**Provably Fair Solution:**
- House commits to random number BEFORE player acts
- Player makes decision
- House reveals random number
- Player (or anyone) can verify house didn't cheat

**It's like:**
- House writes number on paper, seals envelope
- You make your bet
- House opens envelope
- You verify the seal wasn't broken

### 4.2 Commit-Reveal Scheme

**Step-by-Step Example: Dice Game**

**Before Game:**
1. House generates random seed: `"secret_12345"`
2. House computes hash: `SHA256("secret_12345")` = `0x7a3f9b2c...`
3. House publishes hash on-chain (commitment)
4. Player sees hash but not the actual seed

**Player's Turn:**
5. Player sees commitment: `0x7a3f9b2c...`
6. Player bets 100 GOR on "over 50"
7. Player's bet recorded on-chain
8. Player CANNOT change bet now

**Reveal:**
9. House reveals seed: `"secret_12345"`
10. Anyone can verify: `SHA256("secret_12345")` = `0x7a3f9b2c...` ✅
11. Dice roll calculated: `hash(seed + player_wallet + timestamp) % 100` = 67
12. Player wins! (67 > 50)

**Why This Is Fair:**

✅ House committed to seed before knowing player's bet  
✅ House cannot change seed (hash would mismatch)  
✅ Player cannot change bet after seeing commitment  
✅ Anyone can verify house didn't cheat  
✅ Random number is deterministic but unpredictable

**If House Tries to Cheat:**

❌ **Scenario: House reveals different seed**
- House committed: `0x7a3f9b2c...`
- House reveals: `"different_seed"`
- Verification: `SHA256("different_seed")` = `0x9f2e8a1b...` ❌
- **Mismatch detected! → Automatic refund to player**

### 4.3 SDK Enforcement

**For HvP Games: Provably Fair RNG is REQUIRED**

```javascript
// SDK enforces provably fair RNG for HvP games
const game = await GorboySDK.createGame({
  type: "HouseVsPlayer",
  houseStake: 1000,
  playerStake: 100,
  rngMode: "provably-fair", // REQUIRED for HvP, cannot be disabled
});

// SDK handles commit-reveal automatically
const commitment = await game.commitRNG(); // House commits
const playerAction = await game.waitForPlayerAction(); // Player acts
const result = await game.revealRNG(); // House reveals, SDK verifies
```

**SDK Automatically:**
- Generates cryptographic commitments
- Records commitments on-chain
- Verifies reveals match commitments
- Refunds player if mismatch detected
- Logs all RNG events for transparency

**For PvP Games: Optional**
- Players trust each other, not house
- Can use simpler RNG if desired
- Provably fair still available if wanted

**For Mining Games: Not Applicable**
- No RNG needed (score-based rewards)
- Server-side validation sufficient

---

## 5. Game State and Data Storage

### 5.1 Hybrid Storage Model

**On-Chain (Blockchain):**
- Escrow lock transactions
- Escrow amounts and wallet addresses
- Final game results
- Payout transactions
- Verification token commitments
- Dispute resolutions

**Off-Chain (Game Servers):**
- Detailed game state (positions, health, inventory, etc.)
- Player actions and moves
- Chat messages
- Temporary session data

**Merkle Root On-Chain:**
- Cryptographic summary of off-chain data
- Allows verification without storing everything on-chain
- Like a "fingerprint" of the game state

### 5.2 What Is a Merkle Root?

**Simple Explanation:**

Imagine you have 1,000 pages of game data. Storing all 1,000 pages on blockchain would be expensive. Instead:

1. Create a "fingerprint" (hash) of all 1,000 pages
2. Store only the fingerprint on blockchain
3. If anyone questions the data, you can prove it matches the fingerprint

**Technical Explanation:**

```
Game Events:
├─ Player A moved to (10, 20)
├─ Player B attacked Player A
├─ Player A used health potion
└─ Player B won

Hash each event:
├─ Hash1 = SHA256("Player A moved to (10, 20)")
├─ Hash2 = SHA256("Player B attacked Player A")
├─ Hash3 = SHA256("Player A used health potion")
└─ Hash4 = SHA256("Player B won")

Combine hashes:
├─ Hash12 = SHA256(Hash1 + Hash2)
└─ Hash34 = SHA256(Hash3 + Hash4)

Merkle Root:
└─ Root = SHA256(Hash12 + Hash34)
```

**Store on blockchain:** Only the Merkle Root (32 bytes)  
**Store off-chain:** All 1,000 pages of game data

**If dispute:** Provide game data + hashes, anyone can verify it matches the root

### 5.3 Benefits of Hybrid Approach

**Cost Efficiency:**
- Storing 1 KB on-chain: ~$0.01 in gas fees
- Storing 1 MB on-chain: ~$10 in gas fees
- Storing Merkle root (32 bytes): ~$0.0001 in gas fees

**Speed:**
- On-chain transaction: 1-5 seconds
- Off-chain update: < 100 milliseconds
- Games feel instant, not blockchain-slow

**Verifiability:**
- Anyone can verify final results on-chain
- Detailed game data available off-chain
- Cryptographic proof links the two

---

## 6. SDK Access Control

### 6.1 Three-Tier Access System

**Tier 1: Sandbox/Testnet (Open Access)**

**Purpose:** Learning and experimentation

**Requirements:**
- Email verification
- Accept terms of service

**Limits:**
- Testnet only ($gGOR tokens, no real value)
- Unlimited games
- Full SDK features
- No real money at risk

**Benefits:**
- Learn SDK without risk
- Test game mechanics
- Debug integration issues
- Get community feedback

---

**Tier 2: Production (Approved Access)**

**Purpose:** Launch real games with real money

**Requirements:**
- Complete Tier 1 testing
- Submit application with:
  - Game description and mechanics
  - Developer identity verification
  - Game demo/video
  - Terms of service for players
- Basic KYC (Know Your Customer) for compliance

**Initial Limits:**
- Max 1,000 GOR per game
- Max 100 concurrent games
- Max 10,000 GOR total volume per day

**Reputation Building:**
- Limits increase as reputation improves
- At 70+ reputation: Max 5,000 GOR per game
- At 90+ reputation: Max 10,000 GOR per game

**Benefits:**
- Launch on mainnet
- Earn developer fees
- Build player base
- Establish reputation

---

**Tier 3: Verified Developer (Earned Status)**

**Purpose:** Trusted, high-volume games

**Requirements:**
- 90+ developer reputation
- 1,000+ games completed
- < 5% invalidation rate
- < 10% player complaint rate
- Security audit of game (for high-stakes games)

**Limits:**
- Max 50,000 GOR per game (with insurance)
- Unlimited concurrent games
- Unlimited daily volume

**Benefits:**
- "Verified Developer" badge
- Featured in game marketplace
- Reduced SDK fees (if any)
- Priority support
- Early access to new features

### 6.2 Revocation and Suspension

**Automatic Revocation (Immediate):**
- Verified fraud or cheating
- Attempting to bypass verification system
- Manipulating escrow or game results
- 10+ proven cases of unfair outcomes
- Developer reputation drops to "Suspended" (< 30)

**Temporary Suspension (Under Review):**
- Pattern of technical issues (> 20% invalidation rate)
- Multiple player complaints under investigation
- Suspicious escrow patterns detected
- Pending security audit

**Appeal Process:**
1. Developer receives notice with specific violations
2. Developer has 14 days to submit appeal with evidence
3. Policy & Enforcement Team reviews appeal
4. Decision made within 7 days of appeal submission
5. If approved, access restored with probation period

**Probation:**
- Reduced limits for 30 days
- Enhanced monitoring
- One strike policy (immediate revocation if violation)

---

## 7. Gas Fees and Transaction Costs

### 7.1 Who Pays Gas Fees?

| Transaction Type | Who Pays | Deduction Method |
|:----------------|:---------|:-----------------|
| **Escrow Lock** | Players (split) | Paid at lock time |
| **Normal Payout** | Players (from winnings) | Deducted before distribution |
| **Timeout Airdrop** | Players (from escrow) | Deducted before distribution |
| **Invalidation Refund** | Players (split) | Deducted from refund |
| **Developer Fee Collection** | Developer | Paid when claiming fees |
| **Reputation Updates** | Gorboy (subsidized) | No cost to players/developers |

### 7.2 Gas Fee Transparency

**Before Escrow Lock:**
```
┌─────────────────────────────┐
│ Escrow Summary              │
├─────────────────────────────┤
│ Your Stake:      100.0 GOR  │
│ Estimated Gas:     0.1 GOR  │
│ Total Required: 100.1 GOR  │
│                             │
│ If you win:     ~199.8 GOR  │
│ (opponent stake + yours     │
│  minus gas fees)            │
└─────────────────────────────┘
```

**After Game Completion:**
```
┌─────────────────────────────┐
│ Payout Receipt              │
├─────────────────────────────┤
│ Escrow Total:   200.0 GOR   │
│ Gas Fees:        -0.15 GOR  │
│ Developer Fee:   -2.0 GOR   │
│ Your Winnings:  197.85 GOR  │
│                             │
│ Transaction:                │
│ 0x7a3f9b2c...               │
└─────────────────────────────┘
```

### 7.3 Gas Fee Optimization

**Batching:**
- Multiple payouts combined into single transaction
- Reduces per-game gas cost by 50-70%
- Optional "instant payout" (higher fee) vs "batched payout" (lower fee, 1-hour delay)

**Example:**
- Instant payout: 0.15 GOR gas fee, immediate
- Batched payout: 0.05 GOR gas fee, next hour

**Layer-1 Efficiency:**
- Gorbagana (Solana fork) has very low gas fees
- Typical transaction: $0.001 - $0.01 USD
- Much cheaper than Ethereum L1 ($5-$50)

---

## 8. Anti-Cheat and Integrity Monitoring

### 8.1 SDK Responsibilities

**What SDK Detects and Prevents:**

**Wallet/Escrow Manipulation:**
- Attempts to double-spend tokens
- Attempts to modify escrow amounts
- Attempts to withdraw before game completion
- Replay attacks (reusing old transactions)

**Game State Tampering:**
- Checksum mismatches (game state modified)
- Impossible outcomes (negative scores, overflows)
- Timestamp manipulation
- Signature verification failures

**Bot Detection:**
- Superhuman reaction times (< 50ms consistently)
- Perfect play patterns (statistically impossible)
- Rapid-fire game creation (> 20 games/hour)
- Coordinated multi-account behavior

**Rate Limiting:**
- Max 20 games per hour per wallet
- Max 50,000 GOR daily volume per wallet
- Max 10 simultaneous active games per wallet

### 8.2 Developer Responsibilities

**What Developers Must Implement:**

**Game-Specific Anti-Cheat:**
- Aimbots, wallhacks (for shooters)
- Card counting detection (for card games)
- Collision detection (for racing games)
- Input validation (for all games)

**Server-Side Validation:**
- Verify all player actions are legal
- Validate game physics and rules
- Detect impossible player positions/states
- Log all actions for replay verification

**SDK Integration:**
- Call SDK checksum functions regularly
- Report suspicious behavior to SDK
- Use SDK's encrypted communication
- Implement SDK's replay verification

### 8.3 SDK Provides Tools

**Checksum Verification:**
```javascript
// SDK provides checksum for game state
const checksum = GorboySDK.generateChecksum(gameState);

// Later, verify game state hasn't been tampered with
const isValid = GorboySDK.verifyChecksum(gameState, checksum);
if (!isValid) {
  // Game state was modified - invalidate game
  await game.invalidate("State tampering detected");
}
```

**Encrypted Communication:**
```javascript
// SDK encrypts sensitive game data
const encrypted = await GorboySDK.encrypt(gameState);
// Send to server
// Server decrypts and verifies
const decrypted = await GorboySDK.decrypt(encrypted);
```

**Replay Verification:**
```javascript
// SDK records all game events
GorboySDK.recordEvent("player_moved", { x: 10, y: 20 });
GorboySDK.recordEvent("player_attacked", { target: "player2" });

// After game, generate replay proof
const replayProof = await GorboySDK.generateReplayProof();

// Anyone can verify game was played fairly
const isValid = await GorboySDK.verifyReplay(replayProof);
```

---

## 9. Security Audit and Insurance

### 9.1 Smart Contract Audits

**Before Mainnet Launch:**
- Independent security audit by reputable firm
- Penetration testing
- Formal verification (mathematical proof of correctness)
- Bug bounty program

**Ongoing:**
- Annual re-audits
- Community bug bounty (rewards for finding vulnerabilities)
- Continuous monitoring for exploits

### 9.2 Insurance Fund

**Purpose:** Compensate users if exploit occurs despite audits

**Funding:**
- 1% of all developer fees go to insurance fund
- Gorboy contributes matching funds
- Target: 1M GOR reserve

**Coverage:**
- Verified exploits of smart contracts
- Proven bugs causing loss of funds
- Does not cover: user error, phishing, compromised wallets

**Claims Process:**
1. User reports loss with evidence
2. Policy team investigates
3. If verified exploit, user compensated from fund
4. Smart contract paused until fix deployed

---

## 10. Summary

Section 2 establishes the security foundation for the Gorboy SDK:

**Blockchain Architecture:**
- Built on Gorbagana (Solana fork L1)
- Hybrid smart contract model (on-chain escrow, off-chain gameplay)
- Rust-based smart contracts for trustless escrow

**Verification System:**
- QR code + visual pattern + rotating symbol
- Time-limited, wallet-bound tokens
- Prevents fake transparency displays

**Wallet Security:**
- Non-custodial (SDK never holds private keys)
- Backpack and Phantom support
- Rate limits and spending caps

**Provably Fair Gaming:**
- Commit-reveal scheme for HvP games
- Cryptographic proof of fairness
- Automatic refunds if cheating detected

**Access Control:**
- Three-tier system (Sandbox, Production, Verified)
- Reputation-based limit increases
- Clear revocation and appeal process

**Gas Fees:**
- Players pay, deducted from escrow
- Transparent estimates before lock
- Batching optimization available

**Anti-Cheat:**
- SDK handles wallet/escrow security
- Developers handle game-specific anti-cheat
- SDK provides tools (checksums, encryption, replay verification)

These systems work together to create a secure, transparent, and trustworthy gaming platform where players can confidently stake real value.
# GORBOY SDK Specification - Section 3: Proof-of-Play Mining Mechanics

**Version:** 2.0  
**Status:** Approved for Implementation

---

## 1. Self-Regulating Mining System Overview

The Gorboy mining system uses a dynamic adjustment mechanism that automatically regulates rewards based on the remaining token pool size. This creates a self-balancing economy where early adopters earn more tokens at lower value, while late adopters earn fewer tokens at higher value.

### 1.1 Core Formula

**Base Rate:** 1 GORBOY per mining unit (game action)

**Adjustment Formula:**
```
tokens_per_action = (current_pool_size / 200,000,000) × base_rate
```

**Examples:**
- Pool = 220M → 1.1 GORBOY per action
- Pool = 180M → 0.9 GORBOY per action
- Pool = 100M → 0.5 GORBOY per action

**Total Mining Pool:** 2,000,000,000 GORBOY tokens allocated for mining rewards

### 1.2 Economic Guarantees

**For Investors:**
- Holdings shielded from gameplay volume fluctuations
- Controlled supply becomes more valuable with adoption
- Inflation is predictable and diminishes over time
- No ecosystem fee extraction (already accounted for in 2B allocation)

**For Gamers:**
- Early entry means higher rewards per action
- As player base grows, rewards per action decrease but token value increases
- Solo and multiplayer modes both earn from same pool
- 100% of mined tokens go to players (no ecosystem deduction)

**For Developers:**
- Transparent reward math
- No risk of draining treasury
- Build once, scale indefinitely
- Separate mining rewards independent of player rewards

---

## 2. Dual Mining System: Players and Developers

The Gorboy SDK implements a **dual mining system** where players and developers earn tokens independently through different mechanisms.

### 2.1 Player Mining (Time-Gated, Proof-of-Play)

**Mining Interval:** 30 seconds

Every 30 seconds of active gameplay, one **potential mining unit** becomes available to the player. Whether the player actually receives this unit depends on the game's design and the player's actions.

**Key Concepts:**

**Potential Mining Units:**
- Time-based availability (1 unit per 30 seconds)
- Represents maximum possible rewards
- Does not automatically grant tokens

**Actual Mining Units:**
- Units actually awarded to player
- Controlled by game developer's design
- Requires proof-of-play validation
- Can be withheld, accumulated, or released in batches

**Example Flow:**

```
T=0-30s: Player plays
├─ 1 potential unit available
└─ Game decides: Award now, withhold, or accumulate

T=30-60s: Player continues
├─ 1 potential unit available (2 total if previous withheld)
└─ Game decides: Award accumulated units or continue withholding

T=60-90s: Player achieves milestone
├─ 1 potential unit available (3 total if all withheld)
└─ Game awards all 3 units at once!
```

**Developer Control:**

Game developers have full flexibility in how they distribute mining units:
- **Immediate**: Award every 30 seconds automatically
- **Milestone-based**: Award when player reaches level, score, or achievement
- **Accumulated**: Build up units and release in batches for dramatic effect
- **Conditional**: Require specific actions or performance thresholds

### 2.2 Developer Mining (Time-Gated, Inverse Incentive)

**Mining Interval:** 10 minutes

Every 10 minutes of active player gameplay, one **potential mining unit** becomes available to the developer. However, the developer's actual reward is **inversely proportional** to how generous they are with player rewards.

**Inverse Incentive Formula:**

```
developer_multiplier = max(0, 1.0 - (user_success_rate - 0.4) / 0.6)
developer_actual_units = developer_potential_units × developer_multiplier
```

**Where:**
- `user_success_rate` = actual_mined / potential_available (e.g., 0.5 = 50%)
- Formula creates inverse relationship with sweet spot at 40-60%

**Multiplier Table:**

| User Success Rate | Developer Multiplier | Developer Gets | Interpretation |
|:------------------|:--------------------|:---------------|:---------------|
| **0-40%** | 1.0x | Full unit | Too stingy, users unhappy |
| **40%** | 1.0x | Full unit | Lower bound of sweet spot |
| **50%** | 0.83x | 0.83 units | Sweet spot center |
| **60%** | 0.67x | 0.67 units | Upper bound of sweet spot |
| **70%** | 0.50x | 0.50 units | Generous to players |
| **80%** | 0.33x | 0.33 units | Very generous |
| **90%** | 0.17x | 0.17 units | Extremely generous |
| **100%** | 0.0x | Zero units | Suspiciously generous |

**The Sweet Spot (40-60% User Success):**

This range represents the optimal balance:
- **Players**: Receive 40-60% of potential units (decent rewards)
- **Developers**: Receive 0.67-1.0x multiplier (good income)
- **Game Health**: Neutral score (no penalties)
- **Sustainability**: Both parties incentivized to continue

**Why This Works:**

**Too Stingy (< 40%):**
- Developer gets full rewards → Short-term gain
- Players get minimal rewards → Bad experience, churn
- Game gets negative score → Penalties, reputation damage
- **Result**: Developer loses long-term

**Too Generous (> 80%):**
- Players get most/all rewards → Suspicious
- Developer gets minimal/zero rewards → No sustainable income
- Game gets flagged → Possible farming scheme investigation
- **Result**: Unsustainable or fraudulent

**Balanced (40-60%):**
- Players happy with rewards
- Developer earns sustainable income
- Game maintains good reputation
- **Result**: Long-term success for everyone

### 2.3 Developer Mining Trigger Mechanism

**Per Active Player Session:**

Developer mining units accumulate based on **active player sessions**, not total game runtime.

**Example:**

```
Scenario 1: Single Player
- Player A plays for 10 minutes
- Developer earns 1 potential unit (subject to multiplier)

Scenario 2: Multiple Players (Concurrent)
- Player A plays for 10 minutes
- Player B plays for 10 minutes (same time period)
- Player C plays for 10 minutes (same time period)
- Developer earns 3 potential units (1 per player session)

Scenario 3: Multiple Players (Sequential)
- Player A plays for 10 minutes, then leaves
- Player B starts and plays for 10 minutes
- Player C starts and plays for 10 minutes
- Developer earns 3 potential units (1 per session)
```

**Why Per-Session?**

This rewards developers for:
- Creating engaging games that attract many players
- Retaining players for meaningful session lengths
- Building sustainable player bases

**Not rewarded for:**
- Keeping one player in game for hours (caps at session length)
- Artificially inflating session times

---

## 3. Weighted Scoring System (Game Reputation)

Games are continuously monitored and scored based on their player mining success rate. This score affects game visibility, reputation, and potential penalties.

### 3.1 Score Weights by User Success Rate

| User Success Rate | Score Weight | Per 10-Min Period | Interpretation |
|:------------------|:-------------|:------------------|:---------------|
| **< 40%** | -5 | -5 points | Heavy penalty: Too stingy |
| **40-60%** | 0 | 0 points | Neutral: Sweet spot |
| **60-80%** | +1 | +1 point | Good: Player-friendly |
| **80-90%** | +2 | +2 points | Great: Very generous |
| **90-95%** | +3 | +3 points | Excellent: Highly generous |
| **95-100%** | +10 | +10 points | Suspicious: Possible abuse |

**Score Accumulation:**

Scores accumulate over time based on rolling averages of player mining success rates across all active sessions.

**Example:**

```
Hour 1 (6 periods of 10 minutes):
- Average user success: 55% (sweet spot)
- Score change: 0 × 6 = 0 points
- Total score: 0 (neutral)

Hour 2:
- Average user success: 98% (suspicious)
- Score change: +10 × 6 = +60 points
- Total score: +60 (flagged for review)

Hour 3:
- Average user success: 25% (too stingy)
- Score change: -5 × 6 = -30 points
- Total score: +30 (still concerning)
```

### 3.2 Score Thresholds and Actions

**Score Range: -∞ to +∞**

| Score Range | Status | Actions |
|:------------|:-------|:--------|
| **-50 to +50** | 🟢 Green (Healthy) | No action, normal operation |
| **+51 to +100** | 🟡 Yellow (Generous) | Warning notification to developer |
| **+101 to +200** | 🟠 Orange (Suspicious) | Manual review, mining rate limited to 50% |
| **+201+** | 🔴 Red (Abuse) | Mining suspended, SDK access under review |
| **-51 to -100** | 🟡 Yellow (Stingy) | Warning notification to developer |
| **-101 to -200** | 🟠 Orange (Poor UX) | Player warning displayed, reputation penalty |
| **-201+** | 🔴 Red (Predatory) | Game delisted, SDK access revoked |

**Score Decay:**

Scores decay toward zero over time to allow games to recover from temporary issues:
- **Decay Rate**: -1 point per hour (if score > 0) or +1 point per hour (if score < 0)
- **Purpose**: Rewards consistent good behavior, allows redemption

**Example Recovery:**

```
Game hits +150 (Orange status):
- Developer adjusts game to 50% user success rate
- New periods generate 0 score (neutral)
- Score decays: +150 → +149 → +148 → ... → +50 (back to Green)
- Time to recover: ~100 hours of neutral behavior
```

### 3.3 Player Notifications

When a game's score enters Orange or Red status, **players are notified before starting a session**:

**Orange Status Message:**
```
⚠️ Warning: Mining Rate Limited

This game is currently under review for unusual mining patterns.
Mining rewards are temporarily limited to 50% of normal rate.

You can still play, but rewards may be reduced.
```

**Red Status Message (Generous):**
```
🚫 Mining Temporarily Suspended

This game's mining rewards are suspended due to suspected abuse.
You can still play for fun, but no mining rewards will be earned.

The developer is working with Gorboy to resolve this issue.
```

**Red Status Message (Stingy):**
```
⚠️ Poor Mining Experience

This game has a history of very low mining rewards for players.
Consider trying other games with better reward rates.

Current average player success rate: 15%
Recommended minimum: 40%
```

---

## 4. Externalized Configuration System

All mining parameters are stored in an **external configuration file** that can be updated without modifying the SDK code. This allows for rapid tuning and experimentation.

### 4.1 Configuration File Structure

**File:** `gorboy_mining_config.json`

```json
{
  "version": "2.0",
  "last_updated": "2026-01-07T00:00:00Z",
  
  "mining_pool": {
    "total_allocation": 2000000000,
    "current_pool_size": 2000000000,
    "base_rate": 1.0,
    "adjustment_denominator": 200000000
  },
  
  "player_mining": {
    "interval_seconds": 30,
    "max_accumulation_intervals": 3,
    "proof_of_play": {
      "min_input_diversity": 10,
      "min_timing_variance_ms": 50,
      "max_session_gap_seconds": 10,
      "wallet_age_minimum_days": 0
    }
  },
  
  "developer_mining": {
    "interval_seconds": 600,
    "trigger_mode": "per_active_player_session",
    "inverse_multiplier": {
      "formula": "max(0, 1.0 - (user_rate - 0.4) / 0.6)",
      "sweet_spot_min": 0.40,
      "sweet_spot_max": 0.60
    },
    "max_fee_percentage": 0.15
  },
  
  "scoring_system": {
    "weights": {
      "under_40": -5,
      "40_to_60": 0,
      "60_to_80": 1,
      "80_to_90": 2,
      "90_to_95": 3,
      "95_to_100": 10
    },
    "thresholds": {
      "green_max": 50,
      "yellow_max": 100,
      "orange_max": 200
    },
    "decay_rate_per_hour": 1
  },
  
  "audit_system": {
    "wallet_diversity": {
      "max_new_wallet_percentage": 0.20,
      "new_wallet_age_days": 7
    },
    "pattern_detection": {
      "min_timing_variance_bot_threshold_ms": 20,
      "max_mining_success_rate_flag": 0.95,
      "min_unique_inputs_flag": 5
    }
  },
  
  "distribution": {
    "player_percentage": 1.0,
    "developer_percentage": 0.0,
    "ecosystem_percentage": 0.0,
    "note": "Developer earns separately via inverse multiplier system"
  },
  
  "progress_metrics": {
    "status": "TBD",
    "note": "Progress metric thresholds are game-specific and too arbitrary for global config. Will be defined per-game in future versions."
  }
}
```

### 4.2 Configuration Hot-Reloading

The SDK checks for configuration updates every 60 seconds and hot-reloads changes without requiring restart.

**Supported Hot-Reload Parameters:**
- Mining intervals (player and developer)
- Score weights and thresholds
- Proof-of-play requirements
- Audit thresholds
- Wallet age requirements

**NOT Hot-Reloadable (Requires SDK Update):**
- Core formula structure
- Distribution model (player/developer split)
- Smart contract addresses

### 4.3 Configuration Versioning

Each configuration change is versioned and logged on-chain for transparency:

```json
{
  "config_version": "2.0",
  "previous_version": "1.0",
  "changes": [
    "Reduced player_mining.interval_seconds from 60 to 30",
    "Changed developer_mining.max_fee_percentage from 0.30 to 0.15",
    "Set wallet_age_minimum_days to 0 (was 7)"
  ],
  "effective_date": "2026-01-07T00:00:00Z",
  "on_chain_hash": "0x7a3f9b2c..."
}
```

---

## 5. Proof-of-Play Requirements

To prevent automated farming, games must provide **cryptographic proof** that a real player performed meaningful actions during each mining interval.

### 5.1 Minimum Requirements (SDK Enforced)

| Proof Type | Requirement | Configurable | Current Value |
|:-----------|:-----------|:-------------|:--------------|
| **Input Diversity** | ≥ N unique inputs per interval | Yes | 10 |
| **Timing Variance** | Stddev > N ms | Yes | 50ms |
| **Wallet Signature** | Player signs session start | No | Required |
| **Session Continuity** | No gaps > N seconds | Yes | 10s |
| **Wallet Age** | Wallet > N days old | Yes | 0 days |

**All thresholds configurable in `gorboy_mining_config.json`**

### 5.2 Progress Metrics (TBD - Future State)

**Current Status:** Progress metrics are **too game-specific** to define globally.

**Future Implementation:**

Developers will define progress metrics per-game during SDK initialization:

```javascript
// Future API (not yet implemented)
const game = await GorboySDK.createMiningGame({
  progressMetric: {
    type: "score", // or "level", "distance", "kills", etc.
    minDeltaPerInterval: 50,
    maxDeltaPerInterval: 1000,
  }
});
```

**Why TBD?**

Different game genres have vastly different progress metrics:
- **Arcade**: Score (highly variable)
- **Platformer**: Distance or level (predictable)
- **Shooter**: Kills or accuracy (skill-based)
- **Puzzle**: Puzzles solved (discrete)
- **Survival**: Time survived (time-based)

Defining universal thresholds would either be too restrictive or too permissive. This will be addressed in V2 with per-genre guidelines.

---

## 6. Anti-Abuse Mechanisms

### 6.1 Self-Farming Detection (Developer Playing Own Game)

**The Risk:**

A developer could:
1. Create a game
2. Generate 100 fake player wallets
3. Play their own game with all 100 wallets
4. Earn both player rewards (100 wallets) AND developer rewards

**Mitigation Strategies:**

**Wallet Diversity Auditing:**

The SDK tracks wallet age distribution for each game:

```json
{
  "game_id": "space_shooter_pro",
  "total_unique_wallets": 100,
  "wallet_age_distribution": {
    "0_7_days": 85,    // 85% new wallets ⚠️
    "7_30_days": 10,   // 10%
    "30_plus_days": 5  // 5%
  },
  "flag_status": "SUSPICIOUS",
  "reason": "85% of wallets created in last 7 days"
}
```

**Threshold:** If > 20% of wallets are < 7 days old, game is flagged for review.

**Pattern Detection:**

The SDK analyzes gameplay patterns across wallets:

```json
{
  "game_id": "space_shooter_pro",
  "pattern_analysis": {
    "input_similarity": 0.92,  // 92% similar (⚠️ suspicious)
    "timing_similarity": 0.88,  // 88% similar (⚠️ suspicious)
    "session_duration_variance": 0.05,  // Very consistent (⚠️)
    "progress_rate_variance": 0.03  // Very consistent (⚠️)
  },
  "flag_status": "SUSPICIOUS",
  "reason": "Wallets show bot-like consistency"
}
```

**IP Analysis:**

The SDK tracks IP address diversity:

```json
{
  "game_id": "space_shooter_pro",
  "total_unique_ips": 5,
  "total_unique_wallets": 100,
  "ratio": 0.05,  // 5% (⚠️ suspicious)
  "flag_status": "SUSPICIOUS",
  "reason": "100 wallets from only 5 IP addresses"
}
```

**Combined Scoring:**

When multiple red flags appear, confidence increases:

```
Game: "Space Shooter Pro"
- 85% new wallets ⚠️
- 92% input similarity ⚠️
- 5% IP diversity ⚠️
- Developer receiving 80% of total mined tokens ⚠️

Confidence: 95% likely self-farming
Action: Mining suspended, manual review required
```

### 6.2 Automated Detection Systems

**Pattern Detection (SDK-Side):**

| Red Flag | Threshold | Action |
|:---------|:----------|:-------|
| **Superhuman consistency** | Timing variance < 20ms | Auto-reject proofs |
| **Identical input patterns** | > 80% similarity | Flag for review |
| **Wallet age clustering** | > 20% wallets < 7 days | Flag for review |
| **IP concentration** | < 10% unique IPs per wallet | Flag for review |
| **Developer token ratio** | > 50% of total mined | Flag for review |
| **Perfect mining rate** | > 95% success for 100+ intervals | Flag for review |

### 6.3 Limitations and Future Improvements

**Current System Limitations:**

⚠️ **Sophisticated Attackers Can Evade:**
- Use aged wallets (buy old accounts)
- Use VPN/proxy to diversify IPs
- Introduce artificial variance in inputs
- Spread farming across multiple games

**Future Enhancements (V2+):**

✅ **Machine Learning Pattern Detection:**
- Train models on known-good gameplay
- Detect subtle anomalies humans miss
- Continuous learning from new attack patterns

✅ **Social Graph Analysis:**
- Track wallet interaction history
- Identify suspicious wallet clusters
- Detect Sybil attacks (one person, many wallets)

✅ **Behavioral Biometrics:**
- Mouse movement patterns
- Keystroke dynamics
- Reaction time distributions
- Unique per-human signatures

✅ **Economic Disincentives:**
- Require stake to play mining games
- Slash stake if caught farming
- Make self-farming unprofitable

---

## 7. Token Distribution and Minting

### 7.1 Player Mining Distribution

When a player successfully claims a mining reward:

**Step 1: Calculate Reward**
```
current_pool = 200,000,000 GOR
base_rate = 1.0
tokens_per_unit = (200,000,000 / 200,000,000) × 1.0 = 1.0 GOR
```

**Step 2: Mint Tokens**
```
Smart contract mints 1.0 GOR from mining pool
Pool decreases: 200,000,000 → 199,999,999 GOR
```

**Step 3: Distribute**
```
Player wallet: +1.0 GOR (100%)
Developer wallet: +0.0 GOR (0% - earns separately)
Ecosystem: +0.0 GOR (0% - already accounted for in 2B allocation)
```

**Step 4: Record On-Chain**
```
Transaction hash: 0x7a3f9b2c...
Proof-of-play hash: 0x9f2e8a1b...
Timestamp: 2026-01-07 14:32:15 UTC
```

### 7.2 Developer Mining Distribution

When a developer successfully claims a mining reward (every 10 minutes per active player):

**Step 1: Calculate Base Reward**
```
current_pool = 199,999,999 GOR
base_rate = 1.0
tokens_per_unit = (199,999,999 / 200,000,000) × 1.0 ≈ 1.0 GOR
```

**Step 2: Apply Inverse Multiplier**
```
user_success_rate = 0.55 (55% - in sweet spot)
developer_multiplier = max(0, 1.0 - (0.55 - 0.4) / 0.6)
                     = max(0, 1.0 - 0.25)
                     = 0.75

developer_reward = 1.0 × 0.75 = 0.75 GOR
```

**Step 3: Mint Tokens**
```
Smart contract mints 0.75 GOR from mining pool
Pool decreases: 199,999,999 → 199,999,998.25 GOR
```

**Step 4: Distribute**
```
Developer wallet: +0.75 GOR (100% of developer reward)
Player wallet: +0.0 GOR (0% - already received separately)
Ecosystem: +0.0 GOR (0% - no ecosystem fee)
```

**Step 5: Record On-Chain**
```
Transaction hash: 0x3c5d7e9a...
User success rate: 55%
Multiplier applied: 0.75x
Timestamp: 2026-01-07 14:42:15 UTC
```

---

## 8. SDK Implementation

### 8.1 Developer Integration

**Minimal Integration:**

```javascript
// Initialize mining game
const game = await GorboySDK.createMiningGame({
  gameName: "Space Shooter Pro",
  
  // Progress metrics TBD - not required in V1
  // progressMetric: { type: "score", minDelta: 100, maxDelta: 2000 },
  
  // Developer fee removed - earns via inverse multiplier instead
  // developerFee: 0.20,
  
  // Mining interval loaded from config (30s default)
  // miningInterval: 30,
});

// Start player session
await game.startSession({
  playerWallet: "0x7a3f9b2c...",
});

// Developer controls when to award mining units
// Option 1: Award immediately every interval
await game.awardMiningUnit(); // Called every 30s

// Option 2: Accumulate and award on milestone
if (player.score >= 1000) {
  await game.awardAccumulatedMiningUnits(); // Awards all pending units
}

// Option 3: Let SDK auto-award (simplest)
await game.enableAutoAward(); // SDK awards every interval automatically

// SDK automatically:
// - Tracks inputs and timing
// - Validates proof of play
// - Mints and distributes tokens
// - Calculates developer rewards
// - Records transactions on-chain
// - Monitors for abuse patterns
```

### 8.2 SDK Automatic Handling

**What SDK Does Automatically:**

✅ Tracks player inputs (keystrokes, clicks, touches)  
✅ Analyzes timing variance  
✅ Validates wallet signatures  
✅ Enforces time intervals (configurable)  
✅ Generates proof-of-play  
✅ Calculates user success rate  
✅ Applies inverse multiplier to developer rewards  
✅ Mints and distributes tokens  
✅ Records transactions on-chain  
✅ Monitors for abuse patterns  
✅ Updates game reputation score  
✅ Notifies players of game status  

**What Developer Must Do:**

📝 Decide when to award mining units (immediate, milestone, or auto)  
📝 Handle mining reward notifications (optional)  
📝 Monitor game reputation score  

**What Developer Cannot Do:**

❌ Bypass proof-of-play validation  
❌ Mint tokens without player wallet  
❌ Manipulate inverse multiplier formula  
❌ Fake input patterns or timing data  
❌ Award more than available potential units  

---

## 9. Monitoring and Analytics

### 9.1 Developer Dashboard

Developers can view real-time mining analytics:

**Example Dashboard:**

```
┌─────────────────────────────────────────────────┐
│ Mining Analytics - Space Shooter Pro            │
├─────────────────────────────────────────────────┤
│ Reputation Score: +12 🟢 Green (Healthy)        │
│                                                 │
│ PLAYER MINING                                   │
│ Total Mined: 8,715 GOR                          │
│ Success Rate: 55% (Sweet Spot ✅)               │
│ Potential Units: 15,845                         │
│ Actual Units: 8,715                             │
│                                                 │
│ DEVELOPER MINING                                │
│ Total Mined: 1,247 GOR                          │
│ Current Multiplier: 0.75x                       │
│ Active Sessions: 23                             │
│ Units This Hour: 138 (23 players × 6 periods)  │
│                                                 │
│ WALLET DIVERSITY                                │
│ Unique Wallets: 1,247                           │
│ Wallet Age Distribution:                        │
│ ├─ > 30 days: 78% ✅                            │
│ ├─ 7-30 days: 18% ✅                            │
│ └─ < 7 days: 4% ✅                              │
│                                                 │
│ GAMEPLAY PATTERNS                               │
│ Avg Session: 18 minutes                         │
│ Input Diversity: 11.3 avg unique                │
│ Timing Variance: 134ms avg                      │
│                                                 │
│ ⚠️ No issues detected                           │
│                                                 │
│ OPTIMIZATION TIPS                               │
│ • Your 55% success rate is optimal!             │
│ • Consider milestone-based rewards for drama    │
│ • Player retention is strong (18 min avg)       │
└─────────────────────────────────────────────────┘
```

### 9.2 Player Transparency

Players can view their mining history:

**Example Player Dashboard:**

```
┌─────────────────────────────────────────────────┐
│ Your Mining History                             │
├─────────────────────────────────────────────────┤
│ Total Mined: 45.2 GOR                           │
│ Sessions: 67                                    │
│ Avg Success Rate: 58%                           │
│                                                 │
│ Recent Sessions:                                │
│ ├─ Jan 7, 14:32 - Space Shooter Pro            │
│ │  Duration: 12 min (24 intervals)             │
│ │  Potential: 24 units                         │
│ │  Mined: 14 units (58%) ✅                    │
│ │  Earned: 0.7 GOR                             │
│ │  Proof: 0x7a3f... [Verify]                   │
│ │                                               │
│ ├─ Jan 7, 13:15 - Space Shooter Pro            │
│ │  Duration: 8 min (16 intervals)              │
│ │  Potential: 16 units                         │
│ │  Mined: 0 units (0%) ❌                      │
│ │  Earned: 0.0 GOR                             │
│ │  Reason: Insufficient input diversity        │
│ │                                               │
│ └─ Jan 6, 19:45 - Space Shooter Pro            │
│    Duration: 22 min (44 intervals)             │
│    Potential: 44 units                         │
│    Mined: 28 units (64%) ✅                    │
│    Earned: 1.4 GOR                             │
│    Proof: 0x9f2e... [Verify]                   │
│                                                 │
│ GAME COMPARISON                                 │
│ ├─ Space Shooter Pro: 58% avg success ⭐⭐⭐    │
│ ├─ Puzzle Master: 42% avg success ⭐⭐         │
│ └─ Racing Thunder: 71% avg success ⭐⭐⭐⭐     │
└─────────────────────────────────────────────────┘
```

---

## 10. Summary

The Proof-of-Play Mining system ensures fair, abuse-resistant token distribution through an innovative dual mining model:

**Self-Regulating Economics:**
- Dynamic adjustment based on pool size (2B total allocation)
- Early adopters earn more tokens at lower value
- Late adopters earn fewer tokens at higher value
- No ecosystem fee (already accounted for)

**Dual Mining Model:**
- **Players**: Earn every 30 seconds, 100% of mined tokens
- **Developers**: Earn every 10 minutes per active player, subject to inverse multiplier

**Inverse Incentive System:**
- Developer rewards inversely proportional to player success rate
- Sweet spot at 40-60% player success
- Naturally balances player satisfaction with developer income
- Prevents both stinginess and suspicious generosity

**Weighted Scoring:**
- Games scored based on player mining success rate
- Scores accumulate and trigger warnings/penalties
- Players notified of problematic games
- Score decay allows recovery over time

**Externalized Configuration:**
- All parameters in `gorboy_mining_config.json`
- Hot-reloadable without SDK restart
- Versioned and logged on-chain
- Enables rapid tuning and experimentation

**Anti-Abuse Mechanisms:**
- Proof-of-play validation (input diversity, timing variance)
- Wallet diversity auditing
- Pattern detection across sessions
- IP analysis and clustering detection
- Self-farming mitigation (though not perfect)

**Transparency:**
- Developer dashboard with real-time analytics
- Player dashboard with mining history
- On-chain proof-of-play verification
- Public game reputation scores

This system creates a sustainable mining economy that rewards genuine gameplay, incentivizes balanced game design, and prevents automated farming—all while remaining flexible and tunable through external configuration.
# GORBOY SDK Specification - Section 4: House vs Player Mechanics

**Version:** 1.0  
**Status:** Approved for Implementation

---

## 1. Overview

House vs Player (HvP) games represent a distinct category where the game developer (or a designated "house") acts as the opponent, offering fixed or progressive jackpots to players. Unlike Player vs Player games where stakes are matched, HvP games require the house to maintain sufficient escrow to cover all potential payouts.

This section defines the requirements, safeguards, and best practices for HvP games in the Gorboy SDK.

---

## 2. House Escrow Requirements

### 2.1 Full Escrow Mandate

**Core Principle:** The house MUST have the full maximum potential payout in escrow before any player can enter the game.

**Example:**
```
Game: Mega Slots
Maximum Payout: 1,000,000 GOR
House Escrow Required: ≥ 1,000,000 GOR

If house has:
- 1,500,000 GOR → Display shows "1,000,000 GOR" (capped at max)
- 1,000,000 GOR → Display shows "1,000,000 GOR" (fully funded)
- 750,000 GOR → Display shows "750,000 GOR" (underfunded)
```

**Why Full Escrow?**
- Ensures players can always be paid if they win
- Prevents house from advertising jackpots they can't afford
- Builds trust and transparency
- Protects players from insolvency risk

### 2.2 Escrow Display Moniker (Color-Coded)

The SDK provides a **standardized display widget** that shows house escrow status with color-coding:

| Color | Escrow Level | Meaning | Player Action |
|:------|:------------|:--------|:--------------|
| **🟢 Green** | ≥ 100% of max payout | Fully funded | Play normally |
| **🟡 Yellow** | 75-99% of max payout | Slightly underfunded | Lower bet or proceed with caution |
| **🟠 Orange** | 50-74% of max payout | Significantly underfunded | Lower bet recommended |
| **🔴 Red** | < 50% of max payout | Critically underfunded | Lower bet or don't play |

**Display Behavior:**

**Green (Fully Funded):**
```
┌─────────────────────────────┐
│ 🟢 Maximum Payout Available │
│    1,000,000 GOR            │
│                             │
│ House Escrow: Fully Funded  │
│ Verification: 7a3f9b2c      │
└─────────────────────────────┘
```

**Yellow/Orange/Red (Underfunded):**
```
┌─────────────────────────────┐
│ ⚠️ HOUSE UNDERFUNDED ⚠️     │
│    750,000 GOR Available    │
│    (75% of max payout)      │
│                             │
│ ⚠️ Lower your bet to see    │
│    green, or maximum payout │
│    may not be available.    │
│                             │
│ Verification: 7a3f9b2c      │
└─────────────────────────────┘
```

**Display blinks** when yellow/orange/red to draw attention.

### 2.3 Dynamic Bet Adjustment

When house is underfunded, players can **lower their bet** until display turns green.

**Example:**
```
House Escrow: 500,000 GOR
Player wants to bet: 100 GOR with 10,000x multiplier (1M max payout)

SDK calculates:
- Max payout at 100 GOR bet: 1,000,000 GOR
- House only has: 500,000 GOR
- Display: 🔴 Red (50% funded)

Player lowers bet to 50 GOR:
- Max payout at 50 GOR bet: 500,000 GOR
- House has: 500,000 GOR
- Display: 🟢 Green (100% funded)

Player can now play safely!
```

### 2.4 Partial Payout on Underfunded Win

If a player chooses to play when house is underfunded (yellow/orange/red) and wins:

**Payout Rules:**
1. Player receives **up to the available escrow amount**
2. If win exceeds available escrow, player gets **partial payout**
3. House escrow depletes to **zero**
4. Game **locks immediately** for all players
5. SDK displays: "House bankrupt - game locked pending refund"

**Example:**
```
House Escrow: 750,000 GOR (🟡 Yellow)
Player bets: 100 GOR
Player wins: 1,000,000 GOR jackpot

Payout:
- Player receives: 750,000 GOR (all available escrow)
- Player does NOT receive: 250,000 GOR (house can't cover)
- House escrow: 0 GOR
- Game status: LOCKED

All other players see:
"🚫 House bankrupt! Game locked. House must refund to reopen."
```

**Player Warning Before Playing:**

When house is underfunded, SDK shows warning before allowing play:

```
┌─────────────────────────────────────────────┐
│ ⚠️ WARNING: HOUSE UNDERFUNDED ⚠️            │
├─────────────────────────────────────────────┤
│ House only has 750,000 GOR in escrow.       │
│ Maximum payout is advertised as 1,000,000.  │
│                                             │
│ If you win the jackpot, you will receive:   │
│ • 750,000 GOR (all available funds)         │
│ • NOT the full 1,000,000 GOR                │
│                                             │
│ Recommendation: Lower your bet until green. │
│                                             │
│ [ Lower Bet ] [ Play Anyway ] [ Cancel ]    │
└─────────────────────────────────────────────┘
```

---

## 3. Verification System

### 3.1 Real-Time Verification Code

The escrow display includes a **constantly updating verification code** (similar to Section 2's visual verification token).

**How It Works:**

1. **SDK generates code** based on:
   - Current escrow amount
   - House wallet address
   - Timestamp
   - Game ID
   - Cryptographic hash

2. **Code updates every 30 seconds**

3. **Players can verify** on Gorboy website:
   - Enter code or scan QR
   - Website confirms:
     - ✅ Valid Gorboy SDK game
     - ✅ House wallet has X GOR in escrow
     - ✅ Escrow matches displayed amount
     - ✅ Game is not suspended/banned

**Example Verification:**

```
Player sees display:
┌─────────────────────────────┐
│ 🟢 Maximum Payout Available │
│    1,000,000 GOR            │
│                             │
│ Verification: 7a3f9b2c      │
│ [QR Code]                   │
└─────────────────────────────┘

Player visits verify.gorboy.com and enters "7a3f9b2c"

Website shows:
✅ Valid Game: "Mega Slots" by CasinoDev
✅ House Wallet: 0x3c5d7e9a...
✅ Escrow Amount: 1,247,583 GOR
✅ Max Payout: 1,000,000 GOR
✅ Status: Fully Funded (🟢 Green)
✅ Game Status: Active
✅ House Reputation: 87 (Gold)
```

### 3.2 Anti-Fraud Protection

**What Verification Prevents:**

❌ **Fake displays**: Malicious developer can't show fake escrow amounts  
❌ **Outdated displays**: Code expires every 30 seconds  
❌ **Screenshot reuse**: Code changes constantly  
❌ **Wallet manipulation**: Website checks actual on-chain escrow  

**If verification fails:**

```
Player enters code "7a3f9b2c"

Website shows:
❌ Invalid Code
This verification code does not match any active game.

Possible reasons:
• Code expired (codes are valid for 30 seconds)
• Game is not using official Gorboy SDK
• Display is fake or manipulated

⚠️ DO NOT PLAY THIS GAME
```

---

## 4. Concurrent Winner Handling

### 4.1 Win Window Definition

**Win Window:** The time period from when the first player commits an action until that player's result is determined.

**Example (Slot Machine):**

```
Timeline:
14:00:00.000 - Player A clicks "Spin" (window opens)
14:00:00.050 - Player B clicks "Spin" (within window)
14:00:00.100 - Player C clicks "Spin" (within window)
14:00:00.150 - Player A's result determined: WIN! (window closes)
14:00:00.200 - Player D clicks "Spin" (outside window, not included)

Result:
- Players A, B, C are in the win window
- If any/all win, they share the jackpot
- Player D plays in next round with separate escrow
```

**Key Principle:** SDK should determine win/loss **instantly** (milliseconds), even if visual reveal is delayed for UX.

### 4.2 Jackpot Splitting

When multiple players win within the same win window:

**Step 1: Lock Escrow**
- Moment first win is detected, escrow locks
- No new players can enter win window

**Step 2: Determine All Winners**
- Check all players in win window
- Identify who won

**Step 3: Split Jackpot**
- Divide locked escrow equally among winners

**Step 4: Payout**
- Distribute shares to all winners
- Record transactions on-chain

**Step 5: Reset**
- If house has remaining escrow, unlock and continue
- If house is depleted, lock game

**Example:**

```
Jackpot: 1,000,000 GOR
Win Window: Players A, B, C

Results:
- Player A: WIN
- Player B: LOSE
- Player C: WIN

Payout:
- Player A: 500,000 GOR (50%)
- Player C: 500,000 GOR (50%)
- Player B: 0 GOR (lost)

House Escrow After:
- Started with: 1,500,000 GOR
- Paid out: 1,000,000 GOR
- Remaining: 500,000 GOR
- Display now shows: 500,000 GOR (if max payout still 1M, shows 🟡 Yellow)
```

### 4.3 Configurable Concurrent Win Limits

Developers can configure how many concurrent winners are allowed before auto-lock:

**Configuration Variable:** `concurrent_win_limit`

**Default:** 5 winners

**Behavior:**

| Concurrent Winners | Action |
|:-------------------|:-------|
| **1-4 winners** | Normal payout, continue |
| **5 winners (limit)** | Payout, then auto-lock for review |
| **> 5 winners** | Should not occur (lock happens at 5) |

**Why Auto-Lock at Limit?**
- Protects house from rapid depletion
- Allows developer to review for exploits
- Prevents cascading losses if bug exists

**Example:**

```
Game: Mega Slots
Concurrent Win Limit: 5
Jackpot: 100,000 GOR each

Scenario:
- 5 players win simultaneously
- Total payout: 500,000 GOR
- House pays all 5 winners
- Game auto-locks with message:
  "🔒 Multiple winners detected! Game locked for review.
   Developer will review and reopen soon."
```

---

## 5. Exploit Detection and Emergency Locks

### 5.1 Exploit Detection Threshold

**Configuration Variable:** `exploit_detection_threshold`

**Default:** 10 wins per minute

**Behavior:**

If win rate exceeds threshold, game auto-locks and notifies developer:

```
Game: Mega Slots
Threshold: 10 wins/minute

Scenario:
- 15 players win in 1 minute
- Threshold exceeded (15 > 10)
- Game auto-locks
- Developer receives alert:
  "⚠️ EXPLOIT ALERT: 15 wins in 1 minute (threshold: 10)
   Game auto-locked. Review immediately."
```

**Statistical Probability Check:**

SDK can also compare actual win rate to expected win rate:

```
Expected win rate: 1% (house edge 99%)
Actual win rate: 15% (over 1 hour)

Statistical deviation: 15x expected
Confidence: 99.9% likely exploit

Action: Auto-lock, notify developer
```

### 5.2 Massive Jackpot Protection

**Configuration Variable:** `massive_jackpot_threshold`

**Purpose:** For very large jackpots, lock after **1 win** (not 5) to allow review.

**Example:**

```
Game: Mega Lottery
Jackpot: 10,000,000 GOR (massive)
Massive Jackpot Threshold: 5,000,000 GOR
Concurrent Win Limit: 5 (normal)

Behavior:
- Jackpot (10M) > Threshold (5M)
- Lock after 1 win (not 5)
- Developer reviews before reopening

Message to players:
"🎉 JACKPOT WON! 10,000,000 GOR!
 Game locked for verification.
 Come back tomorrow when we can afford to play again! 😅"
```

**Why Lock After 1 Win?**
- Massive payouts require careful review
- Ensures no exploit before continuing
- Allows house to refund/adjust if needed
- Prevents catastrophic losses

**Progressive Jackpot Reset:**

After massive jackpot win, progressive jackpots can reset to low amount (like lottery):

```
Before Win:
- Progressive Jackpot: 10,000,000 GOR (accumulated over time)

After Win:
- Winner receives: 10,000,000 GOR
- Progressive resets to: 100,000 GOR (starting amount)
- Players see: "Jackpot won! Starting fresh at 100K!"
```

### 5.3 Manual Emergency Lock

Developers can **manually lock** the game at any time:

**SDK API:**

```javascript
// Developer detects suspicious activity
await game.emergencyLock({
  reason: "Suspected exploit - investigating",
  duration: "24 hours",
  message: "Game temporarily locked. We'll be back soon!",
});

// All players see:
"🔒 Game Locked
 Reason: Suspected exploit - investigating
 Expected return: 24 hours
 Your funds are safe in escrow."
```

**When to Use:**
- Suspicious win patterns detected
- Bug discovered in game logic
- Unusual player behavior
- Precautionary review needed

---

## 6. Jackpot Win Event Hooks (Drama & Excitement)

### 6.1 Winner Broadcast System

When a jackpot is won and escrow locks, SDK provides **event hooks** for developers to create excitement.

**SDK Event:**

```javascript
game.on('jackpot_won', (event) => {
  // event.winner_location: "California"
  // event.amount: 1000000
  // event.lock_duration: "10 minutes"
  
  game.broadcastToAllPlayers({
    type: 'jackpot_won',
    title: '🎰 JACKPOT WON! 🎰',
    message: `Someone in ${event.winner_location} just won ${event.amount} GOR!`,
    animation: 'house_bust',
    sound: 'jackpot_fanfare',
  });
});
```

**Example Messages:**

```
🎉 WINNER IN CALIFORNIA! 🎉
Someone just won 1,000,000 GOR!
House is busted! Going to the bank...
Come back in 10 minutes!

---

💰 HOUSE CLEANED OUT! 💰
A lucky player just ran away with everything!
We're refunding now... stand by!

---

🎰 JACKPOT HIT! 🎰
10,000,000 GOR WINNER!
Game locked for verification.
This is HUGE! 🚀
```

**Why This Matters:**

✅ **Builds excitement**: Winning feels like a big event  
✅ **Creates FOMO**: Other players want to win too  
✅ **Keeps players engaged**: They wait for unlock instead of leaving  
✅ **Social proof**: "Real people are winning!"  
✅ **Marketing**: Winners create buzz and attract new players  

### 6.2 Lock Duration Display

While game is locked, show countdown timer:

```
┌─────────────────────────────────────────┐
│ 🔒 GAME LOCKED - JACKPOT WON! 🔒        │
├─────────────────────────────────────────┤
│ Someone just won 1,000,000 GOR!         │
│                                         │
│ House is refunding escrow...            │
│                                         │
│ ⏰ Reopening in: 08:32                  │
│    (8 minutes, 32 seconds)              │
│                                         │
│ [ Notify Me When Ready ]                │
└─────────────────────────────────────────┘
```

**Notification When Unlocked:**

```
🎰 GAME REOPENED! 🎰
Mega Slots is back online!
New jackpot: 1,000,000 GOR
Come play now!
```

---

## 7. House Reputation System

### 7.1 Reputation Scoring

Houses (game developers running HvP games) have **reputation scores** similar to the player reputation system.

**Reputation Factors:**

| Factor | Weight | Description |
|:-------|:-------|:------------|
| **Escrow Reliability** | 40% | How often house is fully funded |
| **Payout Speed** | 20% | Time from win to payout |
| **Player Satisfaction** | 20% | Player ratings and reviews |
| **Exploit Frequency** | 10% | How often exploits are found |
| **Complaint Rate** | 10% | Player complaints per 1000 games |

**Reputation Tiers:**

| Tier | Score | Badge | Meaning |
|:-----|:------|:------|:--------|
| **Platinum** | 95-100 | 🟣 | Exemplary house, fully trusted |
| **Gold** | 80-94 | 🟢 | Good house, reliable |
| **Silver** | 60-79 | ⚪ | Neutral, new or average |
| **Bronze** | 40-59 | 🟡 | Needs improvement, caution advised |
| **Tin** | 20-39 | 🟠 | Poor reputation, high risk |
| **Restricted** | 0-19 | 🔴 | Avoid, likely scam or incompetent |

### 7.2 Reputation Penalties

**Automatic Penalties:**

| Violation | Penalty | Cumulative Effect |
|:----------|:--------|:------------------|
| **Underfunded game** | -5 points per occurrence | Frequent underfunding → Tin/Restricted |
| **Failed payout** | -10 points | Can't pay winners → Suspended |
| **Exploit found** | -15 points | Security issues → Review required |
| **Player complaint** | -1 point per complaint | High complaint rate → Bronze/Tin |
| **Slow payout (> 1 hour)** | -2 points | Delays frustrate players |

**Reputation Recovery:**

- +1 point per 100 successful games (no issues)
- +5 points for 30 consecutive days of full escrow
- +10 points for player satisfaction > 90%

### 7.3 Suspension and Banning

**Suspension Triggers:**

| Condition | Action | Duration |
|:----------|:-------|:---------|
| **Reputation < 40 (Bronze)** | Warning notification | N/A |
| **Reputation < 20 (Tin)** | Temporary suspension | 7 days |
| **Failed payout (bankrupt)** | Immediate suspension | Until refunded |
| **10+ complaints in 24 hours** | Temporary suspension | 48 hours |
| **Exploit with losses > 1M GOR** | Immediate suspension | Until resolved |

**Ban Triggers:**

| Condition | Action |
|:----------|:-------|
| **Reputation 0 (Restricted)** | Permanent ban |
| **Repeated exploits (3+)** | Permanent ban |
| **Fraud or scam confirmed** | Permanent ban |
| **Refusal to pay winners** | Permanent ban |

**Player-Facing Messages:**

**Suspended:**
```
⚠️ GAME SUSPENDED
This game is temporarily suspended pending review.

Reason: House failed to maintain adequate escrow
Expected resolution: 7 days

Your funds in escrow are safe and will be returned.
```

**Banned:**
```
🚫 GAME BANNED
This game has been permanently banned from Gorboy SDK.

Reason: Repeated exploits and failure to pay winners

DO NOT PLAY THIS GAME.
All active escrows will be refunded.
```

### 7.4 House Reputation Display

Players can view house reputation before playing:

```
┌─────────────────────────────────────────┐
│ House Reputation: CasinoDev            │
├─────────────────────────────────────────┤
│ Score: 87 🟢 Gold                       │
│                                         │
│ Escrow Reliability: 95% ⭐⭐⭐⭐⭐        │
│ Payout Speed: 2.3 min avg ⭐⭐⭐⭐         │
│ Player Satisfaction: 4.2/5 ⭐⭐⭐⭐        │
│ Exploit Frequency: 0 in 6 months ✅     │
│ Complaint Rate: 0.3% ✅                 │
│                                         │
│ Total Games: 12,450                     │
│ Total Payouts: 8,234,567 GOR            │
│                                         │
│ ✅ Verified House                       │
│ ✅ Fully Insured                        │
│ ✅ No Active Complaints                 │
└─────────────────────────────────────────┘
```

---

## 8. House Edge Transparency (Recommended)

### 8.1 What Is House Edge?

**House edge** is the mathematical advantage the house has over players, expressed as a percentage.

**Examples:**
- **Roulette**: ~5.26% house edge (due to 0 and 00)
- **Blackjack**: ~0.5% house edge (with perfect play)
- **Slot machines**: 2-15% house edge (varies)

**What It Means:**
- Over time, house keeps X% of all money wagered
- Players lose X% on average
- Higher edge = worse odds for players

### 8.2 Display Recommendation

**SDK Recommendation:** Display house edge for transparency, but **not required** for compliance.

**Example Display:**

```
┌─────────────────────────────────────────┐
│ Game: Mega Slots                        │
│ Max Payout: 1,000,000 GOR              │
│                                         │
│ House Edge: 5.0%                        │
│ (You keep 95% on average)               │
│                                         │
│ RTP (Return to Player): 95.0%           │
│                                         │
│ ℹ️ Lower house edge = better odds      │
└─────────────────────────────────────────┘
```

**Benefits of Displaying:**
- ✅ Transparency builds trust
- ✅ Players can compare games
- ✅ Encourages fair house edges
- ✅ Reduces complaints ("I didn't know odds were bad")

**Optional Maximum:**

While not enforced, SDK can **recommend** maximum house edge:
- **Recommended max: 20%**
- Games with > 20% edge get warning badge
- Players see: "⚠️ High house edge - poor odds"

---

## 9. Configuration Parameters

All HvP parameters are configurable in `gorboy_hvp_config.json`:

```json
{
  "version": "1.0",
  "last_updated": "2026-01-07T00:00:00Z",
  
  "escrow_requirements": {
    "full_escrow_required": true,
    "display_color_thresholds": {
      "green_min": 1.00,
      "yellow_min": 0.75,
      "orange_min": 0.50,
      "red_max": 0.49
    },
    "verification_code_update_interval_seconds": 30
  },
  
  "concurrent_winners": {
    "win_window_mode": "instant",
    "concurrent_win_limit": 5,
    "massive_jackpot_threshold": 5000000,
    "massive_jackpot_win_limit": 1
  },
  
  "exploit_detection": {
    "threshold_wins_per_minute": 10,
    "statistical_deviation_threshold": 10.0,
    "auto_lock_on_threshold": true
  },
  
  "house_reputation": {
    "weights": {
      "escrow_reliability": 0.40,
      "payout_speed": 0.20,
      "player_satisfaction": 0.20,
      "exploit_frequency": 0.10,
      "complaint_rate": 0.10
    },
    "penalties": {
      "underfunded_game": -5,
      "failed_payout": -10,
      "exploit_found": -15,
      "player_complaint": -1,
      "slow_payout": -2
    },
    "suspension_threshold": 20,
    "ban_threshold": 0
  },
  
  "house_edge": {
    "display_required": false,
    "display_recommended": true,
    "recommended_maximum": 0.20,
    "warning_threshold": 0.20
  },
  
  "jackpot_events": {
    "broadcast_enabled": true,
    "lock_duration_default_seconds": 600,
    "notification_enabled": true
  }
}
```

---

## 10. SDK Implementation

### 10.1 Developer Integration

**Creating HvP Game:**

```javascript
// Initialize House vs Player game
const game = await GorboySDK.createHvPGame({
  gameName: "Mega Slots",
  maxPayout: 1000000, // GOR
  houseWallet: "0x3c5d7e9a...",
  
  // Optional: Configure concurrent winner limits
  concurrentWinLimit: 5,
  massiveJackpotThreshold: 5000000,
  
  // Optional: Configure exploit detection
  exploitDetectionThreshold: 10, // wins per minute
  
  // Optional: Display house edge (recommended)
  houseEdge: 0.05, // 5%
  
  // Optional: Progressive jackpot
  progressiveJackpot: {
    enabled: true,
    startingAmount: 100000,
    contributionRate: 0.01, // 1% of each bet
  },
});

// SDK automatically:
// - Verifies house has sufficient escrow
// - Displays color-coded escrow status
// - Generates verification codes
// - Handles concurrent winners
// - Detects exploits
// - Broadcasts jackpot wins
// - Manages reputation

// Handle jackpot win event
game.on('jackpot_won', (event) => {
  console.log(`Winner in ${event.winner_location}!`);
  console.log(`Amount: ${event.amount} GOR`);
  
  // Broadcast to all players
  game.broadcastToAllPlayers({
    title: '🎰 JACKPOT WON! 🎰',
    message: `Someone in ${event.winner_location} just won ${event.amount} GOR!`,
    animation: 'house_bust',
  });
  
  // Optionally lock for review
  if (event.amount > 5000000) {
    await game.lock({
      reason: "Massive jackpot - reviewing before reopen",
      duration: "24 hours",
    });
  }
});

// Handle exploit detection
game.on('exploit_detected', (event) => {
  console.error(`Exploit detected: ${event.reason}`);
  
  // Game auto-locks, developer reviews
  await reviewGameLogs();
  
  // If false positive, unlock
  await game.unlock();
});

// Manual emergency lock
await game.emergencyLock({
  reason: "Suspected bug - investigating",
  duration: "2 hours",
});
```

### 10.2 Player Experience

**Before Playing:**

1. Player sees escrow display (green/yellow/orange/red)
2. Player can verify escrow on Gorboy website
3. Player sees house reputation score
4. Player sees house edge (if displayed)
5. Player decides whether to play

**During Play:**

1. SDK determines win/loss instantly
2. Visual reveal can be delayed for UX
3. If win, SDK checks for concurrent winners
4. Escrow locks if needed

**After Win:**

1. SDK calculates payout (split if concurrent winners)
2. Tokens distributed to winner(s)
3. Transaction recorded on-chain
4. If massive jackpot, game locks and broadcasts to all players
5. House reputation updated

**If Underfunded:**

1. Player sees warning before playing
2. Player can lower bet until green
3. If plays anyway and wins, receives partial payout
4. Game locks if house depletes to zero

---

## 11. Summary

Section 4 establishes comprehensive safeguards and best practices for House vs Player games:

**Escrow Requirements:**
- Full escrow required (100% of max payout)
- Color-coded display (green/yellow/orange/red)
- Real-time verification codes
- Dynamic bet adjustment for underfunded games

**Concurrent Winner Handling:**
- Instant win determination (milliseconds)
- Win window from action start to result
- Jackpot splitting among concurrent winners
- Configurable limits (default: 5 winners)

**Exploit Protection:**
- Auto-lock at win rate threshold (default: 10/min)
- Statistical deviation detection
- Massive jackpot protection (lock after 1 win)
- Manual emergency lock capability

**Jackpot Win Events:**
- Broadcast system for excitement/FOMO
- Lock duration display with countdown
- Notification when game reopens
- Encourages player retention

**House Reputation:**
- Scoring based on reliability, speed, satisfaction
- Automatic penalties for violations
- Suspension and banning for severe issues
- Transparent display for players

**House Edge Transparency:**
- Recommended but not required
- Helps players make informed decisions
- Suggested maximum: 20%

**Configuration:**
- All parameters externalized
- Hot-reloadable without SDK restart
- Developer flexibility with recommended defaults

This system ensures fair, transparent, and exciting House vs Player games while protecting both players and houses from exploits and insolvency.
# GORBOY SDK Specification - Section 5: Tech Stack & SDK Architecture

**Version:** 1.0  
**Status:** Approved for Implementation

---

## 1. Overview

The Gorboy SDK is designed as a **language-agnostic RESTful API** that enables game developers to integrate blockchain-based gaming features without requiring blockchain expertise. The architecture prioritizes simplicity, scalability, and developer experience while maintaining security and performance.

**Core Philosophy:**
- **Language Agnostic**: Any language with HTTP support can integrate
- **Backend-Driven**: Core logic lives in Gorboy's Azure-hosted API
- **Blockchain-Abstracted**: Developers don't need to understand smart contracts
- **Optional Client SDKs**: Thin wrappers for better developer experience

---

## 2. Architecture Overview

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Game Developer's Code                    │
│              (JavaScript, Unity, Python, etc.)               │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP/REST
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                    Gorboy RESTful API                        │
│              (Azure Functions / App Service)                 │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Game API   │  │  Mining API  │  │  Escrow API  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Reputation   │  │ Verification │  │   Audit      │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└──────────────────────────┬──────────────────────────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ↓                 ↓                 ↓
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   Azure SQL     │ │  Redis Cache    │ │  Blob Storage   │
│  (Core Data)    │ │  (Hot Data)     │ │  (Logs, Replay) │
└─────────────────┘ └─────────────────┘ └─────────────────┘
         │
         ↓
┌─────────────────────────────────────────────────────────────┐
│              Gorbagana Blockchain (L1)                       │
│         (Smart Contracts, Escrow, Payouts)                   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Component Responsibilities

**Gorboy API (Azure-Hosted):**
- Game session management
- Proof-of-play validation
- Mining reward calculation and distribution
- Escrow verification and monitoring
- Reputation scoring and auditing
- Smart contract interaction
- Rate limiting and authentication

**Azure SQL Database:**
- Game metadata and configuration
- Player sessions and transactions
- Reputation scores and history
- Audit logs and compliance data
- Developer accounts and API keys

**Redis Cache:**
- Live game state (active sessions)
- Leaderboards and statistics
- Configuration hot-reload
- Rate limiting counters

**Blob Storage:**
- Proof-of-play logs
- Game replay data
- Transaction receipts
- Audit trail archives

**Gorbagana Blockchain:**
- Smart contracts (escrow, mining, payouts)
- Immutable transaction records
- Token transfers and balances
- On-chain verification

---

## 3. RESTful API Design

### 3.1 Why REST?

**Advantages:**
- ✅ **Language agnostic**: Works with any HTTP client
- ✅ **Simple and familiar**: Developers already know REST
- ✅ **Easy to test**: Postman, curl, browser
- ✅ **Scalable**: Standard HTTP load balancing
- ✅ **Cacheable**: HTTP caching built-in
- ✅ **Versionable**: URL-based versioning

**REST is NOT outdated** - it remains the standard for most APIs. Alternatives (GraphQL, gRPC) have specific use cases, but REST is the best default for broad compatibility.

### 3.2 API Structure

**Base URL:** `https://api.gorboy.com/v1/`

**Versioning:** URL-based (`/v1/`, `/v2/`, etc.)

**Authentication:** API Keys (developers) + JWT (player sessions)

**Content Type:** `application/json`

**HTTP Methods:**
- `GET`: Retrieve data (games, reputation, stats)
- `POST`: Create resources (games, sessions)
- `PUT`: Update resources (configuration, state)
- `DELETE`: Remove resources (sessions, locks)

### 3.3 API Endpoints (Overview)

**Game Management:**
```
POST   /v1/games                    # Create new game
GET    /v1/games/{game_id}          # Get game details
PUT    /v1/games/{game_id}          # Update game config
DELETE /v1/games/{game_id}          # Delete game
GET    /v1/games                    # List developer's games
```

**Session Management:**
```
POST   /v1/games/{game_id}/sessions           # Start player session
GET    /v1/games/{game_id}/sessions/{session_id}  # Get session details
PUT    /v1/games/{game_id}/sessions/{session_id}  # Update session state
DELETE /v1/games/{game_id}/sessions/{session_id}  # End session
```

**Mining:**
```
POST   /v1/games/{game_id}/sessions/{session_id}/mining/claim  # Claim mining reward
GET    /v1/games/{game_id}/mining/stats                        # Get mining statistics
POST   /v1/games/{game_id}/sessions/{session_id}/progress      # Report progress
```

**Escrow (HvP):**
```
GET    /v1/games/{game_id}/escrow              # Get escrow status
POST   /v1/games/{game_id}/escrow/lock         # Lock escrow (on win)
POST   /v1/games/{game_id}/escrow/unlock       # Unlock escrow
GET    /v1/games/{game_id}/escrow/verify       # Get verification code
```

**Reputation:**
```
GET    /v1/reputation/player/{wallet_address}     # Get player reputation
GET    /v1/reputation/developer/{developer_id}    # Get developer reputation
GET    /v1/reputation/house/{house_id}            # Get house reputation
```

**Verification:**
```
GET    /v1/verify/{code}                       # Verify escrow/game code
```

**Configuration:**
```
GET    /v1/config                              # Get current SDK config
```

### 3.4 Example API Calls

**Create Mining Game:**

```http
POST /v1/games
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json

{
  "gameName": "Space Shooter Pro",
  "gameType": "ProofOfPlay",
  "developerFee": 0.15,
  "progressMetric": {
    "type": "score",
    "minDelta": 100,
    "maxDelta": 2000
  }
}

Response 201 Created:
{
  "gameId": "abc123",
  "gameName": "Space Shooter Pro",
  "gameType": "ProofOfPlay",
  "status": "active",
  "createdAt": "2026-01-07T14:32:15Z",
  "apiEndpoint": "https://api.gorboy.com/v1/games/abc123"
}
```

**Start Player Session:**

```http
POST /v1/games/abc123/sessions
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json

{
  "playerWallet": "0x7a3f9b2c...",
  "walletSignature": "0x9f2e8a1b...",
  "clientInfo": {
    "platform": "web",
    "userAgent": "Mozilla/5.0..."
  }
}

Response 201 Created:
{
  "sessionId": "xyz789",
  "gameId": "abc123",
  "playerWallet": "0x7a3f9b2c...",
  "startedAt": "2026-01-07T14:35:00Z",
  "miningInterval": 30,
  "potentialUnits": 0,
  "actualUnits": 0
}
```

**Report Progress:**

```http
POST /v1/games/abc123/sessions/xyz789/progress
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json

{
  "currentScore": 1250,
  "timestamp": "2026-01-07T14:36:00Z",
  "proofOfPlay": {
    "inputCount": 47,
    "uniqueInputs": 12,
    "timingVariance": 127,
    "sessionGaps": [3.2, 1.8, 4.5]
  }
}

Response 200 OK:
{
  "sessionId": "xyz789",
  "progressAccepted": true,
  "currentScore": 1250,
  "potentialUnits": 2,
  "actualUnits": 0,
  "nextClaimAvailable": "2026-01-07T14:36:30Z"
}
```

**Claim Mining Reward:**

```http
POST /v1/games/abc123/sessions/xyz789/mining/claim
Authorization: Bearer YOUR_API_KEY
Content-Type: application/json

{
  "unitsToAward": 2
}

Response 200 OK:
{
  "sessionId": "xyz789",
  "unitsAwarded": 2,
  "playerReward": 1.4,
  "developerReward": 0.3,
  "transactionHash": "0x3c5d7e9a...",
  "playerWallet": "0x7a3f9b2c...",
  "developerWallet": "0x5f8g9h3k...",
  "timestamp": "2026-01-07T14:37:00Z"
}
```

---

## 4. Authentication and Authorization

### 4.1 Developer Authentication (API Keys)

**Purpose:** Authenticate game developers making server-to-server API calls.

**How It Works:**

1. Developer signs up on Gorboy platform
2. Developer receives API key
3. Developer includes API key in all requests

**Request Header:**
```http
Authorization: Bearer YOUR_API_KEY
```

**API Key Format:**
```
gorboy_live_abc123def456ghi789...  (production)
gorboy_test_xyz789uvw456rst123...  (testnet)
```

**Security:**
- API keys are long, random, and cryptographically secure
- Keys are hashed and salted in database
- Keys can be rotated without downtime
- Keys are scoped to specific games (optional)

**Rate Limiting by Tier:**

| Tier | Reputation | Rate Limit | Burst |
|:-----|:-----------|:-----------|:------|
| **Sandbox** | N/A | 100 req/min | 200 |
| **Production (New)** | 50-69 | 500 req/min | 1,000 |
| **Established** | 70-89 | 2,000 req/min | 4,000 |
| **Verified** | 90-100 | 10,000 req/min | 20,000 |

### 4.2 Player Authentication (Wallet Signatures)

**Purpose:** Verify player identity without passwords or accounts.

**How It Works:**

1. Player connects wallet (Backpack, Phantom)
2. Game requests signature from wallet
3. Wallet signs challenge message
4. Game sends wallet address + signature to API
5. API verifies signature cryptographically
6. Player authenticated by wallet address

**Challenge Message Format:**
```
Sign this message to authenticate with Gorboy:

Game: Space Shooter Pro
Session: xyz789
Timestamp: 2026-01-07T14:35:00Z
Nonce: a1b2c3d4e5f6

This request will not cost any gas fees.
```

**API Verification:**

```http
POST /v1/games/abc123/sessions
{
  "playerWallet": "0x7a3f9b2c...",
  "walletSignature": "0x9f2e8a1b...",
  "challenge": "Sign this message to authenticate..."
}

API verifies:
1. Signature matches wallet address
2. Challenge includes correct game ID and timestamp
3. Timestamp is recent (< 5 minutes old)
4. Nonce hasn't been used before (replay protection)
```

**Benefits:**
- ✅ No passwords, no email, no KYC
- ✅ Anonymous by default
- ✅ Wallet = identity = reputation = payment
- ✅ Very web3-native

### 4.3 Session Tokens (JWT - Optional)

**Purpose:** Maintain session state without requiring wallet signature on every request.

**How It Works:**

1. Player authenticates with wallet signature (once)
2. API issues short-lived JWT token
3. Game includes JWT in subsequent requests
4. JWT expires after 1 hour (configurable)

**JWT Payload:**
```json
{
  "sub": "0x7a3f9b2c...",  // Player wallet address
  "gameId": "abc123",
  "sessionId": "xyz789",
  "iat": 1704672000,        // Issued at
  "exp": 1704675600         // Expires (1 hour later)
}
```

**Request Header:**
```http
Authorization: Bearer JWT_TOKEN
X-API-Key: YOUR_API_KEY
```

**When to Use:**
- ✅ High-frequency requests (progress updates, state sync)
- ✅ Reduces wallet interaction (better UX)
- ✅ Temporary, expires quickly (security)

**When NOT to Use:**
- ❌ Critical operations (escrow lock, payouts) - require fresh wallet signature
- ❌ Long-lived sessions - use wallet signature instead

---

## 5. Database Architecture

### 5.1 Azure SQL Database (Core Data)

**Why SQL?**

Despite hesitation, SQL is the right choice for Gorboy because:
- ✅ **ACID guarantees**: Critical for money (escrow, payouts)
- ✅ **Relationships**: Games → Sessions → Players → Reputation
- ✅ **Auditing**: Immutable transaction logs with foreign keys
- ✅ **Compliance**: Easy to generate reports for legal/regulatory
- ✅ **Consistency**: No "eventual consistency" issues

**Schema Overview:**

**Games Table:**
```sql
CREATE TABLE Games (
  game_id VARCHAR(36) PRIMARY KEY,
  developer_id VARCHAR(36) NOT NULL,
  game_name VARCHAR(255) NOT NULL,
  game_type ENUM('ProofOfPlay', 'HouseVsPlayer', 'PlayerVsPlayer'),
  status ENUM('active', 'suspended', 'banned'),
  developer_fee DECIMAL(5,4),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (developer_id) REFERENCES Developers(developer_id)
);
```

**Sessions Table:**
```sql
CREATE TABLE Sessions (
  session_id VARCHAR(36) PRIMARY KEY,
  game_id VARCHAR(36) NOT NULL,
  player_wallet VARCHAR(66) NOT NULL,
  started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  ended_at TIMESTAMP NULL,
  potential_units INT DEFAULT 0,
  actual_units INT DEFAULT 0,
  player_reward DECIMAL(18,8) DEFAULT 0,
  developer_reward DECIMAL(18,8) DEFAULT 0,
  status ENUM('active', 'completed', 'invalidated'),
  FOREIGN KEY (game_id) REFERENCES Games(game_id)
);
```

**Transactions Table:**
```sql
CREATE TABLE Transactions (
  transaction_id VARCHAR(36) PRIMARY KEY,
  session_id VARCHAR(36) NOT NULL,
  transaction_type ENUM('mining_reward', 'escrow_lock', 'payout', 'refund'),
  amount DECIMAL(18,8) NOT NULL,
  recipient_wallet VARCHAR(66) NOT NULL,
  blockchain_tx_hash VARCHAR(66) NOT NULL,
  timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (session_id) REFERENCES Sessions(session_id)
);
```

**Reputation Table:**
```sql
CREATE TABLE Reputation (
  wallet_address VARCHAR(66) PRIMARY KEY,
  reputation_type ENUM('player', 'developer', 'house'),
  score INT DEFAULT 75,
  tier ENUM('Platinum', 'Gold', 'Silver', 'Bronze', 'Tin', 'Restricted'),
  last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

**Audit Logs Table:**
```sql
CREATE TABLE AuditLogs (
  log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
  event_type VARCHAR(100) NOT NULL,
  entity_id VARCHAR(36) NOT NULL,
  entity_type VARCHAR(50) NOT NULL,
  details JSON,
  timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_timestamp (timestamp),
  INDEX idx_entity (entity_type, entity_id)
);
```

### 5.2 Redis Cache (Hot Data)

**Purpose:** Fast, in-memory storage for frequently accessed data.

**What Goes in Redis:**

**Active Sessions:**
```
Key: session:{session_id}
Value: {
  "gameId": "abc123",
  "playerWallet": "0x7a3f9b2c...",
  "startedAt": 1704672000,
  "potentialUnits": 2,
  "actualUnits": 0,
  "lastActivity": 1704672120
}
TTL: 1 hour (auto-expire inactive sessions)
```

**Rate Limiting Counters:**
```
Key: ratelimit:{api_key}:{minute}
Value: 47  (request count)
TTL: 60 seconds
```

**Configuration:**
```
Key: config:mining
Value: {
  "player_interval_seconds": 30,
  "developer_interval_seconds": 600,
  "concurrent_win_limit": 5,
  ...
}
TTL: None (manually invalidated on update)
```

**Leaderboards:**
```
Key: leaderboard:{game_id}:daily
Type: Sorted Set
Members: player_wallet → score
TTL: 24 hours
```

**Verification Codes:**
```
Key: verify:{code}
Value: {
  "gameId": "abc123",
  "escrowAmount": 1000000,
  "houseWallet": "0x3c5d7e9a...",
  "generatedAt": 1704672000
}
TTL: 30 seconds
```

### 5.3 Blob Storage (Logs and Replay)

**Purpose:** Cheap, scalable storage for large, infrequently accessed data.

**What Goes in Blob Storage:**

**Proof-of-Play Logs:**
```
Path: /proof-of-play/{game_id}/{session_id}.json
Content: {
  "sessionId": "xyz789",
  "inputs": [...],
  "timings": [...],
  "progressHistory": [...]
}
```

**Replay Data:**
```
Path: /replays/{game_id}/{session_id}.replay
Content: Binary replay file (game-specific format)
```

**Transaction Receipts:**
```
Path: /receipts/{transaction_id}.json
Content: {
  "transactionHash": "0x3c5d7e9a...",
  "blockNumber": 12345678,
  "gasUsed": 21000,
  ...
}
```

**Audit Trail Archives:**
```
Path: /audit-logs/{year}/{month}/{day}.jsonl
Content: Newline-delimited JSON logs (compressed)
```

---

## 6. Blockchain Integration

### 6.1 Gorbagana Node Connection

**Options:**

**Option A: Self-Hosted Node (Azure VM)**
- Run Gorbagana validator node on Azure VM
- Full control, no third-party dependency
- Higher cost, requires maintenance

**Option B: External Node Provider**
- Connect to third-party Gorbagana RPC endpoint
- Lower cost, easier maintenance
- Dependency on external service

**Recommendation:** Start with Option B (external provider), move to Option A if needed for performance/reliability.

### 6.2 Smart Contract Interaction

**Web3 Library:** Use Rust or JavaScript libraries to interact with Gorbagana smart contracts.

**Example (Rust):**
```rust
use solana_client::rpc_client::RpcClient;
use solana_sdk::{
    pubkey::Pubkey,
    signature::{Keypair, Signer},
    transaction::Transaction,
};

// Connect to Gorbagana node
let client = RpcClient::new("https://rpc.gorbagana.com");

// Mint mining reward
let mint_instruction = create_mint_instruction(
    player_wallet,
    amount,
    proof_of_play_hash,
);

let transaction = Transaction::new_signed_with_payer(
    &[mint_instruction],
    Some(&payer.pubkey()),
    &[&payer],
    recent_blockhash,
);

let signature = client.send_and_confirm_transaction(&transaction)?;
```

**API Abstracts Blockchain:**

Developers never interact with blockchain directly - API handles all smart contract calls.

---

## 7. Real-Time Events (Optional Enhancement)

### 7.1 WebSockets for Real-Time Updates

**When to Add WebSockets:**

While REST polling is sufficient for V1, WebSockets can enhance UX for real-time features:

**Use Cases:**
- ✅ Live escrow status updates
- ✅ Instant jackpot win notifications
- ✅ Real-time player count
- ✅ Exploit alerts / game lock events

**WebSocket Endpoint:**
```
wss://api.gorboy.com/v1/games/{game_id}/events
```

**Example:**
```javascript
const ws = new WebSocket('wss://api.gorboy.com/v1/games/abc123/events');

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  
  switch (data.type) {
    case 'jackpot_won':
      showJackpotAnimation(data);
      break;
    case 'escrow_locked':
      updateEscrowDisplay(data);
      break;
    case 'game_suspended':
      showSuspensionNotice(data);
      break;
  }
};
```

**Implementation:**

- **Azure SignalR Service** (managed WebSocket service)
- **Fallback to REST polling** if WebSocket unavailable
- **Automatic reconnection** on disconnect

**Priority:** V2 enhancement, not required for V1.

---

## 8. Client SDKs (Optional Wrappers)

### 8.1 Purpose

Client SDKs are **thin wrappers** around the REST API that make integration easier. They are **optional** - developers can call REST directly.

**Benefits:**
- ✅ Cleaner, more idiomatic code
- ✅ Type safety (TypeScript, C#)
- ✅ Automatic error handling
- ✅ Built-in retry logic
- ✅ Better developer experience

### 8.2 Supported Languages (Planned)

**Priority 1 (V1):**
- **JavaScript/TypeScript** (npm package)
- **Unity C#** (Unity Asset Store)

**Priority 2 (V2):**
- **Python** (pip package)
- **Rust** (crates.io)
- **Go** (go modules)

### 8.3 Example: JavaScript SDK

**Installation:**
```bash
npm install @gorboy/sdk
```

**Usage:**
```javascript
import { GorboySDK } from '@gorboy/sdk';

// Initialize SDK
const sdk = new GorboySDK({
  apiKey: 'gorboy_live_abc123...',
  environment: 'production', // or 'testnet'
});

// Create game
const game = await sdk.createGame({
  gameName: "Space Shooter Pro",
  gameType: "ProofOfPlay",
  developerFee: 0.15,
});

// Start player session
const session = await game.startSession({
  playerWallet: "0x7a3f9b2c...",
  walletSignature: "0x9f2e8a1b...",
});

// Report progress
await session.reportProgress({
  currentScore: 1250,
});

// Claim mining reward
const reward = await session.claimMiningReward({
  unitsToAward: 2,
});

console.log(`Player earned: ${reward.playerReward} GOR`);
console.log(`Developer earned: ${reward.developerReward} GOR`);
```

**Under the Hood:**

SDK just makes REST calls:
```javascript
class GorboySDK {
  async createGame(options) {
    const response = await fetch('https://api.gorboy.com/v1/games', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.apiKey}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(options),
    });
    return response.json();
  }
}
```

---

## 9. Scalability and Performance

### 9.1 Expected Load (Initial Estimates)

**Conservative Estimates (Year 1):**
- **Games**: 100-500 active games
- **Players**: 10,000-50,000 monthly active
- **Sessions**: 100,000-500,000 per month
- **API Requests**: 10M-50M per month (~4-20 req/sec avg)

**Optimistic Estimates (Year 2):**
- **Games**: 1,000-5,000 active games
- **Players**: 100,000-500,000 monthly active
- **Sessions**: 1M-5M per month
- **API Requests**: 100M-500M per month (~40-200 req/sec avg)

### 9.2 Scaling Strategy

**Phase 1: Serverless (Azure Functions)**
- **Capacity**: Up to ~200 req/sec
- **Cost**: Pay per request (~$0.20 per 1M requests)
- **Scaling**: Automatic, instant
- **Best for**: Variable traffic, low initial volume

**Phase 2: App Service (Dedicated Instances)**
- **Capacity**: 1,000+ req/sec per instance
- **Cost**: Fixed monthly (~$100-500/month per instance)
- **Scaling**: Horizontal (add more instances)
- **Best for**: Consistent high traffic

**Phase 3: Multi-Region (Global)**
- **Capacity**: 10,000+ req/sec globally
- **Cost**: Higher (multiple regions)
- **Scaling**: Geographic distribution
- **Best for**: Global player base, low latency

### 9.3 Caching Strategy

**Cache Aggressively:**

| Data Type | Cache Location | TTL | Invalidation |
|:----------|:---------------|:----|:-------------|
| **Configuration** | Redis | None | Manual on update |
| **Reputation Scores** | Redis | 5 minutes | On reputation change |
| **Active Sessions** | Redis | 1 hour | On session end |
| **Leaderboards** | Redis | 1 minute | On score update |
| **Verification Codes** | Redis | 30 seconds | Auto-expire |
| **Game Metadata** | Redis | 1 hour | On game update |

**Cache Invalidation:**

- **Write-through**: Update database and cache simultaneously
- **TTL-based**: Auto-expire after time limit
- **Event-based**: Invalidate on specific events (game update, reputation change)

### 9.4 Database Optimization

**Indexing:**
```sql
-- Critical indexes for performance
CREATE INDEX idx_sessions_player ON Sessions(player_wallet);
CREATE INDEX idx_sessions_game ON Sessions(game_id);
CREATE INDEX idx_transactions_session ON Transactions(session_id);
CREATE INDEX idx_reputation_wallet ON Reputation(wallet_address);
CREATE INDEX idx_audit_timestamp ON AuditLogs(timestamp);
```

**Read Replicas:**
- Use Azure SQL read replicas for analytics queries
- Write to primary, read from replicas
- Reduces load on primary database

**Partitioning:**
- Partition large tables by date (e.g., AuditLogs by month)
- Improves query performance on historical data

---

## 10. Monitoring and Observability

### 10.1 Azure Application Insights

**Metrics to Track:**
- **API Latency**: p50, p95, p99 response times
- **Error Rate**: 4xx, 5xx errors per minute
- **Request Volume**: Requests per second by endpoint
- **Database Performance**: Query execution times
- **Cache Hit Rate**: Redis cache effectiveness
- **Blockchain Latency**: Smart contract call times

**Alerts:**
- API latency > 1 second (p95)
- Error rate > 5%
- Database CPU > 80%
- Cache hit rate < 70%
- Blockchain node unreachable

### 10.2 Logging

**Structured Logging (JSON):**
```json
{
  "timestamp": "2026-01-07T14:32:15Z",
  "level": "INFO",
  "service": "gorboy-api",
  "endpoint": "/v1/games/abc123/sessions",
  "method": "POST",
  "apiKey": "gorboy_live_***",
  "playerWallet": "0x7a3f***",
  "duration_ms": 123,
  "status": 201
}
```

**Log Levels:**
- **DEBUG**: Detailed debugging info (disabled in production)
- **INFO**: Normal operations (session start, mining claim)
- **WARN**: Unusual but handled (rate limit hit, cache miss)
- **ERROR**: Errors that need attention (database timeout, blockchain failure)
- **CRITICAL**: System-level failures (service down, data corruption)

### 10.3 Distributed Tracing

**Azure Application Insights** provides distributed tracing:

```
Request: POST /v1/games/abc123/sessions/xyz789/mining/claim
├─ Validate API key (5ms)
├─ Query session from Redis (12ms)
├─ Validate proof-of-play (23ms)
├─ Calculate mining reward (8ms)
├─ Call smart contract (450ms) ⚠️ Slow
├─ Update database (34ms)
├─ Update cache (7ms)
└─ Return response (2ms)
Total: 541ms
```

Helps identify bottlenecks (in this case, blockchain call).

---

## 11. API Documentation

### 11.1 OpenAPI/Swagger

**Auto-Generated Documentation:**

API specification defined in OpenAPI 3.0 format:

```yaml
openapi: 3.0.0
info:
  title: Gorboy SDK API
  version: 1.0.0
  description: RESTful API for blockchain gaming integration

servers:
  - url: https://api.gorboy.com/v1
    description: Production
  - url: https://api-testnet.gorboy.com/v1
    description: Testnet

paths:
  /games:
    post:
      summary: Create a new game
      security:
        - ApiKeyAuth: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateGameRequest'
      responses:
        '201':
          description: Game created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Game'
```

**Interactive Documentation:**

Hosted at `https://docs.gorboy.com/api` with:
- ✅ Interactive "Try It Out" feature
- ✅ Code examples in multiple languages
- ✅ Authentication testing
- ✅ Response schema validation

### 11.2 Developer Portal

**Comprehensive Documentation:**

- **Getting Started Guide**: 5-minute quickstart
- **API Reference**: Complete endpoint documentation
- **Tutorials**: Step-by-step integration guides
- **Code Examples**: JavaScript, Unity, Python
- **Best Practices**: Performance, security, UX
- **Troubleshooting**: Common errors and solutions
- **Changelog**: API version history

**Developer Dashboard:**

- View API keys
- Monitor usage and rate limits
- View game analytics
- Access support tickets
- Download SDKs

---

## 12. Testing and Debugging

### 12.1 Testnet Environment

**Separate Testnet API:**
```
Production: https://api.gorboy.com/v1
Testnet:    https://api-testnet.gorboy.com/v1
```

**Testnet Features:**
- Uses $gGOR tokens (no real value)
- Unlimited API calls (no rate limiting)
- Faster blockchain confirmations
- Reset-able state (for testing)

**API Key Prefix:**
```
Production: gorboy_live_...
Testnet:    gorboy_test_...
```

### 12.2 Mock Mode (Local Development)

**SDK Mock Mode:**

```javascript
const sdk = new GorboySDK({
  apiKey: 'gorboy_test_...',
  environment: 'mock', // No API calls, simulated responses
});

// All calls return mock data instantly
const game = await sdk.createGame({...}); // Instant, no network
```

**Benefits:**
- ✅ Develop offline
- ✅ Instant responses (no latency)
- ✅ Predictable behavior (no blockchain variability)
- ✅ No API key required

### 12.3 Debug Logging

**Enable Verbose Logging:**

```javascript
const sdk = new GorboySDK({
  apiKey: 'gorboy_test_...',
  debug: true, // Logs all API calls and responses
});

// Console output:
// [GorboySDK] POST https://api-testnet.gorboy.com/v1/games
// [GorboySDK] Request: {"gameName": "Space Shooter Pro", ...}
// [GorboySDK] Response (201): {"gameId": "abc123", ...}
```

### 12.4 Error Handling

**Standardized Error Responses:**

```json
{
  "error": {
    "code": "INSUFFICIENT_ESCROW",
    "message": "House does not have sufficient escrow to cover maximum payout",
    "details": {
      "required": 1000000,
      "available": 750000,
      "shortfall": 250000
    },
    "timestamp": "2026-01-07T14:32:15Z",
    "requestId": "req_abc123"
  }
}
```

**Error Codes:**

| Code | HTTP Status | Meaning |
|:-----|:------------|:--------|
| `INVALID_API_KEY` | 401 | API key missing or invalid |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `GAME_NOT_FOUND` | 404 | Game ID doesn't exist |
| `SESSION_EXPIRED` | 410 | Session ended or expired |
| `INSUFFICIENT_ESCROW` | 400 | House underfunded |
| `PROOF_OF_PLAY_FAILED` | 400 | Invalid proof-of-play |
| `BLOCKCHAIN_ERROR` | 503 | Blockchain node unreachable |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## 13. Summary

Section 5 defines a modern, scalable, and developer-friendly tech stack:

**Architecture:**
- ✅ **RESTful API** (language-agnostic, Azure-hosted)
- ✅ **Azure Functions** (serverless, scales automatically)
- ✅ **Azure SQL** (critical data with ACID guarantees)
- ✅ **Redis Cache** (hot data, sub-millisecond latency)
- ✅ **Blob Storage** (logs, replays, archives)
- ✅ **Gorbagana Blockchain** (smart contracts, immutable records)

**Authentication:**
- ✅ **API Keys** for developers (tiered rate limiting)
- ✅ **Wallet Signatures** for players (web3-native, anonymous)
- ✅ **JWT** for sessions (optional, reduces wallet interaction)

**Scalability:**
- ✅ **Serverless → Dedicated** scaling path
- ✅ **Aggressive caching** (Redis, HTTP)
- ✅ **Database optimization** (indexes, read replicas)
- ✅ **Multi-region** ready (future)

**Developer Experience:**
- ✅ **OpenAPI/Swagger** docs (interactive)
- ✅ **Thin client SDKs** (JavaScript, Unity, Python)
- ✅ **Testnet environment** ($gGOR, unlimited calls)
- ✅ **Mock mode** (offline development)
- ✅ **Comprehensive tutorials** and examples

**Monitoring:**
- ✅ **Application Insights** (metrics, alerts)
- ✅ **Structured logging** (JSON, searchable)
- ✅ **Distributed tracing** (bottleneck identification)

This architecture is production-ready, scales from 0 to millions of users, and provides an excellent developer experience while maintaining security and performance.

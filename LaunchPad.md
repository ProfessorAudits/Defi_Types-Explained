# Crypto LaunchPad
A crypto launchpad is a crowdfunding platform that helps early-stage blockchain projects 
raise capital by selling tokens to retail investors before public market exchange listings.

## LaunchPad Types/Common Models 
1. Fixed Price IDO:
     - Users deposit USDT/ETH
     - Tokens distributed after sale
2. Fair Launch:
     - No cap, everyone gets proportional allocation
3. Tier-Based Launchpad:
     - Users staked tokens → tiers → allocations
4. Whitelist-Based IDO:
     - Only approved addresses participate
  
## Phase 2: Core Smart Contract Architecture
1️⃣ TokenSale (Main Contract)

Responsibilities:

- Accept user funds
- Track contributions
- Enforce caps & timing
- Handle refunds / claims

### Key Variables:

```solidity
address saleToken;
address paymentToken; // ETH or ERC20
uint256 tokenPrice;
uint256 hardCap;
uint256 softCap;
uint256 startTime;
uint256 endTime;
```

2️⃣ Vesting Contract (Critical)
Most launches do not release 100% tokens immediately.

Features:

- Cliff
- Linear vesting
- Claim function

3️⃣ Whitelist / Allocation Manager

Options:

- Merkle tree (best)
- Mapping-based (simple but gas heavy)

Merkle whitelist:

- Off-chain whitelist generation
- On-chain proof verification

4️⃣ Admin / Factory Contract (Optional but Pro)

Used to:

- Create multiple IDOs
- Reduce deployment cost
- Make platform scalable

Phase 3: Sale Logic (Core Implementation)
Contribution Logic

Check:
- Sale active
- User allocation limit
- Hard cap not exceeded
- require(block.timestamp >= startTime && block.timestamp <= endTime);
- require(totalRaised + amount <= hardCap);

Finalization Logic:

After sale ends:
If soft cap reached → enable claims
Else → enable refunds

Phase 4: Token Distribution

Claim Flow
- User calls claim()

Contract checks:
- Sale successful
- Vesting unlocked
- Not already claimed
- Transfers tokens

⚠️ Never auto-distribute (gas griefing risk)

## 🧩 System Overview (Mental Model)

Actors:

- Investor (User)
- Project Team
- Launchpad Admin
- Smart Contracts

Core contracts:

- Staking / Tier Contract
- IDO Factory
- IDO Sale Contract (per IDO)
- Vesting Contract (per IDO or shared)
- Treasury

## 👤 INVESTOR (USER) FLOW — STEP BY STEP

NOTE IMP :: 

```solidity
Users stake to earn the right to participate and to get a guaranteed allocation in IDOs.
```

🔹 Step 0: Wallet Connection
- User connects wallet (MetaMask)

Frontend reads:
- Stake amount
- Tier level
- Active IDOs

📌 On-chain: No tx yet

🔹 Step 1: Stake Tokens → Get Tier

User stakes Launchpad Token (LPD).

On-chain flow:

stake(amount)
→ tokens transferred to staking contract
→ userStake[msg.sender] updated
→ tier recalculated

Tier logic example:
Tier	  Stake Required	Allocation
Tier 1	 1,000 LPD	        $100
Tier 2	 5,000 LPD	        $500
Tier 3	 20,000 LPD	      $2,000

⚠️ Important:

Tier should snapshot at whitelist time
Prevent last-minute stake abuse

🔹 Step 2: IDO Announcement (Read-Only)

User sees:

- Token info
- Sale price
- Vesting schedule
- Tier allocations
- Timeline

📌 No tx

🔹 Step 3: Whitelist Snapshot (Critical)

At snapshotTime:

Contract records:
userTier[msg.sender] → allocation

This prevents:
❌ stake → buy → unstake instantly

🔹 Step 4: Buy Tokens (IDO Participation)

User contributes funds.

On-chain checks:
```solidity
require(block.timestamp >= startTime)
require(block.timestamp <= endTime)
require(userContribution + amount <= tierAllocation)
```

Flow:
contribute(amount)
→ payment token transferred in
→ userContribution updated
→ totalRaised updated

📌 Only one or multiple buys, but capped by tier allocation.

🔹 Step 5: Sale Ends

Two outcomes:

✅ Soft cap reached
- Sale finalized
- Claim enabled

❌ Soft cap NOT reached
- Refund enabled

User does nothing yet.

🔹 Step 6A: Claim Tokens (Successful Sale)

User claims vested tokens.

claim()
→ calculate unlocked amount
→ transfer tokens
→ update claimed

Vesting example:

20% TGE
80% linear over 6 months

🔹 Step 6B: Refund (Failed Sale)
refund()
→ payment token returned
→ contribution reset

🔹 Step 7: Unstake Tokens

After lock period:

unstake()
→ return LPD tokens

⚠️ Lock recommended until IDO ends.


## 🏗️ PROJECT / ADMIN FLOW (MULTIPLE IDOS)
🔸 Step 1: Create New IDO (Factory)

Admin or project calls factory.

createIDO(params)
→ deploy new IDO contract
→ register IDO

Params:

- Sale token
- Price
- Hard/Soft cap
- Start/end
- Vesting config
- Tier allocation map

🔸 Step 2: Fund IDO Contract

Project deposits tokens:

depositSaleTokens()
→ tokens locked

⚠️ Prevent underfunded sales.

🔸 Step 3: Whitelist Snapshot Trigger

Admin calls:
snapshotTiers()

This:
Locks tiers
Prevents manipulation

🔸 Step 4: Sale Live
Users participate.

Admin:

Cannot change price
Cannot withdraw funds yet

🔸 Step 5: Finalize Sale
After end time:

finalize()

If success:

Funds → Treasury
Claim enabled

If fail:

Refund enabled
🔸 Step 6: Fund Withdrawal

Admin withdraws raised funds:

withdrawFunds()

⚠️ Use multisig

🔸 Step 7: Post-IDO Ops
Vesting continues
Analytics
Token listing

5️⃣ What Happens If There Is NO Staking?

Let’s be blunt:

- No staking	Result
- Open sale	Whales win
- First come first serve	Gas wars
- Whitelist only	Sybil attacks
- Lottery	User frustration

That’s why serious launchpads always use staking.

6️⃣ Why Locking Is Important

If staking wasn’t locked:

Stake → buy → unstake → repeat
Same capital abused multiple times

So:

Lock during IDO
Snapshot tiers


# What Actually Happens (Step by Step)

1️⃣ Project Issues Tokens
- Project creates its token (ERC20)
- Total supply is fixed or capped
- A portion is reserved for the IDO

Example:

- Total supply: 1,000,000,000
- IDO allocation: 50,000,000 (5%)

2️⃣ Project Deposits Tokens into Launchpad
Before the sale starts:

Project → deposits IDO tokens → IDO smart contract

This is critical:

- Prevents rug pulls
- Guarantees token availability
- Auditors always check this

3️⃣ Investors Provide Money

Investors pay:
- ETH
- USDT / USDC
- Chain-native token
- Investor → pays money → IDO contract

4️⃣ Exchange Happens at Fixed Price

Price is pre-defined:

Example:
1 IDO token = $0.02
User invests $200 → gets 10,000 tokens

No AMM pricing. No slippage.

5️⃣ Funds Go to Project (If Successful)

After sale ends and soft cap is reached:

IDO contract → sends raised funds → project treasury

Usually via:

- Multisig
- DAO treasury

6️⃣ Tokens Go to Investors (With Vesting)

Investors don’t always get all tokens instantly.

Example vesting:
20% at TGE
80% over 6 months

Tokens are:
- Claimed gradually
- Never pushed automatically

What If Sale Fails?
If soft cap not met:

Investors → claim refunds
Project → gets no funds

Tokens stay locked or returned.

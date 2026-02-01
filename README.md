# 🗳️ MyGovernor DAO (OpenZeppelin Governor + Timelock)

A minimal on-chain governance system built with **OpenZeppelin Governor** + **TimelockController**, tested with **Foundry**.

This repo shows the full governance lifecycle:
✅ **Propose** → ✅ **Vote** → ✅ **Queue** (timelock delay) → ✅ **Execute**  
…where the DAO updates a simple `Box` contract, and **no one can update it directly**.

---

## ✨ What’s inside

### Contracts
- **`GovToken.sol`**
  - ERC20 governance token with **Votes** + **Permit** (OZ `ERC20Votes`)
  - Token holders **delegate** voting power to enable voting
- **`MyGovernor.sol`**
  - OpenZeppelin **Governor** implementation:
    - `GovernorSettings` (voting delay, voting period, proposal threshold)
    - `GovernorCountingSimple` (For/Against/Abstain)
    - `GovernorVotes` (uses `GovToken` voting power)
    - `GovernorVotesQuorumFraction` (quorum fraction)
    - `GovernorTimelockControl` (execution gated by timelock)
- **`TimeLock.sol`**
  - Wrapper around `TimelockController`
  - Enforces a **minimum delay** between queuing and execution
- **`Box.sol`**
  - Simple storage contract (`store(uint256)`)
  - **Ownable**, and ownership is transferred to the **Timelock**
  - This means only governance (through the timelock) can call `store()`

---

## 🧠 Governance design

### Ownership + Roles
- `Box` is owned by the **Timelock**
- `MyGovernor` is granted the **PROPOSER_ROLE** on the Timelock
- **Anyone** can execute queued proposals because `EXECUTOR_ROLE` is granted to `address(0)` (open execution)
- The deployer/test contract renounces admin power by revoking `DEFAULT_ADMIN_ROLE`

In short:
- **Timelock owns Box**
- **Governor controls Timelock**
- ✅ `Box.store()` can only happen after a successful vote + delay

---

## ⚙️ Key parameters

In `MyGovernor.sol`:
- **Voting Delay:** `7200`  
- **Voting Period:** `50400`  
- **Quorum:** `4%` (via `GovernorVotesQuorumFraction(4)`)
- **Proposal Threshold:** `0` (anyone with voting power can propose)

In tests:
- **Min Timelock Delay:** `3600` seconds (1 hour)

> Note: comments in the test mention “1 block / 1 week”, but the code uses raw numbers. In local testing we simulate passing time/blocks using `vm.warp` and `vm.roll`.

---

## ✅ Test flow (end-to-end)

The main test `testGovernanceUpdatesBox()` demonstrates the full pipeline:

1. **Setup**
   - Mint `GovToken` to `USER`
   - `USER` delegates to themselves (activates voting power)
   - Deploy `TimeLock` + `MyGovernor`
   - Grant governor proposer role, open executor role, revoke admin role
   - Deploy `Box`, transfer ownership to timelock

2. **Propose**
   - Create calldata for `Box.store(777)`
   - Call `governor.propose(targets, values, calldatas, description)`

3. **Vote**
   - Move forward past voting delay using:
     - `vm.warp(...)` and `vm.roll(...)`
   - `USER` votes **For** using:
     - `governor.castVoteWithReason(proposalId, 1, reason)`

4. **Queue**
   - After voting period ends, queue the operation through the timelock:
     - `governor.queue(..., descriptionHash)`
   - Wait `MIN_DELAY`

5. **Execute**
   - Execute the queued proposal:
     - `governor.execute(..., descriptionHash)`
   - Assert the box value is updated

Also included:
- `testCantUpdateBoxWihtoutGovernance()` verifies direct calls to `box.store()` revert.

---

## 🧪 Running tests (Foundry)

```bash
forge test -vv
Optional extras:

forge test --match-test testGovernanceUpdatesBox -vvvv
forge coverage
🔍 Repo structure (typical)
src/
  Box.sol
  GovToken.sol
  MyGovernor.sol
  TimeLock.sol

test/
  MyGovernorTest.t.sol
🧷 Notes / gotchas
Delegation is required: minting tokens alone does not give voting power.

The Timelock enforces delay: even after a proposal passes, it must be queued and wait MIN_DELAY.

Setting EXECUTOR_ROLE to address(0) means anyone can execute queued proposals (common pattern for decentralization).

🛣️ Next upgrades (easy wins)
Add proposal threshold > 0 to prevent spam

Add multiple voters + quorum tests

Add cancel flow test

Add events + indexing for proposal lifecycle

Add a deployment script to deploy Governor/Timelock/Token/Box on a testnet

📜 License
MIT
# DAO

## Super DCA Liquid Network
[Contest Details] (https://audits.sherlock.xyz/contests/1171/report)


### [High-01] Using trade.startTime as epoch anchor leads to incorrect completed-epoch rewards

## Summary
SuperDCACashback anchors epoch calculations to each trade's trade.startTime instead of the campaign epoch anchor cashbackClaim.startTime. As a result, trades started before the campaign can be (incorrectly) treated as having completed campaign epochs and receive claimable cashback earlier than intended.

## Root Cause
https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-cashback/src/SuperDCACashback.sol#L160

`SuperDCACashback::_calculateEpochData` computes epochs using `block.timestamp - trade.startTime` and ignores `cashbackClaim.startTime`. Downstream functions `(SuperDCACashback::_calculateCompletedEpochsCashback, SuperDCACashback::_calculatePendingCashback, SuperDCACashback::getTradeStatus, SuperDCACashback::claimAllCashback)` consume those incorrect epoch counts and calculate rewards using a per-trade epoch anchor rather than the global campaign epoch anchor.

## Internal Pre-conditions
- cashbackClaim.startTime is set to a campaign anchor (T0).
- cashbackClaim.duration > 0.
- The trade record (ISuperDCATrade.Trade) has trade.startTime earlier than cashbackClaim.startTime (trade started before campaign).
- The trade still has non-zero flow rate meeting minRate.

## External Pre-conditions
- A user (or attacker) can start a trade before the campaign startTime (this is normal behaviour in many deployments).
- The user is the owner of that trade (so they can call SuperDCACashback::claimAllCashback).
- The attacker can wait until after the campaign starts + a partial epoch, then call the cashback contract.

## Attack Path
- Deploy campaign with cashbackClaim.startTime = T0 and epoch duration = D.
- Attacker starts a trade with trade.startTime = T0 - x (x > 0 but < D).
- Wait until time T0 + y where y < D (so fewer than a full campaign epoch has passed from the campaign anchor).
- Call SuperDCACashback::getTradeStatus or SuperDCACashback::claimAllCashback as the trade owner.
- Contract uses trade-anchored elapsed time -> computes at least one completed epoch -> returns claimable > 0 and allows claim.
- Attacker receives cashback that should not be claimable under campaign-anchored rules.

## Impact
- Incorrectly credited completed epochs allow users to claim cashback for epochs that are not fully completed relative to the campaign epoch grid.
- Overpayments against the campaign budget; inconsistent forfeiture semantics (trade may appear to have completed epochs that should be forfeited).
- Funds can be drained faster than intended; accounting/analytics will be incorrect.

## PoC
- Place the test in the SuperDCACashback.t.sol file
- Place the test in the GetTradeStatus contract section.

```javascript
/// @notice Demonstrates the epoch-anchoring vulnerability where epochs are  
  /// computed relative to the trade's startTime instead of the campaign's  
  /// `cashbackClaim.startTime`. A trade that begins slightly before the  
  /// campaign start can be (incorrectly) credited with a completed epoch  
  /// earlier than intended.  
  ///  
  /// Scenario:  
  /// - Campaign startTime = T0, duration = D  
  /// - Trade starts at T0 - 50 (before campaign)  
  /// - Current time = T0 + 75  
  ///  
  /// Expected (correct) completed epochs anchored to campaign start: 0  
  /// Observed (current contract behavior anchored to trade.startTime): 1  
  function test_EpochAnchoring_Vulnerability_TradeStartedBeforeCampaignGetsCredit()  
    public  
  {  
    uint256 epochStartTime = block.timestamp + 1000;  
    uint256 duration = 100;  
    int96 flowRate = int96(1e15);  
    uint256 maxRate = uint256(int256(flowRate)) + 1000;  
  
    SuperDCACashback testContract =  
      _deployContractWithClaim(100, duration, flowRate, maxRate, epochStartTime);  
  
    // Start the trade before the campaign start  
    vm.warp(epochStartTime - 50);  
    uint256 tradeId = _createTrade(trader, flowRate, 100, 1000);  
  
    // Move to a time after campaign start but before a full epoch has elapsed  
    // when measured from the campaign start (75s into first campaign epoch)  
    vm.warp(epochStartTime + 75);  
  
    (uint256 claimable,,) = testContract.getTradeStatus(tradeId);  
  
    // Compute expected values for clarity (what correct logic anchored to  
    // campaign start would yield)  
    // From campaign anchor: tradeActiveStart = max(trade.startTime, epochStartTime)  
    // tradeActiveEnd = block.timestamp -> elapsed = 75 -> 0 completed epochs  
    uint256 expectedByCampaign = 0;  
  
    // By current contract logic (anchored to trade.startTime): elapsed = 125 -> 1 completed epoch  
    uint256 elapsedByTradeAnchor = block.timestamp - (epochStartTime - 50); // 125  
    uint256 completedEpochsByTradeAnchor = elapsedByTradeAnchor / duration; // 1  
    uint256 effectiveFlow = uint256(int256(flowRate));  
    uint256 completedAmount = effectiveFlow * duration * completedEpochsByTradeAnchor;  
    uint256 expectedByTradeAnchor = (completedAmount * 100) / 10_000 / 1e12;  
  
    // The contract currently reports a claimable amount (>0) even though the  
    // correct anchoring to campaign start would give 0. This demonstrates the  
    // vulnerability and a mismatch between spec and implementation.  
    assertGt(claimable, 0);  
    assertEq(claimable, expectedByTradeAnchor);  
    assertEq(expectedByCampaign, 0);  
  }  

```

## Mitigation
Anchor epoch math to cashbackClaim.startTime (the campaign global epoch anchor), and compute eligible epochs as the intersection between the campaign epoch intervals and the trade's active interval. Replace the per-trade elapsed-time logic in SuperDCACashback::_calculateEpochData with campaign-anchored logic.


### [High-02] [Overwriting per-token lastRewardIndex] leads to loss of pending token rewards

## Summary
Calling `SuperDCAStaking::stake` or `SuperDCAStaking::unstake` overwrites a token bucket's lastRewardIndex with the global rewardIndex. If the gauge has not yet called `SuperDCAStaking::accrueReward` for that token, the previously-accrued per-token delta (and therefore pending rewards) is silently discarded, and as a result, expected minting/donations never occur for that time window.

## Root Cause
https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-gauge/src/SuperDCAStaking.sol#L204
https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-gauge/src/SuperDCAStaking.sol#L237

`SuperDCAStaking::stake` and `SuperDCAStaking::unstake` set tokenRewardInfo.lastRewardIndex = rewardIndex when updating a token's stakedAmount. This wipes the outstanding delta (rewardIndex - old lastRewardIndex) that determines how many tokens should be minted/donated for that token when the gauge later calls accrueReward.

## Internal Pre-conditions
- The staking contract has non-zero mintRate and rewardIndex has advanced since the token's lastRewardIndex.
- tokenRewardInfo.stakedAmount > 0 (token has stakers), so there is a non-zero pending amount for that token.
- No intervening accrueReward(token) call has settled the pending amount yet.

## External Pre-conditions
- A malicious or benign external actor can call SuperDCAStaking::stake or SuperDCAStaking::unstake for the affected token (these are public functions; caller is msg.sender).
- The gauge is expected to call SuperDCAStaking::accrueReward(token) later (but it hasn't yet).

## Attack Path
- users stake Token T and rewardIndex advances (via time/mintRate) but SuperDCAStaking::accrueReward(T) has not yet been called.
- Attacker (arbitrary msg.sender) calls SuperDCAStaking::stake(T, smallAmount) or ::unstake(T, someAmount).
- The call updates tokenRewardInfo.lastRewardIndex = rewardIndex, removing the pending delta.
- Later, when the gauge calls SuperDCAStaking::accrueReward(T), it computes zero and returns zero; the previously pending tokens are lost.

## Impact
- Value loss / withheld rewards: pending rewards for the token are never returned to developer/community because the delta is erased before accrueReward runs.
- Violates accounting invariants: per-token accrual isolation and "no accrual leakage on stake/unstake" invariants are broken.
- High severity: direct economic loss and manipulation are possible.

## PoC
- Place the test in the SuperDCACashback.t.sol file
- Place the test in the GetTradeStatus contract section.

```javascript
// SPDX-License-Identifier: Apache-2.0  
pragma solidity ^0.8.22;  
  
import "forge-std/Test.sol";  
import {SuperDCAStakingTest} from "../SuperDCAStaking.t.sol";  
  
contract StakeOverwrite is SuperDCAStakingTest {  
    /// @notice Demonstrates that calling `stake` on a token can overwrite the  
    /// token's `lastRewardIndex` and thus wipe previously-accrued pending rewards  
    /// that were not yet settled via `accrueReward`.  
    function test_StakeWipesPendingRewards() public {  
        // user stakes initially  
        uint256 initialStake = 100e18;  
        vm.prank(user);  
        staking.stake(tokenA, initialStake);  
  
        // advance time so rewards would accrue if updated  
        uint256 start = staking.lastMinted();  
        uint256 elapsed = 1 days;  
        vm.warp(start + elapsed);  
  
        // previewPending should reflect elapsed * mintRate when only this token is staked  
        uint256 pendingBefore = staking.previewPending(tokenA);  
        uint256 expected = rate * elapsed; // when single staker, token receives all minted amount  
        assertEq(pendingBefore, expected);  
  
        // Now an attacker stakes to the same token. The buggy implementation  
        // sets tokenInfo.lastRewardIndex = rewardIndex on stake, which wipes the pending  
        address attacker = makeAddr("Attacker");  
        _mintAndApprove(attacker, 1_000e18);  
        vm.prank(attacker);  
        staking.stake(tokenA, 1e18);  
  
        // After the attacker stake, the pending rewards for the token are erased  
        // because lastRewardIndex was updated to the current rewardIndex.  
        uint256 pendingAfter = staking.previewPending(tokenA);  
        assertEq(pendingAfter, 0);  
  
        // If the gauge were to call accrueReward now, it would return 0, meaning  
        // the previously pending rewards are never minted/distributed.  
        vm.prank(gauge);  
        uint256 accrued = staking.accrueReward(tokenA);  
        assertEq(accrued, 0);  
    }  
}  

```

## Mitigation

No response.


### [Medium-01] Failure to advance lastMinted when totalStakedAmount == 0 leads to retroactive reward issuance and inflation

## Summary
When SuperDCAStaking::_updateRewardIndex returns early while totalStakedAmount == 0 it does not advance lastMinted. Time accumulates off-chain and later gets applied to rewardIndex once there are stakers, causing newly joined stakers to receive rewards for periods when no one was staked. This mints tokens retroactively, inflates supply, and breaks fairness of the reward model.

## Root Cause
https://github.com/sherlock-audit/2025-09-super-dca/blob/main/super-dca-gauge/src/SuperDCAStaking.sol#L181

When SuperDCAStaking::_updateRewardIndex returns early while totalStakedAmount == 0, it does not advance lastMinted. Time accumulates off-chain and is later applied to the rewardIndex once there are stakers, causing newly joined stakers to receive rewards for periods when no one was staked. This minting process retroactively inflates the supply and breaks the fairness of the reward model.

## Internal Pre-conditions
- mintRate > 0
- lastMinted is older than current block timestamp
- totalStakedAmount == 0 for a non-trivial time window
- The gauge or a caller subsequently triggers _updateRewardIndex (via accrueReward, stake, or other code paths) after the idle period

## External Pre-conditions
- An external actor (honest or adversarial) calls SuperDCAStaking::stake or the authorized gauge calls SuperDCAStaking::accrueReward after the idle period.
- Standard ERC20 transfers/approvals succeed for staking proceeds.

## Attack Path
- Let the pool remain empty (no stakers) for some period T.
- Attacker (or honest user) calls SuperDCAStaking::stake (or the gauge calls accrueReward) after T.
- Because lastMinted was not advanced during the idle period, rewardIndex is advanced by the full elapsedTmintRate amount and distributed to current stakers.
- Attacker receives an outsized portion of minted tokens despite not staking during the idle period.
Note: the action executes in the context of msg.sender (the staker) so no privileged call is needed.


## Impact
- Retroactive reward issuance: first staker(s) after idle period receive rewards for time when no stake existed.
- Inflationary minting and unfair reward allocation.
- Financial loss to protocol or intended recipients, and potential exploitation to farm rewards.

This occurs naturally if the staking pool becomes empty (which is common) and then later receives new stakes. No special privileges are required to trigger the problem; any user staking after an idle period will trigger it.

## PoC
- Place the test inside the `SuperDCAStaking.t.sol` file.
```javascript
contract Leakage is SuperDCAStakingTest {  
    /// @notice Demonstrates the retroactive reward issuance bug:  
    /// If time elapses while totalStakedAmount == 0, lastMinted is not updated.  
    /// When a user stakes afterwards, previewPending() shows rewards for the elapsed  
    /// time even though the user was not staked during that period.  
    function test_RewardsAreRetroactivelyIssuedAfterIdlePeriod() public {  
        // Fast-forward time while there are no stakers  
        uint256 start = staking.lastMinted();  
        uint256 elapsed = 1 days;  
        vm.warp(start + elapsed);  
  
        // User stakes immediately after the idle period  
        uint256 stakeAmt = 100e18;  
        vm.prank(user);  
        staking.stake(tokenA, stakeAmt);  
  
        // Because lastMinted was not updated during the idle period, previewPending  
        // for the newly-staked token will include the entire elapsed * mintRate amount.  
        uint256 pending = staking.previewPending(tokenA);  
        uint256 expected = elapsed * rate; // mintRate * elapsed  
  
        // This should be zero under correct behavior, but due to the bug it equals expected  
        assertEq(pending, expected);  
    }  
}  

```

## Mitigation
Fix _updateRewardIndex so that:
lastMinted is advanced when `totalStakedAmount == 0` (consume the idle time), avoiding backdating of rewards.


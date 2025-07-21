## Zaros Part 2
[Contest Details] (https://codehawks.cyfrin.io/c/2025-01-zaros-part-2/results?lt=contest&page=1&sc=reward&sj=reward&t=report)

### [High-01] Incorrect credit capacity checks in the `VaultRouterBranch::redeem` allows for over withdraw the vault

**Summary**
Ther is a flawed credit capacity check in the redeem function that allows users to withdraw more collateral assets than the vault’s available unlocked liquidity. This can drain the vault’s reserves, rendering it insolvent and unable to honor legitimate withdrawal requests.

**Vulnerability**
flawed code
```
// if the credit capacity delta is greater than the locked credit capacity before the state transition, revert
    if (
        ctx.creditCapacityBeforeRedeemUsdX18.sub(vault.getTotalCreditCapacityUsd()).lte(
            ctx.lockedCreditCapacityBeforeRedeemUsdX18.intoSD59x18()
        )
    ) {
        revert Errors.NotEnoughUnlockedCreditCapacity();
    }
```        
The condition checks if the withdrawal amount (delta) is ≤ locked credit capacity, not unlocked. This allows withdrawals to exceed unlocked liquidity if delta > lockedCredit.

A Scenario of the flawed vault check
Vault State:

1. Total Credit Capacity: 1,000 USD (assets available).

2. Locked Credit Capacity: 300 USD (reserved for pending withdrawals).

3. Unlocked Credit Capacity: 700 USD (available for immediate withdrawals).

Malicious Redemption:

1. A user attempts to redeem shares worth 800 USD (exceeding the unlocked 700 USD).

2. The flawed check incorrectly approves the withdrawal because 800 <= 300 evaluates to false.

3. The vault transfers 800 USD to the user, overdrawing its liquidity by 100 USD.

**Attack path**
Nil

**Impact**
Nil

**Mitigation**
The correct approach should have been recalculating the unlocked credit correctly and ensuring that the redeemed amount doesn't exceed it. The corrected condition should check if the delta (redeemed amount) is greater than the unlocked credit and revert if true.
```
+   SD59x18 unlockedCreditBeforeRedeemX18 = ctx.creditCapacityBeforeRedeemUsdX18.sub(
    ctx.lockedCreditCapacityBeforeRedeemUsdX18.intoSD59x18()
);
+   SD59x18 deltaCreditX18 = ctx.creditCapacityBeforeRedeemUsdX18.sub(vault.getTotalCreditCapacityUsd());
​
+   if (deltaCreditX18.gt(unlockedCreditBeforeRedeemX18)) {
    revert Errors.NotEnoughUnlockedCreditCapacity();
}
```

### [Medium-01] BaseFee will underflow if amountIn is lesser than it

**Summary**
During the StabilityBranch::initiateSwap if the amountIn is lesser than the baseFee user wont be able to get a refund in the refunSwap function because the transaction will revert when calculating the refundAmountUsd
Thou, the baseFeeUsd is a uint256 and the amountIn is a uint128, so the baseFeeUsd will underflow if the amountIn is lesser than it.

https://github.com/Cyfrin/2025-01-zaros-part-2/blob/35deb3e92b2a32cd304bf61d27e6071ef36e446d/src/market-making/branches/StabilityBranch.sol#L474

**Mitigation**
Inside the initiateSwap function there should be a check that the amountIn is greater than the baseFee and if it is not should revert the transaction.

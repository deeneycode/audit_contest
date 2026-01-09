## Dependency on Uniswap V3 slot0 for asset valuation enables share price manipulation and vault draining
## Summary
https://github.com/sherlock-audit/2025-11-stnxm-by-easedefi/blob/main/stNXM-Contracts/contracts/core/stNXM.sol#L514
https://github.com/sherlock-audit/2025-11-stnxm-by-easedefi/blob/main/stNXM-Contracts/contracts/core/stNXM.sol#L463
The StNXM vault calculates its share price based on a modified totalSupply that subtracts "virtual shares" held in a Uniswap V3 liquidity position. The calculation of these virtual shares relies on dex.slot0(), which represents the instantaneous spot price of the underlying pool. This vulnerability allows an attacker to manipulate the Uniswap pool price within a single block to artificially inflate the vault's share price and drain its assets.

## Root Cause
The root cause is the reliance on the manipulation-prone dex.slot0 (spot price) within StNXM::dexBalances to value the protocol's LP position, combined with the accounting logic in StNXM::totalSupply that subtracts this manipulatable value from the actual token supply.

## Internal Pre-conditions
- The StNXM contract must have an active liquidity position in the Uniswap V3 pool (i.e., dexTokenIds array is not empty).
- The contract must hold wNXM assets available for withdrawal.
## External Pre-conditions
- A Uniswap V3 pool for wNXM/stNXM exists.
- The pool has sufficient liquidity to facilitate a flash swap or large trade.
## Attack Path
-The attacker acquires a small amount of stNXM (shares).
- The attacker performs a flash swap or large swap on the Uniswap V3 pool, selling stNXM for wNXM.
- This action crashes the spot price of stNXM and significantly increases the amount of stNXM held in the Uniswap pool (the LP position).
- The StNXM::dexBalances function, reading the manipulated slot0 and pool balances, reports a massive increase in virtualShares (the stNXM held in LP).
- The StNXM::totalSupply function calculates super.totalSupply() - virtualShares. As virtualShares increases towards the value of super.totalSupply(), the returned totalSupply approaches zero.
- The vault's share price is calculated as totalAssets() / totalSupply(). With the denominator artificially crushed to near zero, the share price skyrockets.
- The attacker calls StNXM::redeem or StNXM::withdraw using their small stNXM balance. Due to the inflated price, this small balance is valued at more than the vault's entire wNXM holdings.
- The attacker drains the vault.
## Impact
An attacker can manipulate the share price to be arbitrarily high, allowing them to redeem a negligible amount of stNXM shares for the entire wNXM balance of the vault, effectively draining all protocol funds.

## PoC
- Attacker holds 1 stNXM and the vault holds 1000 wNXM.
- Attacker calls UniswapV3Pool::swap, selling a large amount of stNXM into the pool.
- The pool's stNXM balance (virtual shares) spikes.
- Attacker calls StNXM::totalSupply.
- super.totalSupply() (Total Minted) = 10,000 (example)
- virtualShares (LP Position) is manipulated to be 9,999.
- Resulting totalSupply = 1.
- Attacker calls StNXM::redeem(1).
- The vault calculates assets owed: (1 share * totalAssets) / totalSupply.
- 1 * 1000 / 1 = 1000.
- The vault transfers 1000 wNXM to the attacker for 1 share.
## Mitigation
Do not subtract the LP position balances from the totalSupply. Instead, treat the LP position as an asset of the vault like any other strategy. Additionally, switch from using the spot price (slot0) to a Time-Weighted Average Price (TWAP) oracle or a fair-value invariant to calculate the value of the LP position.

## Use of onlyAuthorizedMatcher modifier enables authorized matchers to modify fees
## Summary
Critical admin functions in MarketController are protected by onlyAuthorizedMatcher instead of onlyOwner. As implemented, authorised off‑chain matchers can change setFeeRate, setTreasury, setUserFeeRate, and pause trading — operations that must be restricted to the contract owner/governance.

## Root Cause
https://github.com/sherlock-audit/2025-10-index-fun-order-book-contest/blob/main/orderbook-solidity/src/Market/MarketController.sol#L212

Several owner-intended functions were given the onlyAuthorizedMatcher modifier. Access checks rely on authorizedMatchers[msg.sender] rather than onlyOwner, so privileged matcher accounts can perform sensitive state changes.

## Internal Pre-conditions
- authorizedMatchers[msg.sender] == true.
- Target functions use onlyAuthorizedMatcher.
## External Pre-conditions
.

Attack Path
.

## Impact
Unauthorised operation can be performed.

PoC
.

## Mitigation
Change the modifier on admin functions to onlyOwner

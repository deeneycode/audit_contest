## Plaza Finance
[Contest Details] (https://audits.sherlock.xyz/contests/682/report)

### [High-01] Collateral Level Manipulation via Bondsupply

**Summary**
The unprotected integer division precision loss in the collateral level calculation will cause an exploitable creation rate manipulation for protocol users as attackers will strategicaly modify bond supply to force creation rate below 1.2 threshold.

**Root cause**
The issues originate from how the the protocol calculates the collateral level in the getCreateAmount
https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/Pool.sol#L326

when this level crosses below the collateral_threshold` (1200000), the protocol swithches to an entirely different creation rate formula. The integer division create small gaps in possible collateral level values, which attackers can exploit by carefully choosing bond supply amounts.
The core issues is worsened by two design choices:

the sharp transition at exactly 1.2, creating clear target for manipulation.
The dramatic difference between the two creation rate formulas either side of this threshold.

**Attack path**
1. Initial state

```javascript
tvl = 1500e18 
bondSupply = 10e18
/ calculate collateral level
collateralLevel = (1500e18 *1000000) / (10e18 * 100) = 1500000 // 1.5  in decimal
 initial state is above threshold 1.5 > 1.2
 ```
2. Attack setup
```javascript
// Attacker needs to push collateral level below 1.2
// Formula: newBondSupply = (TVL * PRECISION) / (COLLATERAL_THRESHOLD * BOND_TARGET_PRICE)
targetBondSupply = (1500e18 * 1000000) / (1200000 * 100)
                 = 1500e24 / 120000000
                 = 12.5e18  // Need at least 13 bonds to push below threshold

// Mint additional bonds to reach target
bondsToMint = 13e18 - 10e18 = 3e18 bonds
```

3. Attack Execution Step 1: Manipulation
```javascript
newBondSupply = 13e18   // After minting

// New collateral level
newCollateralLevel = (1500e18 * 1000000) / (13e18 * 100)
                   = 1500e24 / 1300e18
                   = 1153846   // 1.153846 in decimal

// Success: 1.153846 < 1.2 (COLLATERAL_THRESHOLD)
```

4 Attack Impact on Creation Rate
```javascript
// BEFORE ATTACK (collateralLevel = 1.5):
adjustedValue = 1500e18 - (100 * 10e18)     // TVL - (BOND_TARGET_PRICE * bondSupply)
              = 1500e18 - 1000e18 
              = 500e18
creationRate = (500e18 * 1000000) / leverageSupply

// AFTER ATTACK (collateralLevel = 1.153846):
creationRate = (1500e18 * 200000) / leverageSupply
            = 300000e18 / leverageSupply

// Formulas completely switch due to threshold breach!
```

5 Profit calculation example
```javascript
// Assume leverageSupply = 1000e18
beforeCreationRate = (500e18 * 1000000) / 1000e18 
                   = 500000

afterCreationRate = (1500e18 * 200000) / 1000e18
                  = 300000

// For victim depositing 1 ETH deposit at 2000 USD/ETH:
Normal Tokens = (1e18 * 2000 * 1000000) / 500000 = 4e18 tokens
Attack Tokens = (1e18 * 2000 * 1000000) / 300000 = 6.67e18 tokens
```
// ~66.7% more tokens for same deposit!
The summary of the attack exploiit is through a precise sequence.

Identify when the collateral level is slightly above 1.2
Calculate the exact bond supply needed to push the level below 1.2
Front-run user transactions with bond minting to manipulate the collateral level
Allow victim transactions to execute with disadvantageous rates
Back-run to restore the original state and extract profit
For example, with TVL = 1500e18, increasing bond supply from 10 to 13 can push the collateral level from 1.5 to 1.15, triggering the profitable rate switch.

**impact**
When this attack is successfully executed, this attack leads to significant value extraction from the protocol and its users

1. For protocol user
- receives incorrect amounts of tokens when minting
- May overpay by up to 66.7% in worst cases.
2. For protocol
- Loss of price stability and predictability
- Reduced user trust due to inconsistent minting rates
- Potential for cascading liquidations if leveraged positions are affected

### [Medium-01] The excess amount in the preDeposit::_deposit() is not returned back to the user

**Summary**
The _deposit() function inside the preDeposit contract allows an amount to be deposited on behalfOf. When the deposit exceeds the reserve cap (reserveCap), the function silently reduces the amount to fit within the cap. However, the excess portion of the deposit is neither processed nor refunded.

**Root cause**
https://github.com/sherlock-audit/2024-12-plaza-finance/blob/main/plaza-evm/src/PreDeposit.sol#L118
```javascript
  function _deposit(uint256 amount, address onBehalfOf) private checkDepositStarted checkDepositNotEnded {
    if (reserveAmount >= reserveCap) revert DepositCapReached();

    address recipient = onBehalfOf == address(0) ? msg.sender : onBehalfOf;

    // if user would like to put more than available in cap, fill the rest up to cap and add that to reserves
    if (reserveAmount + amount >= reserveCap) { 
      amount = reserveCap - reserveAmount;
    }

    balances[recipient] += amount;
    reserveAmount += amount;
    IERC20(params.reserveToken).safeTransferFrom(msg.sender, address(this), amount);
    emit Deposited(recipient, amount);
  }
```
However, the function does not handle scenario where the user sends more amount leading to overdeposit. This could result in the financial losses for users if they mistakenly send more amount.
While the function alows user to put more than available in cap.

**Attack path**
when the deposit exceeds the reservecap, the function silently reduces the amount to fit within the cap. however, the excess portion of the deposit is neither processed nor refunded
Example:

- Reserve cap: 6000
- Current reserve amount: 3000
- User attempts to deposit: 5000
- Adjusted deposit amount: 3000 (to fill the cap)
- Excess: 2000 (ignored without refund)

**Impact**
Users who send more amount will not receive a refund for the excess amount

**Mitigation**
After performing the reserveCap id deducted from the reserveAmount, the excess amount should be calculated and return any excess amount back to the user.


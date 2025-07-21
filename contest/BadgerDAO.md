## BadgerDAO / badger-ebtc-bsm
[Contest Details] (https://cantina.xyz/code/f57ffb47-0ded-4f04-bcec-ecd7d47fad58/findings?created_by=deeney)

### [High-01] Inverse Price Calculation in tBTCChainlinkAdapter Contract

**Summary**
The tBTCChainlinkAdapter contract incorrectly calculates the BTC/tBTC price instead of the intended tBTC/BTC price due to a flawed formula in the _convertAnswer function. This results in the contract returning the reciprocal of the expected value, potentially causing significant errors in downstream financial applications. https://cantina.xyz/code/f57ffb47-0ded-4f04-bcec-ecd7d47fad58/src/tBTCChainlinkAdapter.sol#L61

**Finding Description**
The tBTCChainlinkAdapter contract aims to provide the tBTC/BTC exchange rate by leveraging two Chainlink price feeds: tBTC/USD and BTC/USD. The intended output is the price of tBTC in BTC (tBTC/BTC), scaled to 18 decimals. However, the _convertAnswer function computes the BTC/tBTC price instead due to an incorrect formula.

The current implementation in the _convertAnswer: https://cantina.xyz/code/f57ffb47-0ded-4f04-bcec-ecd7d47fad58/src/tBTCChainlinkAdapter.sol#L62
```javascript
/// @dev Uses the prices from the tBtc feed and the BTC feed to compute tBTC->BTC
    function _convertAnswer(int256 btcUsdPrice, int256 tBtcUsdPrice) private view returns (int256) {
        return
            (btcUsdPrice * TBTC_USD_PRECISION * ADAPTER_PRECISION) / 
            (BTC_USD_PRECISION * tBtcUsdPrice);
    }
```
Downstream systems expect the tBTC/BTC price but receive its inverse, leading to incorrect financial calculations. For example, if tBTC/USD = 51,000 and BTC/USD = 50,000, the correct tBTC/BTC should be 1.02, but the contract returns approximately 0.9804.

**Impact Explanation**
The impact is assessed as High because; Price feeds are critical for DeFi applications where precise pricing ensures economic fairness. Returning the inverse price can lead to substantial financial errors.

**Likelihood Explanation**
The likelihood is High because: Constant Occurrence: The error is inherent in the formula and affects every price query, not just edge cases.

**Proof of Concept**
Here is a coded proof of concept. It should be included in the test suite of chainlinkAdapterTests:
```javascript
function testIncorrectPriceCalculation() public {
        // Set mock prices
        // tBTC/USD = 51,000 USD, scaled by 10^8
        int256 tBtcUsdPrice = 51_000 * int256(10**8);
        usdtBtcAggregator.setPrice(tBtcUsdPrice);
        usdtBtcAggregator.setLatestRoundId(1);
        usdtBtcAggregator.setUpdateTime(block.timestamp);

        // BTC/USD = 50,000 USD, scaled by 10^8
        int256 btcUsdPrice = 50_000 * int256(10**8);
        btcUsdAggregator.setPrice(btcUsdPrice);
        btcUsdAggregator.setLatestRoundId(1);
        btcUsdAggregator.setUpdateTime(block.timestamp);

        // Get the price from the adapter
        (, int256 answer,,,) = tBTCchainlinkAdapter.latestRoundData();

        // Expected tBTC/BTC price: 51,000 / 50,000 = 1.02
        // Scaled to 18 decimals: 1.02 * 10^18
        int256 expectedTBtcBtc = 1.02 ether; // 1.02 * 10^18

        // Incorrect BTC/tBTC price computed by contract: 50,000 / 51,000 ≈ 0.980392
        // Scaled to 18 decimals
       int256 expectedBtcTBtc = int256((uint256(50_000) * 1e18) / 51_000); // ≈ 0.9803921568627451 * 10^18

        // Log values for clarity
        emit log_named_int("Computed answer", answer);
        emit log_named_int("Expected tBTC/BTC", expectedTBtcBtc);
        emit log_named_int("Expected BTC/tBTC", expectedBtcTBtc);

        // Assert that the answer matches the incorrect BTC/tBTC price
        assertEq(answer, expectedBtcTBtc, "Answer should match incorrect BTC/tBTC price");

        // Assert that the answer does NOT match the expected tBTC/BTC price
        assertTrue(answer != expectedTBtcBtc, "Answer should not match expected tBTC/BTC price");

        // Compute the correct answer for reference
        int256 correctAnswer = (tBtcUsdPrice * int256(10**8) * int256(10**18)) /
            (int256(10**8) * btcUsdPrice);
        assertEq(correctAnswer, expectedTBtcBtc, "Correct answer should be tBTC/BTC scaled to 18 decimals");
    }
```

**Recommended Mitigation**
Fix the _convertAnswer function to compute the correct tBTC/BTC price:
```
function _convertAnswer(int256 btcUsdPrice, int256 tBtcUsdPrice) private view returns (int256) {
    return (tBtcUsdPrice * BTC_USD_PRECISION * ADAPTER_PRECISION) / (TBTC_USD_PRECISION * btcUsdPrice);
}
```

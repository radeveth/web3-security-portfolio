# Good Entry Audit Contest Findings By Radoslav Radev

## High Severity Issue: GetVault `poolMatchesOracle` calculation may overflow


## Lines of code
https://github.com/code-423n4/2023-08-goodentry/blob/main/contracts/GeVault.sol#L367-L378

## Vulnerability details
### Impact
Overflow.

### Proof of Concept
The GetVault derivative contract implements the poolMatchesOracle() function, which is used by deposit(), withdraw() and rebalance() functions.
The poolMatchesOracle() function checks that the pool price isn't manipulated using a Uniswap V3 pool.
The function queries the pool to fetch the sqrtPriceX96 and does the following calculation:

solidity priceX8 = (priceX8 * uint(sqrtPriceX96 / 2 ** 12) ** 2 * 1e8) / 2 ** 168; 

The main issue here is that the multiplications in the expression sqrtPriceX96 * (uint(sqrtPriceX96)) * (1e18) may eventually overflow. This case is taken into consideration by the implementation of the OracleLibrary.getQuoteAtTick function which is part of the Uniswap V3 periphery set of contracts.

```jsx
49:     function getQuoteAtTick(
50:         int24 tick,
51:         uint128 baseAmount,
52:         address baseToken,
53:         address quoteToken
54:     ) internal pure returns (uint256 quoteAmount) {
55:         uint160 sqrtRatioX96 = TickMath.getSqrtRatioAtTick(tick);
56:
57:         // Calculate quoteAmount with better precision if it doesn't overflow when multiplied by itself
58:         if (sqrtRatioX96 <= type(uint128).max) {
59:             uint256 ratioX192 = uint256(sqrtRatioX96) * sqrtRatioX96;
60:             quoteAmount = baseToken < quoteToken
61:                 ? FullMath.mulDiv(ratioX192, baseAmount, 1 << 192)
62:                 : FullMath.mulDiv(1 << 192, baseAmount, ratioX192);
63:         } else {
64:             uint256 ratioX128 = FullMath.mulDiv(sqrtRatioX96, sqrtRatioX96, 1 << 64);
65:             quoteAmount = baseToken < quoteToken
66:                 ? FullMath.mulDiv(ratioX128, baseAmount, 1 << 128)
67:                 : FullMath.mulDiv(1 << 128, baseAmount, ratioX128);
68:         }
69:     }
```
Note that this implementation guards against different numerical issues. In particular, the if in line 58 checks for a potential overflow of sqrtRatioX96 and switches the implementation to avoid the issue.

### Tools Used
Manual Review

### Recommended Mitigation Steps
The poolPrice function can delegate the calculation directly to the OracleLibrary.getQuoteAtTick function of the v3-periphery package:

```jsx
function poolPrice() private view returns (uint256) {
address rocketTokenRETHAddress = RocketStorageInterface(
ROCKET_STORAGE_ADDRESS
).getAddress(
keccak256(
abi.encodePacked("contract.address", "rocketTokenRETH")
)
);
IUniswapV3Factory factory = IUniswapV3Factory(UNI_V3_FACTORY);
IUniswapV3Pool pool = IUniswapV3Pool(
factory.getPool(rocketTokenRETHAddress, W_ETH_ADDRESS, 500)
);
(, int24 tick, , , , , ) = pool.slot0();
return OracleLibrary.getQuoteAtTick(tick, 1e18, rocketTokenRETHAddress, W_ETH_ADDRESS);
}
```

### Assessed type
Uniswap
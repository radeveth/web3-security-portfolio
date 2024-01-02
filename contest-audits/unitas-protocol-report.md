# Unitas Protocol Sherlock Contest - Radev's Findings Report

# Findings Summary

| ID     | Title                                                              | Severity |
| ------ | ------------------------------------------------------------------ | -------- |
| [M-01] | Missing Slippage protection | Medium |

---

# **Detailed Findings**

# [M-01] Missing Slippage protection

## Summary

The `swap()` function in `Unitas.sol` contract and related functions, does not appear to contain `slippage protection`. While there are several checks and balances within the code to ensure the integrity of the swap, including a reserve ratio check and an oracle price check, there is no mechanism for users to define a maximum tolerable slippage for their trades. This absence potentially exposes users to higher than anticipated price changes during token swaps, which may result in financial loss.

## Vulnerability Detail

Slippage occurs when the actual price of a trade differs from the expected price due to changes in liquidity between the time the trade is requested and the time it is executed. In DeFi protocols, slippage is a common issue, especially in larger trades or low-liquidity pools.

Explicit slippage protection allows users to specify a maximum tolerable slippage, usually through a parameter in the swap function. If the actual slippage exceeds this amount, the transaction is reverted, protecting the user from excessive slippage. The provided code does not include this protection mechanism.

## Impact

The absence of explicit slippage protection may `expose users to financial risk`. Without the ability to define a maximum tolerable slippage, a `user's trade might be executed at a price significantly different from what they expected when initiating the trade`. In extreme cases, this could lead to substantial financial loss. Also `swap can be sandwiched`, whitch also causing a loss of funds for users you withdraw their rewards.

## Code Snippet

https://github.com/sherlock-audit/2023-04-unitasprotocol/blob/main/Unitas-Protocol/src/Unitas.sol#L208-L237

## Tool used

Manual Review

## Recommendation

To prevent this vulnerability, consider implementing an explicit slippage protection mechanism. This could be done by adding a parameter to the swap function that allows users to specify a minimum output amount (if they're providing the input amount) or a maximum input amount (if they're specifying the output amount). Then, add a check in the `_getSwapResult()` function to ensure that the calculated swap outcomes do not violate these user-defined limits. This should be carefully implemented!

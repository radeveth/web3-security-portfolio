# Decent - Radev's Findings Report

# Findings Summary
| Number     | Title                                                              | Severity |
| ------ | ------------------------------------------------------------------ | -------- |
| [M-01] | Execution of `swaps` in the `before bridging operation` and in the `same-chain txs` lacks deadline | Medium |
| [M-02] | Removing an already added `swappers` and `bridge adapters` contract address in `UTB.sol` is not possible | Medium |
| [M-03] | Potential LayerZero Channel Blocking in Decent Protocol | Medium |

---

#

## [M-01] Execution of `swaps` in the `before bridging operation` and in the `same-chain txs` lacks deadline

## GitHub Links

- [UniSwapper.sol Contract](https://github.com/code-423n4/2024-01-decent/blob/main/src/swappers/UniSwapper.sol)

## Proof of Concept

In `UniSwapper.sol`, part of the Decent protocol, swaps are executed as part of the token exchange process. This contract interacts with Uniswap or similar decentralized exchanges (DEXs) to perform token swaps. However, the execution of these swaps, particularly before bridging operations and in same-chain transactions, does not include a deadline parameter. In typical DEX interactions, a deadline is used to specify the time by which a swap must be completed, providing a safeguard against unfavorable changes in market conditions or potential manipulation.

In the `UniSwapper.sol` contract, functions that interact with DEXs for swapping tokens (e.g., `swapExactIn`, `swapExactOut`) do not include a deadline parameter. This can be confirmed by reviewing the contract code and noting the absence of a time-bound condition for swap execution.

```solidity
    function swap(
        bytes memory swapPayload
    )
        external
        onlyUtb
        returns (address tokenOut, uint256 amountOut)
    {
        (SwapParams memory swapParams, address receiver, address refund) = abi
            .decode(swapPayload, (SwapParams, address, address));
        tokenOut = swapParams.tokenOut;
        if (swapParams.path.length == 0) {
            return swapNoPath(swapParams, receiver, refund);
        }
        if (swapParams.direction == SwapDirection.EXACT_IN) {
            amountOut = swapExactIn(swapParams, receiver);
        } else {
            swapExactOut(swapParams, receiver, refund);
            amountOut = swapParams.amountOut;
        }
    }

    function swapExactIn(
        SwapParams memory swapParams, // SwapParams is a struct
        address receiver
    ) public payable routerIsSet returns (uint256 amountOut) {
        swapParams = _receiveAndWrapIfNeeded(swapParams);

			  //@audit missing deadline parameter
        IV3SwapRouter.ExactInputParams memory params = IV3SwapRouter
            .ExactInputParams({
                path: swapParams.path,
                recipient: address(this),
                amountIn: swapParams.amountIn,
                amountOutMinimum: swapParams.amountOut
            });

        IERC20(swapParams.tokenIn).approve(uniswap_router, swapParams.amountIn);
        amountOut = IV3SwapRouter(uniswap_router).exactInput(params);

        _sendToRecipient(receiver, swapParams.tokenOut, amountOut);
    }

    function swapExactOut(
        SwapParams memory swapParams,
        address receiver,
        address refundAddress
    ) public payable routerIsSet returns (uint256 amountIn) {
        swapParams = _receiveAndWrapIfNeeded(swapParams);
				//@audit missing deadline parameter
        IV3SwapRouter.ExactOutputParams memory params = IV3SwapRouter
            .ExactOutputParams({
                path: swapParams.path,
                recipient: address(this),
                //deadline: block.timestamp,
                amountOut: swapParams.amountOut,
                amountInMaximum: swapParams.amountIn
            });

        IERC20(swapParams.tokenIn).approve(uniswap_router, swapParams.amountIn);
        amountIn = IV3SwapRouter(uniswap_router).exactOutput(params);

        // refund sender
        _refundUser(
            refundAddress,
            swapParams.tokenIn,
            params.amountInMaximum - amountIn
        );

        _sendToRecipient(receiver, swapParams.tokenOut, swapParams.amountOut);
    }
```

## Impact

The absence of a deadline in swap transactions can lead to several issues:

1. **Front-Running Vulnerability**: Without a deadline, transactions are more susceptible to front-running, where malicious actors can observe a pending swap transaction and execute their own transactions first to affect the market price.
2. **Market Risk**: Users are exposed to market risk for a longer duration, as the swap can be executed at any time, potentially leading to execution at less favorable prices.
3. **User Experience**: Users have no control over the maximum duration for which their transaction can remain pending, which can lead to uncertainty and potential financial losses.

The issue presents a clear risk in terms of potential financial losses and exploitation through market manipulation. However, it does not directly lead to immediate loss of funds or critical contract failures. The severity could be higher in scenarios where the market is highly volatile, or the protocol is heavily utilized, increasing the likelihood of exploitation.

## Recommendations

To mitigate the risks associated with the lack of a deadline in swap operations, consider the following recommendations:

1. **Implement Deadline Parameter**: Modify the swap functions in `UniSwapper.sol` to include a deadline parameter. This parameter should be passed along to the DEX interface to ensure that swaps are only executed within the specified time frame.
2. **User-Defined Deadlines**: Allow users to specify the deadline when initiating a swap. This gives users control over the timing of their transactions and reduces their exposure to market risks.

---

## [M-02] Removing an already added `swappers` and `bridge adapters` contract address in `UTB.sol` is not possible

## GitHub Links

- [UTB.sol Contract](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTB.sol)

## Proof of Concept

In the `UTB.sol` contract, there are functions to register swappers (`registerSwapper`) and bridge adapters (`registerBridge`). These functions allow the contract owner to add new swappers and bridge adapters by updating the `swappers` and `bridgeAdapters` mappings, respectively. However, there is no functionality provided to remove or update these addresses once they have been set. This limitation means that once a swapper or bridge adapter is registered, it cannot be removed or replaced, which could lead to potential issues if the address needs to be changed or if it becomes compromised.

```solidity
		/**
     * @dev Registers and maps a swapper to a swapper ID.
     * @param swapper The address of the swapper.
     */
    function registerSwapper(address swapper) public onlyOwner {
        ISwapper s = ISwapper(swapper);
        swappers[s.getId()] = swapper;
    }

    /**
     * @dev Registers and maps a bridge adapter to a bridge adapter ID.
     * @param bridge The address of the bridge adapter.
     */
    function registerBridge(address bridge) public onlyOwner {
        IBridgeAdapter b = IBridgeAdapter(bridge);
        bridgeAdapters[b.getId()] = bridge;
    }

```

The removing of already added `swappers` and `bridge adapters` contract address with the functions above is impossible.

The contract mappings `swappers` and `bridgeAdapters` are used to store the addresses of swappers and bridge adapters. The functions `registerSwapper` and `registerBridge` allow adding addresses to these mappings. However, there are no functions to remove or update these addresses. This can be confirmed by reviewing the contract code and noting the absence of such functionality.

## Impact

The inability to remove or update swappers and bridge adapters can have several impacts:

1. **Lack of Flexibility**: The contract owner cannot update the system to use a new or upgraded swapper or bridge adapter, limiting the ability to adapt to new requirements or improvements.
2. **Security Risk**: If a swapper or bridge adapter is compromised, the contract owner cannot remove the compromised address, potentially putting user funds at risk if the compromised swapper or bridge adapter is used.
3. **Error Recovery**: Mistakes made during the registration of swappers or bridge adapters cannot be corrected, which could lead to operational issues or security vulnerabilities.

The issue presents a clear limitation in contract functionality and poses potential security risks if external dependencies (swappers or bridge adapters) are compromised.

## Recommendations

To mitigate the issues and enhance the flexibility and security of the `UTB.sol` contract, consider implementing the following recommendations:

**Implement Removal Functions**: Add functions to remove swappers and bridge adapters from the `swappers` and `bridgeAdapters` mappings. Ensure that these functions can only be called by the contract owner or another authorized entity.


---

## [M-03] Potential LayerZero Channel Blocking in Decent Protocol

### GitHub Links

- [UTB.sol Contract](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTB.sol)
- **Component**: `UTB.sol` and related contracts

### Description

The Decent Protocol, particularly in its `UTB.sol` contract and related components (especially the execute() functions), might be vulnerable to an issue where insufficient gas checks for LayerZero operations could allow an attacker to block the LayerZero channel. This vulnerability arises from the lack of minimum gas validation when sending messages through LayerZero, potentially leading to transaction reverts and blocking the message pathway between chains.

### Proof of Concept

- In the context of LayerZero operations, the sender specifies the gas amount for the Relayer to deliver the payload to the destination chain.
- If the gas amount specified is insufficient, it could cause the transaction to revert on the destination chain, resulting in a blocked pathway and StoredPayload.
- The `UTB.sol` contract and related components in the Decent Protocol handle cross-chain operations but do not explicitly include logic for minimum gas checks in LayerZero transactions.

### Tools Used

- Manual review of the `UTB.sol` contract and related components in the Decent Protocol.

### Recommended Mitigation Steps

**Implement Minimum Gas Checks**: Introduce logic in the `UTB.sol` contract and related components to validate the minimum gas required for LayerZero operations. This should cover all scenarios where LayerZero is used for cross-chain communication.

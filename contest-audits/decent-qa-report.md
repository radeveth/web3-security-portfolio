

# QA Report - Decent Audit Contest | 19 Jan 2024 - 23 Jan 2024

---

# Executive summary

### Overview

|              |                                              |
| :----------- | :------------------------------------------- |
| Project Name | Decent                                       |
| Repository   | https://github.com/code-423n4/2024-01-decent |
| Website      | [Link](https://www.decent.xyz/)              |
| Twitter      | [Link](https://twitter.com/decentxyz)        |
| Methods      | Manual Review                                |
| Total nSLOC  | 1193 over 11 contracts                       |

---

---

# Findings Summary

| ID                | Title | Severity                  |
| ----------------- | ----- | ------------------------- |
| [L-01](#L-01)     | Missing `payable` Modifier in `UTBExecutor.sol#execute()` Function |  _Low_ |
| [L-02](#L-02)     | Lack of Ether Withdrawal Mechanism in Contracts with `receive/fallback` Functions | _Low_ |
| [L-03](#L-03)     | Inheritance Issues in `UTBFeeCollector.sol` and `UTBExecutor.sol` | _Low_ |
| [L-04](#L-04)     | Use `quoteLayerZeroFee` instead of sending entire msg.value as gas fee for swap call (Inefficient Gas Fee Calculation for Stargate Router Swap in `StargateBridgeAdapter.sol`) | _Low_ |
| [NC-01](#NC-01)   | Optimization of `_receiveAndWrapIfNeeded()` in `UniSwapper.sol#swap()` | _Non Critical_ |
| [NC-02](#NC-02)   | Use of `else if` in Conditional Statements | _Non Critical_ |
| [NC-03](#NC-03)   | Inheritance and Use of `ecrecover` in `UTBFeeCollector.sol` | _Non Critical_ |
| [NC-04](#NC-04)   | Inconsistency in WETH Token Addresses Across Contracts | _Non Critical_ |
| [NC-05](#NC-05)   | Multiple Usage of `address(this)` in `UTBExecutor.sol` | _Non Critical_ |
| [NC-06](#NC-06)   | Single Use of Stack Variables | _Non Critical_ |
| [NC-07](#NC-07)   | Use of Assembly for Math Equations | _Non Critical_ |
| [NC-08](#NC-08)   | Efficient Back-to-Back Calls Using Assembly | _Non Critical_ |
| [NC-09](#NC-09)   | Use of `storage` Instead of `memory` for Structs/Arrays | _Non Critical_ |
| [NC-10](#NC-10)   | Inefficient Use of `+=` and `-=` for State Variables | _Non Critical_ |

---

---

## <a name="L-01"></a>[L-01] Missing `payable` Modifier in `UTBExecutor.sol#execute()` Function

### GitHub Links

- [UTBExecutor.sol#execute()](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBExecutor.sol#L41-L81) function

### Description

The `execute()` function in the `UTBExecutor.sol` contract is intended to execute payment transactions, potentially involving the transfer of Ether. However, the function lacks the `payable` modifier, which is necessary for functions in Solidity to accept Ether. Without this modifier, any attempt to send Ether to this function will fail, limiting its utility in scenarios where Ether transfers are required.

```js
    function execute(
        address target,
        address paymentOperator,
        bytes memory payload,
        address token,
        uint amount,
        address payable refund,
        uint extraNative
    ) public onlyOwner {
        bool success;
        if (token == address(0)) {
            (success, ) = target.call{value: amount}(payload);
            if (!success) {
                (refund.call{value: amount}(""));
            }
            return;
        }

        uint initBalance = IERC20(token).balanceOf(address(this));

        IERC20(token).transferFrom(msg.sender, address(this), amount);
        IERC20(token).approve(paymentOperator, amount);

        if (extraNative > 0) {
            (success, ) = target.call{value: extraNative}(payload);
            if (!success) {
                (refund.call{value: extraNative}(""));
            }
        } else {
            (success, ) = target.call(payload);
        }

        uint remainingBalance = IERC20(token).balanceOf(address(this)) -
            initBalance;

        if (remainingBalance == 0) {
            return;
        }

        IERC20(token).transfer(refund, remainingBalance);
    }
```

### Recommendations

To resolve this issue, add the `payable` modifier to the `execute()` function definition in `UTBExecutor.sol`. This change will allow the function to accept Ether, enabling it to handle transactions involving native token transfers as intended.

---

## <a name="L-02"></a>[L-02] Lack of Ether Withdrawal Mechanism in Contracts with `receive/fallback` Functions

### GitHub Links

**Contracts with receive/fallback:**
- [UTB.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTB.sol)
- [UTBFeeCollector.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBFeeCollector.sol)
- [DecentBridgeAdapter.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/bridge_adapters/DecentBridgeAdapter.sol)
- [StargateBridgeAdapter.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/bridge_adapters/StargateBridgeAdapter.sol)
- [UniSwapper.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/swappers/UniSwapper.sol)

### Description

Several contracts in the repository have `receive()` and/or `fallback()` functions to accept Ether. However, there is no implemented mechanism for withdrawing Ether that is sent to these contracts. This oversight can lead to situations where Ether becomes locked within these contracts with no way to retrieve it, potentially resulting in loss of funds.

### Recommendations

Implement a secure withdrawal function in each contract that has a `receive()` or `fallback()` function. This function should allow authorized parties (such as the contract owner) to withdraw Ether from the contract. Ensure that proper access control mechanisms are in place to prevent unauthorized withdrawals.

---

## <a name="L-03"></a>[L-03] Inheritance Issues in `UTBFeeCollector.sol` and `UTBExecutor.sol`

### GitHub Links

- [UTBFeeCollector.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBFeeCollector.sol)
- [UTBExecutor.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBExecutor.sol)

### Description

The `UTBFeeCollector.sol` contract is expected to implement the functionalities of the `IUTBFeeCollector.sol` interface, and similarly, `UTBExecutor.sol` should implement `IUTBExecutor.sol`. However, neither contract currently inherits from their respective interfaces. This omission can lead to issues with contract interoperability and may result in the contracts not adhering to the expected standard behaviors defined in the interfaces.

### Recommendations

Update the `UTBFeeCollector.sol` and `UTBExecutor.sol` contracts to explicitly inherit from `IUTBFeeCollector.sol` and `IUTBExecutor.sol` interfaces, respectively. Ensure that all functions defined in the interfaces are properly implemented in these contracts. This change will enhance contract standardization and ensure compliance with the defined interface specifications.

---

## <a name="L-04"></a>[L-04] Use `quoteLayerZeroFee` instead of sending entire msg.value as gas fee for swap call (Inefficient Gas Fee Calculation for Stargate Router Swap in `StargateBridgeAdapter.sol`)

### GitHub Links

- [StargateBridgeAdapter.sol#callBridge()](https://github.com/code-423n4/2024-01-decent/blob/main/src/bridge_adapters/StargateBridgeAdapter.sol#L163-L181) function

### Explanation (of the code logic)

In the `StargateBridgeAdapter.sol` contract of the Decent Protocol, the `callBridge()` function is responsible for executing swap operations using the Stargate router. The function currently sends the entire `msg.value` (the native asset balance sent with the transaction) as the value for the swap call. This approach does not utilize the `quoteLayerZeroFee` method provided by Stargate's router, which is designed to accurately calculate the required gas fee based on the transaction parameters.

Essentially, the current implementation in `StargateBridgeAdapter.sol` of sending the entire `msg.value` as the gas fee for swap calls is not aligned with Stargate's recommended practices. Implementing the `quoteLayerZeroFee` method for precise fee calculation would optimize the contract's efficiency and fund usage.

```js
    function callBridge(
        uint256 amt2Bridge,
        uint256 dstChainId,
        bytes memory bridgePayload,
        bytes calldata additionalArgs,
        address payable refund
    ) private {
        router.swap{value: msg.value}(
            getDstChainId(additionalArgs), //lzBridgeData._dstChainId, // send to LayerZero chainId
            getSrcPoolId(additionalArgs), //lzBridgeData._srcPoolId, // source pool id
            getDstPoolId(additionalArgs), //lzBridgeData._dstPoolId, // dst pool id
            refund, // refund adddress. extra gas (if any) is returned to this address
            amt2Bridge, // quantity to swap
            (amt2Bridge * (10000 - SG_FEE_BPS)) / 10000, // the min qty you would accept on the destination, fee is 6 bips
            getLzTxObj(additionalArgs), // additional gasLimit increase, airdrop, at address
            abi.encodePacked(getDestAdapter(dstChainId)),
            bridgePayload // bytes param, if you wish to send additional payload you can abi.encode() them here
        );
    }
```

### Impact

- **Overpayment of Gas Fees**: The contract may overpay for gas fees by sending the entire balance, leading to inefficient use of funds.
- **Non-Optimal Fee Calculation**: The lack of precise fee calculation could result in either excess payment (with subsequent refund) or insufficient fee allocation, potentially affecting the success of the swap operation.
- **Refund Mechanism Dependency**: Relying on the gas refund mechanism may not always be optimal, especially in scenarios of high network congestion or fluctuating gas prices.

### Proof of Concept

- The `callBridge` function in `StargateBridgeAdapter.sol` uses `msg.value` for the swap call's gas fee without calculating the exact fee requirement.
- Stargate's documentation recommends using `quoteLayerZeroFee` for an accurate fee calculation, which is not implemented in the current contract code.

### Recommendations

- **Implement `quoteLayerZeroFee`**: Modify the `callBridge` function to use `quoteLayerZeroFee` from Stargate's router to calculate the exact gas fee required for the swap operation.

---

## <a name="NC-01"></a>[NC-01] Optimization of `_receiveAndWrapIfNeeded()` in `UniSwapper.sol#swap()`

### GitHub Links

- [UniSwapper.sol#swap() Function](https://github.com/code-423n4/2024-01-decent/blob/main/src/swappers/UniSwapper.sol)

### Description

The `_receiveAndWrapIfNeeded()` function in `UniSwapper.sol#swap()` is called in all three branches of the if-else statement. This redundant placement increases gas usage as the function is repeatedly called in each conditional branch.

### Recommendations

To optimize gas usage, move the `_receiveAndWrapIfNeeded()` function call before the if-else checks. This change ensures that the function is executed only once, regardless of which branch is taken, thereby reducing the overall gas cost of the `swap()` function.

---

## <a name="NC-02"></a>[NC-02] Use of `else if` in Conditional Statements

### GitHub Links

- [Conditional Block in UniSwapper.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/swappers/UniSwapper.sol#L73-L76)

### Description

In the provided code snippet, the use of `else` instead of `else if` for the second condition in the if-else block can lead to less readable and potentially error-prone code, especially if more conditions are added in the future.

### Recommendations

Replace the `else` with `else if (condition)` to explicitly state the condition for which the code block should execute. This enhances code readability and maintainability.

```js
    function swap(
        bytes memory swapPayload
    )
        external
        onlyUtb
        returns (address tokenOut, uint256 amountOut)
    {
        // ... code ...
        else {
            swapExactOut(swapParams, receiver, refund);
            amountOut = swapParams.amountOut;
        }
    }
```

---

## <a name="NC-03"></a>[NC-03] Inheritance and Use of `ecrecover` in `UTBFeeCollector.sol`

### GitHub Links

- [UTBFeeCollector.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBFeeCollector.sol)

### Description

The `UTBFeeCollector.sol` contract does not inherit from the `IUTBFeeCollector.sol` interface. Additionally, the contract uses the `ecrecover` function directly instead of the more secure and recommended OpenZeppelin's `ECDSA` library. The absence of a check for `address(0)` can lead to potential vulnerabilities.

### Recommendations

- Update `UTBFeeCollector.sol` to explicitly inherit from `IUTBFeeCollector.sol`.
- Replace direct usage of `ecrecover` with OpenZeppelin's `ECDSA` library for enhanced security and readability.
- Add a check to ensure the recovered address is not `address(0)` to prevent potential null address vulnerabilities.

---

## <a name="NC-04"></a>[NC-04] Inconsistency in WETH Token Addresses Across Contracts

### GitHub Links

- [Various Contracts in the Repository](https://github.com/code-423n4/2024-01-decent)

### Description

There is an inconsistency in the handling of WETH token addresses across different contracts. In some contracts, the WETH address can be changed, while in others, it cannot. This inconsistency can lead to issues, especially if the WETH address needs to be updated or synchronized across contracts.

### Recommendations

Implement a consistent mechanism for managing the WETH token address across all contracts. Consider using a central configuration contract or a similar pattern to ensure that the WETH address remains consistent and can be updated uniformly when necessary.

---

## <a name="NC-05"></a>[NC-05] Multiple Usage of `address(this)` in `UTBExecutor.sol`

### GitHub Links

- [UTBExecutor.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBExecutor.sol)

### Description

The `address(this)` is used multiple times within the `execute()` function in `UTBExecutor.sol`. Repeatedly accessing `address(this)` can lead to unnecessary gas costs.

### Recommendations

Cache `address(this)` in a local variable at the beginning of the function and use this variable throughout the function. This change will reduce gas costs by minimizing redundant access to the contract's address.

---

## <a name="NC-06"></a>[NC-06] Single Use of Stack Variables

### GitHub Links

- [Various Files and Lines](https://github.com/code-423n4/2024-01-decent)

### Description

Several instances in the code involve declaring stack variables that are only used once. This practice incurs additional gas costs due to unnecessary stack assignment.

### Recommendations

Directly use the assigned values in the expressions where they are needed, instead of assigning them to stack variables. This change will save gas by avoiding unnecessary stack operations.

---

## <a name="NC-07"></a>[NC-07] Use of Assembly for Math Equations

### GitHub Links

- [StargateBridgeAdapter.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/bridge_adapters/StargateBridgeAdapter.sol)

### Description

The use of standard arithmetic operations in Solidity can be less gas-efficient compared to using inline assembly, especially for complex mathematical expressions.

### Recommendations

Utilize assembly for the math equation `(amt2Bridge * (1e4 - SG_FEE_BPS)) / 1e4` to optimize gas usage. Assembly allows for more efficient computation by directly manipulating EVM opcodes.

---

## <a name="NC-08"></a>[NC-08] Efficient Back-to-Back Calls Using Assembly

### GitHub Links

- [Various Files and Lines](https://github.com/code-423n4/2024-01-decent)

### Description

Back-to-back external calls with similar function signatures and parameters can be optimized using assembly. This optimization can reuse the same memory space and potentially avoid memory expansion costs.

### Recommendations

Implement assembly code to handle back-to-back external calls efficiently. This approach can reuse function signatures, parameters, and memory space, leading to significant gas savings.

---

## <a name="NC-09"></a>[NC-09] Use of `storage` Instead of `memory` for Structs/Arrays

### GitHub Links

- [Various Files and Lines](https://github.com/code-423n4/2024-01-decent)

### Description

Assigning data fetched from storage to a `memory` variable incurs high gas costs due to multiple `SLOAD` operations. This issue is prevalent in several contracts where structs or arrays are unnecessarily stored in memory.

### Recommendations

Declare structs or arrays with the `storage` keyword instead of `memory` when they are fetched from storage. Cache any frequently accessed fields in local variables to minimize `SLOAD` operations.

---

## <a name="NC-10"></a>[NC-10] Inefficient Use of `+=` and `-=` for State Variables

### GitHub Links

- [DecentEthRouter.sol](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentEthRouter.sol)

### Description

Using `+=` and `-=` for state variables is less efficient than direct arithmetic operations. This inefficiency is found in `DecentEthRouter.sol`.

### Recommendations

Replace `+=` and `-=` with direct arithmetic operations (e.g., `x = x + y`) for state variables to optimize gas usage. This change will reduce the gas cost of these operations.

### GitHub Links

- [UTBFeeCollector.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBFeeCollector.sol)
- [UTBExecutor.sol](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBExecutor.sol)

### Description

The `UTBFeeCollector.sol` contract is expected to implement the functionalities of the `IUTBFeeCollector.sol` interface, and similarly, `UTBExecutor.sol` should implement `IUTBExecutor.sol`. However, neither contract currently inherits from their respective interfaces. This omission can lead to issues with contract interoperability and may result in the contracts not adhering to the expected standard behaviors defined in the interfaces.

### Recommendations

Update the `UTBFeeCollector.sol` and `UTBExecutor.sol` contracts to explicitly inherit from `IUTBFeeCollector.sol` and `IUTBExecutor.sol` interfaces, respectively. Ensure that all functions defined in the interfaces are properly implemented in these contracts. This change will enhance contract standardization and ensure compliance with the defined interface specifications.

---

1. Move `_receiveAndWrapIfNeeded()` function before if-else checks in `UniSwapper.sol#swap()` function, because in all of the three if-else cases this function is executed, this will reduce contract gas
2. In the code block below use `else if` rather than just `else`

    ```solidity
    if (swapParams.direction == SwapDirection.EXACT_IN) {
    	amountOut = swapExactIn(swapParams, receiver);
    } else {
    	swapExactOut(swapParams, receiver, refund);
      amountOut = swapParams.amountOut;
    }
    ```

3. Use OpenZeppelin's `ECDSA` library rather than `ecrecover` in `UTBFeeCollector.sol#collectFees()` function and also add check for `address(0)`
4. in some contracts weth can be changed, but in other cannot. also there is no mechanism that ensure the **sameness of this weth token addresses**
5. `address(this)` should be cached when used multiple times

    There is one instance of this issue

    ```solidity
    ```solidity

    📁 File: src/UTBExecutor.sol

    /// @audit 'address(this)' used 3 times
    41: function execute(
    42: address target,
    43: address paymentOperator,
    44: bytes memory payload,
    45: address token,
    46: uint amount,
    47: address payable refund,
    48: uint extraNative
    49: ) public onlyOwner {

    ```
    [41](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBExecutor.sol/#L41-L49)
    ```

6. Stack variable is only used once
If the variable is only accessed once, it's cheaper to use the assigned value directly that one time, and save the 3 gas the extra stack assignment would spend.

Gas saved per Instance: ~3 *(Total: ~45)*
<details>
<summary><i>There are 15 instances of this issue:</i></summary>

```solidity
📁 File: lib/decent-bridge/src/DecentBridgeExecutor.sol

30:         uint256 balanceBefore = weth.balanceOf(address(this));

40:         uint256 remainingAfterCall = amount -
```

[30](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentBridgeExecutor.sol/#L30-L30), [40](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentBridgeExecutor.sol/#L40-L40)

```solidity
📁 File: lib/decent-bridge/src/DecentEthRouter.sol

96:         uint256 GAS_FOR_RELAY = 100000;
97:         uint256 gasAmount = GAS_FOR_RELAY + _dstGasForCall;

170:         ICommonOFT.LzCallParams memory callParams = ICommonOFT.LzCallParams({
```

[96](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentEthRouter.sol/#L96-L97), [170](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentEthRouter.sol/#L170-L170)

```solidity
📁 File: src/UTB.sol

287:         bool native = approveAndCheckIfNative(instructions, amt2Bridge);

326:         ISwapper s = ISwapper(swapper);

335:         IBridgeAdapter b = IBridgeAdapter(bridge);
```

[287](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTB.sol/#L287-L287), [326](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTB.sol/#L326-L326), [335](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTB.sol/#L335-L335)

```solidity
📁 File: src/UTBExecutor.sol

59:         uint initBalance = IERC20(token).balanceOf(address(this));
```

[59](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBExecutor.sol/#L59-L59)

```solidity
📁 File: src/UTBFeeCollector.sol

49:         bytes32 constructedHash = keccak256(

53:         address recovered = ecrecover(constructedHash, v, r, s);
```

[49](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBFeeCollector.sol/#L49-L49), [53](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBFeeCollector.sol/#L53-L53)

```solidity
📁 File: src/bridge_adapters/DecentBridgeAdapter.sol

51:         SwapParams memory swapParams = abi.decode(

96:         uint64 dstGas = abi.decode(additionalArgs, (uint64));

103:         SwapParams memory swapParams = abi.decode(
```

[51](https://github.com/code-423n4/2024-01-decent/blob/main/src/bridge_adapters/DecentBridgeAdapter.sol/#L51-L51), [96](https://github.com/code-423n4/2024-01-decent/blob/main/src/bridge_adapters/DecentBridgeAdapter.sol/#L96-L96), [103](https://github.com/code-423n4/2024-01-decent/blob/main/src/bridge_adapters/DecentBridgeAdapter.sol/#L103-L103)

```solidity
📁 File: src/swappers/UniSwapper.sol

129:         IV3SwapRouter.ExactInputParams memory params = IV3SwapRouter
```

[129](https://github.com/code-423n4/2024-01-decent/blob/main/src/swappers/UniSwapper.sol/#L129-L129)

</details>

7. Use assembly for math equations

It is cheaper to use `div(mul(a, b), c)` instead of `(a * b) / c`. https://gist.github.com/notbozho/9ca20f4c6c837a33fab723fecd861f24

Gas saved per Instance: ~260

<i>There is one instance of this issue:</i>

```solidity
📁 File: src/bridge_adapters/StargateBridgeAdapter.sol

65:     ) external pure returns (uint256) {
66:         return (amt2Bridge * (1e4 - SG_FEE_BPS)) / 1e4;
67:     }
```

[65](https://github.com/code-423n4/2024-01-decent/blob/main/src/bridge_adapters/StargateBridgeAdapter.sol/#L65-L67)

##### Recommended Mitigation Steps:

```solidity
function exampleFunction(uint256 a, uint256 b) external view returns (uint256 result) {
  assembly {
      result := div(mul(a,b), 2)
  }
}
```

---

8. Use assembly to perform efficient back-to-back calls

If a similar external call is performed back-to-back, we can use assembly to reuse any function signatures and function parameters that stay the same. In addition, we can also reuse the same memory space for each function call (`scratch space` + `free memory pointer` + `zero slot`), which can potentially allow us to avoid memory expansion costs.

Gas saved per Instance: ~300 *(Total: ~1,200)*
<details>
<summary><i>There are 4 instances of this issue:</i></summary>

```solidity
📁 File: lib/decent-bridge/src/DecentBridgeExecutor.sol

36:             weth.transfer(from, amount);

44:         weth.transfer(from, remainingAfterCall);
```

[36](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentBridgeExecutor.sol/#L36-L36), [44](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentBridgeExecutor.sol/#L44-L44)

```solidity
📁 File: src/UTBFeeCollector.sol

71:             payable(owner).transfer(amount);

73:             IERC20(token).transfer(owner, amount);
```

[71](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBFeeCollector.sol/#L71-L71), [73](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTBFeeCollector.sol/#L73-L73)

</details>

---

9. Using `storage` instead of `memory` for structs/arrays saves gas

When fetching data from a storage location, assigning the data to a `memory` variable causes all fields of the struct/array to be read from storage, which incurs a Gcoldsload (**2100 gas**) for *each* field of the struct/array. If the fields are read from the new memory variable, they incur an additional `MLOAD` rather than a cheap stack read. Instead of declearing the variable with the `memory` keyword, declaring the variable with the `storage` keyword and caching any fields that need to be re-read in stack variables, will be much cheaper, only incuring the Gcoldsload for the fields actually read. The only time it makes sense to read the whole struct/array into a `memory` variable, is if the full struct/array is being returned by the function, is being passed to a function that requires `memory`, or if the array/struct is being read from another `memory` array/struct

Gas saved per Instance: ~2,100 *(Total: ~18,900)*
<details>
<summary><i>There are 9 instances of this issue:</i></summary>

```solidity
📁 File: lib/decent-bridge/src/DecentEthRouter.sol

170:         ICommonOFT.LzCallParams memory callParams = ICommonOFT.LzCallParams({
171:             refundAddress: payable(msg.sender),
172:             zroPaymentAddress: address(0x0),
173:             adapterParams: adapterParams
174:         });
```

[170](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentEthRouter.sol/#L170-L174)

```solidity
📁 File: src/UTB.sol

69:         SwapParams memory swapParams = abi.decode(
70:             swapInstructions.swapPayload,
71:             (SwapParams)
72:         );

181:         SwapParams memory newPostSwapParams = abi.decode(
182:             instructions.postBridge.swapPayload,
183:             (SwapParams)
184:         );
```

[69](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTB.sol/#L69-L72), [181](https://github.com/code-423n4/2024-01-decent/blob/main/src/UTB.sol/#L181-L184)

```solidity
📁 File: src/bridge_adapters/DecentBridgeAdapter.sol

51:         SwapParams memory swapParams = abi.decode(
52:             postBridge.swapPayload,
53:             (SwapParams)
54:         );

103:         SwapParams memory swapParams = abi.decode(
104:             postBridge.swapPayload,
105:             (SwapParams)
106:         );

134:         SwapParams memory swapParams = abi.decode(
135:             postBridge.swapPayload,
136:             (SwapParams)
137:         );
```

[51](https://github.com/code-423n4/2024-01-decent/blob/main/src/bridge_adapters/DecentBridgeAdapter.sol/#L51-L54), [103](https://github.com/code-423n4/2024-01-decent/blob/main/src/bridge_adapters/DecentBridgeAdapter.sol/#L103-L106), [134](https://github.com/code-423n4/2024-01-decent/blob/main/src/bridge_adapters/DecentBridgeAdapter.sol/#L134-L137)

```solidity
📁 File: src/bridge_adapters/StargateBridgeAdapter.sol

202:         SwapParams memory swapParams = abi.decode(
203:             postBridge.swapPayload,
204:             (SwapParams)
205:         );
```

[202](https://github.com/code-423n4/2024-01-decent/blob/main/src/bridge_adapters/StargateBridgeAdapter.sol/#L202-L205)

```solidity
📁 File: src/swappers/UniSwapper.sol

129:         IV3SwapRouter.ExactInputParams memory params = IV3SwapRouter
130:             .ExactInputParams({
131:                 path: swapParams.path,
132:                 recipient: address(this),
133:                 amountIn: swapParams.amountIn,
134:                 amountOutMinimum: swapParams.amountOut
135:             });

149:         IV3SwapRouter.ExactOutputParams memory params = IV3SwapRouter
150:             .ExactOutputParams({
151:                 path: swapParams.path,
152:                 recipient: address(this),
153:                 //deadline: block.timestamp,
154:                 amountOut: swapParams.amountOut,
155:                 amountInMaximum: swapParams.amountIn
156:             });
```

[129](https://github.com/code-423n4/2024-01-decent/blob/main/src/swappers/UniSwapper.sol/#L129-L135), [149](https://github.com/code-423n4/2024-01-decent/blob/main/src/swappers/UniSwapper.sol/#L149-L156)

</details>

---

10. x + y is more efficient than using += for state variables (likewise for -=)

In instances found where either += or -= are used against state variables use x = x + y instead

Gas saved per Instance: ~248 *(Total: ~496)*

<i>There are 2 instaces of this issue:</i>

```solidity
📁 File: lib/decent-bridge/src/DecentEthRouter.sol

56:         balanceOf[msg.sender] += amount;

64:         balanceOf[msg.sender] -= amount;
```

[56](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentEthRouter.sol/#L56), [64](https://github.com/decentxyz/decent-bridge/blob/7f90fd4489551b69c20d11eeecb17a3f564afb18/src/DecentEthRouter.sol/#L64)

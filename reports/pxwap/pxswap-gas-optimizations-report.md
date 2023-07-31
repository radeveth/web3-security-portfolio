# Introduction
Gas Optimization report of the **pxswap** protocol was done by Radev.

# About **Radev**
Radoslav Radev, or **Radev**, is an independent smart contract security researcher. Reach out on Twitter [@radev_eth](https://twitter.com/radev_eth).

# About **pxswap**
NFT OTC Trading platform, the Future of NFT Finance

# Executive summary

### Overview

|               |                                                                                                                                                   |
| :------------ | :------------------------------------------------------------------------------------------------------------------------------------------------ |
| Project Name  | pxswap                                                                                                                                            |
| Repository    | https://github.com/pxswap-xyz/pxswap/tree/main                                                                                                    |
| Commit hash	  | [4d20e98c5beb02934ae63ac174abe04227a53a0f](https://github.com/pxswap-xyz/pxswap/tree/4d20e98c5beb02934ae63ac174abe04227a53a0f)                    |
| Website       | [Link](https://www.pxswap.xyz/)                                                                                                                   |
| Twitter       | [Link](https://twitter.com/pxswap_xyz)                                                                                                            |

### Gas Optimizations Summary

|Number|Issue|Instances|Total Gas Saved|
|-|:-|:-:|:-:|
| [GAS-1](#GAS-1) | Use `selfbalance()` instead of `address(this).balance` | 1 | - |
| [GAS-2](#GAS-2) | Avoid updating storage when the value hasn't changed | 1 | 1700 |
| [GAS-3](#GAS-3) | Consider activating `via-ir` for deploying | 0 | 250 |
| [GAS-4](#GAS-4) | Use Custom Errors | 2 | 58 |
| [GAS-5](#GAS-5) | Use assembly to emit events | 3 | 114 |
| [GAS-6](#GAS-6) | Function names can be optimized | 1 | 128 |
| [GAS-7](#GAS-7) | Don't initialize variables with default value | 4 | 84 |
| [GAS-8](#GAS-8) | Functions guaranteed to revert when called by normal users can be marked `payable` | 2 | 42 |
| [GAS-9](#GAS-9) | Variables need not be initialized to zero | 4 | - |

👉 Total saving ~ 2126 gas

**❗ Disclaimer:** Gas totals are estimates based on data from the Ethereum Yellowpaper. The estimates use the lower bounds of ranges and count two iterations of each `for`-loop. All values above are runtime, not deployment, values; deployment values are listed in the individual issue descriptions. The table above as well as its gas numbers do not include any of the excluded findings.

---

### <a name="GAS-1"></a>[GAS-1] Use `selfbalance()` instead of `address(this).balance`
Use assembly when getting a contract's balance of ETH.

You can use `selfbalance()` instead of `address(this).balance` when getting your contract's balance of ETH to save gas.
Additionally, you can use `balance(address)` instead of `address.balance()` when getting an external contract's balance of ETH.

*Saves 15 gas when checking internal balance, 6 for external*

<details>
<summary><i>Instances (1):</i></summary>

```solidity
File: src/Pxswap.sol

152:         (bool sent, ) = payable(owner()).call{value: address(this).balance}(""); 

```

</details>

### <a name="GAS-2"></a>[GAS-2] Avoid updating storage when the value hasn't changed
If the old value is equal to the new value, not re-storing the value will avoid a Gsreset (2900 gas), potentially at the expense of a Gcoldsload (2100 gas) or a Gwarmaccess (100 gas)


Gas saved per Instance: ~1700
<details>
<summary><i>Instances (1):</i></summary>

```solidity
File: src/Pxswap.sol

147:     function setFee(uint256 _fee) external onlyOwner { 
148:         fee = _fee; // 🔥 Found
149:     } 

```

</details>

### <a name="GAS-3"></a>[GAS-3] Consider activating `via-ir` for deploying
The IR-based code generator was introduced with an aim to not only allow code generation to be more transparent and auditable but also to enable more powerful optimization passes that span across functions.You can enable it on the command line using `--via-ir` or with the option `{"viaIR": true}`.This will take longer to compile, but you can just simple test it before deploying and if you got a better benchmark then you can add --via-ir to your deploy commandMore on: https://docs.soliditylang.org/en/v0.8.17/ir-breaking-changes.html

### <a name="GAS-4"></a>[GAS-4] Use Custom Errors
[Source](https://blog.soliditylang.org/2021/04/21/custom-errors/)
Instead of using error strings, to reduce deployment and runtime cost, you should use Custom Errors. This would save both deployment and runtime cost.


Gas saved per Instance: ~29
<details>
<summary><i>Instances (2):</i></summary>

```solidity
File: src/Pxswap.sol

153:         require(sent, "Failed to send Ether"); 

```

</details>

### <a name="GAS-5"></a>[GAS-5] Use assembly to emit events
We can use assembly to emit events efficiently by utilizing `scratch space` and the `free memory pointer`. This will allow us to potentially avoid memory expansion costs.
Note: In order to do this optimization safely, we will need to cache and restore the free memory pointer.

For example, for a generic `emit` event for `eventSentAmountExample`: 
```solidity
// uint256 id, uint256 value, uint256 amount
emit eventSentAmountExample(id, value, amount);
```

We can use the following assembly emit events:

```solidity
        assembly {
            let memptr := mload(0x40)
            mstore(0x00, calldataload(0x44))
            mstore(0x20, calldataload(0xa4))
            mstore(0x40, amount) 
            log1(
                0x00,
                0x60,
                // keccak256("eventSentAmountExample(uint256,uint256,uint256)")
                0xa622cf392588fbf2cd020ff96b2f4ebd9c76d7a4bc7f3e6b2f18012312e76bc3
            )
            mstore(0x40, memptr)
        }
```


Gas saved per Instance: ~38
<details>
<summary><i>Instances (3):</i></summary>

```solidity
File: src/Pxswap.sol

58:         emit IPxswap.TradeOpened(newTradeId, offerNfts, requestNfts); 

83:         emit IPxswap.TradeCanceled(tradeId); 

131:        emit IPxswap.TradeAccepted(tradeId); 

```

</details>

### <a name="GAS-6"></a>[GAS-6] Function names can be optimized
Function that are `public`/`external` and `public` state variable names can be optimized to save gas.

Method IDs that have two leading zero bytes can save **128 gas** each during deployment, and renaming functions to have lower method IDs will save **22 gas** per call, per sorted position shifted. [Reference](https://blog.emn178.cc/en/post/solidity-gas-optimization-function-name/)


Gas saved per Instance: ~128
<details>
<summary><i>Instances (1):</i></summary>

```solidity
File: src/Pxswap.sol

27: contract Pxswap is IPxswap, ERC721Holder, ReentrancyGuard, Ownable { 

```

</details>

### <a name="GAS-7"></a>[GAS-7] Don't initialize variables with default value

Gas saved per Instance: ~21
<details>
<summary><i>Instances (4):</i></summary>

```solidity
File: src/Pxswap.sol

49:         for (uint256 i = 0; i < lNft;) { 

74:         for (uint256 i = 0; i < lOfferedNfts;) { 

109:        for (uint256 i = 0; i < lNft;) { 

122:        for (uint256 i = 0; i < lOfferedNfts;) { 

```

</details>

### <a name="GAS-8"></a>[GAS-8] Functions guaranteed to revert when called by normal users can be marked `payable`
If a function modifier such as `onlyOwner` is used, the function will revert if a normal user tries to pay the function. Marking the function as `payable` will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided.


Gas saved per Instance: ~21
<details>
<summary><i>Instances (2):</i></summary>

```solidity
File: src/Pxswap.sol

147:     function setFee(uint256 _fee) external onlyOwner { 

151:     function withdrawFees() external onlyOwner { 

```

</details>

### <a name="GAS-09"></a>[GAS-09] Variables need not be initialized to zero
The default value for variables is zero, so initializing them to zero is superfluous.

<details>
<summary><i>Instances (4):</i></summary>

```solidity
File: src/Pxswap.sol

49:         for (uint256 i = 0; i < lNft;) { 

74:         for (uint256 i = 0; i < lOfferedNfts;) { 

109:         for (uint256 i = 0; i < lNft;) { 

122:         for (uint256 i = 0; i < lOfferedNfts;) { 

```

</details>

<details>
<summary><i>Code Snippet (4):</i></summary>

- https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol#L49

- https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol#L74

- https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol#L109

- https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol#L122
        
</details>

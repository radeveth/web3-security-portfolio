# Introduction
A time-boxed security review of the **pxswap** protocol was done by **Radev**, with a focus on the security aspects of the application's smart contracts implementation.

# Disclaimer
A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# About **Radev**
Radoslav Radev, or **Radev**, is an independent smart contract security researcher. Reach out on Twitter [@radev_eth](https://twitter.com/radev_eth).

# About **pxswap**
NFT OTC Trading platform, the Future of NFT Finance

# Severity classification
| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

### Impact
- **High** - leads to a significant material loss of assets in the protocol or significantly harms a group of users.
- **Medium** - only a small amount of funds can be lost (such as leakage of value) or a core functionality of the protocol is affected.
- **Low** - can lead to any kind of unexpected behaviour with some of the protocol's functionalities that's not so critical.

### Likelihood
- **High** - attack path is possible with reasonable assumptions that mimic on-chain conditions and the cost of the attack is relatively low to the amount of funds that can be stolen or lost.
- **Medium** - only conditionally incentivized attack vector, but still relatively likely.
- **Low** - has too many or too unlikely assumptions or requires a huge stake by the attacker with little or no incentive.

### Actions required by severity level
- **Critical** - client **must** fix the issue.
- **High** - client **must** fix the issue.
- **Medium** - client **should** fix the issue.
- **Low** - client **could** fix the issue.

# Executive summary

### Overview

|               |                                                                                                                                                   |
| :------------ | :------------------------------------------------------------------------------------------------------------------------------------------------ |
| Project Name  | pxswap                                                                                                                                            |
| Repository    | https://github.com/pxswap-xyz/pxswap/tree/main                                                                                                    |
| Commit hash	  | [4d20e98c5beb02934ae63ac174abe04227a53a0f](https://github.com/pxswap-xyz/pxswap/tree/4d20e98c5beb02934ae63ac174abe04227a53a0f)                    |
| Website       | [Link](https://www.pxswap.xyz/)                                                                                                                   |
| Twitter       | [Link](https://twitter.com/pxswap_xyz)                                                                                                            |
| Methods       | Manual review                                                                                                                                     |


### Scope

| File                                                                                                                                                                         | SLOC | nSLOC |
| :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---: | ----: |
| _Contracts (1)_                                                                                                                                                              |      |       |
| [src/Pxswap.sol](https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol)                                                          |  156 |   106 |
| _Abstracts (1)_                                                                                                                                                              |      |       |
| [abstract/ReentrancyGuard.sol](https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/abstract/ReentrancyGuard.sol)                          |   37 |    37 |
| _Interfaces (1)_                                                                                                                                                             |      |       |
| [interfaces/IPxswap.sol](https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/interfaces/IPxswap.sol)                                      |   10 |    10 |
| _Total (3)_                                                                                                                                                                  |  203 |   153 |

---

# Findings Summary

| ID     | Title                                                                                     | Severity         |
| ------ | ------------------------------------------------------------------------------------------| ---------------- |
| [M-01] | Manipulations of `setFee()`                                                               | Medium           |
| [L-01] | Users can trade over the protocol without paying any fee                                  | Low              |
| [L-02] | Missing deadline checks allow pending transactions to be maliciously executed             | Low              |
| [L-03] | External call recipient may consume all transaction gas (gas grief pssible)               | Low              |
| [L-04] | Unnecessary Storage Retention of completed trades in `trades` mapping                     | Low              |
| [I-01] | `Pxswap.sol` contract uses both `require()`/`revert()` as well as custom errors           | Informational    |
| [I-02] | Consider disabling `renounceOwnership()` in `Pxswap.sol` contract                         | Informational    |
| [I-03] | Functions calling contracts/addresses with transfer hooks are missing reentrancy guards   | Informational    |
| [I-04] | Consider adding emergency-stop functionality                                              | Informational    |
| [I-05] | Import declarations should import specific identifiers, rather than the whole file        | Informational    |
| [I-06] | Consider using named mappings                                                             | Informational    |
| [I-07] | Use solidity version 0.8.21 to gain some gas boost                                        | Informational    |

# Detailed Findings 👇



# Medium Severity

## [M-01] Manipulations of `setFee()`

### Severity

**Impact:**
High, as it will charge users more than the should be charged.

**Likelihood:**
Low, as it requires a malicious/compromised owner.


### Summary
Currently, the `setFee` function in `Pxswap.sol` contract do not have an upper bound on the fee rate being set by the owner. This opens up a centralization attack vector, where the owner can front-run trades by setting a bigger fee.


### Vulnerability Detail
If we consider that the fee variable is meaningfully applied, there will still be several problems with this:

1. Admin can set the fee up to whatever ether he wants. This is bad for users!

3. Owner could front run the `acceptTrade()` function. Let's consider the following scenario:
  - (2.1) Considering fee is 1 ETH, Alice is okay with that and call `acceptTrade()` function and is ready to pay the 1 ETH fee. She submits a transaction to the mempool.
  - (2.2) Meanwhile, the contract owner monitor the mempool, notices Alice's transaction and quickly calls `setFee()` with a higher fee, let's say 5 ETH, and pays a higher gas price to prioritize their transaction.
  - (2.3) When Alice's transaction is processed, she ends up paying 5 ETH as the fee instead of the intended 1 ETH, as the owner successfully `front-ran her transaction`. The owner can then call withdrawFees() to collect the all fees.


### Code Snippet
https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol#L147-L149

### Recommandation
- Fees should have a reasonable upper limit for prevent potential griefing.
- Add a timelock to change fees. In that way, frontrunning wouldn't be possible and users would know the fees they are agreeing with.



# Low Severity

## [L-01] Users can trade over the protocol without paying any fee
### Severity

**Impact:**
Low.

**Likelihood:**
Low.


### Summary
The default value of uint256 is 0, so when the `fee` state variable in `Pxswap.sol` is declared it will be 0.

### Vulnerability Detail
This open a time to users to trade over the protocol without paying any fee, before the owner call `setFee()` function and set the fee.

### Code Snippet
https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol#L32-L34

### Recommandation
Add constructor to the `Pxswap.sol` contract with _fee parameter, so the msg.sender (creator/owner) to can set initial fee. Also you can set initial fee in the declaration of the fee variable. 

```solidity
    constructor(uint256 _fee) {
        fee = _fee
    }
```
or
```solidity
    uint256 public fee = 1;
```


## [L-02] Missing deadline checks allow pending transactions to be maliciously executed

### Severity

**Impact:**
Medium, as users can unknowingly perform bad trades.

**Likelihood:**
Low.


### Summary
The `Pxswap.sol` contract does not allow users to submit a deadline for their trades. This missing feature enables pending transactions to be maliciously executed at a later point.


### Vulnerability Detail
The most common solution is to include a deadline timestamp as a parameter. If such an option is not present, users can unknowingly perform bad trades:

1. Alice wants to execute particular trade. She signs the transaction calling `Pxswap.sol#acceptTrade()`.
2. The transaction is submitted to the mempool, however, Alice chose a transaction fee that is too low for miners to be interested in including her transaction in a block. The transaction stays pending in the mempool for extended periods, which could be hours, days, weeks, or even longer.
3. When the average gas fee dropped far enough for Alice's transaction to become interesting again for miners to include it, her trade will be executed. In the meantime, the price of ETH could have drastically changed. She has unknowingly performed a bad trade due to the pending transaction she forgot about.

### Code Snippet
https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol#L86-L132

### Recommandation
Introduce a deadline parameter to the `acceptTrade()` function.


## [L-03] External call recipient may consume all transaction gas (gas grief pssible)
### Severity

**Impact:**
Low.

**Likelihood:**
Low.

### Summary
`withdrawFees()` function contains low level call.
```solidity
        (bool sent, ) = payable(owner()).call{value: address(this).balance}("");
```

### Vulnerability Detail
There is no limit specified on the amount of gas used, so the recipient can use up all of the transaction's gas, causing it to revert.

### Code Snippet
https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol#L152

### Recommandation
Use `addr.call{gas: <amount>}("")` or [this](https://github.com/nomad-xyz/ExcessivelySafeCall) library instead.


## [L-04] Unnecessary Storage Retention of completed trades in `trades` mapping
### Severity

**Impact:**
Medium: Unnecessary storage consumption due to completed trades remaining in the contract's state.

**Likelihood:**
Low.

### Summary
Completed trades remain permanently in the `trades` mapping, leading to inefficiencies and potential state bloat. (There are no way to remove trade from `trades` mapping which can leads to a problem)

### Vulnerability Detail
- The contract includes functions to open, accept, and cancel trades, but once a trade is added to the trades mapping, it remains there permanently.
- Even after a trade is accepted or canceled, its details remain in the contract's state. This includes the addresses and tokens involved in the trade.
- Over time, as more trades are completed, this could contribute to a continuous growth in the contract's storage size, leading to increased costs and potential state bloat.
- While not a direct security vulnerability, this behavior could have implications for the efficiency, usability, and privacy of the contract.

### Code Snippet
https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol#L31

### Recommandation
Completed trades from the trades mapping can be deleted in the `acceptTrade()` function, because if one trade is "accept trade", it will not be used anymore.

Also, the private variable `_tradeId` should be decremented. 



# Informational

## [I-01] `Pxswap.sol` contract uses both `require()`/`revert()` as well as custom errors

### Summary
Consider using just one method in a single file.
```solidity
        require(msg.value == fee, "Incorrect fee sent!");
        if(msg.value != fee){
            revert Errors.PAY_FEE();
        }
```

### Code Snippet
https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol#L91-L94


## [I-02] Consider disabling `renounceOwnership()` in `Pxswap.sol` contract

### Summary
Typically, the contract's owner is the account that deploys the contract. As a result, the owner is able to perform certain privileged activities. The OpenZeppelin's Ownable is used in this project contract implements renounceOwnership. This can represent a certain risk if the ownership is renounced for any other reason than by design. Renouncing ownership will leave the contract without an owner, thereby removing any functionality that is only available to the owner.


## Functions calling contracts/addresses with transfer hooks are missing reentrancy guards

### Summary
Adherence to the check-effects-interaction pattern is commendable, but without a reentrancy guard in functions, especially with transfer hooks (in the context of `_beforeTokenTransfer` and `_afterTokenTransfer` in [ERC721](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/ERC721.sol)), users are exposed to read-only reentrancy risks. This can lead to malicious actions without altering the contract state. Adding a reentrancy guard is vital for security, protecting against both traditional and read-only reentrancy attacks, ensuring a robust and safe protocol.

###
Code Snippet:
https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol#L35-L60


## [I-04] Consider adding emergency-stop functionality

### Summary
In the event of a security breach or any unforeseen emergency, swiftly suspending all protocol operations becomes crucial. Having a mechanism in place to halt all functions collectively, instead of pausing individual contracts separately, substantially enhances the efficiency of mitigating ongoing attacks or vulnerabilities. This not only quickens the response time to potential threats but also reduces operational stress during these critical periods. Therefore, consider integrating a 'circuit breaker' or 'emergency stop' function into the smart contract system architecture. Such a feature would provide the capability to suspend the entire protocol instantly, which could prove invaluable during a time-sensitive crisis management situation.


## [I-05] Import declarations should import specific identifiers, rather than the whole file

### Summary
Using import declarations of the form `import {<identifier_name>} from "some/file.sol"` avoids polluting the symbol namespace making flattened files smaller, and speeds up compilation (but does not save any gas).

### Code Snippet
https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol#L11-L18


## [I-06] Consider using named mappings

### Summary
Consider moving to solidity version 0.8.18 or later, and using [named mappings](https://ethereum.stackexchange.com/questions/51629/how-to-name-the-arguments-in-mapping/145555#145555) to make it easier to understand the purpose of each mapping.


## [I-07] Use solidity version 0.8.21 to gain some gas boost

### Summary
Upgrade to the latest solidity version 0.8.21 to get additional gas savings

See latest release for reference: https://blog.soliditylang.org/2023/05/10/solidity-0.8.20-release-announcement/

# Introduction
NatSpec Review of the **pxswap** protocol was done by Radev.

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


### Files NatSpec Review Summary

|Files|
|-|
| [Pxswap.sol](https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol) |
| [IPxswap.sol](https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/interfaces/IPxswap.sol) |
| [ReentrancyGuard.sol](https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/abstract/ReentrancyGuard.sol) |
| [DataTypes.sol](https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/types/DataTypes.sol) |





## [Pxswap.sol](https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/Pxswap.sol)

<details>

<summary><i>Changed File Version</i></summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.19;

import "lib/openzeppelin-contracts/contracts/token/ERC721/ERC721.sol";
import "lib/openzeppelin-contracts/contracts/token/ERC721/utils/ERC721Holder.sol";
import "lib/openzeppelin-contracts/contracts/utils/Counters.sol";
import "lib/openzeppelin-contracts/contracts/access/Ownable.sol";
import {ReentrancyGuard} from "./abstract/ReentrancyGuard.sol";
import {IPxswap} from "./interfaces/IPxswap.sol";
import {DataTypes} from "./types/DataTypes.sol";
import {Errors} from "./libraries/Errors.sol";

/**
 * @title pxswap
 * @author pxswap (https://github.com/pxswap-xyz/pxswap)
 * @author Ali Konuk - @alikonuk1
 * @dev This contract is for P2P trading non-fungible tokens (NFTs)
 * @dev Please reach out to ali@pxswap.xyz regarding this contract
 */
contract Pxswap is IPxswap, ERC721Holder, ReentrancyGuard, Ownable {
    using Counters for Counters.Counter;

    /// @notice Counter for trade IDs
    Counters.Counter private _tradeId;
    
    /// @notice Mapping of trade IDs to Trade structures
    mapping(uint256 => DataTypes.Trade) public trades;

    /// @notice Trading fee amount
    uint256 public fee;

    /**
     * @notice Open a new trade
     * @param offerNfts Addresses of the NFTs being offered
     * @param offerNftIds IDs of the NFTs being offered
     * @param requestNfts Addresses of the NFTs being requested
     */
    function openTrade(
        address[] calldata offerNfts,
        uint256[] calldata offerNftIds,
        address[] calldata requestNfts
    ) external {
        // Implementation
    }

    /**
     * @notice Cancel an existing trade
     * @param tradeId ID of the trade to be canceled
     */
    function cancelTrade(uint256 tradeId) external nonReentrant {
        // Implementation
    }

    /**
     * @notice Accept an existing trade
     * @param tradeId ID of the trade to be accepted
     * @param tokenIds IDs of the tokens involved in the trade
     */
    function acceptTrade(uint256 tradeId, uint256[] calldata tokenIds)
        external
        payable
        nonReentrant
    {
        // Implementation
    }

    /**
     * @notice Retrieve the offers of a given trade
     * @param tradeId ID of the trade
     * @return Addresses and IDs of the offered NFTs, requested NFTs, and trade open status
     */
    function getOffers(uint256 tradeId)
        external
        view
        returns (address[] memory, uint256[] memory, address[] memory, bool)
    {
        // Implementation
    }

    /**
     * @notice Set the trading fee
     * @param _fee The new fee amount
     */
    function setFee(uint256 _fee) external onlyOwner {
        fee = _fee;
    }

    /**
     * @notice Withdraw collected fees
     */
    function withdrawFees() external onlyOwner {
        // Implementation
    }
}
```

</details>



## [IPxswap.sol](https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/interfaces/IPxswap.sol)

<details>

<summary><i>Changed File Version</i></summary>

```solidity

// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.19;

/**
 * @title IPxswap
 * @dev Interface for the Pxswap contract, defining the main events for trading operations.
 */
interface IPxswap {
    /**
     * @dev Emitted when a trade has been opened.
     * @param tradeId The unique identifier for the trade.
     * @param nfts Array of NFT addresses being offered in the trade.
     * @param requestNfts Array of NFT addresses that the initiator wants to receive.
     */
    event TradeOpened(
        uint256 indexed tradeId, address[] indexed nfts, address[] indexed requestNfts
    );

    /**
     * @dev Emitted when a trade has been canceled.
     * @param tradeId The unique identifier for the trade that was canceled.
     */
    event TradeCanceled(uint256 indexed tradeId);

    /**
     * @dev Emitted when a trade has been accepted and completed.
     * @param tradeId The unique identifier for the trade that was accepted.
     */
    event TradeAccepted(uint256 indexed tradeId);
}

```

</details>


## [ReentrancyGuard.sol](https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/abstract/ReentrancyGuard.sol)

<details>

<summary><i>Changed File Version</i></summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.19;

/**
 * @title ReentrancyGuard
 * @author OpenZeppelin
 * @dev Contract module that helps prevent reentrant calls to a function.
 * Inheriting contracts must simply use the nonReentrant modifier to protect functions.
 */
abstract contract ReentrancyGuard {
    uint256 private constant _NOT_ENTERED = 1;
    uint256 private constant _ENTERED = 2;

    uint256 private _status;

    error ReentrancyGuardReentrantCall();

    /**
     * @dev Initializes the contract setting the deployer as the initial owner.
     */
    constructor() {
        _status = _NOT_ENTERED;
    }

    /**
     * @dev Modifier to prevent reentrant calls.
     * Mark a function with this to require that it is never simultaneously entered by multiple callers.
     */
    modifier nonReentrant() {
        _nonReentrantBefore();
        _;
        _nonReentrantAfter();
    }

    /**
     * @dev Hook that is called before any function marked as nonReentrant.
     * Any custom pre-logic can be added here.
     */
    function _nonReentrantBefore() private {
        if (_status == _ENTERED) {
            revert ReentrancyGuardReentrantCall();
        }

        _status = _ENTERED;
    }

    /**
     * @dev Hook that is called after any function marked as nonReentrant.
     * Any custom post-logic can be added here.
     */
    function _nonReentrantAfter() private {
        _status = _NOT_ENTERED;
    }

    /**
     * @dev Return the current status of the reentrancy guard.
     * @return True if the contract is currently entered, otherwise false.
     */
    function _reentrancyGuardEntered() internal view returns (bool) {
        return _status == _ENTERED;
    }
}
```

</details>



## [DataTypes.sol](https://github.com/pxswap-xyz/pxswap/blob/4d20e98c5beb02934ae63ac174abe04227a53a0f/src/types/DataTypes.sol)

<details>

<summary><i>Changed File Version</i></summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.19;

library DataTypes {
    /**
     * @dev Struct representing a trade on the Pxswap platform.
     * @param initiator The address initiating the trade.
     * @param offeredNfts Array of NFT addresses being offered in the trade.
     * @param offeredNftsIds Array of NFT IDs corresponding to the offered NFTs.
     * @param requestNfts Array of NFT addresses that the initiator wants to receive.
     * @param isOpen Boolean representing whether the trade is open or closed.
     */
    struct Trade {
        address initiator;
        address[] offeredNfts;
        uint256[] offeredNftsIds;
        address[] requestNfts;
        bool isOpen;
    }
}
```

</details>

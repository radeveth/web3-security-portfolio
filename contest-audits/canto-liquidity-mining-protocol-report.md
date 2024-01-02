# Canto Liquidity Mining Protocol - Radev's Findings Report

# Findings Summary
| Number     | Title                                                              | Severity |
| ------ | ------------------------------------------------------------------ | -------- |
| [M-01] | Unfair Ambient Reward Distribution: A Gap in Fair Distribution Due to Mid-Week Changes | Medium |
| [M-02] | Front-Running Vulnerability: Exploiting Reward Updates for Maximized Payouts | Medium |


**QA Issues**

|     Number      | Issue                                                                      | Instances |
| :-------------: | :------------------------------------------------------------------------- | :-------: |
| [NC-1](#NC-1)  | Overwriting set values for `poolIdx` and `weekFrom`                        |     1     |
| [NC-2](#NC-2)  | Potential gas issues with `setConcRewards`                                 |     1     |
| [NC-3](#NC-3)  | Improvement in `LiquidityMiningPath#claimConcentratedRewards()`            |     1     |
| [NC-4](#NC-4)  | Function interruption due to "Already claimed" check                       |     2     |
| [NC-5](#NC-5)  | DoS in `accrueAmbientPositionTimeWeightedLiquidity()`                      |     1     |
| [NC-6](#NC-6)  | Skipped weeks issue in `accrueAmbientPositionTimeWeightedLiquidity()`      |     1     |
| [NC-7](#NC-7)  | Checks absence on `lookupPosition`                                         |     1     |
| [NC-8](#NC-8)  | Event emitting in `claimAmbientRewards()`                                  |     1     |
| [NC-9](#NC-9)  | Floating point arithmetic concern                                          |     1     |
| [NC-10](#NC-10) | `claimAmbientRewards()` function is susceptible to gas inefficiency issues |     1     |

---

# [M-01] Unfair Ambient Reward Distribution: A Gap in Fair Distribution Due to Mid-Week Changes

https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/callpaths/LiquidityMiningPath.sol#L65-L72
https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/callpaths/LiquidityMiningPath.sol#L74-L81
https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/mixins/LiquidityMining.sol#L156-L196
https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/mixins/LiquidityMining.sol#L256-L289

## Impact

1. The system allows for unfair reward distribution based on timing and governance decisions. Specifically, users who claim their rewards early (before governance makes a change to rewards) may receive fewer rewards than those who claim later (after the change). This can lead to significant discrepancies in rewards among users based on the timing of claims, undermining the equitable intent of a rewards system.
2. After the `values for weekly Concentrated and Ambient Rewards are changed`, users who claimed before the update `will not receive any rewards afterward`.

## Vulnerability Details

**Let's first analyse the current protocol logic**

1. The [`setConcRewards()`](https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/callpaths/LiquidityMiningPath.sol#L65-L72) and [`setAmbRewards()`](https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/callpaths/LiquidityMiningPath.sol#L74-L81) functions are used to set the weekly reward rate for the liquidity mining sidecar. Reward rates are set by determining a total amount that will be disbursed per week. Governance can choose how many weeks that the reward rate will be set for.

```solidity
    function setConcRewards(bytes32 poolIdx, uint32 weekFrom, uint32 weekTo, uint64 weeklyReward) public payable {
        // require(msg.sender == governance_, "Only callable by governance");
        require(weekFrom % WEEK == 0 && weekTo % WEEK == 0, "Invalid weeks");
        while (weekFrom <= weekTo) {
            concRewardPerWeek_[poolIdx][weekFrom] = weeklyReward;
            weekFrom += uint32(WEEK);
        }
    }

    function setAmbRewards(bytes32 poolIdx, uint32 weekFrom, uint32 weekTo, uint64 weeklyReward) public payable {
        // require(msg.sender == governance_, "Only callable by governance");
        require(weekFrom % WEEK == 0 && weekTo % WEEK == 0, "Invalid weeks");
        while (weekFrom <= weekTo) {
            ambRewardPerWeek_[poolIdx][weekFrom] = weeklyReward;
            weekFrom += uint32(WEEK);
        }
    }
```

2. `LiquidityMining.sol` contains all of the logic for accruing and claiming rewards. This is the most important part of the codebase and should be the main focus for wardens.
   Via [claimAmbientRewards()](https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/mixins/LiquidityMining.sol#L256-L289) and [claimConcentratedRewards()](https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/mixins/LiquidityMining.sol#L156-L196) ([functions in LiquidityMiningPath.sol](https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/callpaths/LiquidityMiningPath.sol#L54-L63)) functions users can claim their accrued rewards.

```solidity
    function claimConcentratedRewards(
        address payable owner,
        bytes32 poolIdx,
        int24 lowerTick,
        int24 upperTick,
        uint32[] memory weeksToClaim
    ) internal {
        accrueConcentratedPositionTimeWeightedLiquidity(
            owner,
            poolIdx,
            lowerTick,
            upperTick
        );
        CurveMath.CurveState memory curve = curves_[poolIdx];
        // Need to do a global accrual in case the current tick was already in range for a long time without any modifications that triggered an accrual
        accrueConcentratedGlobalTimeWeightedLiquidity(poolIdx, curve);
        bytes32 posKey = encodePosKey(owner, poolIdx, lowerTick, upperTick);
        uint256 rewardsToSend;
        for (uint256 i; i < weeksToClaim.length; ++i) {
            uint32 week = weeksToClaim[i];
            require(week + WEEK < block.timestamp, "Week not over yet");
            require(
                !concLiquidityRewardsClaimed_[poolIdx][posKey][week],
                "Already claimed"
            );
            uint256 overallInRangeLiquidity = timeWeightedWeeklyGlobalConcLiquidity_[poolIdx][week];
            if (overallInRangeLiquidity > 0) {
                uint256 inRangeLiquidityOfPosition;
                for (int24 j = lowerTick + 10; j <= upperTick - 10; ++j) {
                    inRangeLiquidityOfPosition += timeWeightedWeeklyPositionInRangeConcLiquidity_[poolIdx][posKey][week][j];
                }
                // Percentage of this weeks overall in range liquidity that was provided by the user times the overall weekly rewards
                rewardsToSend += inRangeLiquidityOfPosition * concRewardPerWeek_[poolIdx][week] / overallInRangeLiquidity;
            }
            concLiquidityRewardsClaimed_[poolIdx][posKey][week] = true;
        }
        if (rewardsToSend > 0) {
            (bool sent, ) = owner.call{value: rewardsToSend}("");
            require(sent, "Sending rewards failed");
        }
    }
```

---

```solidity
    function claimAmbientRewards(
        address owner,
        bytes32 poolIdx,
        uint32[] memory weeksToClaim
    ) internal {
        CurveMath.CurveState memory curve = curves_[poolIdx];
        accrueAmbientPositionTimeWeightedLiquidity(payable(owner), poolIdx);
        accrueAmbientGlobalTimeWeightedLiquidity(poolIdx, curve);
        bytes32 posKey = encodePosKey(owner, poolIdx);
        uint256 rewardsToSend;
        for (uint256 i; i < weeksToClaim.length; ++i) {
            uint32 week = weeksToClaim[i];
            require(week + WEEK < block.timestamp, "Week not over yet");
            require(
                !ambLiquidityRewardsClaimed_[poolIdx][posKey][week],
                "Already claimed"
            );
            uint256 overallTimeWeightedLiquidity = timeWeightedWeeklyGlobalAmbLiquidity_[
                    poolIdx
                ][week];
            if (overallTimeWeightedLiquidity > 0) {
                uint256 rewardsForWeek = (timeWeightedWeeklyPositionAmbLiquidity_[
                    poolIdx
                ][posKey][week] * ambRewardPerWeek_[poolIdx][week]) /
                    overallTimeWeightedLiquidity;
                rewardsToSend += rewardsForWeek;
            }
            ambLiquidityRewardsClaimed_[poolIdx][posKey][week] = true;
        }
        if (rewardsToSend > 0) {
            (bool sent, ) = owner.call{value: rewardsToSend}("");
            require(sent, "Sending rewards failed");
        }
    }
```

3. When a user claims their `Ambient Rewards` for a specific week or set of weeks, they cannot reclaim for the same period due to the `ambLiquidityRewardsClaimed_[poolIdx][posKey][week] = true;` line in [claimAmbientRewards()](https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/mixins/LiquidityMining.sol#L256-L289). The same restriction applies when claiming `Concentrated Rewards` via [claimConcentratedRewards()](https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/mixins/LiquidityMining.sol#L156-L196).

```solidity
           // claimAmbientRewards()
            ambLiquidityRewardsClaimed_[poolIdx][posKey][week] = true;
```

```solidity
           // claimConcentratedRewards()
            concLiquidityRewardsClaimed_[poolIdx][posKey][week] = true;
```

In essence, users cannot claim their rewards multiple times within the same week.

## Proof of Concept

1. Consider a week where user A and user B both participate and accrue Ambient rewards for example.
2. User A decides to claim his Ambient rewards for week 10 to week 15.
3. Later, governance observes heightened activity from week 10 to week 15 and opts to increase `ambRewardPerWeek_` for that weeks to further incentivize participants.
4. User B claims rewards after this increase.
5. User A try to get his new rewards but actually can't. (the function revert)

**Result:**
User A receives rewards based on the original rate, while user B gets rewards based on the increased rate. This leads to User B receiving a significantly larger reward for the same set of weeks, even if both users had similar activity.

## Tools Used

- Manual Inspection.
- Logical and timing analysis based on the provided functions behavior.

## Recommended Mitigation Steps

1. **Immutable Week Rewards**: Once rewards for a week are set, they should be made immutable. This prevents the governance or admin from making changes that can impact users who've already claimed their rewards.
2. **Reward Adjustments for Future Weeks Only**: If governance wishes to increase the reward amount due to increased activity, this change should only apply to subsequent weeks. This ensures all users within a particular week receive rewards based on the same rate.
3. **Reclaim Mechanism**: Implement a mechanism where, if rewards for a week are adjusted upwards after some users have already claimed, those users can reclaim the difference. However, this adds complexity and could be gas-inefficient.


# [M-02] Front-Running Vulnerability: Exploiting Reward Updates for Maximized Payouts

https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/callpaths/LiquidityMiningPath.sol#L65-L72
https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/callpaths/LiquidityMiningPath.sol#L74-L81
https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/mixins/LiquidityMining.sol#L156-L196
https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/mixins/LiquidityMining.sol#L256-L289

## Impact

Malicious users **claim rewards at a higher rate than what was intended** by front-running governance actions meant to reduce rewards. This allows them to claim rewards at a higher rate than what was intended, undermining the protocol's intended economic incentives and potentially causing undue inflation.

## Proof of Concept

**Attack Scenario:**

1. **Monitoring for Reward Update Transactions:** An malicious user sets up a bot or script to monitor pending transactions sent from the governance address or any transaction calling `setConcRewards` or `setAmbRewards`. This can be done using tools like Etherscan's API or Web3 libraries to listen to mempool transactions.

2. **Identification of the Reduction:** Once the user's script identifies a transaction intending to reduce the rewards, it triggers the next steps automatically.

3. **Swift Claiming:** The malicious user immediately sends a transaction calling `claimAmbientRewards` or `claimConcentratedRewards`. To ensure their transaction gets processed first, they set a gas price significantly higher than the current average or the gas price of the governance's transaction.

4. **Transaction Confirmation:** Given the higher gas price, miners prioritize the malicious user transaction over the governance's reward adjustment transaction. As a result, the malicious user claim transaction gets confirmed first, allowing them to receive rewards at the old, higher rate.

5. **Result:** By the time the governance transaction gets confirmed and the reward rate is reduced, the malicious user has already claimed a significant portion of the rewards at the higher rate. Over time and repeated instances, this can lead to substantial losses for the protocol.

**Evidence:**

- Monitoring the mempool can reveal a pattern of such front-running transactions frequently getting confirmed just before the governance's reward adjustment transactions.

## Code Snippet

1. The [`setConcRewards()`](https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/callpaths/LiquidityMiningPath.sol#L65-L72) and [`setAmbRewards()`](https://github.com/code-423n4/2023-10-canto/blob/main/canto_ambient/contracts/callpaths/LiquidityMiningPath.sol#L74-L81) functions are used to set the weekly reward rate for the liquidity mining sidecar. Reward rates are set by determining a total amount that will be disbursed per week. Governance can choose how many weeks that the reward rate will be set for.

```solidity
    function setConcRewards(bytes32 poolIdx, uint32 weekFrom, uint32 weekTo, uint64 weeklyReward) public payable {
        // require(msg.sender == governance_, "Only callable by governance");
        require(weekFrom % WEEK == 0 && weekTo % WEEK == 0, "Invalid weeks");
        while (weekFrom <= weekTo) {
            concRewardPerWeek_[poolIdx][weekFrom] = weeklyReward;
            weekFrom += uint32(WEEK);
        }
    }

    function setAmbRewards(bytes32 poolIdx, uint32 weekFrom, uint32 weekTo, uint64 weeklyReward) public payable {
        // require(msg.sender == governance_, "Only callable by governance");
        require(weekFrom % WEEK == 0 && weekTo % WEEK == 0, "Invalid weeks");
        while (weekFrom <= weekTo) {
            ambRewardPerWeek_[poolIdx][weekFrom] = weeklyReward;
            weekFrom += uint32(WEEK);
        }
    }
```

---

```solidity
    function claimConcentratedRewards(
        address payable owner,
        bytes32 poolIdx,
        int24 lowerTick,
        int24 upperTick,
        uint32[] memory weeksToClaim
    ) internal {
        accrueConcentratedPositionTimeWeightedLiquidity(
            owner,
            poolIdx,
            lowerTick,
            upperTick
        );
        CurveMath.CurveState memory curve = curves_[poolIdx];
        // Need to do a global accrual in case the current tick was already in range for a long time without any modifications that triggered an accrual
        accrueConcentratedGlobalTimeWeightedLiquidity(poolIdx, curve);
        bytes32 posKey = encodePosKey(owner, poolIdx, lowerTick, upperTick);
        uint256 rewardsToSend;
        for (uint256 i; i < weeksToClaim.length; ++i) {
            uint32 week = weeksToClaim[i];
            require(week + WEEK < block.timestamp, "Week not over yet");
            require(
                !concLiquidityRewardsClaimed_[poolIdx][posKey][week],
                "Already claimed"
            );
            uint256 overallInRangeLiquidity = timeWeightedWeeklyGlobalConcLiquidity_[poolIdx][week];
            if (overallInRangeLiquidity > 0) {
                uint256 inRangeLiquidityOfPosition;
                for (int24 j = lowerTick + 10; j <= upperTick - 10; ++j) {
                    inRangeLiquidityOfPosition += timeWeightedWeeklyPositionInRangeConcLiquidity_[poolIdx][posKey][week][j];
                }
                // Percentage of this weeks overall in range liquidity that was provided by the user times the overall weekly rewards
                rewardsToSend += inRangeLiquidityOfPosition * concRewardPerWeek_[poolIdx][week] / overallInRangeLiquidity;
            }
            concLiquidityRewardsClaimed_[poolIdx][posKey][week] = true;
        }
        if (rewardsToSend > 0) {
            (bool sent, ) = owner.call{value: rewardsToSend}("");
            require(sent, "Sending rewards failed");
        }
    }
```

---

```solidity
    function claimAmbientRewards(
        address owner,
        bytes32 poolIdx,
        uint32[] memory weeksToClaim
    ) internal {
        CurveMath.CurveState memory curve = curves_[poolIdx];
        accrueAmbientPositionTimeWeightedLiquidity(payable(owner), poolIdx);
        accrueAmbientGlobalTimeWeightedLiquidity(poolIdx, curve);
        bytes32 posKey = encodePosKey(owner, poolIdx);
        uint256 rewardsToSend;
        for (uint256 i; i < weeksToClaim.length; ++i) {
            uint32 week = weeksToClaim[i];
            require(week + WEEK < block.timestamp, "Week not over yet");
            require(
                !ambLiquidityRewardsClaimed_[poolIdx][posKey][week],
                "Already claimed"
            );
            uint256 overallTimeWeightedLiquidity = timeWeightedWeeklyGlobalAmbLiquidity_[
                    poolIdx
                ][week];
            if (overallTimeWeightedLiquidity > 0) {
                uint256 rewardsForWeek = (timeWeightedWeeklyPositionAmbLiquidity_[
                    poolIdx
                ][posKey][week] * ambRewardPerWeek_[poolIdx][week]) /
                    overallTimeWeightedLiquidity;
                rewardsToSend += rewardsForWeek;
            }
            ambLiquidityRewardsClaimed_[poolIdx][posKey][week] = true;
        }
        if (rewardsToSend > 0) {
            (bool sent, ) = owner.call{value: rewardsToSend}("");
            require(sent, "Sending rewards failed");
        }
    }
```

## Tools Used

Manual Inspection

## Recommended Mitigation Steps

**Time-Locked Updates**: Introduce a delay on governance actions that adjust rewards, providing users with a clear window of upcoming changes.



# NC-1 Overwriting set values for `poolIdx` and `weekFrom`

There's no check to prevent overwriting already set values for a particular `poolIdx` and `weekFrom` combination in the `concRewardPerWeek_` mapping. This might lead to unforeseen results and data integrity issues.

Example:

- If `concRewardPerWeek_[1][5] = 20;` has already been set,
- And the user (or governance, when the commented out permission check is enabled) calls `setConcRewards` with `poolIdx = 1`, `weekFrom = 5`, and `weeklyReward = 10`,
- Then after the function execution, the value would indeed be overwritten to `concRewardPerWeek_[1][5] = 10;`.
  If there's a desire to prevent overwriting already set values, an additional check would need to be added, such as:

```solidity
require(concRewardPerWeek_[poolIdx][weekFrom] == 0, "Reward already set for this week");
```

This would throw an exception and halt the function if there's already a reward set for the given pool and week.

# NC-2 Potential gas issues with `setConcRewards`

The usage of the `while` loop without bounds can result in high gas consumption, making transactions expensive or even impossible to process in extreme cases. It also opens up possibilities for malicious actors to grief the contract.

**Gas Griefing**:
A malicious actor (or even an unintentional user) could call this function with a very large range between `weekFrom` and `weekTo`, causing the function to execute the `while` loop an excessive number of times. This could result in a transaction that consumes a vast amount of gas, potentially causing the transaction to run out of gas or become prohibitively expensive to execute.

If there are no restrictions on who can call this function (the governance check is commented out), it could be used as an attack vector by malicious users to "grief" the contract, by repeatedly calling the function with large ranges to waste the gas of any monitoring or interacting services.

**Gas Limit Issues**:
If the range between `weekFrom` and `weekTo` is large enough, it could exceed the block gas limit. This would make it impossible for the transaction to be processed, effectively blocking the operation.

**Gas Griefing Explained**:

Gas griefing is when someone purposely makes certain actions on the Ethereum network (like using a function in a smart contract) use up more gas, or computing power, than they should. This can cause problems and inconveniences.

**Impacts**:

1. **Costly Actions**: Making an action on the network can become more expensive for users.
2. **Blocked Functions**: Some actions might become impossible to do if they use up too much gas.
3. **Slower Network**: Too many of these "expensive" actions can make the whole network slow down.
4. **Loss of Trust**: Users might start doubting the reliability of a project if it's often targeted by gas griefing.
5. **Good for Miners, Bad for Users**: Miners, who validate transactions, might earn more because of the higher fees. But this is bad for regular users who have to pay these fees.
6. **Network Traffic Jams**: A lot of these "expensive" actions can cause a traffic jam on the network, making everything slow.
7. **Opens Doors for Other Attacks**: Gas griefing can create opportunities for other harmful actions on the network.
8. **Extra Costs for Projects**: Projects have to spend more to monitor for gas griefing and might need to fix or update their systems if attacked.

In simple words, gas griefing is like someone intentionally causing traffic jams on a highway, making journeys longer and more costly for everyone. It's essential to design roads (or in this case, smart contracts) to prevent these jams and keep everything running smoothly.

**Recommendation**:

1. **Add an Upper Bound**: Limit the difference between `weekFrom` and `weekTo` to a maximum value, e.g.,

   ```solidity
   require(weekTo - weekFrom <= MAX_WEEKS_RANGE, "Range too large");
   ```

   where `MAX_WEEKS_RANGE` is a predefined constant that sets a reasonable maximum number of weeks for which rewards can be set in a single transaction.

2. **Re-enable Governance Check**: If it's intended for only certain addresses (like a governance mechanism) to call this function, ensure that you uncomment and properly configure the `require` check for permissions. This way, you can limit potential abuse by restricting who can set rewards.

# NC-3 Improvement in `LiquidityMiningPath#claimConcentratedRewards()`

The direct usage of `msg.sender` in this function can be optimized. It's generally good practice to provide flexibility and make functions more versatile by allowing parameters instead.

# NC-4 Function interruption due to "Already claimed" check

These checks, though essential for security, might end up stopping the entire function process. An alternative design should be considered, perhaps skipping the loop iteration instead of stopping the whole function.

```solidity
        require(
            !concLiquidityRewardsClaimed_[poolIdx][posKey][week],
            "Already claimed"
        );
```

```solidity
        require(
            !ambLiquidityRewardsClaimed_[poolIdx][posKey][week],
            "Already claimed"
        );
```

# NC-5 DoS in `accrueAmbientPositionTimeWeightedLiquidity()`

A potential Denial of Service could occur if the difference between `lastAccrued` and `block.timestamp` is significantly large, making the function very gas-heavy.

# NC-6 Skipped weeks issue in `accrueAmbientPositionTimeWeightedLiquidity()`

In case this function isn't called for a prolonged period, weeks in between might not be correctly accounted for, potentially resulting in inaccurate reward calculations.

# NC-7 Checks absence on `lookupPosition`

Assuming always correct data without validation might result in unexpected behaviors. Validation checks should be added.

# NC-8 Event emitting in `claimAmbientRewards()`

Emitting events is crucial for dApps to monitor and track. Their omission might make interaction and monitoring difficult.

# NC-9 Floating point arithmetic concern

Solidity doesn't handle floating-point numbers natively. It's essential to be cautious when performing divisions to avoid precision loss.

# NC-10 `claimAmbientRewards()` function is susceptible to gas inefficiency issues

## Impact

The `claimAmbientRewards()` function in the provided code may be susceptible to gas inefficiency issues because of the unbounded loop based on user input. Malicious actors can exploit this by introducing an exceptionally large array for `weeksToClaim`, causing the function to use excessive gas. This can lead to a denial-of-service for genuine users, especially when trying to claim rewards for multiple weeks simultaneously.

## Proof of Concept

- **Loop with Dynamic Length**:
  Within the `claimAmbientRewards()` function, there exists a loop that iterates over `weeksToClaim`:

  ```solidity
  for (uint256 i; i < weeksToClaim.length; ++i) {
      ...
  }
  ```

  By feeding an excessively large array for `weeksToClaim`, malevolent actors could potentially cause the function to deplete all available gas.

- **External Calls to `msg.sender`**:
  The function initiates an external call to send value:
  ```solidity
  (bool sent, ) = msg.sender.call{value: rewardsToSend}("");
  ```
  The potential complexities stemming from `msg.sender` receiving value should be acknowledged, especially if `msg.sender` is a fallback function in a contract.

## Recommendations for Enhanced Gas Management

```solidity
function claimAmbientRewards(...) external {
    ...
+   uint256 gasBefore = gasleft();
    ... // original code
+   require(gasleft() > gasBefore/64, "Unexpected gas consumption");
    ...
}
```

## Tools Used

- Manual Code Review

## Recommended Mitigation Steps

1. **Bounded Loops**: Introduce a limitation on the permissible length of the `weeksToClaim` array that can be processed within a single transaction. This acts as a preventive measure to ensure that the function doesn't run out of gas because of disproportionately large arrays.

---

# Canto Liquidity Mining Protocol - Analysis Report

## Canto Liquidity Mining Protocol Audit

- **Purpose**: Canto introduces a new liquidity mining feature specifically for Ambient Finance.
- **Implementation**: This feature is a sidecar contract integrated into Ambient using their proxy contract pattern.
- **New Contracts**:

  1. LiquidityMiningPath.sol: Allows user interactions.
  2. LiquidityMining.sol: Contains core logic.

- **About LiquidityMining Sidecar**:

  - This sidecar is designed to implement a liquidity mining protocol for Ambient. Its goal is to incentivize liquidity for Ambient pools on Canto.

- **Incentive Mechanism**:

  - Incentives focus on a width of liquidity based on the current tick, specifically ranging from currentTick-10 to currentTick+10. Users must provide liquidity within this range to qualify for rewards.

- **Rewards**:

  - The protocol tracks time-weighted liquidity (both global and per-user) for various positions, helping calculate user rewards based on their contribution. Rewards are then disbursed proportionately among liquidity providers.

- **Implementation Details**:

  - Reward rates are set weekly, based on a predetermined total disbursement amount. Governance determines the duration of these reward rates.
  - LiquidityMining.sol is central for rewards and needs special attention during audits.

- **Ambient Overview**:

  - Previously known as CrocSwap, Ambient is a dex allowing liquidity providers to deposit either "ambient" liquidity or concentrated liquidity into token pairs.
  - CrocSwapDex is Ambient's main contract that users interact with.
  - Ambient uses modular "sidecar" contracts to manage functionalities not accommodated in CrocSwapDex due to EVM contract limit.

- **Noteworthy Sidecar Contracts**:
  - BootPath: Installs other sidecars.
  - ColdPath: Creates new pools.
  - WarmPath: Manages liquidity.
  - LiquidityMiningPath: Canto's sidecar for liquidity mining.
  - KnockoutPath, LongPath, MicroPaths, SafeModePath: Handle various specific functionalities.

---

---

## Examples & Scenarios

### **Loop Execution in `LiquidityMining.sol#accrueAmbientPositionTimeWeightedLiquidity()`**:

Let's assume:

1. `WEEK` = 7 days = 604800 seconds.
2. `time` (the starting point from which we are updating) = some arbitrary timestamp representing a Wednesday at noon.
3. `block.timestamp` (current Ethereum block's timestamp) = a timestamp representing 9 days later, which is a Friday evening.

This means the loop will have to account for a full week from the starting Wednesday to the next Wednesday, and then an additional 2 days up to Friday evening.

**Initial Data**:

- `time` = 1,678,688,000 (Wednesday, Noon)
- `block.timestamp` = 1,693,568,000 (Friday, 9 days later)
- `liquidity` = 1000 (just an arbitrary value for this example)

**First Iteration**:

- `currWeek` = `time` rounded down to the last Sunday = 1,676,160,000 (Last Sunday)
- `nextWeek` = `currWeek` + `WEEK` = 1,682,960,000 (Next Sunday)
- `dt` = Since `nextWeek` (Next Sunday) is before `block.timestamp` (Friday, 9 days later), we'll take a full week, i.e., `nextWeek - time` = 1,682,960,000 - 1,678,688,000 = 4,272,000 seconds (5 days from Wednesday Noon to Sunday).
- The liquidity for the current week (from the Last Sunday to the Next Sunday) is incremented by `dt * liquidity` = 4,272,000 \* 1000.
- Now, increment `time` by `dt`: `time` = 1,678,688,000 + 4,272,000 = 1,682,960,000 (Next Sunday).

**Second Iteration**:

- `currWeek` = `time` (which is now the Next Sunday) = 1,682,960,000.
- `nextWeek` = `currWeek` + `WEEK` = 1,689,760,000 (Sunday after the Next Sunday).
- `dt` = Since `block.timestamp` (Friday, 9 days later) is before `nextWeek` (Sunday after the Next Sunday), we'll take the difference from the current `time` (Next Sunday) up to `block.timestamp` (Friday, 9 days later) = 1,693,568,000 - 1,682,960,000 = 10,608,000 seconds (2 days from Sunday to Tuesday).
- The liquidity for the current week (from the Next Sunday to the following Sunday) is incremented by `dt * liquidity` = 10,608,000 \* 1000.
- Now, increment `time` by `dt`: `time` = 1,682,960,000 + 10,608,000 = 1,693,568,000 (Friday, 9 days later).

At this point, `time` is equal to `block.timestamp`, so the loop ends.

**Results**:

- For the week starting at the Last Sunday, the time-weighted liquidity was incremented by the equivalent of 5 days of liquidity.
- For the week starting at the Next Sunday, the time-weighted liquidity was incremented by the equivalent of 2 days of liquidity.

### **`claimAmbientRewards()` Function Walkthrough**:

### Scenario:

1. Assume `owner` is `0x1234567890abcdef1234567890abcdef12345678`.
2. The `poolIdx` is a hypothetical index (hash) `0xabcdef1234567890`.
3. There are 2 weeks to claim in `weeksToClaim` array: `[100, 101]` (These are hypothetical week numbers).
4. Let's assume that the `block.timestamp` is `102`.

### Step-by-Step Execution:

1. **Initialization**:

   - `curve` is fetched from `curves_` based on `poolIdx`.
   - The function `accrueAmbientPositionTimeWeightedLiquidity()` and `accrueAmbientGlobalTimeWeightedLiquidity()` are called.
   - The `posKey` (a unique identifier for the user's position) is generated using `encodePosKey()`.

2. **Rewards Calculation**:

   - Start loop for each week in `weeksToClaim`:

     - For `week = 100`:

       - Check if week + WEEK < block.timestamp. Since 100 + WEEK < 102, this is true.
       - Check if the rewards for this week have already been claimed. Assume they haven't.
       - Fetch the overall time-weighted liquidity for this week from `timeWeightedWeeklyGlobalAmbLiquidity_`. Let's assume this is `1000`.
       - If the overall liquidity is greater than 0, calculate the rewards for the week:
         - Get the time-weighted position ambient liquidity for this week for the specific user, assume it's `10`.
         - Fetch the total rewards available for the week (`ambRewardPerWeek_`). Let's assume it's `500`.
         - Calculate rewards: (10 \* 500) / 1000 = `5`. Add this to `rewardsToSend`.
       - Mark the rewards as claimed for this week.

     - For `week = 101`:
       - Same steps repeated. Let's assume the user gets `4` as the reward for this week.
       - Add this to `rewardsToSend`. Now, `rewardsToSend = 9`.

3. **Sending Rewards**:
   - Check if `rewardsToSend` is greater than 0.
   - Send the rewards to the owner. Here, `9` will be sent to the `owner`.

### Outcome:

The owner would receive `9` as rewards for the weeks `[100, 101]` in our hypothetical scenario.

---

---

## Questions & Notes:

### **During Documentation Review**:

1. Who is the intended audience for interaction with the `LiquidityMining` and `LiquidityMiningPath` contracts?
2. What is the correct procedure to engage with Ambient Contracts?
3. Clarification on the term "sidecar contract".

### **During Audit**:

1. Canto is developing a liquidity mining protocol tailored for the Ambient decentralized exchange.
2. Differentiation between Ambient rewards and Concentrated rewards.
3. Insight into the `CurveState` Struct.

### **Potential Attack Vectors**:

1. Possibility of liquidity providers withdrawing more rewards than they have accumulated.
2. Scenario where liquidity providers might receive lesser rewards than they are entitled to.


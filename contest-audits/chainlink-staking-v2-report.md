# Chainlink Staking v0.2 Code4rena Contest - Radev's Findings Report

# Findings Summary

| ID     | Title                                                                                                                                 | Severity |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------- | -------- |
| [M-01] | Stakers can bypass the "Cannot Claim Reward When Paused" requirement        | Medium   |
| [M-02] | Claiming accumulated rewards while the contract is underfunded can lead to a loss of rewards     | Medium   |

### Low Issues

|Number|Issue Title|
|-|:-|
| [L-01](#L-01) | RewardVault does not revert on zero staker amount in `claimReward()` function |
| [L-02](#L-02) | Sybil Attack |
| [L-03](#L-03) | Risk of Slashing Below the minPrincipalPerStaker Threshold |
| [L-04](#L-04) | Risk of Slashing Below the minPrincipalPerStaker Threshold |

### Non Critical Issues

|Number|Issue Title|
|-|:-|
| [NC-01](#NC-01) | Front Running Concern in the Unbonding Process |
| [NC-02](#NC-02) | Important functions doesn't emit an events |

### Suggestions

|Number|Suggestion Title|
|-|:-|
| [S-01](#S-01) | Revert `claimReward()` if the msg.sender is not the staker or staking pool |
| [S-02](#S-02) | Inconsistent Comparison Methods with block.timestamp in _inClaimPeriod() Function |

---

# **Detailed Findings**

# [M-01] Stakers can bypass the "Cannot Claim Reward When Paused" requirement

## Lines of code
https://github.com/code-423n4/2023-08-chainlink/blob/main/src/pools/StakingPoolBase.sol#L459-L505
https://github.com/code-423n4/2023-08-chainlink/blob/main/src/pools/StakingPoolBase.sol#L464-L466

## Impact
The ability for stakers to bypass the "Cannot Claim Reward When Paused" requirement presents a serious security concern for the Chainlink staking protocol. The pause mechanism is typically implemented to provide administrators with the capability to halt certain operations, often in response to security threats, bugs, or other unforeseen incidents. When users can bypass this, several scenarios could occur:

1. Unexpected Financial Impacts: If the pause was initiated due to a miscalculation in reward distribution or any financial discrepancies, allowing users to claim could result in financial losses to the pool or disproportionate rewards for certain users.
2. Loss of Control


## Proof of Concept
1. Scenario Setup: Assume that the protocol has paused the staking pool due to a detected issue. During this pause, reward claims should be halted until the issue is resolved.

2. Bypass Initiation: A user, being aware of the bypass, starts the process by calling the unstake() function. Instead of trying to claim the reward directly (which would be blocked by the pause), they set shouldClaimReward = false.

3. Bypass Execution: By setting the claim flag to false, the function bypasses the paused() && shouldClaimReward check. This is the protocol's weak point. The function then proceeds to invoke finalizeReward().

- This function performs the necessary calculations and sets the staker's rewards in a state where they are "finalized" and ready to be claimed.
- This action is executed without any hindrance, even though the pool is paused, because the pause check is only done for direct reward claims.

4. Claiming the Finalized Reward: After finalizing the reward, the user simply calls claimReward(). This function doesn’t have the same pause restrictions as unstake(), allowing the user to successfully claim their reward even when they technically shouldn't be able to.

5. Repeating for Maximum Gain: The user, or any other aware of this, can repeat the process: unstaking without claiming and then separately claiming the reward, each time successfully bypassing the pause.

6. Potential Mass Exploit: If the vulnerability becomes widely known before it's fixed, multiple stakers might rush to use this method, leading to a cascade of unexpected reward claims during a critical paused period.

7. `unstake()` function in StakingBaseReward contract where the RewardVault.sol#finalizeReward() is called

```js
  function unstake(uint256 amount, bool shouldClaimReward) external override {
    Staker storage staker = s_stakers[msg.sender];
    if (!_canUnstake(staker)) {
      revert StakerNotInClaimPeriod(msg.sender);
    }
    if (paused() && shouldClaimReward) {
      revert CannotClaimRewardWhenPaused();
    }
    
    /*
    ...
    */

    uint256 claimedReward = s_rewardVault.finalizeReward({
      staker: msg.sender,
      oldPrincipal: stakerPrincipal,
      unstakedAmount: amount,
      shouldClaim: shouldClaimReward,
      stakedAt: stakedAt
    });

    /*
    ...
    */
  }
```

- `finalizeReward()` function inRewardVault.sol contract

```js
  function finalizeReward(
    address staker,
    uint256 oldPrincipal,
    uint256 stakedAt,
    uint256 unstakedAmount,
    bool shouldClaim
  ) external override onlyStakingPool returns (uint256) {
    if (paused() && shouldClaim) revert CannotClaimRewardWhenPaused();

    _updateRewardPerToken();
    
    bool isOperator = msg.sender == address(i_operatorStakingPool);
    bool shouldForfeit = unstakedAmount != 0;
    StakerReward memory stakerReward = _calculateStakerReward({
      staker: staker,
      isOperator: isOperator,
      stakerPrincipal: oldPrincipal
    });
    uint256 fullForfeitedRewardAmount = _applyMultiplier({
      stakerReward: stakerReward,
      shouldForfeit: shouldForfeit,
      stakerStakedAtTime: stakedAt
    });

    if (fullForfeitedRewardAmount != 0) {
      _forfeitStakerBaseReward({
        stakerReward: stakerReward,
        fullForfeitedRewardAmount: fullForfeitedRewardAmount,
        unstakedAmount: unstakedAmount,
        oldPrincipal: oldPrincipal,
        isOperator: isOperator
      });
    }

    delete stakerReward.earnedBaseRewardInPeriod;
    delete stakerReward.claimedBaseRewardsInPeriod;

    uint256 claimedAmount;

    if (shouldClaim) {
      claimedAmount = _transferRewards(staker, stakerReward);
    }

    s_rewards[staker] = stakerReward;

    /*
    ...
    */

    return claimedAmount;
  }
```

- `claimReward()` function in RewardVault.sol contract

```js
  function claimReward() external override whenNotPaused returns (uint256) {
    _updateRewardPerToken();

    bool isOperator = _isOperator(msg.sender);
    IStakingPool stakingPool =
      isOperator ? IStakingPool(i_operatorStakingPool) : IStakingPool(i_communityStakingPool);
    uint256 stakerPrincipal = _getStakerPrincipal(msg.sender, stakingPool);
    StakerReward memory stakerReward = _calculateStakerReward({
      staker: msg.sender,
      isOperator: isOperator,
      stakerPrincipal: stakerPrincipal
    });

    _applyMultiplier({
      stakerReward: stakerReward,
      shouldForfeit: false,
      stakerStakedAtTime: _getStakerStakedAtTime(msg.sender, stakingPool)
    });

    stakerReward.claimedBaseRewardsInPeriod += stakerReward.earnedBaseRewardInPeriod.toUint112();
    delete stakerReward.earnedBaseRewardInPeriod;

    uint256 claimableRewards = _transferRewards(msg.sender, stakerReward);
    s_rewards[msg.sender] = stakerReward;

    /*
    ...
    */

    return claimableRewards;
  }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
I believe there are two possible solutions:

1. Ensure that the 'staker' cannot call the claimRewards() function when the Staking Pool is in a paused state and the 'staker' is in the unbonding process.
2. Consider introducing a state variable that tracks the number of base rewards the 'staker' has accumulated during his/her unbonding process while the Staking Pool is in a paused state. This way, when the 'staker' calls the claimRewards() function when the Staking Pool is in paused state, his/her claimable rewards (only claimable rewards) can be decreased by the accumulated rewards. (tricky one)

## Assessed type
Other


# [M-02] Claiming accumulated rewards while the contract is underfunded can lead to a loss of rewards

## Lines of code
https://github.com/code-423n4/2023-08-chainlink/blob/main/src/rewards/RewardVault.sol#L623-L658

## Impact
Claiming accumulated rewards while the contract is underfunded can lead to a loss of rewards.

## Proof of Concept
Stakers can claim their rewards by calling the claimReward() function on the RewardVault.sol contract they wish to claim rewards from. The vault calculates the amount of rewards the staker has earned and transfers it from the RewardVault to the staker.

Let's analyze the logic of `claimReward()` function:

1. `claimReward()` function: This function allows a user to claim their reward.
 - It first updates the reward per token if needed.
 - It then calculates the reward for the staker, applies any multipliers, and then updates and transfers the reward to the user.
2. `_updateRewardPerToken()` function: This function updates the reward per token for the community and the operator if not updated in the current block.
3. `_calculatePoolsRewardPerToken()` function: This function calculates the reward per token for community and operators. It returns the reward per token values for the community, operator, and operator delegated.
 - It retrieves the total principal amount staked in both community and operator staking pools and then calculates the reward per token for each.
4. `_calculateVestedRewardPerToken()` function: This function calculates the reward per token for a specific reward bucket based on the time elapsed since the last reward was emitted and the total amount staked in the associated pool.
 - If the total principal staked is 0, it just returns the current vestedRewardPerToken from the reward bucket.
 - If no rewards have been emitted since the last update, it also just returns the current vestedRewardPerToken.
 - Otherwise, it calculates the rewards earned per token based on the time elapsed and the reward bucket's emission rate.


From all of this logic, the logic that we are most interested is how rewards are calculated and updated, particularly when totalPrincipal is 0.
If totalPrincipal is 0, the rewards allocated during this period become unclaimable, thereby negatively affecting the stakers who join afterward.

- `_calculateVestedRewardPerToken()` function:

```js
function _calculateVestedRewardPerToken(
    RewardBucket memory rewardBucket,
    uint256 totalPrincipal
) private view returns (uint256) {
    if (totalPrincipal == 0) return rewardBucket.vestedRewardPerToken;
    ...
}
```
The behavior can be characterized as:

1. Prior to Any Stakes: If rewards are deposited into the contract when no staker has participated (totalPrincipal is zero), the _calculateVestedRewardPerToken function simply returns the existing vestedRewardPerToken value. This means the newly allocated rewards are not acknowledged. Subsequent stakers will not benefit from these particular rewards since the metric for rewards-per-token doesn't account for the new rewards.

2. After All Stakers Exit: If all stakers retract their stakes and additional rewards are deposited afterward, the same issue arises.

3. Outcome: The rewards essentially become "locked" inside the contract due to the non-updated vestedRewardPerToken value in scenarios where totalPrincipal is zero.

**Implications:**
1. Token Utility: The presence of locked rewards reduces the tokens' utility, potentially affecting the protocol's perceived trustworthiness.

2. Economic Impact: New and returning stakers face an undeserved disadvantage, which may make the protocol less enticing to prospective participants.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Consider reverting if there are insufficient LINK tokens available as rewards. This is the best immediate solution to the problem.

## Assessed type
Other

---

## Low Issues

### <a name="L-01"></a>[L-01] RewardVault does not revert on zero staker amount in `claimReward()` function
#### Overview:
The [`claimReward()`](https://github.com/code-423n4/2023-08-chainlink/blob/main/src/rewards/RewardVault.sol#L623-L658) function is designed to allow stakers or staking pools to claim their rewards, but the function doesn't handle cases where the staker's staked LINK is 0.

#### Detailed Analysis:
**Designed Functionality:** 
- The function is called to claim a staker’s rewards.
- Returns the amount of rewards that were claimed.
**Issue:** 
- The function lacks a check to revert if the staker’s staked LINK is zero, what is actually writen in the [Technical Specification](https://github.com/code-423n4/2023-08-chainlink/blob/main/docs/specs.pdf).

#### Recommendation:
Revert if the staker’s staked LINK is 0

---

### <a name="L-02"></a>[L-02] Sybil Attack
#### Overview:
Since the formula to calculate rewards in `RewardVault` contract is as following:
rewards = ((stakeStakedLINKAmount * emittedRewardAmount) / totalStakedLINKAmount)

An Staker might create multiple accounts (or addresses) to act as distinct entities and gain undue advantages in reward systems that favor a larger number of participants.

#### Recommendation:
Unsure. Introduce Robust Identity Verification.

---

### <a name="L-03"></a>[L-03] Risk of Slashing Below the minPrincipalPerStaker Threshold
#### Overview:
In the process of slashing an operator's staking principals in the `_slashOperators()` function, there's no verification to ensure that the staking amount remains above the predefined `minPrincipalPerStaker` threshold.

#### Description:
The `_slashOperators()` function deducts `principalAmount` from the staking principal of each operator. While it does ensure that an operator is not slashed below zero by the line:

```solidity
uint256 slashedAmount = principalAmount > operatorPrincipal ? operatorPrincipal : principalAmount;
```

It doesn't check if the post-slash principal remains above a potentially predefined minimum threshold, `minPrincipalPerStaker`. If such a threshold exists to maintain system integrity or fulfill certain conditions, slashing without considering this limit may lead to undesired system states.

For instance, consider an operator with 1100 LINK staked and a `minPrincipalPerStaker` of 1000 LINK. If the operator is slashed by 200 LINK without the threshold check, their resulting balance would be 900 LINK— violating the system's requirement.

#### Proof of Concept:
The following snippet from the `_slashOperators()` function depicts the lack of threshold check:

```solidity
uint224 history = staker.history.latest();
uint256 operatorPrincipal = uint112(history >> 112);
uint256 slashedAmount = principalAmount > operatorPrincipal ? operatorPrincipal : principalAmount;
uint256 updatedPrincipal = operatorPrincipal - slashedAmount;
```

The above code ensures that the `updatedPrincipal` doesn't go negative, but doesn't validate against the `minPrincipalPerStaker` threshold.

#### Recommendation:
Introduce a check to ensure the post-slash principal doesn't drop below `minPrincipalPerStaker`. If a slash would take an operator below this amount, either adjust the slash amount or implement an appropriate exception handling mechanism.

```solidity
uint256 potentialUpdatedPrincipal = operatorPrincipal - slashedAmount;
if(potentialUpdatedPrincipal < minPrincipalPerStaker) {
    // Handle this case, either by adjusting the slash amount or skipping the operation with an appropriate event/notification
}
```

---

### <a name="L-03"></a>[L-03] Risk of Slashing Below the minPrincipalPerStaker Threshold
#### Overview:
In the [Technical Specification](https://github.com/code-423n4/2023-08-chainlink/blob/main/docs/specs.pdf) states that attempting to slash beyond a slasher's capacity will result in a contract revert. However, the provided code does not revert in this scenario; instead, it adjusts the `principalAmount` based on the available slashing capacity. This inconsistency can lead to unexpected behavior and can mislead users or developers interacting with the contract.

#### Description:
The [Technical Specification](https://github.com/code-423n4/2023-08-chainlink/blob/main/docs/specs.pdf) on "Rate Limited Slashing" explicitly mentions:

> When attempting to slash over this capacity, the contract will revert until sufficient capacity has been refilled.

Contrastingly, in the `slashAndReward` function within the provided code, if the combined amount to slash exceeds the slasher's capacity, the contract modifies the `principalAmount` rather than reverting:

```solidity
if (combinedSlashAmount > remainingSlashCapacity) {
      principalAmount = remainingSlashCapacity / stakers.length;
}
```

This action does not align with the documented behavior, leading to potential misunderstandings.

#### Proof of Inconsistency:

The documentation claims a contract revert upon slashing beyond capacity. However, the provided code adjusts the `principalAmount` to fit within the available slashing capacity without reverting.

#### Recommendation:

To align the code's behavior with the documented expectation, you should introduce a `revert` statement when the `combinedSlashAmount` exceeds the `remainingSlashCapacity`.

Replace:

```solidity
if (combinedSlashAmount > remainingSlashCapacity) {
      principalAmount = remainingSlashCapacity / stakers.length;
}
```

With:

```solidity
require(combinedSlashAmount <= remainingSlashCapacity, "Slashing amount exceeds slasher's capacity");
```

---


## Non Critical Issues

### <a name="NC-01"></a>[NC-01] Front Running Concern in the Unbonding Process
#### Overview:
The `StakingPoolBase.sol` contract provides a mechanism for stakers to "unbond" their staked tokens in the pool.
Front-running is a concern in blockchain networks where malicious actors can view pending transactions and attempt to place their own transactions ahead with a higher gas fee, gaining an advantage.
So, Front-running attacks may occur during the unbonding period process.

#### Vulnerability Details
1. Stake call unbond().
2. Non friendly admin view the pending transaction in mempool and front run it with setUnbondingPeriod() function call. The admin increase the unbondingPeriod.

#### Recommendation:
Introduce timelock between `setUnbondingPeriod()` and `unbond()` functions.

---

### <a name="NC-02"></a>[NC-02] Important functions doesn't emit an events
#### Overview:
Consider to emit an events in important state changing functions.

### Example Instance
- [setPoolConfig()](https://github.com/code-423n4/2023-08-chainlink/blob/main/src/pools/StakingPoolBase.sol#L400-L405)

---

## Suggestions

### <a name="S-01"></a>[S-01] Revert `claimReward()` if the msg.sender is not the staker or staking pool
#### Overview
Leading to the [Technical Specification](https://github.com/code-423n4/2023-08-chainlink/blob/main/docs/specs.pdf) the `claimReward()` function in RewardVault.sol should be called by a staker or staking pool to
claim a staker’s rewards and returns the amount claimed.
However the `claimReward()` function doesn't revert if the msg.sender is not staker or staking pool. It will be better to revert if the msg.sender is not staker or staking pool.

---

### <a name="S-02"></a>[S-02] Inconsistent Comparison Methods with block.timestamp in _inClaimPeriod() Function
#### Overview
Ensure consistent comparison methods when using block.timestamp.

Example Instance:
Within the `_inClaimPeriod()` function, the comparisons involving `block.timestamp` are inconsistent. Initially, a strict less than `<` comparison is used (`block.timestamp < staker.unbondingPeriodEndsAt`), while later a less than or equal to `<=` comparison is employed (`block.timestamp <= staker.claimPeriodEndsAt`).

```solidity
  function _inClaimPeriod(Staker storage staker) private view returns (bool) {
    if (staker.unbondingPeriodEndsAt == 0 || block.timestamp < staker.unbondingPeriodEndsAt) {
      return false;
    }

    return block.timestamp <= staker.claimPeriodEndsAt;
  }
```

For clarity and to prevent potential edge case issues, it might be beneficial to standardize the approach for both comparisons.

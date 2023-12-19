# GameSwift Security Review

A security review of the [GameSwift](https://gameswift.io/). 

The first modular gaming blockchain based on zkEVM.

Author: [gkrastenov](https://twitter.com/gkrastenov) Independent Security Researcher

This audit report includes all the vulnerabilities, issues and code improvements found during the security review.

## Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

## Risk classification

| Severity           | Impact: High | Impact: Medium | Impact: Low |
| :----------------- | :----------: | :------------: | :---------: |
| Likelihood: High   |   Critical   |      High      |   Medium    |
| Likelihood: Medium |     High     |     Medium     |     Low     |
| Likelihood: Low    |    Medium    |      Low       |     Low     |

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

## Executive summary

### Overview
|               |                                                                                                                                             |
| :------------ | :------------------------------------------------------------------------------------------------------------------------------------------ |
| Project Name  | GameSwift 
| Language  | Solidity 
| Review Commit hash   | 25f850c21d29846b14df6c546bcc6f341cde3235 |
| Fixes Review Commit hash   | 978a3731c6d3c21606b21f9ad7c26cf489eb180d |

### Scope

| File    | 
| :-------------- | 
| Contracts (2)  |
| [LockStaking.sol](https://github.com/KaliszukTomasz/LockStakingContractAudit/blob/25f850c21d29846b14df6c546bcc6f341cde3235/contracts/LockStaking.sol)  |
| [WaitingList.sol](https://github.com/KaliszukTomasz/LockStakingContractAudit/blob/25f850c21d29846b14df6c546bcc6f341cde3235/contracts/WaitingList.sol) |
| **nSLOC (545)** | 

### Issues found

| Severity        | Count | Fixed | Acknowledged |
| :-------------  | ----: | ----: |  ---------:  |
| High risk       |     1 |     1 |            0 |
| Medium risk     |     3 |     3 |            1 |
| Low risk        |     2 |     2 |            1 |
| Informational   |     1 |     0 |            1 |
| **Total**       |   **7** |   **6** |            **1** |

# Findings

| ID     |  Title                                                                                        | Severity      |
| :------| :------------------------------------------------------------------------------------------   | :------------ |
| [H-01] | Unstaking will not work because the unstaked amount is always zero                            | High          |
| [M-01] | Lack of access control in the stakeTokens function                                            | Medium        |
| [M-02] | Unwanted extending staking time                                                               | Medium        |
| [M-03] | Locking user's funds if they stake before the first reward period                             | Medium        |
| [L-01] | The getUserInfo function will return the wrong value for end time                             | Low           |
| [L-02] | Avoid unnecessary external call in checkEthFeeAndRefundDust modifier                          | Low           |
| [I-01] | Requrement for stakingPhase == StakingPhase.Open is unnesery in migrateToTier                 | Informational |


# High
## [H-01] Unstaking will not work because the unstaked amount is always zero
### Impact 
During the creation of `UnbondInfo` for every user who decides to start unstaking their tokens, the amount and release time are recorded. Before that, the user's amount is set to zero, resulting in the creation of `UnbondInfo` with an amount equal to zero. This will block the `finishUnstaking` function because `releasedAmount += unbonding.amount` will be equal to zero, causing it to revert every time it is called.

```solidity
user.amount = 0;
        user.unbonding = UnbondInfo({
        amount : user.amount,
        release : block.timestamp + unbondTime
        });
```

### Recommended Mitigation Steps
Make the following changes:
```diff
-        user.amount = 0;
        user.unbonding = UnbondInfo({
        amount : user.amount,
        release : block.timestamp + unbondTime
        });
+       user.amount = 0;
        emit UnstakeStarted(msg.sender, user.unbonding.amount);
```

### Fixes Review 
Fixed.

# Medium

## [M-01] Lack of access control in the stakeTokens function
### Impact 
The `stakeTokens` function should be accessible only during the `Open` phase. Unfortunately, this is not true because everyone can access it in the `WaitingList` or `Whitelist` phase and stake their tokens. The problem comes from the `notClosedPhase` modifier, where it is checked if the phase is different than `Close`.

```solidity 
 modifier notClosedPhase() {
        require(activePhase != Phase.Closed, "Phase is Closed");
        _;
    }
```

### Recommended Mitigation Steps
Make the following changes:
```diff
- function stakeTokens(uint256 _amount) external payable checkEthFeeAndRefundDust(msg.value) notClosedPhase whenNotPaused {
+ function stakeTokens(uint256 _amount) external payable checkEthFeeAndRefundDust(msg.value) isOpenPhase whenNotPaused {
        stakeTokensInternal(_amount, msg.sender);
    }
```
### Fixes Review 
Fixed.

## [M-02] Unwanted extending staking time
### Impact 
When the users call the `claimRewards` function to claim their rewards, the reward is calculated in the `updateRewards` function, where their staking start time will be updated. This will lead to an extending of their staking time by another 180 days. This behavior limits the user's ability to claim their tokens before the completion of the staking period. So, if someone decides to stake their tokens and after 90 days, decides to claim half of their reward, they have to wait a total of 270 days instead of the expected 180 days at the beginning of staking.

### Recommended Mitigation Steps
Create two a different maps where will be stored last `stakingStartTimes` and when user start his first staking.

```diff
    mapping(address => uint256) public stakingStartTimes;
+   mapping(address => uint256) public stakingFirstTime;
```

### Fixes Review 
Fixed.

## [M-03] Locking user's funds if they stake before the first reward period
### Impact 
If user stakes tokens before starting of first reward period his funds will be locked, no way for unstaking or migrating, in the contract because every time when `calculateReward` is called it will revert. His `lastStakeTime` will be always `<` than `rewardPeriods[i].start ` and durign calculation of `timeDelta` will lead to underflow error. 

```solidity
 uint256 tempStart = rewardPeriods[i].start < user.lastStakeTime ? user.lastStakeTime : rewardPeriods[i].start;

 timeDelta = block.timestamp - tempStart;
```

The first reward start is set in the constructor, equal to the `_stake` variable. Users can stake their tokens if `_stake > block.timestamp` because the phase is set by default to `Open`. If the start of the first reward period is planned a few days after the deployment of the contract and users stake their tokens during this time difference, they will lose their tokens.

### Recommended Mitigation Steps
Set `currentPhase` to `Phase.Closed` by default and change it when the first reward period is planned to start.

### Fixes Review 
Fixed.

# Low

## [L-01] The getUserInfo function will return wrong value for end time
### Impact 
If unbonding is started for the user, the `endTime` will be equal to `lockupDuration` because the `stakingStartTimes` of the user is reset at the beginning of unbonding. The `endTime` should be equal to when the user is able to start unstaking of his tokens.

### Fixes Review 
Fixed.

## [L-02] Avoid unnecessary external call in checkEthFeeAndRefundDust modifier
### Impact 
When a user calls payable function, he needs to pay a fee that is greater than or equal to the fee. If `msg.value > ethFee`, the remaining fee will be returned back to the user `dust = value - ethFee`. In the case where `msg.value == ethFee`, the variable `dust` will be zero and an unnecessary external call to `msg.sender` will be executed.

### Recommended Mitigation Steps
Make the following changes:
```diff
 modifier checkEthFeeAndRefundDust(uint256 value) {
        require(value >= ethFee, "Insufficient fee: the required fee must be covered");

        uint256 dust = value - ethFee;
-        (bool sent,) = address(msg.sender).call{value : dust}("");
-        require(sent, "Failed to return overpayment");

+        if(dust != 0) {
+           (bool sent,) = address(msg.sender).call{value : dust}("");
+           require(sent, "Failed to return overpayment");
        }
        _;
    }
```

### Fixes Review 
Fixed.

# Information

## [I-01] Requrement for stakingPhase == StakingPhase.Open is unnesery in migrateToTier
### Impact 
Requirement for` stakingPhase == StakingPhase.Open` is unnecessary in the `migrateToTier` function because staking in the `LockStaking` contract will be possible only when `stakingPhase == StakingPhase.WaitingList`. If the user calls the `migrateToTier` function when `stakingPhase == StakingPhase.Open`, it will revert the transaction.

`lockStaking.stakeWithWaitingList(migrationAmount, userAddress);`

### Recommended Mitigation Steps
Make the following changes:
```diff
-require(stakingPhase == StakingPhase.WaitingList || stakingPhase == StakingPhase.Open, "Phase not in WaitingList or Open");
+require(stakingPhase == StakingPhase.WaitingList, "Phase not in WaitingList");
```

### Fixes Review 
Acknowledged.
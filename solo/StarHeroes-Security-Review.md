# StarHeroes Security Review

A security review of the [StarHeroes](https://twitter.com/StarHeroes_game).

The first multiplayer third-person space shooter designed for esports. With its top-quality dynamic and competitive gameplay, StarHeroes provides players with what they crave most: true gaming emotions.

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

|                          |                                          |
| :----------------------- | :--------------------------------------- |
| Project Name             | StarHeroes                               |
| Review Commit hash       | 9010b162adfae5f78ff913047dfe9fd223934deb |
| Fixes Review Commit hash | 4662eec97cfe065f3b857a8109f6b5aa75cbdf11 |

### Scope

| File                                    |
| :-------------------------------------- |
| Contracts (2)                           |
| contracts/staking/FixedStaking.sol      |
| contracts/vesting/FixedVestingCliff.sol |
| **nSLOC (364)**                         |

### Issues found

| Severity         |  Count |  Fixed | Acknowledged |
| :--------------- | -----: | -----: | -----------: |
| High risk        |      0 |      0 |            0 |
| Medium risk      |      5 |      4 |            1 |
| Low risk         |      4 |      3 |            1 |
| Informational    |      9 |      6 |            3 |
| Code improvement |      2 |      2 |            0 |
| Gas              |      7 |      6 |            1 |
| **Total**        | **27** | **21** |        **6** |

# Findings

| ID      | Title                                                                                        | Severity         |
| :------ | :------------------------------------------------------------------------------------------- | :--------------- |
| [M-01]  | It is possible to create UnbondInfo with \_amount == 0                                       | Medium           |
| [M-02]  | Owner can register total vesting amount that exceeds what the contract currently holds       | Medium           |
| [M-03]  | Possible reverting of releasableAmount                                                       | Medium           |
| [M-04]  | Unnecessary external call in checkEthFeeAndRefundDust modifier                               | Medium           |
| [M-05]  | getStakingRewards will revert after changing of APY                                          | Medium           |
| [L-01]  | getStakingRewards return userReward for freezed account                                      | Low              |
| [L-02]  | It is possible to register frozen accounts                                                   | Low              |
| [L-03]  | calculateReward will revert if double staking is made                                        | Low              |
| [L-04]  | The nonReentrant modifier should occur before all other modifiers                            | Low              |
| [I-01]  | Use msg.sender instead of owner()                                                            | Informational    |
| [I-02]  | Add an additional parameter for the FreezeAccount event                                      | Informational    |
| [I-03]  | Using SafeMath when compiler is ^0.8.0                                                       | Informational    |
| [I-04]  | Add NatSpec documentation                                                                    | Informational    |
| [I-05]  | Use safeTransfer/safeTransferFrom instead of transfer/transferFrom                           | Informational    |
| [I-06]  | Include old fee value in event                                                               | Informational    |
| [I-06]  | Import declarations should import specific identifiers, rather than the whole file           | Informational    |
| [I-08]  | amount argument can be bigger than the balance                                               | Informational    |
| [I-09]  | Check if constructor argument is in the past                                                 | Informational    |
| [CI-01] | tempStart can be calculated before if condition                                              | Code improvement |
| [CI-02] | Unnecessary updating of lastStakeTime                                                        | Code improvement |
| [G-01]  | Use external modifier instead of public                                                      | Gas              |
| [G-02]  | Use custom error instead of require statement                                                | Gas              |
| [G-03]  | ++i/i++ should be unchecked{++i}/unchecked{i++} when it is not possible for them to overflow | Gas              |
| [G-04]  | Use != 0 instead of > 0                                                                      | Gas              |
| [G-05]  | Add unchecked {} for subtractions where the operands can not underflow                       | Gas              |
| [G-06]  | Use immutable when is possible                                                               | Gas              |
| [G-07]  | Default int values are manually set                                                          | Gas              |

# Medium

## [M-01] It is possible to create UnbondInfo with \_amount == 0

### Impact

Every user can call the `startUnstaking` function with `_amount` equal to 0 and to create an `UnbondInfo` position. They do not need to stake any amount before that and it will emit the `UnstakeStarted(msg.sender, _amount)` event.

### Recommended Mitigation Steps

```diff
    function startUnstaking(uint256 _amount) public payable checkEthFeeAndRefundDust(msg.value) nonReentrant {
        UserInfo storage user = userInfo[msg.sender];

+        require(_amount > 0, "Zero amount");
        require(user.unbondings.length < unbondLimit, "startUnstaking: limit reached");
        require(user.amount >= _amount, "startUnstaking: not enough staked amount");
```

### Fixes Review

Fixed.

## [M-02] Owner can register total vesting amount that exceeds what the contract currently holds

### Impact

Owner can register a vesting amount for every user. If the total registered vested amount exceeds what the contract currently holds, some users will not be able to receive their tokens.

```solidity
    function registerVestingAccounts(address[] memory _userAddresses, uint256[] memory _amounts) external onlyOwner {
        require(_amounts.length == _userAddresses.length, "Amounts and userAddresses must have the same length");

        for (uint i = 0; i < _userAddresses.length; i++) {
            userTotal[_userAddresses[i]] = _amounts[i];
            //@audit contract has to have enough balance
        }
    }
```

### Fixes Review

Acknowledged.

### Recommended Mitigation Steps

In every for loop iteration, check if the total registered vesting amount is bigger than the balance of the contract. Also, include the expected APY in the calculation.

## [M-03] Possible reverting of releasableAmount

### Impact

The releaseable amount is calculated based on the vested amount minus the amount already released. The vested amount is equal to the total tokens multiplied by the elapsed time and divided by the total vesting duration.

`uint256 vestedAmount = (totalTokens * elapsedTime) / totalVestingTime;`

If the owner initially registers a big amount of vesting for a user, the user is allowed after some time to release a part of this amount based on the above formula. The owner has the rights to change the vesting amount for every user and if he decreases the vested amount of a user, it can open potential vulnerabilities. It is possible for` tokensReleased[userAddress]` to be bigger than the newly calculated vested amount `vestedAmount` and the transaction to revert because of arithmetic underflow.

```solidity
        uint256 elapsedTime = block.timestamp - config.startDate;
        uint256 totalVestingTime = config.endDate - config.startDate;

        uint256 vestedAmount = (totalTokens * elapsedTime) / totalVestingTime;

        return vestedAmount - tokensReleased[userAddress]; //@audit potential vulnerabilities
```

## Recommended Mitigation Steps

Make the following changes:

```diff
-return vestedAmount.sub(tokensReleased[userAddress]);
+return vestedAmount < tokensReleased[userAddress] ? 0 : vestedAmount - tokensReleased[userAddress];
```

### Fixes Review

Fixed.

## [M-04] Unnecessary external call in checkEthFeeAndRefundDust modifier

### Impact

When a user calls payable function, he needs to pay a fee that is greater than or equal to the storage variable `ethFee`. If `msg.value `is larger than `ethFee`, the remaining fee will be returned back to the user` dust = value - ethFee`. In the case where `msg.value == ethFee`, the variable `dust` will be zero, and an unnecessary external call to `msg.sender` will be executed. This will increase the transaction cost.

## Recommended Mitigation Steps

I recommend to add more stricly condiiton for fee

```diff
-require(value >= ethFee, "Insufficient fee: the required fee must be covered");
+require(value == ethFee, "Insufficient fee: the required fee must be covered");
```

or to return remaning fee back to user only when `dust != 0`.

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

## [M-05] getStakingRewards will revert after changing of APY

### Impact

Owner has the right to change APY at any time. He need to add a new reward rate by calling the `setRewardRate` function. The problem comes in the calculation of staking rewards. When a new reward rate is added, `rewardPeriods.length` becomes greater than 1. In the last iteration of the for loop in the `getStakingRewards` function, `elapsedTime` will be calculated as follows:

`uint256 elapsedTime = block.timestamp < config.startDate ? block.timestamp - rewardPeriods[i].start : config.startDate - rewardPeriods[i].start;`

The statement `block.timestamp < config.startDate` is always false, and `elapsedTime` is equal to `config.startDate - rewardPeriods[i].start`. Unfortunately, `rewardPeriods[i].start` is always greater than `config.startDate`, and this will revert with panic code 17, indicating arithmetic underflow.

The issue is marked as `medium` because the owner is trusted.

```solidity
        for (uint256 i = 0; i < rewardPeriods.length; i++) {

            if (i == rewardPeriods.length - 1) {
                //@audit H -> revert if length > 1 in config.startDate - rewardPeriods[i].start;
                uint256 elapsedTime = block.timestamp < config.startDate ? block.timestamp - rewardPeriods[i].start : config.startDate - rewardPeriods[i].start;

                userReward += userTotal[_userAddress] * elapsedTime * rewardPeriods[i].rate / VESTING_DIVIDER / YEAR_DIVIDER;
            } else {
                uint256 duration = rewardPeriods[i + 1].start - rewardPeriods[i].start;
                userReward += userTotal[_userAddress] * duration * rewardPeriods[i].rate / VESTING_DIVIDER / YEAR_DIVIDER;
            }
        }
```

### Proof of concept

```javascript
import { time } from "@nomicfoundation/hardhat-network-helpers";
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { expect } from "chai";
import { Contract } from "ethers";
import { ethers } from "hardhat";

describe.only("TokenVesting", () => {
  let vesting: Contract;

  let user: SignerWithAddress;
  let owner: SignerWithAddress;

  beforeEach(async () => {
    [owner, user] = await ethers.getSigners();

    const Vesting = await ethers.getContractFactory("FixedVestingCliff");

    await time.increaseTo(1699900000);
    const endDate = 1700050000 + 2629743; // end after 1 month

    vesting = await Vesting.deploy(
      1700000000, // tge time
      1700050000, // start time
      endDate,
      "0x580e933d90091b9ce380740e3a4a39c67eb85b4c", // token address
      5000, // 5%
      0 // zero fee
    );
  });

  it.only("PoC", async () => {
    const amount = ethers.utils.parseUnits("9", 18);

    // After 5k block, the owner registers vesting accounts.
    await time.increaseTo(1700040000);
    await vesting
      .connect(owner)
      .registerVestingAccounts([user.address], [amount]);

    expect(
      await vesting.connect(user.address).getStakingRewards(user.address)
    ).to.not.be.eq("0");

    // After 10k block, the owner updates the reward rate to 3% (the percentage does not matter)
    await time.increaseTo(1700140000);
    await vesting.connect(owner).setRewardRate(3000);

    // After 1k block, user calls the getStakingRewards function.
    await time.increaseTo(1700150000);

    // Revert with panic code 17: Arithmetic operation underflowed
    await vesting.connect(user).getStakingRewards(user.address);
  });
});
```

### Recommended Mitigation Steps

Add an additional check for `block.timestamp < config.startDate` in the `setRewardRate` function..

### Fixes Review

Fixed.

# Low

## [L-01] getStakingRewards return userReward for freezed account

### Impact

Function `getStakingRewards` returns the calculated `userReward` for frozen accounts, which are expected to not receive staking rewards.

### Recommended Mitigation Steps

Add `accountNotFrozen` modifier for `getStakingRewards` function.

### Fixes Review

Fixed.

## [L-02] It is possible to register frozen accounts

### Impact

Frozen accounts can be registered by the owner. This should not be possible because when an account is frozen, it is not able to participate in vesting.

```solidity
        require(_amounts.length == _userAddresses.length, "Amounts and userAddresses must have the same length");

        for (uint i = 0; i < _userAddresses.length; i++) {
            //@audit can register frozen account
            userTotal[_userAddresses[i]] = _amounts[i];
        }
```

### Recommended Mitigation Steps

Add an additional check for every account that will be registered.

### Fixes Review

Acknowledged.

## [L-03] calculateReward will revert if double staking is made

### Impact

The `calculateReward` function will revert if a user made a double staking before `rewardPeriods[i].start`. The variable `tempStart` will be equal to `rewardPeriods[i].start`, which is less than `block.timestamp` because start is in the future.

```solidity
        uint256 tempStart = rewardPeriods[i].start < user.lastStakeTime ? user.lastStakeTime : rewardPeriods[i].start;

        //@audit this will revert if double staking is made before rewardPeriods[i].start
        // tempstart = rewardPeriods[i].start    which is < than block.timestamp because start is in the future
        timeDelta = block.timestamp - tempStart;
```

### Recommended Mitigation Steps

This is a rare case and as we discussed, the difference between deploying the contract and starting the staking will be small. I recommend to notify the user to not stake until the staking period has started.

### Fixes Review

Fixed. An additional check has been added
`require(block.timestamp >= rewardPeriods[0].start, "Staking not started.");`

## [L-04] The nonReentrant modifier should occur before all other modifiers

### Impact

The modifiers are read one by one depending on their order. The `checkEthFeeAndRefundDust` modifier is placed before the `nonReentrant` modifier, which allows the possibility for a user to execute an arbitrary call or attempt a reentrancy attack.

This is a best practice to protect against reentrancy in other modifiers.

### Fixes Review

Fixed.

# Information

## [I-01] Use msg.sender instead of owner()

### Impact

When a function is only accessible by the owner due to the presence of the `onlyOwner` modifier, there is no need to call an internal function to obtain the owner's address. In this case,` msg.sender` is equal to `owner()`.

### Recommended Mitigation Steps

Make the following changes:

```diff
    function withdrawEth(uint256 amount) external onlyOwner {

-       (bool success,) = payable(owner()).call{value : amount}("");
+       (bool success,) = payable(msg.sender).call{value : amount}("");
        if (!success) {
            revert WithdrawFailed();
        }
-        emit WithdrawEth(owner(), amount);
+        emit WithdrawEth(msg.sender, amount);
    }
```

### Fixes Review

Fixed.

## [I-02] Add an additional parameter for the FreezeAccount event

Owner can freeze and unfreeze any user at any time. It would be beneficial to add the `_freeze` parameter to the `FreezeAccount` event to clarify which action is being performed.

```diff
- emit FreezeAccount(_userAddress);
+ emit FreezeAccount(_userAddress, _freeze);
```

### Fixes Review

Fixed.

## [I-03] Using SafeMath when compiler is ^0.8.0

There is no need to use `SafeMath` when compiler is ^0.8.0 because it has built-in under/overflow checks.

### Fixes Review

Fixed.

## [I-04] Add NatSpec documentation

NatSpec documentation to all public methods and variables is essential for better understanding of the code by developers and auditors and is strongly recommended.

### Fixes Review

Acknowledged.

## [I-05] Use safeTransfer/safeTransferFrom instead of transfer/transferFrom

Using safeTransfer/safeTransferFrom instead of transfer/transferFrom. It will automatically handle the returning of the bool parameter.

### Recommended Mitigation Steps

Example:

```diff
-require(token.transferFrom(msg.sender, address(this), _amount), "Stake transfer failed.");`
+token.safeTransferFrom(msg.sender, address(this), _amount);`
```

### Fixes Review

Acknowledged.

## [I-06] Include old fee value in event

After changing the fee value in the `updateEthFee` function, it would be good to include the old value of the fee in the `UpdateFee` event.

```diff
-emit UpdateFee(_newFee);
+emit UpdateFee(oldFee, _newFee);
```

### Fixes Review

Fixed.

## [I-07] Import declarations should import specific identifiers, rather than the whole file

Using import declarations of the form `import {<identifier_name>} from "some/file.sol"` avoids polluting the symbol namespace making flattened files smaller and speeds up compilation

### Fixes Review

Fixed.

## [I-08] amount argument can be bigger than the balance

The `amount` argument can be bigger than the balance of the contract and transaction will revert.
Add an additional check for that which returns a proper error with good information if this happens.

```solidity
    function withdrawEth(uint256 amount) external onlyOwner {
        //@audit amount > balance
        (bool success,) = payable(owner()).call{value : amount}("");
        if (!success) {
            revert WithdrawFailed();
        }
        emit WithdrawEth(owner(), amount);
    }
```

### Fixes Review

Fixed.

## [I-09] Check if constructor argument is in the past

Check if the `_start` argument is in the past in the constructor of the `FixedStaking` contract. This can also be done for the `_tgeDate` in the constructor of the `FixedVestingCliff` contract.

### Fixes Review

Acknowledged.

# Code Improvements

## [CI-01] tempStart can be calculated before if condition

### Details

Variable `tempStart` can be calculated before if condition.

### Recommended Mitigation Steps

Make the following changes:

```diff
        for (uint256 i = startIndex; i < rewardPeriods.length; i++) {

            uint256 timeDelta;
+           uint256 tempStart = rewardPeriods[i].start < user.lastStakeTime ? user.lastStakeTime : rewardPeriods[i].start;
            if (i < rewardPeriods.length - 1) {

-                uint256 tempStart = rewardPeriods[i].start < user.lastStakeTime ? user.lastStakeTime : rewardPeriods[i].start;
                timeDelta = rewardPeriods[i + 1].start - tempStart;
                reward += user.amount * rewardPeriods[i].rate * timeDelta / YEAR_DIVIDER / RATE_DIVIDER;
            } else {


-                uint256 tempStart = rewardPeriods[i].start < user.lastStakeTime ? user.lastStakeTime : rewardPeriods[i].start;
                timeDelta = block.timestamp - tempStart;
                reward += user.amount * rewardPeriods[i].rate * timeDelta / YEAR_DIVIDER / RATE_DIVIDER;
            }
        }
```

### Fixes Review

Fixed.

## [CI-02] Unnecessary updating of lastStakeTime

### Details

Unnecessary updating of the `lastStakeTime` variable if `pending > 0`. At the end of the function, `lastStakeTime` is updated every time.

```solidity
        user.amount += _amount;
        user.lastStakeTime = block.timestamp;
        emit StakeStarted(msg.sender, _amount);
```

### Recommended Mitigation Steps

Make the following changes:

```diff
        if (user.amount != 0) {
            uint256 pending = calculateReward(msg.sender);

            if (pending > 0) { //@audit-r != 0 better
                user.waitingRewards += pending;

                 //@audit code-improvement
-                user.lastStakeTime = block.timestamp;
            }
        }

        user.amount += _amount;
        user.lastStakeTime = block.timestamp;
        emit StakeStarted(msg.sender, _amount);
```

### Fixes Review

Fixed.

# Gas

## [G-01] Use external modifier instead of public

The `external` modifier should be used instead of `public` for `claimStakingRewards` function since is not used internally.

## [G-02] Use custom error instead of require statement

Use custom errors instead of require statements. It saves more gas.

## [G-03] ++i/i++ should be unchecked{++i}/unchecked{i++} when it is not possible for them to overflow

The unchecked keyword is new in solidity version 0.8.0, so this only applies to that version or higher, which these instances are. This saves 30-40 gas per loop

## [G-04] Use != 0 instead of > 0

Replace spotted instances with != 0 for uints as this uses less gas.

## [G-05] Add unchecked {} for subtractions where the operands can not underflow

In some places in the codebase, `unchecked {}` can be used for subtractions because they are already checked.

## [G-06] Use immutable when is possible

The storage variable `token` can be immutable in the `FixedStaking` contract because it will never be changed.

## [G-07] Default int values are manually set

In instances where a new variable is defined, there is no need to set it to it's default value.

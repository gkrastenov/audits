# GameSwift Security Review

A security review of the [GameSwift](https://gameswift.io/). GameSwift is a one-stop gaming ecosystem that aims to introduce cross-chain interoperability to Web3 gaming.

Author: [gkrastenov](https://twitter.com/gkrastenov) Independent Security Researcher

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

## Executive summary

### Overview
|               |                                                                                                                                             |
| :------------ | :------------------------------------------------------------------------------------------------------------------------------------------ |
| Project Name  | GameSwift                                                                                                                                                 
| Review Commit hash   | 8d6c99af4b7864043234efef63f5bdf968045317|

### Scope

| File    | 
| :-------------- | 
| Contracts (2)  |
| contracts/airdrop/Airdrop.sol|
| contracts/staking/Staking.sol|
| **nSLOC (415)** | 


### Issues found

| Severity      | Count | Fixed | Acknowledged |
| :------------ | ----: | ----: |  ---------:  |
| High risk     |     0 |     - |            - |
| Medium risk   |     2 |     - |            - |
| Low risk      |     2 |     - |            - |
| Informational |     7 |     - |            - |
| Gas           |     4 |     - |            - |
| **Total**         |   **15** |   **-** |            **-** |

# Findings

|ID  |  Title                                                                                | Severity      |
|:---| :------------------------------------------------------------------------------------| :------------ |
|M-1 | Unnecessary external call in checkEthFeeAndRefundDust modifier                       | Medium        |
|M-2 | It is possible to create UnbondInfo with _amount == 0                                | Medium        |
|L-1 | The nonReentrant modifier should occur before all other modifiers                    | Low           |
|L-2 | Use msg.sender instead of owner()                                                    | Low           |
|I-1 | Redundant if condition                                                               | Informational |
|I-2 | Add 100% test coverage                                                               | Informational |
|I-3 | Missing non-bytes32(0) checks                                                        | Informational |
|I-4  | Use safeTransfer instead of transfer                                                | Informational |
|I-5  | Add NatSpec documentation                                                           | Informational |
|I-6  | Missing non-zero address checks                                                     | Informational |
|I-7  | Import declarations should import specific identifiers                              | Informational |
|G-1|  Use custom error instead of require statement                                        | Gas |
|G-2|  ++i/i++ should be unchecked{++i}/unchecked{i++} when it is not possible for them to overflow  | Gas |
|G-3|  Use constant for REWARDS_PRECISION                                                  | Gas |
|G-4|  Use extnernal modifier instead of public                                             | Gas |

# Medium

## [M-01] Unnecessary external call in checkEthFeeAndRefundDust modifier
### Impact 
When a user calls `stake`, `startUnstaking`, `claim`, or `restake` functions, he needs to pay a fee that is greater than or equal to the storage variable `ethFee`. If `msg.value` is larger than `ethFee`, the remaining fee will be returned back to the user `dust = value - ethFee`. In the case where `msg.value == ethFee`, the variable `dust` will be zero, and an unnecessary external call to `msg.sender` will be executed. This will increase the transaction cost.

```solidity
    modifier checkEthFeeAndRefundDust(uint256 value) {
        require(value >= ethFee, "Insufficient fee: the required fee must be covered");

        //@audit if ethFee == value will make unnecessary call. 

        uint256 dust = value - ethFee;
        (bool sent,) = address(msg.sender).call{value : dust}("");
        require(sent, "Failed to return overpayment");
        _;
    }
```

### Recommended Mitigation Steps
Return remaning fee back to user only when `dust != 0`

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

## [M-02] It is possible to create UnbondInfo with _amount == 0    
### Impact 
Every user can call the `startUnstaking` function with `_amount` equal to 0 and to create an `UnbondInfo` position. They do not need to stake any amount before that, and it will emit the `UnstakeStarted(msg.sender, _amount)` event.

### Recommended Mitigation Steps
Add additional check for that.

```diff
    function startUnstaking(uint256 _amount) public payable checkEthFeeAndRefundDust(msg.value) nonReentrant {
        UserInfo storage user = userInfo[msg.sender];

+        require(_amount > 0, "Zero amount");
        require(user.unbondings.length < unbondLimit, "startUnstaking: limit reached");
        require(user.amount >= _amount, "startUnstaking: not enough staked amount");
```

# Low severity

## [L-01] The nonReentrant modifier should occur before all other modifiers
### Impact 
The modifiers are read one by one depending on their order. The `checkEthFeeAndRefundDust` modifier is placed before the `nonReentrant` modifier, which allows the possibility for a user to execute an arbitrary call or attempt a reentrancy attack. 

This is a best practice to protect against reentrancy in other modifiers.
### Recommended Mitigation Steps
Make the following changes for `stake`, `startUnstaking`, `claim`, and `restake` functions:

```diff
-function stake(uint256 _amount) public payable checkEthFeeAndRefundDust(msg.value) nonReentrant
+function stake(uint256 _amount) public payable nonReentrant checkEthFeeAndRefundDust(msg.value)
```

## [L-02] Use msg.sender instead of owner()
### Impact 
When a function is only accessible by the owner due to the presence of the `onlyOwner` modifier, there is no need to call an internal function to obtain the owner's address. In this case, `msg.sender` is equal to `owner()`.

```solidity
function withdrawEth(uint256 amount) external onlyOwner {
        //@audit use directly msg.sender 
        (bool success,) = payable(owner()).call{value : amount}("");
        if (!success) {
            revert WithdrawFailed();
        }

        emit WithdrawEth(owner(), amount);
}
```

### Recommended Mitigation Steps
Make the following changes:

```diff
- (bool success,) = payable(owner()).call{value : amount}("");
+  (bool success,) = payable(msg.sender).call{value : amount}("");

```

# Information

## [I-01] Redundant if condition
The condition `block.timestamp < vestingStartDate` in the `getVestedAmount` function is redundant because there is already a check before calling the function for `block.timestamp > vestingStartDate`. This means that `block.timestamp` will always be greater than `vestingStartDate` in the `getVestedAmount` function.

in `getClaimableAmount`function:

```solidity
if (block.timestamp > vestingStartDate) {
    claimable += getVestedAmount(halfOfAirdrop);
}
```

## [I-02] Add 100% test coverage
Currently, the contracts have 0% test coverage. Increasing the test coverage and testing every unit of code can help prevent unexpected behavior.

## [I-03] Missing non-bytes32(0) checks
Add non-bytes32(0) checks for all places where `merkleRoot` is set from the owner.

## [I-04] Use safeTransfer instead of transfer
Using safeTransfer instead of transfer will automatically handle the returning of the bool parameter.

### Recommended Mitigation Steps
Example:

```diff
-require(token.transfer(msg.sender, claimableAmount), "Airdrop transfer failed.");`
+token.safeTransfer(msg.sender, claimableAmount);`
```

## [I-05] Add NatSpec documentation
NatSpec documentation to all public methods and variables is essential for better understanding of the code by developers and auditors and is strongly recommended.

## [I-06] Missing non-zero address checks
Add non-zero address checks for all address type arguments. 

## [I-07] Import declarations should import specific identifiers, rather than the whole file
Using import declarations of the form `import {<identifier_name>} from "some/file.sol"` avoids polluting the symbol namespace making flattened files smaller and speeds up compilation

# Gas

## [G-01] Use custom error instead of require statement 
Use custom errors instead of require statements. It saves more gas.

Instances:
Staking: [51](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/staking/Staking.sol#L51), [54](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/staking/Staking.sol#L54), [114](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/staking/Staking.sol#L114), [132-133](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/staking/Staking.sol#L132-L133), [170-171](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/staking/Staking.sol#L170-L171), [179](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/staking/Staking.sol#L179), [182](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/staking/Staking.sol#L182), [217](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/staking/Staking.sol#L217)

Airdrop: [71](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/airdrop/Airdrop.sol#L71), [76](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/airdrop/Airdrop.sol#L76), [78](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/airdrop/Airdrop.sol#L78)

## [G-02] ++i/i++ should be unchecked{++i}/unchecked{i++} when it is not possible for them to overflow
The unchecked keyword is new in solidity version 0.8.0, so this only applies to that version or higher, which these instances are. This saves 30-40 gas [per loop](https://gist.github.com/hrkrshnn/ee8fabd532058307229d65dcd5836ddc#the-increment-in-for-loop-post-condition-can-be-made-unchecked)

Instances:
Staking: [166](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/staking/Staking.sol#L166), [208](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/staking/Staking.sol#L208)

Airdrop: [131](https://github.com/GameSwift/gs-evm-contracts/blob/8d6c99af4b7864043234efef63f5bdf968045317/contracts/airdrop/Airdrop.sol#L131)

## [G-03] Use constant for REWARDS_PRECISION
Use a constant variable for REWARDS_PRECISION instead of `1e12`. Constant variables are saved in bytecode and help save gas.

## [G-04] Use external access modifier instead of public
For the functions `stake`, `restake`, `startUnstaking`, `finishUnstaking`, `restake`, `getUserUnbondings`, and claim, use the external access modifier because they will be called only externally and not internally.
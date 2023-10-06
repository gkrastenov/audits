# SpartaDex Security Review

A second security review of the [SpartaDex](https://spartadex.io/) before deploying of their contracts on Arbitrum.

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

## Executive summary

### Overview
|               |                                                                                                                                             |
| :------------ | :------------------------------------------------------------------------------------------------------------------------------------------ |
| Project Name  | SpartaDex                                                                                                                                                 
| Review Commit hash   | [fb6b6a69fe5ecba792d49fa460e94d0b98712a8b](https://github.com/SpartaDEX/sdex-smart-contracts/tree/fb6b6a69fe5ecba792d49fa460e94d0b98712a8b) |
| Fixes Review Commit hash   | [d7e8823e0bbaa6b791f918ffa01c923e55147a9e](https://github.com/SpartaDEX/sdex-smart-contracts/tree/d7e8823e0bbaa6b791f918ffa01c923e55147a9e) |

### Scope

| File    | 
| :-------------- | 
| Contracts (6)  |
| [contracts/tokens/Polis.sol](https://github.com/SpartaDEX/sdex-smart-contracts/blob/fb6b6a69fe5ecba792d49fa460e94d0b98712a8b/contracts/tokens/Polis.sol)   |
| [contracts/dex/core/SpartaDexFactory.sol](https://github.com/SpartaDEX/sdex-smart-contracts/blob/fb6b6a69fe5ecba792d49fa460e94d0b98712a8b/contracts/dex/core/SpartaDexFactory.sol)   |
| [contracts/staking/SpartaStaking.sol](https://github.com/SpartaDEX/sdex-smart-contracts/blob/fb6b6a69fe5ecba792d49fa460e94d0b98712a8b/contracts/staking/SpartaStaking.sol)   |
| [contracts/SpartaRewardLpLinearStaking.sol](https://github.com/SpartaDEX/sdex-smart-contracts/blob/fb6b6a69fe5ecba792d49fa460e94d0b98712a8b/contracts/staking/SpartaRewardLpLinearStaking.sol)   |
| [contracts/ContractsRepository.sol](https://github.com/SpartaDEX/sdex-smart-contracts/blob/fb6b6a69fe5ecba792d49fa460e94d0b98712a8b/contracts/ContractsRepository.sol)   |
| [contracts/vesting/SpartaVesting.sol](https://github.com/SpartaDEX/sdex-smart-contracts/blob/fb6b6a69fe5ecba792d49fa460e94d0b98712a8b/contracts/vesting/SpartaVesting.sol)   |

### Issues found

| Severity      | Count | Fixed | Acknowledged |
| :------------ | ----: | ----: |  ---------:  |
| Critical risk |     3 |     3 |            0 |
| High risk     |     1 |     1 |            0 |
| Medium risk   |     0 |     0 |            0 |
| Low risk      |     0 |     0 |            0 |
| Informational |     6 |     6 |            0 |
| **Total**         |   **10** |   **10** |            **0** |

# Findings
|ID  |  Title                                                                                | Severity      |
|:---| :------------------------------------------------------------------------------------| :------------ |
|C-1 | The `claim()` function will always revert                                            | Critical      |
|C-2 | The `currentImplementation()` function will always revert                            | Critical      |
|C-3 | Users can double their StakedSparta tokens                                           | Critical      |
|H-1 | Only the staked amount is moved without the reward                                   | High          |
|I-1 | Add an additional check: If `claimableAmount > 0`                                    | Informational |
|I-2 | The deactivate function is redundant                                                 | Informational |
|I-3 | Use external modifier instead of public                                              | Informational |
|I-4 | In the RewardTaken event, toTransfer should be used instead of reward                | Informational |
|I-5 | Emit event after changing the `baseTokenURI/contractURI`                             | Informational |
|I-6 | In the event MovedToNextImplementation, `total` should be used instead of` balance ` | Informational |

# Critical
## [C-01] The claim() function will always revert
When the `claim()` function is called, a new Sparta staking address is retrieved from the `Repository` contract. The argument `"SPARTA_STAKING"` is incorrectly used in the `getContract()` function; it should be the `keccak256` hash of `"SPARTA_STAKING"`. The `getContract()` function will return a`ddress(0)` and the function will revert when trying to stake the claimed amount.

### Impact 

```solidity
        address spartaStakingAddress = contractsRepository.getContract(
            "SPARTA_STAKING" //@audit-issue wrong input
        );
```
### Recommended Mitigation Steps
Users have to move the tokens from one iteration to the next without unstaking period and fees

```diff
        address spartaStakingAddress = contractsRepository.getContract(
-            "SPARTA_STAKING"
+            keccak256(“SPARTA_STAKING”) 
        );
```

### Fixes Review 
Fixed.

## [C-02] The currentImplementation() function will always revert
### Impact 
The idea behind `currentImplementation()` is to allow users to move tokens from one iteration to the next without a unstaking period and fees. The `getContract()` function will return the last iteration from the `Repository` contract. The problem arises in the if condition because `spartaStakingAddress == address(spartaStakingAddress)` is always true and the `currentImplementation()` function will always revert when called. Users will not be able to move their tokens.


```
function currentImplementation() public view returns (ISpartaStaking) {
        address spartaStakingAddress = contractsRepository.getContract(
            SPARTA_STAKING_CONTRACT_ID
        );

        if (spartaStakingAddress == address(spartaStakingAddress)) {
            revert CurrentImplementation(); //@audit-issue it will always revert
        }

        return SpartaStaking(spartaStakingAddress);
    }
```

### Proof of Concept
```javascript
it("C-02, PoC", async function () {
    // Some user logic here:
    await mintTokensToContract();
    await mintTokensToStakerAndApprove();
    await initialize();
    await time.setNextBlockTimestamp(stakingStart);
    await instance.connect(staker).stake(stakedTokens);

    // PoC start:
    const contractId = ethers.utils.id("SPARTA_STAKING");

    contractsRepository.setContract(
      ethers.utils.solidityKeccak256(["string"], ["SPARTA_STAKING"]),
      "0x0000000000000000000000000000000000000000"
    );
    
    expect(await contractsRepository.connect(stakingOwner).tryGetContract(contractId)).to.be.eq("0x0000000000000000000000000000000000000000");

   // first call of currentImplementation
   await expect(instance.connect(staker).currentImplementation())
    .to.be.reverted;  

    // Creating new instances of SpartaStaking
    let instance2: Contract;
    
    const SpartaStaking = await ethers.getContractFactory("SpartaStaking");

    instance2 = await SpartaStaking.deploy(
      sparta.address,
      stakedSparta.address,
      acl.address,
      contractsRepository.address,
      treasury.address,
      fees
    );

    // the addresses are different
    expect(instance2.address).to.not.eq(instance.address);

    // overide the address in the repository
    // stakingOwner is REPOSITORY_OWNER

   await expect(await contractsRepository.connect(stakingOwner).setContract(contractId, instance2.address))
    .to.not.be.reverted;

   expect(await contractsRepository.connect(stakingOwner).getContract(contractId)).to.be.eq(instance2.address);

    // second call of currentImplementation
   await expect(instance.connect(staker).currentImplementation())
      .to.be.reverted;
  });
```
### Recommended Mitigation Steps
Make the following changes:

```diff
function currentImplementation() public view returns (ISpartaStaking) {
        address spartaStakingAddress = contractsRepository.getContract(
            SPARTA_STAKING_CONTRACT_ID
        );

-       if (spartaStakingAddress == address(spartaStakingAddress))
+       if (spartaStakingAddress == address(this)) {
            revert CurrentImplementation();
        }

        return SpartaStaking(spartaStakingAddress);
    }
```

### Fixes Review 
Fixed.

## [C-03] Users can double their StakedSparta tokens
### Impact 
When a new staking Sparta implementation is added and users decide to move their staked and reward amounts, they will call the `stakeAs` function, which will mint newly StakedSparta tokens. This is not the correct behavior because it will double their token balance. They should only move their existing balance instead of doubling it.

### Recommended Mitigation Steps
Burn the StakedSparta tokens during moving to the new round implementation.

### Fixes Review 
Fixed.


# High
## [H-01] Only the staked amount is moved without the reward
### Impact
When a user calls `moveToNextSpartaStakingWithReward()`, he have to move both the already staked amount and the reward to the new Sparta staking implementation. Currently, only the user's balance is moved without the reward amount.

```solidity
        uint256 total = balance + reward;
        if (total == 0) {
            revert ZeroAmount();
        }
        balanceOf[msg.sender] = 0;
        rewards[msg.sender] = 0;
        totalSupply -= balance;
        sparta.forceApprove(address(current), balance); //@audit-issue it should be total instead of balance
        current.stakeAs(msg.sender, balance);
```
### Recommended Mitigation Steps
Make the following changes:

```diff
-       sparta.forceApprove(address(current), balance);
-       current.stakeAs(msg.sender, balance);
+       sparta.forceApprove(address(current), total);
+       current.stakeAs(msg.sender, total);
```

### Fixes Review 
Fixed.


# Informational
## [I-01] Add an additional check: If claimableAmount > 0
To avoid zero staking and transferring, add an additional check if `claimableAmount > 0`.
### Fixes Review 
Fixed.

## [I-02] The deactivate function is redundant
The `deactivate()` function is redundant because you can update the restricted variable with the `setActivated()` function.

```soldiity
     //@audit-issue redundant function
    function deactivate() external onlyLiquidityController {
        restricted = false;
    }

    function setActivated(bool value) external onlyLiquidityController {
        if (restricted != value) {
            restricted = value;
        }
    }
```
### Fixes Review 
Fixed.

## [I-03] Use external modifier instead of public
In the `SpartaRewardLpLinearStaking` contract, use the `external` modifier instead of `public` since the functions are not used internally.
### Fixes Review 
Fixed.

## [I-04] In the RewardTaken event, toTransfer should be used instead of reward
In the `RewardTaken` event, `toTransfer` should be used instead of `reward`. The purpose of the event is to emit the wallet address (`msg.sender`, in your case) and the amount he will receive. If `spartaStakingAddress != address(0)` is true, then the user will receive only 25% of the reward.
### Fixes Review 
Fixed.

## [I-05] Emit event after changing the baseTokenURI/contractURI
In the `Polis` contract, the event should be emitted after changing the `baseTokenURI/contractURI`, not before.
### Fixes Review 
Fixed.

## [I-06] In the event MovedToNextImplementation, total should be used instead of balance
In the event `MovedToNextImplementation`, `total` should be used instead of `balance`.

Make the following changes:
```diff
-emit MovedToNextImplementation(msg.sender, balance, reward);
+emit MovedToNextImplementation(msg.sender, total, reward);
```
### Fixes Review 
Fixed.
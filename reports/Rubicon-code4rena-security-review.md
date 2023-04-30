
# Rubicon Code4rena Security Review

A security review of the [Rubicon v2](https://code4rena.com/contests/2023-04-rubicon-v2). An onchain order book protocol for Ethereum, built on L2s.

Author: [gkrastenov](https://twitter.com/gkrastenov)

This audit report includes all the vulnerabilities, issues and code improvements found during the security review.

# Findings

| ID     | Title                                                                       | Severity      | Resolution   |
| :----- | :-------------------------------------------------------------------------- | :------------ | :----------- |
| [M-01] | Market can be inflated with unkillable offers                               | Medium        | Soon         |
| [M-02] | Modifier can_offer is incorrect                                             | Medium        | Soon         |
| [M-03] | Trouble with migration of USDT pool                                         | Medium        | Soon         |
| [L-01] | notifyRewardAmount() will revert if rewardsDuration[token] is zero          | Low           | Soon         |
| [I-01] | Using SafeMath when compiler is ^0.8.0                                      | Informational | Soon         |
| [I-02] | Unused storage variable                                                     | Informational | Soon         |
| [I-03] | Code is not properly formatted                                              | Informational | Soon         |
| [I-04] | NatSpec missing from external functions                                     | Informational | Soon         |
| [I-05] | Consider using custom errors instead of require statements                  | Informational | Soon         |
| [I-06] | Open TODO in code                                                           | Informational | Soon         |

# Medium severity

## [M-01] Market can be inflated with unkillable offers

### Impact
The market can be inflated with offers which can not be killed/canceled or bought. Currently, when a new offer is created owner and recipient address are coming from arguments previously setted from msg.sender [516-517](https://github.com/code-423n4/2023-04-rubicon/blob/main/contracts/RubiconMarket.sol#L516-L517) . Malicious user can create many offers where owner and recipient address are `address(0)` like make this offers to can not be killed/canceled or bought.

### Proof of Concept
Paste this test in ProtocolDeployment.ts

```
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

### Recommended Mitigation Steps
Add one more condition when the user create a new offer.

```
require(owner != address(0) && recipient != address, "Zero address for owner and recipient!");
```

## [M-02] Modifier can_offer is incorrect

### Impact
Modifier can_offer is incorrect because it checks if `!isClosed()` is true [598](https://github.com/code-423n4/2023-04-rubicon/blob/main/contracts/RubiconMarket.sol#L598). The method `isClosed()` always return false and the condition is passed every time. Reading the comment above the modifier [596](https://github.com/code-423n4/2023-04-rubicon/blob/main/contracts/RubiconMarket.sol#L596) I think that the developers wants to stop market when its need and no new offers to be allowed for creating.

### Recommended Mitigation Steps
Instead of calling of `isClosed()` method use `stop()` method which fit more with this condition.

```
/// @dev After close_time has been reached, no new offers are allowed.
modifier can_offer() override {
    require(!stop());
    _;
}
```

Also refactor the stop() method

```
function stop(bool isStopped) external auth {
    stopped = isStopped;
}
```

## [M-03] Trouble with migration of USDT pool

### Impact
During the migration of the USDT pool can be reached some problem with approving of the tokens [53](https://github.com/code-423n4/2023-04-rubicon/blob/main/contracts/V2Migrator.sol#L53). Some tokens (like USDT) have approval race condition protection, which requires the allowance before a call to approve to already be either 0 or UINT_MAX. If this is not the case, the call reverts. I mark this issues as High because Rubicon protocol has already USDT pool witch locked money in him.

### Recommended Mitigation Steps
Set allowance to zero before approve.

# Low severity

## [L-01] notifyRewardAmount() will revert if rewardsDuration[token] is zero
If `rewardsDuration[token]` is equal to zero during the calculation of rewardRatio in `notifyRewardAmount(...)` will revert.
[233](https://github.com/code-423n4/2023-04-rubicon/blob/main/contracts/periphery/BathBuddy.sol#L233)
### Mitigation steps:ß
Add additional check for that

```
require(_rewardsDuration != 0, "Invalid duration time");
```

# Informational

## [I-01] Using SafeMath when compiler is ^0.8.0
There is no need to use SafeMath when compiler is ^0.8.0 because it has built-in under/overflow checks.

## [I-02] Unused storage variable
Storage variable `stopped` is unused and does not affect the contracts. It should be removed or be used for something else.

## [I-03] Code is not properly formatted
Run a formatter on the code, for example use the prettier-solidity plugin.

## [I-04] NatSpec missing from external functions
Add NatSpec docs for all external functions so their intentions and signatures are clear.

## [I-05] Consider using custom errors instead of require statements with string error
Custom errors reduce the contract size and can provide easier integration with a protocol. Consider using those instead of require statements with string error

## [I-06] Open TODO in code
In some places in the code have a open TODOs
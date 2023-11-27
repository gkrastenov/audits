# RootedToken Security Review

A security review of the RootedToken.

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
| Project Name  | RootedToken                                                                                                                                                 

### Scope

| Files    | 
| :-------------- | 
| FeeSplitter.sol|
| SwapPair.sol|
| RootedTransferGate.sol|


### Issues found

| Severity      | Count | Fixed | Acknowledged |
| :------------ | ----: | ----: |  ---------:  |
| High risk     |     2 |     - |            - |
| Medium risk   |     3 |     - |            - |
| Low risk      |     4 |     - |            - |
| **Total**         |   **9** |   **-** |            **-** |

# Findings

|ID  |  Title                                                                                | Severity      |
|:---| :------------------------------------------------------------------------------------| :------------ |
|H-01 | Unspecified slippage allows sandwich attacks                                        | High        |
|H-02 | Some functions do not work properly with meta-transactions                          | High        |
|M-01 | Use of block.timestamp for swap                                                     | Medium        |
|M-02 | Batch transfer/transferFrom will always fail                                        | Medium        |
|M-03 | The calling of the balanceOf function can easily be blocked                         | Medium        |
|L-01 | getDumpTax() function will return 0 for small duration                              | Low           |
|L-02 | Centralization risk                                                                 | Low           |
|L-03 | Splitter should be checked for non-zero address                                     | Low           |
|L-04 |Very small transfers can bypass taxes                                                | Low           |

# High

## [H-01] Unspecified slippage allows sandwich attacks
### Impact
The amount getting swapped will be completely lost.

### Vulnerability Details
The swap in `FeeSplitter.sol` the `payFees` function executes a swap, but the `amountOutMinimum` value, which is accountable for slippage
protection is at 0, which allows the swap to yield 0 tokens in return for the amount provided. This allows for MEVs to pick the transaction up from the mempool and to sandwich it by manipulating the pool, in which the swap is happening.

### Recommendations
Consider setting `amountOutMinimum` to some appropriate value, that includes a conservative amount of tolerance for price impact.

## [H-02] Some functions do not work properly with meta-transactions
### Impact
Some functions do not work properly with meta-transactions which can lead to lose of funds.

### Vulnerability Details
The functions `transfer`, `approve` and `transferFrom` will not work properly with meta-transactions. In meta-transactions, data is signed off-chain by one person and executed by another person who pays the gas fees. In this case, the original sender won't be `msg.sender`. Using of `msg.sender` directly is not the proper way of usage. The replayer who executed the transaction will be affected, even though they were not originally the signer of the transaction.

### Recommendations
`_msgSender()` should be used instead of `msg.sender`.

# Medium

## [M-01] Use of block.timestamp for swap
### Impact
Not allowing users to supply their own deadline could potentially expose them to sandwich attacks.

### Vulnerability Details
The current implementation can lead to significant risks including unfavorable trade outcomes and potential financial loss. Without a userspecified deadline, transactions can remain in the mempool for an extended period, resulting in execution at a potentially disadvantageous time. Moreover, by setting the deadline to `block.timestamp`, a validator can hold the transaction without any time constraints, further exposing users to the risk of price fluctuations.

Advanced protocols like Automated Market Makers (AMMs) can allow users to specify a deadline parameter that enforces a time limit by which
the transaction must be executed. Without a deadline parameter, the transaction may sit in the mempool and be executed at a much later time
potentially resulting in a worse price for the user.

```solidity 
uint256[] memory amounts = router.swapExactTokensForTokens(sellAmount, 0, path, address(this), block.timestamp);
```

### Recommendations
Allow for a deadline to be set by the user, such that after the deadline the transaction never takes place.


## [M-02] Batch transfer/transferFrom will always fail
### Impact
It is not possible to perform batch `transfers/transferFrom` from one user to many other users.

### Vulnerability Details

The problem arises in the require statement within the `allowBalance` function,

```solidity 
require (last.origin != allow.origin || last.blockNumber != allow.blockNumber || last.transferFrom != allow.transferFrom, "Liquidity is locked (Please try again next block)");`
```

The conditions `last.origin != allow.origin`, `last.blockNumber != allow.blockNumber` and `last.transferFrom != allow.transferFrom`
will always evaluate to false in the second transfer or transferFrom call within a batch operation, leading to a transaction revert. Users are
required to wait for one block before executing multiple transfer or they can create a specific contract to bypass this requirement.

## [M-03] The calling of the balanceOf function can easily be blocked
### Impact
Calling of `balanceOf` function easy can be blocked through front-running attack.

### Vulnerability Details
If the `SwapPair` contract calls the `balanceOf` function and `liquidityPairLocked[pair]` is true, then it must satisfy the following require statement:

```solidity 
require (last.origin != allow.origin || last.blockNumber != allow.blockNumber || last.transferFrom != allow.transferFrom);
```

To meet this requirement, the caller must first invoke `transfer` or `transferFrom` to fulfill all conditions. However, if these two calls are not
executed within a single transaction — for instance, if the `SwapPair` calls transfer in one transaction and then calls `balanceOf` in a separate
transaction — there is a risk of being front-runned by a malicious user.

The malicious user can observe the initial transaction of `SwapPair`, which is a `transfer` and subsequently execute another transfer to alter
the value of `last.origin`. Consequently, the subsequent transaction involving the `balanceOf` function for the pair will result in failure.


# Low 
## [L-01] getDumpTax() function will return 0 for small duration
The `getDumpTax()` function will return 0 if a small duration is set up by the owner or the `feeController`. This occurs due to a precision error in the
following calculation:

`dumpTaxStartRate * (dumpTaxEndTimestamp - block.timestamp) * 1e18 / dumpTaxDurationInSeconds / 1e18; // return 2 after 40 days`

## [L-02] Centralization risk
The whole contract carries a significant centralization risk. The owner can easily scam all users by altering fees or blacklisting/whitelisting
addresses.

## [L-03] Splitter should be checked for non-zero address
The splitter address should be checked for a non-zero value, as if the address is zero during the transfer, the fee will be transferred to
`address(0)`. This will lead to funds being stuck.

## [L-04] Very small transfers can bypass taxes
Very small transfers can bypass taxes due to precision loss:

`return amount * feesRate / 10000;`

For instance, if the feesRate is 2, then transfers with an amount of 500 will bypass taxation for the user.

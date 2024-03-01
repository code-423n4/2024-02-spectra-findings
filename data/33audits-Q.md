## Title 

There should be a contract existence check before performing a `delegateCall` the the `RewardsProxy`.

## Link

https://github.com/perspectivefi/spectra-core/blob/fec59dc6720fb4861b07b30845ef2c1a42f947bf/src/tokens/PrincipalToken.sol#L399

## Impact
The PrincipalToken contract uses the delegatecall proxy pattern. If the
implementation contract is incorrectly set or is self-destructed, the contract may not detect and may return. 

A delegateCall to a self-destructed contract will return success. Consider the following notes from the [Solidity Documentation](https://docs.soliditylang.org/en/develop/control-structures.html#error-handling-assert-require-revert-and-exceptions).

```
The low-level call, delegatecall and callcode will return success if the called account is
non-existent, as part of the design of EVM. Existence must be checked prior to calling if desired.
```

## Proof of Concept

Prior to the delegateCall the function should verify that a contract exists the address of the RewardsProxy.

```
    /** @dev See {IPrincipalToken-claimRewards}. */
    function claimRewards(bytes memory _data) external restricted {
        if (rewardsProxy == address(0)) {
            revert NoRewardsProxySet();
        }
        _data = abi.encodeWithSelector(IRewardsProxy(rewardsProxy).claimRewards.selector, _data);
        //@audit- not checking for contract size here
        (bool success, ) = rewardsProxy.delegatecall(_data);
        if (!success) {
            revert ClaimRewardsFailed();
        }
    }
```

## Tools Used

Manual review.

## Recommended Mitigation Steps

Consider adding a contract existence check and reverting if no code exists at the rewardsProxy address.

## Title

Add a deadline check to the initialize function

## Link

https://github.com/perspectivefi/spectra-core/blob/fec59dc6720fb4861b07b30845ef2c1a42f947bf/src/tokens/PrincipalToken.sol#L133

## Description

The expiry date is extremely important to the functionality of the PrincipalToken contract. After the expiry date passed the value of YT will always be zero. If a malicious miner wanted to exploit the system and would benefit by YT being zero they could refuse to mine the deployment/initialize transaction after the expiry date. Since this value is being set by `duration + block.timestamp` it is possible for an attacker to exploit this.

## Recommendations

Consider adding a deadline check so that the expiry date isn't manipulated and if it is the deployment transaction will revert.







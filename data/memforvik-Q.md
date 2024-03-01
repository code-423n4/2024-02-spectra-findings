## The implementation of function maxWithdraw does not comply with EIP Standard

according to `eip-5059`(https://eips.ethereum.org/EIPS/eip-5095), maxWithdraw MUST factor in both global and user-specific limits, like if withdrawals are entirely disabled (even temporarily) it MUST return 0 and MUST NOT revert.

```
    function maxWithdraw(address owner) public view override whenNotPaused returns (uint256) {
        return convertToUnderlying(_maxBurnable(owner));
    }
```

The above function will directly revert when the contract is paused. 
This logic does not comply with EIP-5059.

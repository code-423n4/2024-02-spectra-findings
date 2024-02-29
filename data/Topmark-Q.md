### Report 1:
#### Incomplete Validation
The implementation of initialize(...) function as noted from the pointer shows that validation was done to ensure `_assetDecimals` is not greater than `_ibtDecimals` to ensure consistency in the codebase, however this is not totally right as the correct condition to validate would be to ensure `_assetDecimals` is equal `_ibtDecimals` which guarantees consistency in contract.
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L145
```solidity
  function initialize(
        address _ibt,
        uint256 _duration,
        address _initialAuthority
    ) external initializer {
       ...
        if (
            _assetDecimals < MIN_DECIMALS ||
>>>            _assetDecimals > _ibtDecimals ||
            _ibtDecimals > MAX_DECIMALS
        ) {
            revert InvalidDecimals();
        }
    ...
```
###  Report 2:

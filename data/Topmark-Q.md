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
#### Inconsistent Value conversion
The two functions provided below shows how conversion is done between IBTS and Shares however a look at how _ptRate and _ibtRate is derived shows that one of the function used false parameter with _getPTandIBTRates(...) function call while the other uses true, this will create inconsistency in value conversion and can be taken advantage of by bad actors
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L684/PrincipalToken.sol#L684
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L702
```solidity
   function _convertIBTsToShares(
        uint256 _ibts,
        bool _roundUp
    ) internal view returns (uint256 shares) {
>>>        (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);
        if (_ptRate == 0) {
            revert RateError();
        }
        shares = _ibts.mulDiv(
            _ibtRate,
            _ptRate,
            _roundUp ? Math.Rounding.Ceil : Math.Rounding.Floor
        );
    }
    function _convertIBTsToSharesPreview(uint256 ibts) internal view returns (uint256 shares) {
>>>        (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(true); // to round up the shares, the PT rate must round down
        if (_ptRate == 0) {
            revert RateError();
        }
        shares = ibts.mulDiv(_ibtRate, _ptRate);
    }
```
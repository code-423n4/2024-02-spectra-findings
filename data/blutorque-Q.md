# L-01:  Truncation of `yieldOfUserInIBT` due to division before multiplication in `_computeYield`

There are more than one cases for which user receiving less yield, due to division over muliplication first. Let's consider a particular case, where only positive yield happened; 

If `_oldPTRate == _ptRate && _ibtRate > _oldIBTRate == true`

Yield generated only from positive change in ibtRate here, 
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/libraries/PrincipalTokenUtil.sol#L74-L75

```solidity
        if (_oldPTRate == _ptRate && _ibtRate > _oldIBTRate) {
            // only positive yield happened
            newYieldInIBTRay = ibtOfPTInRay.mulDiv(_ibtRate - _oldIBTRate, _ibtRate);
```

Where `ibtOfPTInRay` is calculated as 
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/libraries/PrincipalTokenUtil.sol#L72
```solidity
        uint256 ibtOfPTInRay = userYTBalanceInRay.mulDiv(_oldPTRate, _oldIBTRate);
```

Here, userYTBalanceInRay is divided by _oldIBTRate, which result `ibtOfPTInRay` later multiplied to `_ibtRate - _oldIBTRate` in newYieldInIBTRay calculation. Due to division before multiplication, some yields from IBTs will be lost in rounding. 

Similarly, when `_oldPTRate > _ptRate` and `_ibtRate < _oldIBTRate` becomes true, the negative yield `expectedNegativeYieldInAssetRay` will generate from the dropped ibtRate. 

This `expectedNegativeYieldInAssetRay` will be less than the expected, due to similar rounding as above. Results in, protocol bearing more losses instead user. 

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/libraries/PrincipalTokenUtil.sol#L104-L107

### Recommendation
Perform multiply before the division. 
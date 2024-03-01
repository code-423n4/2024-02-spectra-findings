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

Here, userYTBalanceInRay is divided by _oldIBTRate, which result ibtOfPTInRay later multiplied to `_ibtRate - _oldIBTRate` in newYieldInIBTRay calculation. Due to rounding issue, this results in less output ibtYield for user. 

https://github.com/crytic/slither/wiki/Detector-Documentation#divide-before-multiply

### Recommendation
Perform multiply before the division. 
## [L-01] anyone can set the ibt/pt rate for anyone through updateYield

The `updateYield` function lacks any access control, allowing anyone to set the ibtRate and ptRate for anyone. 

If this is set before a user interacts with the protocol, the below check will be circumvented and computeYield will called with value set by someone in the past. 

```solidity 
        // Check for skipping yield update when the user deposits for the first time or rates decreased to 0.
        if (_oldIBTRateUser != 0) {
            updatedUserYieldInIBT = PrincipalTokenUtil._computeYield(
                _user,
                yieldOfUserInIBT[_user],
                _oldIBTRateUser,
                _ibtRate,
                _oldPTRateUser,
                _ptRate,
                yt
            );
        }
```

## [L-02] The Mint event gives incorrect information

The `Mint` event in `PrincipalToken.sol` sets the msg.sender as originator ('from')of the minting coins. This is obviously incorrect since it is the PT contract which mints the token, hence there is a Transfer event with address(0) as originator.

The recommandation is to set the 'from' in the Mint event to address(0) to be consistent with standard minting events. 

## [L-03] Broken Invariant: Equal Supply PT==YT

The protocol stated the following invariant in the contest README.md: "PT and its YT should have an equal supply at all times"

In the `_beforeRedeem` function, YT and PT are burned in equal amounts before expiry, but only PT is burned after expiry. 
Therefore, after expiry, the stated invariant is broken. 

This currently has no impact so I submit this as a low. 

## [L-04] Inconsistent application of SAFETY_BOUND

In `_computeYield` in `PrincipalTokenUtil.sol`, a `SAFETY_BOUND` is applied with the stated goal of favoring the protocol in case of approximation. 

However, this safety is only applied in the second part of the if statement. Since it is perfectly possible for the first part to resolve to an amount smaller than the `SAFETY_BOUND`, it should be applied consistenly in all cases. 

``` solidity
        } else {
            if (_oldPTRate > _ptRate) {
                // PT depeg happened
                uint256 yieldInAssetRay;
                if (_ibtRate >= _oldIBTRate) {
                    // both negative and positive yield happened, more positive
                    yieldInAssetRay =
                        _convertToAssetsWithRate(
                            userYTBalanceInRay,
                            _oldPTRate - _ptRate,
                            RayMath.RAY_UNIT,
                            Math.Rounding.Floor
                        ) +
                        _convertToAssetsWithRate(
                            ibtOfPTInRay,
                            _ibtRate - _oldIBTRate,
                            RayMath.RAY_UNIT,
                            Math.Rounding.Floor
                        );
                } else {
                    // either both negative and positive yield happened, more negative
                    // or only negative yield happened
                    uint256 actualNegativeYieldInAssetRay = _convertToAssetsWithRate(
                        userYTBalanceInRay,
                        _oldPTRate - _ptRate,
                        RayMath.RAY_UNIT,
                        Math.Rounding.Floor
                    );
                    uint256 expectedNegativeYieldInAssetRay = Math.ceilDiv(
                        ibtOfPTInRay * (_oldIBTRate - _ibtRate),
                        RayMath.RAY_UNIT
                    );
                    yieldInAssetRay = expectedNegativeYieldInAssetRay >
                        actualNegativeYieldInAssetRay
                        ? 0
                        : actualNegativeYieldInAssetRay - expectedNegativeYieldInAssetRay;
                    yieldInAssetRay = yieldInAssetRay.fromRay(
                        IERC4626(IPrincipalToken(IYieldToken(_yt).getPT()).underlying()).decimals()
                    ) < SAFETY_BOUND 
                        ? 0
                        : yieldInAssetRay;
                }

```

## [NC-01] Incorrect naming of burnWithoutUpdate

The `burnWithoutUpdate` function name in `YieldToken.sol` implies that this will function will not call the `_update` function. Yet it calls the `_burn` function which calls `_update`.  

After checking with the protocol, the without update refers to the `updateYield` function. So the function should be renamed to `burnWithoutYieldUpdate` to avoid confusion. 
 



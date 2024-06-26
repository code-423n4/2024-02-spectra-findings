## [L-01] `PrincipalToken::_depositIBT` doesn't emit an event with `_ytReceiver` address.

The function properly emits the event with the `_ptReceiver` address but it is not emitted for `_ytReceiver` as tokens are also minted for `_ytReceiver` address. We recommend a similar event emission for better off-chain monitoring.

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L750

```js
        emit Mint(msg.sender, _ptReceiver, shares);

        IYieldToken(yt).mint(_ytReceiver, shares);
        // @audit-info no emission for _ytReciever.
```

## [L-02] For `withdraw` functionality, `withdraw` event should be emitted instead of the `redeem` event

The vault is ERC-4626 equivalent. `withdraw` and `redeem` are two different operations performed in ERC-4626.

The function `_withdrawShares` is used to perform `withdraw` not redeem so it should emit `withdraw` event instead of `redeem` to avoid confusion off-chain. 
 
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L780C1-L798C6

```js
    function _withdrawShares(
        uint256 _ibts,
        address _receiver,
        address _owner,
        uint256 _ptRate,
        uint256 _ibtRate
    ) internal returns (uint256 shares) {
        if (_ptRate == 0) {
            revert RateError();
        }
        // convert ibts to shares using provided rates
        shares = _ibts.mulDiv(_ibtRate, _ptRate, Math.Rounding.Ceil);
        // burn owner's shares (YT and PT)
        if (block.timestamp < expiry) {
            IYieldToken(yt).burnWithoutUpdate(_owner, shares);
        }
        _burn(_owner, shares);
@>        emit Redeem(_owner, _receiver, shares);
    }
```

## [L-03] `RayMath::fromRay` should be careful that `_decimals` should be lesser than or equal to 27

The function doesn't take care of a case where the `_decimals` is greater or equal to 27, if this happens the operation will output zero. 

https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol#L35

```js
    function fromRay(uint256 _a, uint256 _decimals) internal pure returns (uint256 b) {
        // @audit-info check decimals is not greater than 27
        uint256 decimals_ratio = 10 ** (27 - _decimals); // @audit greater than 27 will cause problem here.
        assembly {
            b := div(_a, decimals_ratio)
        }
    }
```

## [L-04] `RayMath::fromRay` should enforce a check that the `_a` argument is not zero

The `fromRay` function in the `RayMath` library should check if the input variable `_a` is non-zero. 
 
https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol#L35

```js
    function fromRay(
        uint256 _a,
        uint256 _decimals,
        bool _roundUp
    ) internal pure returns (uint256 b) {
        // @audit-info should check if `_a` is not zero
        uint256 decimals_ratio = 10 ** (27 - _decimals);
        assembly {
            b := div(_a, decimals_ratio)

            if and(eq(_roundUp, 1), gt(mod(_a, decimals_ratio), 0)) {
                b := add(b, 1)
            }
        }
    
```
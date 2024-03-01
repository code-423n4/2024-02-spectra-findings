## Findings Summary

| Label | Description |
| - | - |
| [L-01](#) | Incorrect Return Value for `previewDeposit()`|
| [L-02](#) | Incorrect Return Value for `maxDeposit()`|
| [L-03](#) | Unreasonable Behavior: Transferability of YieldToken After Expiry|

## [L-01] Incorrect Return Value for `previewDeposit()`
The calculation path for `previewDeposit()` and `deposit()` is as follows:

1. previewDeposit() => _convertIBTsToSharesPreview() => _getPTandIBTRates(**true**)
2. deposit()　　　　=> _convertIBTsToShares()　　　　=>_getPTandIBTRates(**false**)

`previewDeposit()` should use the same rounding direction as `deposit()`.

suggest:
```diff
    function _convertIBTsToSharesPreview(uint256 ibts) internal view returns (uint256 shares) {
-       (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(true); // to round up the shares, the PT rate must round down
+       (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false); // to round up the shares, the PT rate must round down
        if (_ptRate == 0) {
            revert RateError();
        }
        shares = ibts.mulDiv(_ibtRate, _ptRate);
    }
```
## [L-02] Incorrect Return Value for `maxDeposit()`
In `maxDeposit()`, the return value is currently `type(uint256).max`. 
However, since deposits are subject to the input/output limits of `ibt`, 
recommend using `IERC4626(ibt).previewDeposit()` instead.

```diff
    function maxDeposit(address) external pure override returns (uint256) {
-       return type(uint256).max;
+       return IERC4626(ibt).previewDeposit(address(this));
    }
```
## [L-03] Unreasonable Behavior: Transferability of YieldToken After Expiry
Currently, even when the `YieldToken.balanceOf()` becomes zero after expiry, but it is still possible to transfer the token. 
This behavior is not reasonable.  suggest that after expiry, transfers should be disallowed.

```diff
    function transfer(
        address to,
        uint256 amount
    ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {
+       require(block.timestamp < IPrincipalToken(pt).maturity(),"Expire");
        IPrincipalToken(pt).beforeYtTransfer(msg.sender, to);
        return super.transfer(to, amount);
    }

    function transferFrom(
        address from,
        address to,
        uint256 amount
    ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {
+       require(block.timestamp < IPrincipalToken(pt).maturity(),"Expire");
        IPrincipalToken(pt).beforeYtTransfer(from, to);
        return super.transferFrom(from, to, amount);
    }
```
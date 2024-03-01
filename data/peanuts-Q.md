### [L-01] withdraw() should burn shares before converting IBT to assets for best practice

The `withdraw()` function calls `ERC4626.withdraw()` first before calling `_withdrawShares()`. This means that at one point in time, the user will have both the asset tokens as well as the PT/YT tokens. This may lead to undesirable circumstances with the user holding both tokens, like potential cross-contract reentrancy.

```
    function withdraw(
        uint256 assets,
        address receiver,
        address owner
    ) public override returns (uint256 shares) {
        _beforeWithdraw(assets, owner);
        (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);
        uint256 ibts = IERC4626(ibt).withdraw(assets, receiver, address(this));
        shares = _withdrawShares(ibts, receiver, owner, _ptRate, _ibtRate);
    }
```

For best practice, like following the CEI pattern, `_withdrawShares()` should be called first before `ERC4626.withdraw()`.

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L278-L287

### [L-02] The functions without the minimum check should not be used

In every function family (deposit,redeem,withdraw), there will be a function that has a minimum check and a function that does not have a minimum check. For example, in the `deposit()` family, there is a function with minShares check and a function without minShares check.

```
    //@audit - no min check
    function deposit(
        uint256 assets,
        address ptReceiver,
        address ytReceiver
    ) public override returns (uint256 shares) {
        IERC20(_asset).safeTransferFrom(msg.sender, address(this), assets);
        IERC20(_asset).safeIncreaseAllowance(ibt, assets);
        uint256 ibts = IERC4626(ibt).deposit(assets, address(this));
        shares = _depositIBT(ibts, ptReceiver, ytReceiver);
    }

    //@audit - min check
    /** @dev See {IPrincipalToken-deposit}. */
    function deposit(
        uint256 assets,
        address ptReceiver,
        address ytReceiver,
        uint256 minShares
    ) external override returns (uint256 shares) {
        shares = deposit(assets, ptReceiver, ytReceiver);
>       if (shares < minShares) {
            revert ERC5143SlippageProtectionFailed();
        }
```

A user can choose the deposit function to deposit their tokens. Best practice will be to just allow users to call only the function with the minShares check to prevent any frontrunning issues.

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L176-L197

### [L-03] `storeRatesAtExpiry()` does not enforce that the rates are store at the exact expiry time

When withdrawing or redeeming PT tokens, if the block.timestamp >= expiry, then `storeRatesAtExpiry()` is called. `storeRatesAtExpiry()` can only be called once, and it sets the `ratesAtExpiryStored` to true. Then, it calls `_getCurrentPTandIBTRates()` to get the `ptRate` and `ibtRate`.

```
    function storeRatesAtExpiry() public override afterExpiry {
        if (ratesAtExpiryStored) {
            revert RatesAtExpiryAlreadyStored();
        }
        ratesAtExpiryStored = true;
        // PT rate not rounded up here
        (ptRate, ibtRate) = _getCurrentPTandIBTRates(false);
        emit RatesStoredAtExpiry(ibtRate, ptRate);
    }
```

The `ptRate` and `ibtRate` is then calculated. Note that if the block.timestamp is greater than the expiry, then the rates will be slightly different, which may affect the actual value of PT tokens. 

Try to ensure that `storeRatesAtExpiry()` is called at the exact time of expiry.


https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L409-L417
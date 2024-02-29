## `previewDeposit()` and `deposit()` both operate on slightly different `ptRates`  

## Vulnerability detail

When triggering the `previewDeposit()` function if it is before the expiry, `_getCurrentPTandIBTRates()` will fetch the ibt and pt rates, rounding up the ptRate, the amount of shares returned is inverse to the ptRate, a higher ptRate will result in less shares. 

However when going through the `deposit()` route the the ptRate will be rounded down, which means slightly more shares in favor of the user. 

## Impact

Slightly less shares returned with the deposit method, while when deposit() is called more shares in favor of the user. There is a slight inconsistency and it is recommended to round in favor of the protocol instead. 

## POC

When calling the [deposit() method](https://github.com/code-423n4/2024-02-spectra/blob/25603ac27c3488423a0739b66e784c01a3db7d75/src/tokens/PrincipalToken.sol#L763)

```solidity
shares = _convertIBTsToShares(_ibts - tokenizationFee, false);
```
We round down which ends up minting more shares than with the [previewDeposit](https://github.com/code-423n4/2024-02-spectra/blob/25603ac27c3488423a0739b66e784c01a3db7d75/src/tokens/PrincipalToken.sol#L703)

```solidity
     (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(true); 
```

because the larger the ptRate the less shares are given ` shares = ibts.mulDiv(_ibtRate, _ptRate);`. 



## Links
https://github.com/code-423n4/2024-02-spectra/blob/25603ac27c3488423a0739b66e784c01a3db7d75/src/tokens/PrincipalToken.sol#L702-L708


https://github.com/code-423n4/2024-02-spectra/blob/25603ac27c3488423a0739b66e784c01a3db7d75/src/tokens/PrincipalToken.sol#L751-L770


## Mitigation route

Although I understand that this step was implemented such as `previewDeposit <= deposit `. When both function called in the same transaction in would be best if the returned the same amount of shares. So consider rounding in favor of the protocol in the deposit function as well.

```diff
    function _depositIBT(
        uint256 _ibts,
        address _ptReceiver,
        address _ytReceiver
    ) internal notExpired nonReentrant whenNotPaused returns (uint256 shares) {
        updateYield(_ytReceiver);
        uint256 tokenizationFee = PrincipalTokenUtil._computeTokenizationFee(
            _ibts,
            address(this),
            registry
        );
        _updateFees(tokenizationFee);
    --  shares = _convertIBTsToShares(_ibts - tokenizationFee, false);
    ++  shares = _convertIBTsToShares(_ibts - tokenizationFee, true);
        if (shares == 0) {
            revert RateError();
        }
        _mint(_ptReceiver, shares);
        emit Mint(msg.sender, _ptReceiver, shares);
        IYieldToken(yt).mint(_ytReceiver, shares);
    }


```
## [L-1] User can execute flashloan with zero flashloan fee
The `flashloan(...)` function checks the flashloan `fee` from the registry contract. 
However, the flashloan `fee` is not set at initialization of the Registry contract but a `setPTFlashLoanFee` is available to set the flashloan fee.

Since the flashloan `fee` is not set at initialization of the registry, users can execute a flashloan without paying any fee before the `setPTFlashLoanFee` is called to set flashlon `fee`.

- https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L617C8-L629C1
```
File: PrincipalToken.sol
function flashLoan(
        IERC3156FlashBorrower _receiver,
        address _token,
        uint256 _amount,
        bytes calldata _data
    ) external override returns (bool) {
        if (_amount > maxFlashLoan(_token)) revert FlashLoanExceedsMaxAmount();

        uint256 fee = flashFee(_token, _amount);
        _updateFees(fee);//@audit check if fee is zero and revert.

        // Initiate the flash loan by lending the requested IBT amount
        IERC20(ibt).safeTransfer(address(_receiver), _amount);

        // Execute the flash loan
        if (_receiver.onFlashLoan(msg.sender, _token, _amount, fee, _data) != ON_FLASH_LOAN)
            revert FlashLoanCallbackFailed();

        // Repay the debt + fee
        IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);

        return true;
    }

```



## Impact
Users can execute flashloan without fee before the `fee` is set.


## Recommendation
Consider implementing a check on the `flashloan()` function to ensure the calculated fee is not zero.

```diff
File: PrincipalToken.sol
function flashLoan(
        IERC3156FlashBorrower _receiver,
        address _token,
        uint256 _amount,
        bytes calldata _data
    ) external override returns (bool) {
        if (_amount > maxFlashLoan(_token)) revert FlashLoanExceedsMaxAmount();

        uint256 fee = flashFee(_token, _amount);
++      require(fee != 0, "PT: Flashloan fee not set")
        _updateFees(fee);//@audit check if fee is zero and revert.

        // Initiate the flash loan by lending the requested IBT amount
        IERC20(ibt).safeTransfer(address(_receiver), _amount);

        // Execute the flash loan
        if (_receiver.onFlashLoan(msg.sender, _token, _amount, fee, _data) != ON_FLASH_LOAN)
            revert FlashLoanCallbackFailed();

        // Repay the debt + fee
        IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);

        return true;
    }

```
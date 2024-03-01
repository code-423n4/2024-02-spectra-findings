# QA Report

## Low Risk 
| Count | Title | Instances |
|:--:|:-------|:--:|
| [L-01] | Potential reentrancy issue in `withdraw()` | 1 |
| [L-02] | Ambiguity in paused state features for PT | 1 |

Total Low Risk Issues: 2

## Non-Critical
| Count | Title | Instances |
|:--:|:-------|:--:|
| [NC-01] | Incorrect code comment in `_convertIBTsToSharesPreview()` | 1 |

Total Non-Critical Issues: 1

## [L-01] Potential reentrancy issue in `withdraw()`

### Bug Description


The `withdraw()` function in the PT contract could lead to a reentrancy vulnerability. This occurs when `IERC4626(ibt).withdraw(assets, receiver, address(this))` is invoked, transferring assets to the receiver. If these assets are of the ERC777 type, they might activate the receiver's `tokenReceived` function. At this point, the PT/YT tokens haven't been burned through the `_withdrawShares()` function, potentially allowing users to interact with external contracts while the PT contract is in an inconsistent state.

While the possibility of reentrancy exists, it doesn't necessarily provide a feasible method to exploit the contract. Nonetheless, the very capability for malicious actors to execute code during a period when the PT contract is not in its intended state deems this a issue.

The readme specifies that "IBT must not be ERC777", yet it does not state the assets within IBT must not be ERC777. In fact, the OpenZeppelin ERC4626 implementation explicitly addresses reentrancy protection within its comments, it's advisable for the PT contract to do the same.

```solidity
    function withdraw(
        uint256 assets,
        address receiver,
        address owner
    ) public override returns (uint256 shares) {
        _beforeWithdraw(assets, owner);
        (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);
>       // @audit: This may call `tokensReceived` of the receiver contract if the asset of ERC4626 is a ERC777, which PT/YT shares are not yet updated, causing a potential reentrancy issue.
>       uint256 ibts = IERC4626(ibt).withdraw(assets, receiver, address(this));
>       shares = _withdrawShares(ibts, receiver, owner, _ptRate, _ibtRate);
    }
```

[Openzeppelin ERC4626](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC4626.sol#L256-L277):
```solidity
    function _withdraw(
        address caller,
        address receiver,
        address owner,
        uint256 assets,
        uint256 shares
    ) internal virtual {
        if (caller != owner) {
            _spendAllowance(owner, caller, shares);
        }

        // If _asset is ERC-777, `transfer` can trigger a reentrancy AFTER the transfer happens through the
        // `tokensReceived` hook. On the other hand, the `tokensToSend` hook, that is triggered before the transfer,
        // calls the vault, which is assumed not malicious.
        //
        // Conclusion: we need to do the transfer after the burn so that any reentrancy would happen after the
        // shares are burned and after the assets are transferred, which is a valid state.
        _burn(owner, shares);
        SafeERC20.safeTransfer(_asset, receiver, assets);

        emit Withdraw(caller, receiver, owner, assets, shares);
    }
```

### Code Snippet

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L278-L287

### Recommendation

Use `IERC4626(ibt).previewWithdraw()` to calculate the amount of ibt to withdraw, and move `_withdrawShares()` in front of `IERC4626(ibt).withdraw()`.

## [L-02] Ambiguity in paused state features for PT

### Bug description

The pause feature within the PT contract is unclear and inconsistent on which functionalities are to be paused. For instance, fundamental operations like `deposit/redeem/withdraw`, and their preview functions, are paused. However, claiming yields and transferring YT (which updates yields) remain active.

Additionally, there is inconsistency in the pausing of "max" related view functions: `maxWithdraw` is paused, whereas `maxRedeem` and `maxDeposit` are not. This selective suspension of features adds to the confusion regarding the pause functionality's scope.

Most importantly, the flash loan feature is not paused. If the PT contract had some severe errors that requires pausing, flash loans should be paused as well.

```solidity
    function flashLoan(
        IERC3156FlashBorrower _receiver,
        address _token,
        uint256 _amount,
        bytes calldata _data
    ) external override returns (bool) {
        if (_amount > maxFlashLoan(_token)) revert FlashLoanExceedsMaxAmount();

        uint256 fee = flashFee(_token, _amount);
        _updateFees(fee);

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

### Code Snippet

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L609-L631

### Recommendation

Make the pauseable states more clear, and also pause flash loan feature.

## [NC-01] Incorrect code comment in `_convertIBTsToSharesPreview()`

### Bug description

The code comment here is incorrect, it should be `// to round down the shares, the PT rate must round up`.

```solidity
    /**
     * @dev Converts amount of IBT to amount of PT shares with current rates.
     * This method also rounds the result of the new PTRate computation in case of negative rate
     * @param ibts amount of IBT to convert to shares
     * @return shares resulting amount of shares
     */
    function _convertIBTsToSharesPreview(uint256 ibts) internal view returns (uint256 shares) {
        (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(true); // to round up the shares, the PT rate must round down
        if (_ptRate == 0) {
            revert RateError();
        }
        shares = ibts.mulDiv(_ibtRate, _ptRate);
    }
```

### Code Snippet

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L701-L707

### Recommendation

Fix the comment.
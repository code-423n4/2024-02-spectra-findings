# ⛽ Gas Report: Spectra

## 📚 Table of Contents

1. [Approach](#Approach)
2. [GAS-01 Insignificant yield updates lead to gas inefficiencies in `PrincipalToken::updateYield`](#GAS-01)
3. [GAS-02 Overridden transfer and transferFrom functions introduce extra gas cost in `YieldToken::transfer` and `YieldToken::transferFrom`](#GAS-02)
4. [GAS-03 Repeated State Variable Accesses is gas inefficient in `PrincipalToken::_updatePTandIBTRates` and `PrincipalToken::_getCurrentPTandIBTRates`](#GAS-03)
5. [GAS-04 Avoid Unnecessary Storage Updates in `PrincipalToken::_updatePTandIBTRates`](#GAS-04)
6. [GAS-05 Batch Operations if possible in `PrincipalToken` to save gas](#GAS-05)
8. [GAS-06 Inline functions to save gas](#GAS-06)
9. [GAS-07 Use shorter variable types like uint8 over uint256](#GAS-07)
10. [GAS-08 Use shorter custom error messages](#GAS-08)
11. [Conclusion](#Conclusion)

## 🛠️ Approach

I utilized a systematic checklist approach to uncover a variety of gas optimization opportunities across the contracts. This revealed significant areas for improvement, including the optimization of yield updates, more efficient storage access patterns, and the strategic use of conditional storage updates. In the report I also highlight the tangible benefits of the targeted code adjustments for enhancing overall contract efficiency and reducing gas costs.


## [GAS-01] Insignificant yield updates lead to gas inefficiencies in `PrincipalToken::updateYield`

The function calculates and updates a user's yield without ensuring the new yield value is reasonable or exceeds a minimum threshold. This might lead to situations where insignificant yield changes consume gas unnecessarily.

```solidity
File: PrincipalToken.sol
340:     function updateYield(address _user) public override returns (uint256 updatedUserYieldInIBT) {
341:         (uint256 _ptRate, uint256 _ibtRate) = _updatePTandIBTRates();
342: 
343:         uint256 _oldIBTRateUser = ibtRateOfUser[_user];
344:         if (_oldIBTRateUser != _ibtRate) {
345:             ibtRateOfUser[_user] = _ibtRate;
346:         }
347:         uint256 _oldPTRateUser = ptRateOfUser[_user];
348:         if (_oldPTRateUser != _ptRate) {
349:             ptRateOfUser[_user] = _ptRate;
350:         }
351: 
352:         // Check for skipping yield update when the user deposits for the first time or rates decreased to 0.
353:         if (_oldIBTRateUser != 0) {
354:             updatedUserYieldInIBT = PrincipalTokenUtil._computeYield(
355:                 _user,
356:                 yieldOfUserInIBT[_user],
357:                 _oldIBTRateUser,
358:                 _ibtRate,
359:                 _oldPTRateUser,
360:                 _ptRate,
361:                 yt
362:             );
363:             yieldOfUserInIBT[_user] = updatedUserYieldInIBT;
364:             emit YieldUpdated(_user, updatedUserYieldInIBT);
365:         }
366:     }

```
[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L340)

### Optimization

Validate the updated yield to ensure meaningful updates. Implement a threshold for yield updates to avoid insignificant changes.

```solidity
uint256 yieldChange = updatedUserYieldInIBT > yieldOfUserInIBT[_user] 
                      ? updatedUserYieldInIBT - yieldOfUserInIBT[_user] 
                      : 0;
if (yieldChange < MIN_YIELD_CHANGE_THRESHOLD) {
    revert YieldChangeTooSmall();
}
```

## [GAS-02] Overridden transfer and transferFrom functions introduce extra gas cost in `YieldToken::transfer` and `YieldToken::transferFrom`

The `transfer` and `transferFrom` functions in `YieldToken` contract override the ERC-20 standard functions to include a call to `beforeYtTransfer`, which updates the yield for both the sender and receiver. While this ensures yield is correctly updated, it also introduces additional gas costs for transfers

```solidity
File: YieldToken.sol
71:     function transfer(
72:         address to,
73:         uint256 amount
74:     ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {
75:         IPrincipalToken(pt).beforeYtTransfer(msg.sender, to);
76:         return super.transfer(to, amount);
77:     }

```
[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L71C5-L77C6)

```solidity
File: YieldToken.sol
095:     function transferFrom(
096:         address from,
097:         address to,
098:         uint256 amount
099:     ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {
100:         IPrincipalToken(pt).beforeYtTransfer(from, to);
101:         return super.transferFrom(from, to, amount);
102:     }
103: 
```

[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L95C5-L102C6)

### Optimization

Ensure the necessity of yield updates on every transfer to balance functionality with cost and potential side effects.

## [GAS-03] Repeated State Variable Accesses is gas inefficient in `PrincipalToken::_updatePTandIBTRates` and `_getCurrentPTandIBTRates`

State variables are accessed multiple times in functions like _updatePTandIBTRates and _getCurrentPTandIBTRates. This could be optimized by caching their values in memory variables at the beginning of the function.

```solidity
File: PrincipalToken.sol
879:     function _updatePTandIBTRates() internal returns (uint256 _ptRate, uint256 _ibtRate) {
880:         if (block.timestamp >= expiry) {
881:             if (!ratesAtExpiryStored) {
882:                 storeRatesAtExpiry();
883:             }
884:         }
885:         (_ptRate, _ibtRate) = _getPTandIBTRates(false);
886:         if (block.timestamp < expiry) {
887:             if (_ibtRate != ibtRate) {
888:                 ibtRate = _ibtRate;
889:             }
890:             if (_ptRate != ptRate) {
891:                 ptRate = _ptRate;
892:             }
893:         }
894:     }
```

[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L879)

```solidity
File: PrincipalToken.sol
901:     function _getCurrentPTandIBTRates(bool roundUpPTRate) internal view returns (uint256, uint256) {
902:         uint256 currentIBTRate = IERC4626(ibt).previewRedeem(ibtUnit).toRay(_assetDecimals);
903:         if (IERC4626(ibt).totalAssets() == 0 && IERC4626(ibt).totalSupply() != 0) {
904:             currentIBTRate = 0;
905:         }
906:         uint256 currentPTRate = currentIBTRate < ibtRate
907:             ? ptRate.mulDiv(
908:                 currentIBTRate,
909:                 ibtRate,
910:                 roundUpPTRate ? Math.Rounding.Ceil : Math.Rounding.Floor
911:             )
912:             : ptRate;
913:         return (currentPTRate, currentIBTRate);
914:     }

```
[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L901C5-L914C6)

### Optimization

Cache state variables in memory at the beginning of functions to avoid multiple state reads.

```solidity
uint256 currentIBTRate = ibtRate;
uint256 currentPTRate = ptRate;
```

## [GAS-04] Avoid Unnecessary Storage Updates in `PrincipalToken::_updatePTandIBTRates`

The contract updates storage variables like ibtRate and ptRate in _updatePTandIBTRates function even if their new values are the same as the old ones.

```solidity
File: PrincipalToken.sol
879:     function _updatePTandIBTRates() internal returns (uint256 _ptRate, uint256 _ibtRate) {
880:         if (block.timestamp >= expiry) {
881:             if (!ratesAtExpiryStored) {
882:                 storeRatesAtExpiry();
883:             }
884:         }
885:         (_ptRate, _ibtRate) = _getPTandIBTRates(false);
886:         if (block.timestamp < expiry) {
887:             if (_ibtRate != ibtRate) {
888:                 ibtRate = _ibtRate;
889:             }
890:             if (_ptRate != ptRate) {
891:                 ptRate = _ptRate;
892:             }
893:         }
894:     }

```

[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L879)

### Optimization

Before updating a storage variable, check if the new value is different from the current one to avoid unnecessary storage writes.

## [GAS-05] Batch Operations if possible in `PrincipalToken` to save gas

Where possible, consider batch-processing updates to reduce the frequency of state changes. For example, if multiple users' yields can be updated together in a single transaction without individual user triggers, it could save gas across multiple transactions.

```solidity
File: PrincipalToken.sol
340:     function updateYield(address _user) public override returns (uint256 updatedUserYieldInIBT) {
341:         (uint256 _ptRate, uint256 _ibtRate) = _updatePTandIBTRates();
342: 
343:         uint256 _oldIBTRateUser = ibtRateOfUser[_user];
344:         if (_oldIBTRateUser != _ibtRate) {
345:             ibtRateOfUser[_user] = _ibtRate;
346:         }
347:         uint256 _oldPTRateUser = ptRateOfUser[_user];
348:         if (_oldPTRateUser != _ptRate) {
349:             ptRateOfUser[_user] = _ptRate;
350:         }
351: 
352:         // Check for skipping yield update when the user deposits for the first time or rates decreased to 0.
353:         if (_oldIBTRateUser != 0) {
354:             updatedUserYieldInIBT = PrincipalTokenUtil._computeYield(
355:                 _user,
356:                 yieldOfUserInIBT[_user],
357:                 _oldIBTRateUser,
358:                 _ibtRate,
359:                 _oldPTRateUser,
360:                 _ptRate,
361:                 yt
362:             );
363:             yieldOfUserInIBT[_user] = updatedUserYieldInIBT;
364:             emit YieldUpdated(_user, updatedUserYieldInIBT);
365:         }
366:     }

```

[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L340C5-L366C6)

## [GAS-06] Inline functions to save gas 

Inlining functions can help reduce gas costs by eliminating the overhead associated with function calls.

For instance, _updateFees() is called in the depositIBT() function:

```solidity
File: PrincipalToken.sol
750:     function _depositIBT(
751:         uint256 _ibts,
752:         address _ptReceiver,
753:         address _ytReceiver
754:     ) internal notExpired nonReentrant whenNotPaused returns (uint256 shares) {
755:         updateYield(_ytReceiver);
756:         uint256 tokenizationFee = PrincipalTokenUtil._computeTokenizationFee(
757:             _ibts,
758:             address(this),
759:             registry
760:         );
761:         _updateFees(tokenizationFee);
762:         shares = _convertIBTsToShares(_ibts - tokenizationFee, false);
763:         if (shares == 0) {
764:             revert RateError();
765:         }
766:         _mint(_ptReceiver, shares);
767:         emit Mint(msg.sender, _ptReceiver, shares);
768:         IYieldToken(yt).mint(_ytReceiver, shares);
769:     }

```

[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L750C5-L769C6)

In _claimYield() function:

```solidity
File: PrincipalToken.sol
848:     function _claimYield() internal returns (uint256 yieldInIBT) {
849:         yieldInIBT = updateYield(msg.sender);
850:         if (yieldInIBT == 0) {
851:             return 0;
852:         } else {
853:             yieldOfUserInIBT[msg.sender] = 0;
854:             uint256 yieldFeeInIBT = PrincipalTokenUtil._computeYieldFee(yieldInIBT, registry);
855:             _updateFees(yieldFeeInIBT);
856:             yieldInIBT -= yieldFeeInIBT;
857:             emit YieldClaimed(msg.sender, msg.sender, yieldInIBT);
858:         }
859:     }

```
[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L848C5-L860C1)

In flashLoan() function:

```solidity
File: PrincipalToken.sol
609:     function flashLoan(
610:         IERC3156FlashBorrower _receiver,
611:         address _token,
612:         uint256 _amount,
613:         bytes calldata _data
614:     ) external override returns (bool) {
615:         if (_amount > maxFlashLoan(_token)) revert FlashLoanExceedsMaxAmount();
616: 
617:         uint256 fee = flashFee(_token, _amount);
618:         _updateFees(fee);
619: 
620:         // Initiate the flash loan by lending the requested IBT amount
621:         IERC20(ibt).safeTransfer(address(_receiver), _amount);
622: 
623:         // Execute the flash loan
624:         if (_receiver.onFlashLoan(msg.sender, _token, _amount, fee, _data) != ON_FLASH_LOAN)
625:             revert FlashLoanCallbackFailed();
626: 
627:         // Repay the debt + fee
628:         IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);
629: 
630:         return true;
631:     }

```

[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L609C5-L631C6)

### Optimization

Inlining _updateFees() could offer minor gas optimizations but at the cost of reduced code clarity and increased maintenance complexity. Note that this increases the risk of inconsistencies or errors in future modifications.

## [GAS-07] Use shorter variable types like uint8 over uint256

The decimals of a token (_ibtDecimals, _assetDecimals) are typically within the range of 0 to 18, making them candidates for a smaller integer type like uint8.

```solidity
File: PrincipalToken.sol
52:     uint256 private _ibtDecimals;
53:     uint256 private _assetDecimals;
```
[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L52C5-L53C36)

### Optimization

```solidity
uint8 private _ibtDecimals;
uint8 private _assetDecimals;
```

## [GAS-08] Use shorter custom error messages

Solidity allows for custom errors, which are more gas-efficient than require statements with string messages. However, the names and parameters of these errors also contribute to the bytecode size and, indirectly, to deployment and execution costs.

### Instance 1

```solidity
File: AMBeacon.sol
26:     error BeaconInvalidImplementation(address implementation);
```
[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L26)

### Optimization

```solidity
error InvalidImpl(address impl);
```

### Instance 2
```solidity
File: YieldToken.sol
44:             revert CallerIsNotPtContract();
```

[Lines of Code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L44)

### Optimization

```solidity
error NotPtCaller();
```

## 📌 Conclusion

In my analysis of gas inefficiencies within Spectra in-scope contracts, I pinpointed specific areas for optimization. The key findings included unnecessary yield updates in `PrincipalToken::updateYield`, which could be streamlined by setting a minimum threshold for yield changes, and the extra gas costs introduced by overridden `transfer` and `transferFrom` methods in `YieldToken`. Additionally, I identified multiple state variable accesses and unnecessary storage updates in `PrincipalToken::_updatePTandIBTRates` as areas where caching values and conditional updates could yield gas savings. These targeted adjustments, alongside recommendations for batch operations and shorter variable types, present valuable opportunities to enhance gas efficiency across the contracts.
## Low Severity Findings

### [L-01] Remove `nonReentrant` modifier from internal functions and  place on the function where this internal functions called

Remove `nonReentrant` modifier from `_beforeRedeem` and `_beforeWithdraw` internal functions and  place on the external/public functions where these internal functions called otherwise it will not do its job since when these functions ends `nonReentrant` will be reset while actual  transfer and interaction is happening after these functions call in their external/public functions.


```solidity
File : src/tokens/PrincipalToken.sol

805:      function _beforeRedeem(uint256 _shares, address _owner) internal nonReentrant whenNotPaused {
 
828:      function _beforeWithdraw(uint256 _assets, address _owner) internal whenNotPaused nonReentrant {    

```
[_beforeRedeem](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L805C5-L805C98), [_beforeWithdraw](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L828C36-L828C100)

`beforeWithdraw` is called in `withdraw` function and non view actual external call transfer occurs after _beforeWithdraw but nonReentrant not present here since it ended when _beforeWithdraw finishes so it is good to add `nonReentrant` here on `withdraw` and remove from _beforeWithdraw.

```solidity
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
Similarly add nonReentrant on `withdrawIBT` function. It also calls _beforeWithdraw.

And also remove nonReentrant from `_beforeRedeem` internal function and add it on `redeemForIBT` and `redeem` function where `_beforeRedeem` called.

### [L-02] roundUp or roundDown ie. Ceil/Floor not specified in `_convertIBTsToSharesPreview` which may lead to different results number in different functions.

```solidity
File : src/tokens/PrincipalToken.sol

701:     function _convertIBTsToSharesPreview(uint256 ibts) internal view returns (uint256 shares) {
702:         (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(true); // to round up the shares, the PT rate must round down
703:         if (_ptRate == 0) {
704:             revert RateError();
705:         }
706:         shares = ibts.mulDiv(_ibtRate, _ptRate);
707:      }

```
(https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L701C1-L707C6)

**Recommendation**
Use one more variable to tell Ceil/Floor as used in others take roundUp var. from caller of this function
```diff
File : src/tokens/PrincipalToken.sol

- 701:     function _convertIBTsToSharesPreview(uint256 ibts) internal view returns (uint256 shares) {
+ 701:     function _convertIBTsToSharesPreview(uint256 ibts, bool _roundUp) internal view returns (uint256 shares) {
702:         (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(true); // to round up the shares, the PT rate must round down
703:         if (_ptRate == 0) {
704:             revert RateError();
705:         }
- 706:         shares = ibts.mulDiv(_ibtRate, _ptRate);
+ 706:         shares = ibts.mulDiv(_ibtRate, _ptRate, _roundUp ? Math.Rounding.Ceil : Math.Rounding.Floor);
707:      }

```

### [L-03] Missing access control in PrincipalToken::updateYield function. anyone can update user's mapping `ibtRateOfUser` and `ptRateOfUser` at any time.
It can result in updating the mapping multiple times according to benfit of malicious attacker. And will result in unwanted loss for user whose mapping is updated.

[src/tokens/PrincipalToken.sol#L340-L350](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L340C5-L350C10)

```solidity
function updateYield(address _user) public override returns (uint256 updatedUserYieldInIBT) {
        (uint256 _ptRate, uint256 _ibtRate) = _updatePTandIBTRates();

        uint256 _oldIBTRateUser = ibtRateOfUser[_user];
        if (_oldIBTRateUser != _ibtRate) {
            ibtRateOfUser[_user] = _ibtRate;
        }
        uint256 _oldPTRateUser = ptRateOfUser[_user];
        if (_oldPTRateUser != _ptRate) {
            ptRateOfUser[_user] = _ptRate;
        }
```

**Recommendation**
When updating mappings `ibtRateOfUser` and `ptRateOfUser`  don't allow anyone to update for anyone instead update only for msg.sender instead of passing user so only msg.sender can update is own mapping not other's.

```diff
File: src/tokens/PrincipalToken.sol

- function updateYield(address _user) public override returns (uint256 updatedUserYieldInIBT) {
+ function updateYield() public override returns (uint256 updatedUserYieldInIBT) {    
        (uint256 _ptRate, uint256 _ibtRate) = _updatePTandIBTRates();

-        uint256 _oldIBTRateUser = ibtRateOfUser[_user];
+        uint256 _oldIBTRateUser = ibtRateOfUser[msg.sender];
        if (_oldIBTRateUser != _ibtRate) {
-            ibtRateOfUser[_user] = _ibtRate;
+            ibtRateOfUser[msg.sender] = _ibtRate;
        }
-        uint256 _oldPTRateUser = ptRateOfUser[_user];
+        uint256 _oldPTRateUser = ptRateOfUser[msg.sender];
        if (_oldPTRateUser != _ptRate) {
-            ptRateOfUser[_user] = _ptRate;
+            ptRateOfUser[msg.sender] = _ptRate;
        }

```
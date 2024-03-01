SPECTEA GAS OPTIMIZATIONS
--
INTRODUCTION
--
Highlighted below are optimizations exclusively targeting state-mutating functions and view/pure functions invoked by state-mutating functions. In the discussion that follows, only runtime gas is emphasized, given its inevitable dominance over deployment gas costs throughout the protocol's lifetime.

Please be aware that some code snippets may be shortened to conserve space, and certain code snippets may include `@audit` tags in comments to facilitate issue explanations."
- [SPECTEA GAS OPTIMIZATIONS](#spectea-gas-optimizations)
- [INTRODUCTION](#introduction)
  - [State variables can be packed into fewer storage slot (saves ~8000 Gas)](#g-01-state-variables-can-be-packed-into-fewer-storage-slot-saves-8000-gas)
  - [Refactor `_beforeWithdraw()` to get better gas optimization](#g-02-refactor-_beforewithdraw-to-get-better-gas-optimization)
  - [Minimize the number of external calls by storing the result of IERC4626(ibt) in a local variable before using it multiple times](#g-03-minimize-the-number-of-external-calls-by-storing-the-result-of-ierc4626ibt-in-a-local-variable-before-using-it-multiple-times)
  - [Refactor functions such that all checks are performed before assigning values to state variables](#g-04-refactor-functions-such-that-all-checks-are-performed-before-assigning-values-to-state-variables)
  - [Refactor `_updatePTandIBTRates()` to get better gas optimization](#g-05-refactor-_updateptandibtrates-to-get-better-gas-optimization)
  - [No need to cache state reads if using once](#g-06-no-need-to-cache-state-reads-if-using-once)


### Gas report

### Table Of Contents

**Note: The issues addressed here were not reported by the bot, for packing variables, notes explaining the how and why are included**

## [G-01] State variables can be packed into fewer storage slot (saves ~8000 Gas)
**Explanation**
### Proof of Code
**'expiry' is specified during initialization and cannot exceed uint128; 'duration', like comment state duration, is set in seconds.So, if we utilize uint128, we could meet a large number of needs. We can avoid slod by tightly packing both variables.(Saves 1 SLOT: 2.1K Gas)
**

```solidity
59:    uint256 private expiry; // date of maturity (set at initialization) //@audit use uint128 or less than uint256
60:   uint256 private duration; // duration to maturity

```
**For _ibtDecimals set in initialization `_ibtDecimals = IERC4626(_ibt).decimals();` so we can store _ibtDecimals in uint8.(Saves 2 SLOT: 4.2K Gas)
**
```solidity
File:
52: uint256 private _ibtDecimals; //@audit use uint8
53: uint256 private _assetDecimals;
```
**MIN_DECIMALS and MAX_DECIMALS are declared as uint128 they won't need more than 128 bits, and they are constants.(Saves 1 SLOT: 2.1K Gas)
**
```solidity
40:  uint256 private constant MIN_DECIMALS = 6; 
41:  uint256 private constant MAX_DECIMALS = 18;
```
### Optimized code:

```diff
diff --git a/src/tokens/PrincipalToken.sol b/src/tokens/PrincipalToken.sol
index 9f4b912..7514415 100644
--- a/src/tokens/PrincipalToken.sol
+++ b/src/tokens/PrincipalToken.sol
@@ -37,27 +37,28 @@ contract PrincipalToken is
     using RayMath for uint256;
     using Math for uint256;
 
-    uint256 private constant MIN_DECIMALS = 6;
-    uint256 private constant MAX_DECIMALS = 18;
+    uint128 private constant MIN_DECIMALS = 6;
+    uint128 private constant MAX_DECIMALS = 18;
     bytes32 private constant ON_FLASH_LOAN = keccak256("ERC3156FlashBorrower.onFlashLoan");
 
     address private immutable registry;
 
     address private rewardsProxy;
     bool private ratesAtExpiryStored;
+    uint8 private _ibtDecimals; 
+    uint8 private _assetDecimals;
     address private ibt; // address of the Interest Bearing Token 4626 held by this PT vault
     address private _asset; // the asset of this PT vault (which is also the asset of the IBT 4626)
     address private yt; // YT corresponding to this PT, deployed at initialization
     uint256 private ibtUnit; // equal to one unit of the IBT held by this PT vault (10^decimals)
-    uint256 private _ibtDecimals;
-    uint256 private _assetDecimals;
 
     uint256 private ptRate; // or PT price in asset (in Ray)
     uint256 private ibtRate; // or IBT price in asset (in Ray)
     uint256 private unclaimedFeesInIBT; // unclaimed fees
     uint256 private totalFeesInIBT; // total fees
-    uint256 private expiry; // date of maturity (set at initialization)
-    uint256 private duration; // duration to maturity
+    uint128 private expiry; // date of maturity (set at initialization)
+    uint128 private duration; // duration to maturity
```

## [G-02] Refactor `_beforeWithdraw()` to get better gas optimization

### Details
In the function `_beforeWithdraw()`, verify the block.timestamp < expiryThis function is called in `withdraw` and `WithdrawIBT` [L302](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L302C1-L313C6) .Then the timestamp is less than the expiry time. Both functions call the function `beforeWithdraw()` [L823](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L828C3-L842C6
), which checks the identical condition, indicating a duplicate check. We can avoid by modifying the `_beforeWithdraw` function.


### Proof of Code
- [PrincipalToken.sol#L780](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L780C4-L799C1)
```solidity
File: src/tokens/PrincipalToken.sol
780:    function _withdrawShares(
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
@>        if (block.timestamp < expiry) {
            IYieldToken(yt).burnWithoutUpdate(_owner, shares);
        }
        _burn(_owner, shares);
        emit Redeem(_owner, _receiver, shares);
    }
```
### Optimized code:

```diff
diff --git a/src/tokens/PrincipalToken.sol b/src/tokens/PrincipalToken.sol
index 9f4b912..d66c16e 100644
--- a/src/tokens/PrincipalToken.sol
+++ b/src/tokens/PrincipalToken.sol
@@ -835,6 +837,7 @@ contract PrincipalToken is
             }
         } else {
             updateYield(_owner);
+             IYieldToken(yt).burnWithoutUpdate(_owner, shares);
         }
         if (maxWithdraw(_owner) < _assets) {
             revert UnsufficientBalance();
```
## [G-03] Minimize the number of external calls by storing the result of IERC4626(ibt) in a local variable before using it multiple times.

### Details
 External calls can be quite expensive, inlining here would be the solution so that we only make the external call once and use again.

### Proof of Code
- [PrincipalToken.sol#L901-L914](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L901C5-L914C6)
```solidity
File: src/tokens/PrincipalToken.sol
901:    function _getCurrentPTandIBTRates(bool roundUpPTRate) internal view returns (uint256, uint256) {
        uint256 currentIBTRate = IERC4626(ibt).previewRedeem(ibtUnit).toRay(_assetDecimals); // get current IBTRate
        if (IERC4626(ibt).totalAssets() == 0 && IERC4626(ibt).totalSupply() != 0) { //@audit cache multiple external call
            currentIBTRate = 0;
        }
        uint256 currentPTRate = currentIBTRate < ibtRate
            ? ptRate.mulDiv(
                currentIBTRate,
                ibtRate,
                roundUpPTRate ? Math.Rounding.Ceil : Math.Rounding.Floor
            )
            : ptRate;
        return (currentPTRate, currentIBTRate);
    }
```

### Optimized code:

```diff
diff --git a/src/tokens/PrincipalToken.sol b/src/tokens/PrincipalToken.sol
index 9f4b912..0479abc 100644
--- a/src/tokens/PrincipalToken.sol
+++ b/src/tokens/PrincipalToken.sol
@@ -893,14 +893,15 @@ contract PrincipalToken is 
      * @dev View function to get current IBT and PT rate
      * @param roundUpPTRate true if the ptRate resulting from mulDiv computation in case of negative rate should be rounded up
      * @return new pt and ibt rates
      */
     function _getCurrentPTandIBTRates(bool roundUpPTRate) internal view returns (uint256, uint256) {
-        uint256 currentIBTRate = IERC4626(ibt).previewRedeem(ibtUnit).toRay(_assetDecimals);
-        if (IERC4626(ibt).totalAssets() == 0 && IERC4626(ibt).totalSupply() != 0) {
+            IERC4626 ibtContract = IERC4626(ibt);
+        uint256 currentIBTRate = ibtContract.previewRedeem(ibtUnit).toRay(_assetDecimals); // get current IBTRate
+        if (ibtContract.totalAssets() == 0 && ibtContract.totalSupply() != 0) { 
             currentIBTRate = 0;
         }
         uint256 currentPTRate = currentIBTRate < ibtRate
```
## [G-04] Refactor functions such that all checks are performed before assigning values to state variables.

### Details
Moving the check maxWithdraw(_owner) < _assets to be the early revert in the _beforeWithdraw function could potentially save gas in scenarios where the check fails, as it would prevent executing the subsequent operations in the function. 

### Proof of Code
Validate the function parameters before making the state reads.

- [PrincipalToken.sol#L828-L842](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L828C4-L842C6)
```solidity
File: /src/tokens/PrincipalToken.sol
828:  function _beforeWithdraw(uint256 _assets, address _owner) internal whenNotPaused nonReentrant {
        if (_owner != msg.sender) {
            revert UnauthorizedCaller();
        }
        if (block.timestamp >= expiry) {
            if (!ratesAtExpiryStored) {
                storeRatesAtExpiry();
            }
        } else {
            updateYield(_owner);
        }
        if (maxWithdraw(_owner) < _assets) { //@audit why this condition after above condition 
            revert UnsufficientBalance();
        }
    }
```
### Optimized code:

```diff
diff --git a/src/tokens/PrincipalToken.sol b/src/tokens/PrincipalToken.sol
index 9f4b912..d9dcc80 100644
--- a/src/tokens/PrincipalToken.sol
+++ b/src/tokens/PrincipalToken.sol
@@ -829,21 +829,21 @@ contract PrincipalToken is
         if (_owner != msg.sender) {
             revert UnauthorizedCaller();
         }
+        if (maxWithdraw(_owner) < _assets) { 
+            revert UnsufficientBalance();
+        }
         if (block.timestamp >= expiry) {
             if (!ratesAtExpiryStored) {
                 storeRatesAtExpiry();
             }
         } else {
             updateYield(_owner);
-        }
-        if (maxWithdraw(_owner) < _assets) {
-            revert UnsufficientBalance();
        }   
     }
```
## [G-05] Refactor `_updatePTandIBTRates()` to get better gas optimization

### Details
`_updatePTandIBTRates()` check block.timestamp >= expiry which make sure that ratesAtExiryStored at a same time function also update state variable ptRate and ibtRate. This check involve state variable which is redundant check. We can refactor this function and avoid duplicate check.

### Proof of Code
By doing this we can avoid block.timestamp < expiry
- [PrincipalToken.sol#L879-L894](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L879C4-L894C6)
```solidity
File: src/tokens/PrincipalToken.sol
879:     function _updatePTandIBTRates() internal returns (uint256 _ptRate, uint256 _ibtRate) {
        if (block.timestamp >= expiry) {
            if (!ratesAtExpiryStored) {
                storeRatesAtExpiry();
            }
        }
        (_ptRate, _ibtRate) = _getPTandIBTRates(false);
        if (block.timestamp < expiry) { //@audit modify logic
            if (_ibtRate != ibtRate) { 
                ibtRate = _ibtRate;
            }
            if (_ptRate != ptRate) {
                ptRate = _ptRate;
            }
        }
    }
```
### Optimized code:

```diff
diff --git a/src/tokens/PrincipalToken.sol b/src/tokens/PrincipalToken.sol
index 9f4b912..c6111a1 100644
--- a/src/tokens/PrincipalToken.sol
+++ b/src/tokens/PrincipalToken.sol
@@ -881,13 +887,11 @@ contract PrincipalToken is
             if (!ratesAtExpiryStored) {
                 storeRatesAtExpiry();
             }
-        }
-        (_ptRate, _ibtRate) = _getPTandIBTRates(false);
-        if (block.timestamp < expiry) {
-            if (_ibtRate != ibtRate) {
+        } else {
+            (_ptRate, _ibtRate) = _getPTandIBTRates(false);
+
+            if (_ibtRate != ibtRate || _ptRate != ptRate) {
                 ibtRate = _ibtRate;
-            }
-            if (_ptRate != ptRate) {
                 ptRate = _ptRate;
             }
         }
```
## [G-06] No need to cache state reads if using once
### Details
Variable _oldPTRate is only used therefore no need to cache ptRateOfUser[_user]

### Proof of Code
- [/PrincipalToken.sol#L560C5](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L560C5-L579C1)
```solidity
File:
562:    function getCurrentYieldOfUserInIBT(
        address _user
    ) external view override returns (uint256 _yieldOfUserInIBT) {
        (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);
        uint256 _oldIBTRate = ibtRateOfUser[_user];
        uint256 _oldPTRate = ptRateOfUser[_user]; //@audit used once
        if (_oldIBTRate != 0) {
            _yieldOfUserInIBT = PrincipalTokenUtil._computeYield(
                _user,
                yieldOfUserInIBT[_user],
                _oldIBTRate,
                _ibtRate,
                _oldPTRate,
                _ptRate,
                yt
            );
            _yieldOfUserInIBT -= PrincipalTokenUtil._computeYieldFee(_yieldOfUserInIBT, registry);
        }
    }
```
### Optimized code:
The variable _oldPTRate is only used once in the function. As such there was no need to cache the value of ptRateOfUser[_user]. Caching only adds to the gas consumption rather than save us some gas.

```diff
diff --git a/src/tokens/PrincipalToken.sol b/src/tokens/PrincipalToken.sol
index 9f4b912..0199be9 100644
--- a/src/tokens/PrincipalToken.sol
+++ b/src/tokens/PrincipalToken.sol
@@ -562,14 +564,13 @@ contract PrincipalToken is
     ) external view override returns (uint256 _yieldOfUserInIBT) {
         (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);
         uint256 _oldIBTRate = ibtRateOfUser[_user];
-        uint256 _oldPTRate = ptRateOfUser[_user];
         if (_oldIBTRate != 0) {
             _yieldOfUserInIBT = PrincipalTokenUtil._computeYield(
                 _user,
                 yieldOfUserInIBT[_user],
                 _oldIBTRate,
                 _ibtRate,
-                _oldPTRate,
+                ptRateOfUser[_user],
                 _ptRate,
                 yt
             );
```
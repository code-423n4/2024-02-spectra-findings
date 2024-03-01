# Gas Optimizations

**Note : _G-03_, _G-07_, _G-08_ and _G-11_ contains only those instances which were missed by bot. Since they are major gas savings so I included those missed instances**

## Table of Contents

- [G-01] [State variables can be packed into fewer storage slot by reducing their size (saves ~4000 Gas)](#g-01-state-variables-can-be-packed-into-fewer-storage-slot-by-reducing-their-size-saves-4000-gas)     

- [G-02] [Refactor the `PrincipalToken::getCurrentYieldOfUserInIBT` function to avoid one function call and one `Gcoldsload` when `oldIBTRate == 0`](#g-02-refactor-the-principaltokengetcurrentyieldofuserinibt-function-to-avoid-one-function-call-and-one-gcoldsload-when-oldibtrate--0)

- [G-03] [State variables can be packed by truncating timestamp(Instance Missed by bot)(Gas Saved ~2000 GAS)](#g-03-state-variables-can-be-packed-by-truncating-timestampinstance-missed-by-botgas-saved-2000-gas)      

- [G-04] [Remove `whenNotPaused` modifier check from the functions (Gas Saved ~100+ Gas)](#g-04-remove-whennotpaused-modifier-check-from-the-functions-gas-saved-100-gas)

- [G-05] [Cache external call to avoid re-calling same external function.](#g-05-cache-external-call-to-avoid-re-calling-same-external-function)                                                                                                                          
- [G-06] [Change the order of `modifier` checks in `functions` to fail early, it can save gas Half of the times(Gas Saved ~22000 Gas)](#g-06-change-the-order-of-modifier-checks-in-functions-to-fail-early-it-can-save-gas-half-of-the-timesgas-saved-22000-gas)                                                                                                                          
- [G-07] [State variables should be cached in stack variables rather than re-reading them from storage(Instances Missed by Bot) (Gas Saved ~100 Gas)](#g-07-state-variables-should-be-cached-in-stack-variables-rather-than-re-reading-them-from-storageinstances-missed-by-bot-gas-saved-100-gas)                                                                                                                          
- [G-08] [Use unchecked{} whenever underflow not possible (Instances Missed by bot)(Gas Saved ~320 Gas)](#g-08-use-unchecked-whenever-underflow-not-possible-instances-missed-by-botgas-saved-320-gas)                                                                                                                          
- [G-09] [Use direct `_admin` immutable var. instead of calling `_proxyAdmin()` saves function call.](#g-09-use-direct-_admin-immutable-var-instead-of-calling-_proxyadmin-saves-function-call)                                                                                                                          
- [G-10] [Cache function result into stack var first instead of state variables if need to read them twice](#g-10-cache-function-result-into-stack-var-first-instead-of-state-variables-if-need-to-read-them-twice)                                                                                                                          
- [G-11] [Check `amount` for `zero` before mint/burn (Missed by bot)](#g-11-check-amount-for-zero-before-mintburn-missed-by-bot)                                                                                                                          


## [G-01] State variables can be packed into fewer storage slot by reducing their size (saves ~4000 Gas)

The EVM works with 32 byte words. Variables less than 32 bytes can be declared next to each other in storage and this will pack the values together into a single 32 byte storage slot (if values combined are <= 32 bytes). If the variables packed together are retrieved together in functions (more likely with structs), we will effectively save ~2000 gas with every subsequent SLOAD for that storage slot. This is due to us incurring a Gwarmaccess (100 gas) versus a Gcoldsload (2100 gas).


### `SAVE: 4000 GAS, 2 SLOT`

### `_ibtDecimals`, `_assetDecimals`, and `rewardsProxy` can be packed in a single slot `SAVES: 4000 Gas, 2 SLOT`

Since `_ibtDecimals` and `_assetDecimals` initialized in `initialize` function with uint8 size of decimals we can see at line 141 and 142 `IERC4626(_ibt).decimals()` and `PrincipalTokenUtil._tryGetTokenDecimals(_asset)` returning values of uint8 type which are directly assigned into `_ibtDecimals` and `_assetDecimals` respectively. So uint8 is enough to hold these decimal values therefore `_ibtDecimals` and `_assetDecimals` size can be reduced to `uint8` each which can be packed with address `rewardsProxy` into 1 slot. **Saves 2 storage slots.**


```solidity
File : src/tokens/PrincipalToken.sol

46:    address private rewardsProxy;
     ...

52:    uint256 private _ibtDecimals;
53:    uint256 private _assetDecimals;

```
[PrincipalToken.sol#L46-L53](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L52-L53)

```solidity
File : src/tokens/PrincipalToken.sol

141:   _ibtDecimals = IERC4626(_ibt).decimals();//@audit returning decimal of uint8 type 
142:   _assetDecimals = PrincipalTokenUtil._tryGetTokenDecimals(_asset);//@audit returning decimal of uint8 type
```

[PrincipalToken.sol#L141-142](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L141-142)

**Recommended Mitigation Steps:**

```diff
File : src/tokens/PrincipalToken.sol

46:    address private rewardsProxy;
+      uint8 private _ibtDecimals;
+      uint8 private _assetDecimals;
     ...

-52:    uint256 private _ibtDecimals;
-53:    uint256 private _assetDecimals;


```

## [G-02] Refactor the `PrincipalToken::getCurrentYieldOfUserInIBT` function to avoid one function call and one `Gcoldsload` when `oldIBTRate == 0`

When `oldIBTRate == 0` then calling `_getPTandIBTRates(false)` (at line 563 below) and reading `ptRateOfUser[_user]` (at line 565) is useless since their result not used until  `_oldIBTRate != 0` So it wastes lot of Gas to call and read them when `oldIBTRate == 0` since returned `_yieldOfUserInIBT` value will be 0 when `oldIBTRate` is 0. So place calling `_getPTandIBTRates(false)` and reading `ptRateOfUser[_user]` lines inside if block when `_oldIBTRate != 0` where their values are being used to calculate returned value `_yieldOfUserInIBT`.

#### When `oldIBTRate == 0` It can save safely multiple SLOADs and external calls inside `_getPTandIBTRates(false)` and 1 SLOAD in ptRateOfUser[_user] reading.

```solidity
File : src/tokens/PrincipalToken

560: function getCurrentYieldOfUserInIBT(
        address _user
     ) external view override returns (uint256 _yieldOfUserInIBT) {
563:        (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);
        uint256 _oldIBTRate = ibtRateOfUser[_user];
565:        uint256 _oldPTRate = ptRateOfUser[_user];
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
578:    }

```
[PrincipalToken.sol#L560C5-L578C6](https://github.com/code-423n4/2024-02-spectra/blob/b35bbf78ad9d0e74e9c8450a0c5c6d35b68f7228/src/tokens/PrincipalToken.sol#L560C5-L578C6)

**Recommended Mitigation Steps:**
```diff
File : src/tokens/PrincipalToken

function getCurrentYieldOfUserInIBT(
        address _user
    ) external view override returns (uint256 _yieldOfUserInIBT) {
-        (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);
        uint256 _oldIBTRate = ibtRateOfUser[_user];
-        uint256 _oldPTRate = ptRateOfUser[_user];
        if (_oldIBTRate != 0) {
+        (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false); 
+        uint256 _oldPTRate = ptRateOfUser[_user];   
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

## [G-03] State variables can be packed by truncating timestamp(Instance Missed by bot)(Gas Saved ~2000 GAS)

The EVM works with 32 byte words. Variables less than 32 bytes can be declared next to each other in storage and this will pack the values together into a single 32 byte storage slot (if values combined are <= 32 bytes). If the variables packed together are retrieved together in functions (more likely with structs), we will effectively save ~2000 gas with every subsequent SLOAD for that storage slot. This is due to us incurring a Gwarmaccess (100 gas) versus a Gcoldsload (2100 gas).

### `expiry` and `address yt` can be packed in a single slot `SAVES: 2000 Gas, 1 SLOT`

Since `expiry` holds time (block.timestamp + duration) in seconds. So `uint64` is more than sufficient to hold any realistic time.

```solidity
File : src/tokens/PrincipalToken.sol

50: address private yt; // YT corresponding to this PT, deployed at initialization
     ...

59: uint256 private expiry; // date of maturity (set at initialization)

```
[PrincipalToken.sol#L50](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L50), [PrincipalToken.sol#L59](https://github.com/code-423n4/2024-02-spectra/blob/b35bbf78ad9d0e74e9c8450a0c5c6d35b68f7228/src/tokens/PrincipalToken.sol#L59)

**Recommended Mitigation Steps:**
```diff
File : src/tokens/PrincipalToken.sol

50: address private yt; // YT corresponding to this PT, deployed at initialization
+   uint64 private expiry; // date of maturity (set at initialization)
     ...

-59: uint256 private expiry; // date of maturity (set at initialization)

```








## [G-04] Remove `whenNotPaused` modifier check from the functions (Gas Saved ~100+ Gas)

### Remove `whenNotPaused` from `previewWithdraw` function

Since `whenNotPaused` modifier also used in `previewWithdrawIBT` function which is called in `previewWithdraw` so it will be redundant to check same thing twice.

Reducing one extra call to whenNotPAused can save 1 SLOAD and other opcodes associated with this modifier.

### Saves At least ~100 GAS 

```solidity
File : src/tokens/PrincipalToken.sol

46:    function previewWithdraw(
47:        uint256 assets
48:    ) external view override whenNotPaused returns (uint256) {
49:        uint256 ibts = IERC4626(ibt).previewWithdraw(assets);
50:        return previewWithdrawIBT(ibts);
51:    }

```
[PrincipalToken.sol#L446C1-L451C6](https://github.com/code-423n4/2024-02-spectra/blob/b35bbf78ad9d0e74e9c8450a0c5c6d35b68f7228/src/tokens/PrincipalToken.sol#L446C1-L451C6)

### Remove `whenNotPaused` from `_beforeWithdraw` function

Since `whenNotPaused` modifier also used in `maxWithdraw` function which is called in `_beforeWithdraw` so it will be redundant to check same thing twice.

```solidity
File : src/tokens/PrincipalToken.sol

828:    function _beforeWithdraw(uint256 _assets, address _owner) internal whenNotPaused nonReentrant {
829:        if (_owner != msg.sender) {
830:            revert UnauthorizedCaller();
831:        }
832:        if (block.timestamp >= expiry) {
833:            if (!ratesAtExpiryStored) {
834:                storeRatesAtExpiry();
835:            }
836:        } else {
837:            updateYield(_owner);
838:        }
839:        if (maxWithdraw(_owner) < _assets) {
840:            revert UnsufficientBalance();
841:        }
    }
```
[PrincipalToken.sol#L828C1-L842C6](https://github.com/code-423n4/2024-02-spectra/blob/b35bbf78ad9d0e74e9c8450a0c5c6d35b68f7228/src/tokens/PrincipalToken.sol#L828C1-L842C6)

## [G-05] Cache external call to avoid re-calling same external function.

Since ` IYieldToken(_yt).decimals()` and `IERC20Metadata(_yt).decimals()` are calling same `decimals()` function on same `_yt` address contract. That means both call will return same result since same function on same contract called just interface to prepare instances are different but underlying decimals definition is same so this call can be cached after at first call instead of re-calling in same function twice.

This call is called to YieldToken decimals function.
So it saves 1 External call which includes another external call in YieldToken decimals function. So it saves a lot of gas caching ` IYieldToken(_yt).decimals()` in PrincipalTokenUtil::_computeYield function.

```solidity
File : src/libraries/PrincipalTokenUtil.sol

55:       function _computeYield(
...        
68:        uint256 userYTBalanceInRay = IYieldToken(_yt).actualBalanceOf(_user).toRay(
69:            IYieldToken(_yt).decimals() //@audit cache this
70:        );
...        
129:        return _userYieldIBT + newYieldInIBTRay.fromRay(IERC20Metadata(_yt).decimals());//@audit use cached valued instead of re-calling decimals()
130:    }
```
(https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L55C1-L130C6)


```solidity

15:    contract YieldToken is IYieldToken, ERC20PermitUpgradeable {
...
105:    function decimals()
106:        public
107:        view
108:        virtual
109:        override(IYieldToken, ERC20Upgradeable)
110:        returns (uint8)
111:    {
112:        return IERC20Metadata(pt).decimals();
113:    }

```

(https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L105C1-L113C6)

**Recommended Mitigation Steps:**

```diff
File : src/libraries/PrincipalTokenUtil.sol

55:       function _computeYield(
...        
+           uint256 _ytDecimals = IYieldToken(_yt).decimals();
  68:        uint256 userYTBalanceInRay = IYieldToken(_yt).actualBalanceOf(_user).toRay(
- 69:            IYieldToken(_yt).decimals()
+ 69:            _ytDecimals
  70:        );

        ...      

- 129:        return _userYieldIBT + newYieldInIBTRay.fromRay(IERC20Metadata(_yt).decimals());
+ 129:        return _userYieldIBT + newYieldInIBTRay.fromRay(_ytDecimals);
130:    }
```

## [G-06] Change the order of `modifier` checks in `functions` to fail early, it can save gas Half of the times(Gas Saved ~22000 Gas)

**Total Gas Saved: ~22000 Half of the times**

### Change the order of `nonReentrant` and `whenNotPaused` modifier checks from to high gas consumer one

The function incorporates three modifiers `notExpired`, `nonReentrant` and `whenNotPaused`. An optimization suggestion is made to reorder these modifiers for potential gas savings.
Place `nonReentrant` modifier at third and `whenNotPaused` at 2nd. Since `nonReentrant` takes almost ~24000 Gas due to it's false to true changing value in storage when executed. While `whenNotPaused` have 1 Sload and some other opcodes of modifier. 

#### So when failing then fail with less gas consuming one first. It can save ~22000 GAS Half of the times.

```solidity
File : src/tokens/PrincipalToken

750:  function _depositIBT(
751:    uint256 _ibts,
752:    address _ptReceiver,
753:    address _ytReceiver
754:   ) internal notExpired nonReentrant whenNotPaused returns (uint256 shares) {

```
[PrincipalToken.sol#L750-L754](https://github.com/code-423n4/2024-02-spectra/blob/b35bbf78ad9d0e74e9c8450a0c5c6d35b68f7228/src/tokens/PrincipalToken.sol#L750-L754)

**Recommended Mitigation Steps:**
```diff
File : src/tokens/PrincipalToken

750:  function _depositIBT(
751:    uint256 _ibts,
752:    address _ptReceiver,
753:    address _ytReceiver
-754:   ) internal notExpired nonReentrant whenNotPaused returns (uint256 shares) {
+754:   ) internal notExpired  whenNotPaused nonReentrant returns (uint256 shares) {

```

## [G-07] State variables should be cached in stack variables rather than re-reading them from storage(Instances Missed by Bot) (Gas Saved ~100 Gas)

### `SAVE: 100 GAS, 1 SLOAD`


The instances below point to the second+ access of a state variable within a function. Caching of a state variable replaces each Gwarmaccess (100 gas) with a much cheaper stack read. Other less obvious fixes/optimizations include having local memory caches of state variable structs, or having local caches of state variable contracts/addresses.

### `ibtRate` can be cached to save 1 SLOAD 100 Gas ( 97 Gas technically)

```solidity
File : src/tokens/PrincipalToken.sol

906:       uint256 currentPTRate = currentIBTRate < ibtRate
907:            ? ptRate.mulDiv(
908:                currentIBTRate,
909:                ibtRate,
910:                roundUpPTRate ? Math.Rounding.Ceil : Math.Rounding.Floor
911:            )
912:            : ptRate;
913:        return (currentPTRate, currentIBTRate);
914:    }

```
[PrincipalToken.sol#L906C2-L914C6](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L906C2-L914C6)

**Recommended Mitigation Steps:**
```diff
File : src/tokens/PrincipalToken.sol

+         uint256 _ibtRate = ibtRate;


-906:       uint256 currentPTRate = currentIBTRate < ibtRate
+906:       uint256 currentPTRate = currentIBTRate < _ibtRate
907:            ? ptRate.mulDiv(
908:                currentIBTRate,
-909:                ibtRate,
+909:                _ibtRate,
910:                roundUpPTRate ? Math.Rounding.Ceil : Math.Rounding.Floor
911:            )
912:            : ptRate;
913:        return (currentPTRate, currentIBTRate);
914:    }

```
  

## [G-08] Use unchecked{} whenever underflow not possible (Instances Missed by bot)(Gas Saved ~320 Gas)

**Note: Analyzer only talks about overflow errors so these underflow related instances also not covered by that** 

### Total Gas Saved : ~320 GAS in 2 Instances.

Because of previous condition check before the operation(-), it can be decided that underflow not possible.

#### Saves ~160  GAS per instance

```solidity
File : src/libraries/PrincipalTokenUtil.sol

73:    if (_oldPTRate == _ptRate && _ibtRate > _oldIBTRate) {

75:     newYieldInIBTRay = ibtOfPTInRay.mulDiv(_ibtRate - _oldIBTRate, _ibtRate); //@audit due it's if condition it's subtraction can be unchecked
  
80:       if (_ibtRate >= _oldIBTRate) {
        ...
95:         } else {


104:    uint256 expectedNegativeYieldInAssetRay = Math.ceilDiv(
105:                        ibtOfPTInRay * (_oldIBTRate - _ibtRate), //@audit since it is in else so subtraction can be unchecked

              ...  
              }

```
(https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L75C12-L75C86), (https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L104C21-L105C65)

```diff
File : src/libraries/PrincipalTokenUtil.sol

-75:     newYieldInIBTRay = ibtOfPTInRay.mulDiv(_ibtRate - _oldIBTRate, _ibtRate);
+        unchecked {
+               uint256 ibtRateMinusOldIbtRate = _ibtRate - _oldIBTRate;
+        }
+        newYieldInIBTRay = ibtOfPTInRay.mulDiv(ibtRateMinusOldIbtRate, _ibtRate);

-104:    uint256 expectedNegativeYieldInAssetRay = Math.ceilDiv(
-105:                        ibtOfPTInRay * (_oldIBTRate - _ibtRate),
+        unchecked {
+                uint256 oldIBTRateMinusIbtRate = _oldIBTRate - _ibtRate;
+       }
+        uint256 expectedNegativeYieldInAssetRay = Math.ceilDiv(
+                            ibtOfPTInRay * (oldIBTRateMinusIbtRate),

```

## [G-09] Use direct `_admin` immutable var. instead of calling `_proxyAdmin()` saves function call.

We can access direct immutable `_admin` variable instead of accessing it through function `_proxyAdmin()`. It can save extra function call and saves some gas 100% safely.

```solidity
File : src/proxy/AMTransparentUpgradeableProxy.sol

88:     ERC1967Utils.changeAdmin(_proxyAdmin());


94:     function _proxyAdmin() internal virtual returns (address) {
95:        return _admin;
96:     }


101:    function _fallback() internal virtual override {
102:        if (msg.sender == _proxyAdmin()) {

```
(https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol#L88C9-L88C49), (https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol#L101C1-L102C43)

```diff
File : src/proxy/AMTransparentUpgradeableProxy.sol

-88:     ERC1967Utils.changeAdmin(_proxyAdmin());
+88:     ERC1967Utils.changeAdmin(_admin);


94:     function _proxyAdmin() internal virtual returns (address) {
95:        return _admin;
96:     }


101:    function _fallback() internal virtual override {
-102:        if (msg.sender == _proxyAdmin()) {
+102:        if (msg.sender == _admin) {

```

## [G-10] Cache function result into stack var first instead of state variables if need to read them twice

`_getCurrentPTandIBTRates` function result can be cached into stack var. instead of  `ptRate` and `ibtRate` state variables when state var. read just after that than better will be to read from stack var. again and assigning those stack var. into state var.  also and emit those stack var. also. which saves 2 SLOAD (~200 gas)

### GAS SAVED 2 SLOAD (~200 GAS)

```solidity
File : src/tokens/PrincipalToken.sol

414:   // PT rate not rounded up here
415:   (ptRate, ibtRate) = _getCurrentPTandIBTRates(false);
416:   emit RatesStoredAtExpiry(ibtRate, ptRate); //@audit avoid this read from state var. use stack var. instead

```
[PrincipalToken.sol#L414-L416](https://github.com/code-423n4/2024-02-spectra/blob/b35bbf78ad9d0e74e9c8450a0c5c6d35b68f7228/src/tokens/PrincipalToken.sol#L414-L416)

**Recommended Mitigation Steps:**

```diff
File : src/tokens/PrincipalToken.sol
+     uint256 _ptRate;
+     uint256 _ibtRate;

-415:   (ptRate, ibtRate) = _getCurrentPTandIBTRates(false);
-416:   emit RatesStoredAtExpiry(ibtRate, ptRate);
+415:   (_ptRate, _ibtRate) = _getCurrentPTandIBTRates(false);
+       ptRate = _ptRate;
+       ibtRate = _ibtRate;
+416:   emit RatesStoredAtExpiry(_ibtRate, _ptRate); //read from stack var saved 2 SLOAD

```

## [G-11] Check `amount` for `zero` before mint/burn (Missed by bot)

**Note: Bot only covers check 0 value transfers in tranfser and transferFrom but not covered when minting and burning 0 amounts.**

0 amount burning and minting doesn't have any effect but just wastes gas when 0 value passed by mistake in mint/burn. Openzeppelin(used here) `mint/burn` will not revert when 0 amount passed. So it is better to revert early when amount is 0 in mint/burn rather than wasting more gas in 0 value minting/burning.

```solidity
File : src/tokens/YieldToken.sol

42:    function burnWithoutUpdate(address from, uint256 amount) external override {
43:        if (msg.sender != pt) {
44:            revert CallerIsNotPtContract();
45:        }
46:        _burn(from, amount); //@audit gas check amount for 0
47:    }


50:    function mint(address to, uint256 amount) external override {
51:        if (msg.sender != pt) {
52:            revert CallerIsNotPtContract();
53:        }
54:        _mint(to, amount);//@audit gas check amount for 0
55:    }


58:    function burn(uint256 amount) public override {
59:        IPrincipalToken(pt).updateYield(msg.sender);
60:        _burn(msg.sender, amount);//@audit gas check amount for 0
61:    }

```
(https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L42C1-L47C6), (https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L50C5-L55C6), (https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L58C5-L61C6)    


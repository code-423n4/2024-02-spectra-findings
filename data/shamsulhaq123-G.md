## Gas Optimisition

## Summary

|No|Issue|Instance|
|--|-----|--------|
|[G-01]|address(this) can be stored in state variable that will ultimately cost less|12|
|[G-02]|Call msg.sender directly instead of caching them|18|
|[G-03]|Comparing to constant boolean|17
|[G-04]|Empty Blocks Should Be Removed Or Emit Something|1|
|[G-05]|Using `delete` statement can save gas|3|
|[G-06]|Use `assembly` to write address storage values|3|
|[G-07]|Use shift Left instead of multiplication if possible to save gas|2|
|[G-08]|Use assembly to check for `address(0)`|6|
|[G-09]|Using assembly to check for zero can save gas|1|
|[G-10]|Write direct outcome, instead of performing mathematical operations|2|
|[G-11]|+= costs more gas than = + for state variables|1|
|[G-12]|Unnecessary look up in if condition|2|
|[G-13]|Use assembly to emit events|11|
|[G-14]|Stack variable cost less while used in emiting event|11|
|[G-15]|Superfluous event fields|8|
|[G-16]|Use constants instead of type(uintx).max|2|
|[G-17]|Use hardcode address instead address(this)|12|
|[G-18]|Default value initialization|3|
|[G-19]|Modifiers are redundant if used only once or not used at all. |2|
|[G-20]|Use function instead of modifiers |2|
|[G-21]|Remove or replace unused state variables|1|
|[G-22]|Using private for constants saves gas|1|
|[G-23]Should use arguments instead of state variable |11|

### [G-01] address(this) can be stored in state variable that will ultimately cost less
than every time calculating this specifically when it used multiple times in a contract

```solidity

file: tokens/PrincipalToken.sol

181   IERC20(_asset).safeTransferFrom(msg.sender, address(this), assets);
        IERC20(_asset).safeIncreaseAllowance(ibt, assets);
183        uint256 ibts = IERC4626(ibt).deposit(assets, address(this));

211   IERC20(ibt).safeTransferFrom(msg.sender, address(this), ibts);

235    assets = IERC4626(ibt).redeem(_convertSharesToIBTs(shares, false), receiver, address(this));

285   uint256 ibts = IERC4626(ibt).withdraw(assets, receiver, address(this));

335  assets = IERC4626(ibt).redeem(ibts, msg.sender, address(this));

372  yieldInAsset = IERC4626(ibt).redeem(yieldInIBT, _receiver, address(this));

499  return IERC4626(ibt).previewRedeem(IERC4626(ibt).balanceOf(address(this)));

588  return IERC4626(ibt).balanceOf(address(this));

628   IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);

646  address(this)

736   address(this)

758   address(this)
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L181
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L211
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L235
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L285
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L335
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L372
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L499
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L588
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L628
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L646
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L736
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L758

### [G-02] Call msg.sender directly instead of caching them

In the instance below, instead of caching `msg.sender` and incurring unnecessary stack manipulation, we can call `msg.sender` directly.

```solidity

file: proxy/AMTransparentUpgradeableProxy.sol

102   if (msg.sender == _proxyAdmin()) 

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol#L102

```solidity

file: tokens/PrincipalToken.sol

181 IERC20(_asset).safeTransferFrom(msg.sender, address(this), assets);

211  IERC20(ibt).safeTransferFrom(msg.sender, address(this), ibts);

330   if (msg.sender != IRegistry(registry).getFeeCollector()) 

335  assets = IERC4626(ibt).redeem(ibts, msg.sender, address(this));
        emit FeeClaimed(msg.sender, ibts, assets);

386 if (msg.sender != yt)

624  if (_receiver.onFlashLoan(msg.sender, _token, _amount, fee, _data) != ON_FLASH_LOAN)

767    emit Mint(msg.sender, _ptReceiver, shares);

806   if (_owner != msg.sender) 

829  if (_owner != msg.sender) 

849  yieldInIBT = updateYield(msg.sender);

853  yieldOfUserInIBT[msg.sender] = 0;

857  emit YieldClaimed(msg.sender, msg.sender, yieldInIBT);
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L181
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L211
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L330
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L335-L336 
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L386
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L624
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L767
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L806
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L829
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L849
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L853
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L857

```solidity

file: YieldToken.sol

43 if (msg.sender != pt) 

51   if (msg.sender != pt) 

59  IPrincipalToken(pt).updateYield(msg.sender);
60   _burn(msg.sender, amount);

75  IPrincipalToken(pt).beforeYtTransfer(msg.sender, to);
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L43
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L51
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L59-L60
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L75

```solidity

file: libraries/PrincipalTokenUtil.sol

166 FEE_DIVISOR - IRegistry(_registry).getFeeReduction(_pt, msg.sender)

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L166


## [G-03] Comparing to constant boolean

There is no need to verify that `== true` or `== false` when the variable checked upon is a boolean as well.

```solidity

file: tokens/PrincipalToken.sol

235  assets = IERC4626(ibt).redeem(_convertSharesToIBTs(shares, false), receiver, address(this));

259      ibts = _convertSharesToIBTs(shares, false);

284  (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);

309  (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);

413  ratesAtExpiryStored = true

415 (ptRate, ibtRate) = _getCurrentPTandIBTRates(false);

467  return _convertSharesToIBTs(_maxBurnable(owner), false);

479  return _convertSharesToIBTs(shares, false);

489    return _convertIBTsToShares(IERC4626(ibt).previewDeposit(underlyingAmount), false);

494 return IERC4626(ibt).previewRedeem(_convertSharesToIBTs(principalAmount, false));

534   (, uint256 _ibtRate) = _getPTandIBTRates(false);

540   (uint256 _ptRate, ) = _getPTandIBTRates(false);

563   (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);

663   (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);

684   (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);

762   shares = _convertIBTsToShares(_ibts - tokenizationFee, false);

885  (_ptRate, _ibtRate) = _getPTandIBTRates(false);
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L235
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L259
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L284
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L309
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L413
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L415
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L467
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L479
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L489
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L494
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L534
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L540
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L563
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L663
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L684
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L762
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L885

## [G-04] Empty Blocks Should Be Removed Or Emit Something

The code should be refactored such that they no longer exist, or the block should do something useful, such as emitting an event or reverting. If the contract is meant to be extended, the contract should be abstract and the function signatures be added without any default implementation. If the block is an empty if-statement block to avoid doing subsequent checks in the else-if/else conditions, the else-if/else conditions should be nested under the negation of the if-statement, because they involve different classes of checks, which may lead to the introduction of errors when the code is later modified (if(x){}else if(y){...}else{...} => if(!x){if(y){...}else{...}})

```solidity

file: proxy/AMProxyAdmin.so

28 constructor(address initialAuthority) AccessManaged(initialAuthority) {}

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol#L28


## [G-05] Using `delete` statement can save gas

```solidity

file: tokens/PrincipalToken.sol

413   unclaimedFeesInIBT = 0;

853  yieldOfUserInIBT[msg.sender] = 0;

904 currentIBTRate = 0;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L413
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L853
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L904

## [G-06] Use `assembly` to write address storage values

```solidity

file: tokens/PrincipalToken.sol

106 registry = _registry;

132  duration = _duration;

423  rewardsProxy = _rewardsProxy;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L106
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L132
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L423

## [G-07] Use shift Left instead of multiplication if possible to save gas

```solidity

file: tokens/PrincipalToken.sol

151  ibtUnit = 10 ** _ibtDecimals;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L151

```solidity

file: libraries/PrincipalTokenUtil.sol

105  ibtOfPTInRay * (_oldIBTRate - _ibtRate),
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L105

### [G-08]  Use assembly to check for `address(0)`

```solidity

file: proxy/AMTransparentUpgradeableProxy.sol

83   initialAuthority != address(0),

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol#L83

```solidity

file: tokens/PrincipalToken.sol

103  if (_registry == address(0)) 

125    if (_ibt == address(0) || _initialAuthority == address(0)) 

395  if (rewardsProxy == address(0)) 

726 if (ytBeacon == address(0)) 

733 IYieldToken(address(0)).initialize.selector,
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L103
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L125
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L395
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L726
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L733

### [G-09] Using assembly to check for zero can save gas

Using assembly to check for zero can save gas by allowing more direct access to the evm and reducing some of the overhead associated with high-level operations in solidity.

```solidity

file: proxy/AMBeacon.sol

76  if (newImplementation.code.length == 0) 

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L76

## [G-10] Write direct outcome, instead of performing mathematical operations

Write direct outcome instead ofperforming mathematical operations saves some gas

```solidity

file: tokens/PrincipalToken.sol

151  ibtUnit = 10 ** _ibtDecimals;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L15

```solidity

file: libraries/PrincipalTokenUtil.sol

105  ibtOfPTInRay * (_oldIBTRate - _ibtRate),
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L105

## [G-11] += costs more gas than = + for state variables
use = + or = - instead to save gas

```solidity

file:tokens/PrincipalToken.sol

714 unclaimedFeesInIBT += _feesInIBT;
715 totalFeesInIBT += _feesInIBT;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L714-L715

## [G-12] Unnecessary look up in if condition

If the || condition isn’t required, the second condition will have been looked up unnecessarily.

```solidity

file:tokens/PrincipalToken.sol

125   if (_ibt == address(0) || _initialAuthority == address(0)) 

144 _assetDecimals < MIN_DECIMALS ||
145   _assetDecimals > _ibtDecimals ||
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L125
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L144-L155 

## [G-13] Use assembly to emit events
We can use assembly to emit events efficiently by utilizing `scratch space` and the `free memory pointer`. This will allow us to potentially avoid memory expansion costs. Note: In order to do this optimization safely, we will need to cache and restore the free memory pointer.

```solidity

file: proxy/AMBeacon.sol

80    emit Upgraded(newImplementation);
``` 
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L80

```solidity

file: tokens/PrincipalToken.sol

236   emit Redeem(owner, receiver, shares);

261 emit Redeem(owner, receiver, shares);

336  emit FeeClaimed(msg.sender, ibts, assets);

364  emit YieldUpdated(_user, updatedUserYieldInIBT);

416  emit RatesStoredAtExpiry(ibtRate, ptRate);

422   emit RewardsProxyChange(rewardsProxy, _rewardsProxy);

740    emit YTDeployed(_yt);

767   emit Mint(msg.sender, _ptReceiver, shares);

797  emit Redeem(_owner, _receiver, shares);

857 emit YieldClaimed(msg.sender, msg.sender, yieldInIBT);
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L236
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L261
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L336
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L364
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L416
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L422
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L740
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L767
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L797
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L857

## [G-14] Stack variable cost less while used in emiting event
Even if the variable is going to be used only one time, caching a state variable and use its cache in an emit would help you reduce the cost by at least

```solidity

file: proxy/AMBeacon.sol

80    emit Upgraded(newImplementation);
``` 
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L80

```solidity

file: tokens/PrincipalToken.sol

236   emit Redeem(owner, receiver, shares);

261 emit Redeem(owner, receiver, shares);

336  emit FeeClaimed(msg.sender, ibts, assets);

364  emit YieldUpdated(_user, updatedUserYieldInIBT);

416  emit RatesStoredAtExpiry(ibtRate, ptRate);

422   emit RewardsProxyChange(rewardsProxy, _rewardsProxy);

740    emit YTDeployed(_yt);

767   emit Mint(msg.sender, _ptReceiver, shares);

797  emit Redeem(_owner, _receiver, shares);

857 emit YieldClaimed(msg.sender, msg.sender, yieldInIBT);
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L236
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L261
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L336
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L364
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L416
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L422
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L740
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L767
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L797
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L857

## [G-15] Superfluous event fields
`block.timestamp` and `block.number` are added to event information by default so adding them manually wastes gas

```solidity

file: tokens/PrincipalToken.sol

86   if (block.timestamp >= expiry)

94 if (block.timestamp < expiry) 

133  expiry = _duration + block.timestamp;

793  if (block.timestamp < expiry) 

812    if (block.timestamp >= expiry)

832    if (block.timestamp >= expiry) 

880   if (block.timestamp >= expiry) {
        
886        if (block.timestamp < expiry) 
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L86
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L94
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L133
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L793
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L812
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L832
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L880-L886

```solidity

file: tokens/YieldToken.sol

124    return (block.timestamp < IPrincipalToken(pt).maturity()) ? super.balanceOf(account) : 0;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L124

## [G-16] Use constants instead of type(uintx).max

type(uint120).max or type(uint128).max, etc. it uses more gas in the distribution process and also for each transaction than constant usage.

```solidity

file: okens/PrincipalToken.sol#

442  return type(uint256).max;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L442

```solidity

file: libraries/PrincipalTokenUtil.sol

143  if (returnedDecimals <= type(uint8).max) 

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L143

## [G-17] Use hardcode address instead address(this)
Instead of using address(this), it is more gas-efficient to pre-calculate and use the hardcoded address.

```solidity

file: tokens/PrincipalToken.sol

181   IERC20(_asset).safeTransferFrom(msg.sender, address(this), assets);
        IERC20(_asset).safeIncreaseAllowance(ibt, assets);
183        uint256 ibts = IERC4626(ibt).deposit(assets, address(this));

211   IERC20(ibt).safeTransferFrom(msg.sender, address(this), ibts);

235    assets = IERC4626(ibt).redeem(_convertSharesToIBTs(shares, false), receiver, address(this));

285   uint256 ibts = IERC4626(ibt).withdraw(assets, receiver, address(this));

335  assets = IERC4626(ibt).redeem(ibts, msg.sender, address(this));

372  yieldInAsset = IERC4626(ibt).redeem(yieldInIBT, _receiver, address(this));

499  return IERC4626(ibt).previewRedeem(IERC4626(ibt).balanceOf(address(this)));

588  return IERC4626(ibt).balanceOf(address(this));

628   IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);

646  address(this)

736   address(this)

758   address(this)
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L181
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L211
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L235
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L285
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L335
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L372
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L499
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L588
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L628
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L646
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L736
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L758

## [G-18] Default value initialization
Problem
If a variable is not set/initialized, it is assumed to have the default value (0, false, 0x0 etc depending on the data type).
Explicitly initializing it with its default value is an anti-pattern and wastes gas.
Instances include:

```solidity

file: tokens/PrincipalToken.sol

413   unclaimedFeesInIBT = 0;

853  yieldOfUserInIBT[msg.sender] = 0;

904 currentIBTRate = 0;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L413
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L853
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L904

## [G-19] Modifiers are redundant if used only once or not used at all. 

```solidity

file: tokens/PrincipalToken.sol

85   modifier notExpired() virtual 

93  modifier afterExpiry() virtual 
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L85
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L93

## [G-20] Use function instead of modifiers 

```solidity

file: tokens/PrincipalToken.sol

85   modifier notExpired() virtual 

93  modifier afterExpiry() virtual 
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L85
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L93

## [G-21] Remove or replace unused state variables

```solidity

file: tokens/PrincipalToken.sol

62  mapping(address => uint256) private ibtRateOfUser; 
    mapping(address => uint256) private ptRateOfUser; 
 64   mapping(address => uint256) private yieldOfUserInIBT;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L62-L64

## [G-22] Using private for constants saves gas
If needed, the values can be read from the verified contract source code, or if there are multiple values there can be a single getter function that returns a tuple of the values of all currently-public constants. Saves 3406-3606 gas in deployment gas due to the compiler not having to create non-payable getter functions for deployment calldata, not having to store the bytes of the value outside of where it's used, and not adding another entry to the method ID table

```solidity

file: proxy/AMProxyAdmin.sol

23 string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";


```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol#L23

## [G-23] Should use arguments instead of state variable 

state variables should not used in emit  ,  This will save near 97 gas  

```solidity

file: proxy/AMBeacon.sol

80    emit Upgraded(newImplementation);
``` 
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L80

```solidity

file: tokens/PrincipalToken.sol

236   emit Redeem(owner, receiver, shares);

261 emit Redeem(owner, receiver, shares);

336  emit FeeClaimed(msg.sender, ibts, assets);

364  emit YieldUpdated(_user, updatedUserYieldInIBT);

416  emit RatesStoredAtExpiry(ibtRate, ptRate);

422   emit RewardsProxyChange(rewardsProxy, _rewardsProxy);

740    emit YTDeployed(_yt);

767   emit Mint(msg.sender, _ptReceiver, shares);

797  emit Redeem(_owner, _receiver, shares);

857 emit YieldClaimed(msg.sender, msg.sender, yieldInIBT);
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L236
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L261
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L336
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L364
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L416
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L422
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L740
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L767
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L797
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L857
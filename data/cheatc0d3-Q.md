# 🌟 QA Report: Spectra

## 📚 Table of Contents

1. [Approach](#Approach)
2. [[L-01] Lack of collateral asset validations leads to potential worthless principal token](#L-01)
3. [[L-02] Unsafe use of delegatecall without validation in PrincipalToken contract](#L-02)
4. [[L-03] Missing Events for Critical State Changes and Administrative Functions](#L-03)
5. [[L-04] Insufficient input validation for Function Parameters in PrincipalToken::initialize](#L-04)
7. [[L-05] Missing Zero Amount Validation](#L-05)
8. [[L-06] Failure to validate output in PrincipalToken.sol::_convertIBTsToShares and PrincipalToken.sol::_convertSharesToIBTs](#L-06)
9. [[L-07] Front-running Attacks possible on AMProxyAdmin::_dispatchUpgradeToAndCall and AMProxyAdmin::upgradeAndCall](#L-07)
10. [Conclusion](#Conclusion)


## 🛠️ Approach

I manually reviewed the code, focusing on logic, external calls, and user inputs. I kicked off with reviewing each contract code manually line by line based on a checklist I drafted for low severity and nuanced issues in solidity smart contracts. This helped quickly flag common problems and areas needing closer inspection.

## [L-01] Lack of collateral asset validations leads to potential worthless principal token

The principal token and yield token values depend directly on the assets deposited in the IBT vault that backs it. However, there are no checks on the type, quality, or risks associated with the collateral asset choice. Poor collateral selection or liquidations can make the tokens lose complete value.

```solidity
File: PrincipalToken.sol
120:     function initialize(
121:         address _ibt,
122:         uint256 _duration,
123:         address _initialAuthority
124:     ) external initializer {
125:         if (_ibt == address(0) || _initialAuthority == address(0)) {
126:             revert AddressError();
127:         }
128:         if (IERC4626(_ibt).totalAssets() == 0) {
129:             revert RateError();
130:         }
131:         _asset = IERC4626(_ibt).asset();
132:         duration = _duration;
133:         expiry = _duration + block.timestamp;
134:         string memory _ibtSymbol = IERC4626(_ibt).symbol();
135:         string memory name = NamingUtil.genPTName(_ibtSymbol, expiry);
136:         __ERC20_init(name, NamingUtil.genPTSymbol(_ibtSymbol, expiry));
137:         __ERC20Permit_init(name);
138:         __Pausable_init();
139:         __ReentrancyGuard_init();
140:         __AccessManaged_init(_initialAuthority);
141:         _ibtDecimals = IERC4626(_ibt).decimals();
142:         _assetDecimals = PrincipalTokenUtil._tryGetTokenDecimals(_asset);
143:         if (
144:             _assetDecimals < MIN_DECIMALS ||
145:             _assetDecimals > _ibtDecimals ||
146:             _ibtDecimals > MAX_DECIMALS
147:         ) {
148:             revert InvalidDecimals();
149:         }
150:         ibt = _ibt;
151:         ibtUnit = 10 ** _ibtDecimals;
152:         ibtRate = IERC4626(ibt).previewRedeem(ibtUnit).toRay(_assetDecimals);
153:         ptRate = RayMath.RAY_UNIT;
154:         yt = _deployYT(
155:             NamingUtil.genYTName(_ibtSymbol, expiry),
156:             NamingUtil.genYTSymbol(_ibtSymbol, expiry)
157:         );
158:     }

```
[Lines of code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L120)

### Mitigation 

```solidity 
// Validate and set minimum collateralization ratio
function validateAndSetIBT(
    address _ibt, 
    uint minCollateralRatio
) internal {

    // Check asset type and volatility
    require(IBTAssetChecker.validateAsset(_ibt), "Unsafe asset");
    
    // Set minimum collateralization level
    require(IERC4626(_ibt).collateralRatio() >= minCollateralRatio, "Undercollateralized"); 

    // Approve trusted price feeds for periodic checks
    IBTPriceFeed(0x...).addConsumer(address(this), _ibt);

    // Set circuit breaker level 
    circuitBreakerPrice = IBTPriceFeed(_ibt).latestPrice() * 0.8; 

    ibt = _ibt;
}

// Check price feed and pause if circuit breaker triggered 
function checkCircuitBreaker() public {
   if (IBTPriceFeed(ibt).latestPrice() < circuitBreakerPrice) {
        _pause();
   }
}
```


## [L-02] Unsafe use of delegatecall without validation in PrincipalToken contract

The use of `delegatecall` with dynamic user-supplied data (`_data`) in `claimRewards` introduces risks, such as unexpected behavior or vulnerabilities due to the execution context (storage, msg.sender, etc.) being shared with the called contract. This could potentially be misused if the `rewardsProxy` contract has vulnerabilities or if there's an error in the way `_data` is handled.

```solidity
File: PrincipalToken.sol
394:     function claimRewards(bytes memory _data) external restricted {
395:         if (rewardsProxy == address(0)) {
396:             revert NoRewardsProxySet();
397:         }
398:         _data = abi.encodeWithSelector(IRewardsProxy(rewardsProxy).claimRewards.selector, _data);
399:         (bool success, ) = rewardsProxy.delegatecall(_data);
400:         if (!success) {
401:             revert ClaimRewardsFailed();
402:         }
403:     }
```
[Lines of code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L394)


### Mitigation
Avoid using `delegatecall` with user-supplied data when possible. If `delegatecall` must be used, ensure rigorous validation of the input data.

## [L-03] Missing Events for Critical State Changes and Administrative Functions

In the `PrincipalToken` contract, the code does not emit events when fees are updated or when critical thresholds are reached. In addition, administrative functions such as `pause`, `unPause`, and `setRewardsProxy` are critical but do not emit events upon their invocation.

Functions to add events:

- `_updateFees` 
- `pause`
- `unpause`


[Lines of code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L713)
[Lines of code](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L161)
[Lines of code](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L166)


## Mitigation

Emit events when significant fee-related actions are taken. This ensures that users and external systems can accurately track fee-related changes and their impact on transactions.

```solidity
event FeesUpdated(uint256 newTokenizationFee, uint256 newYieldFee);
event Paused(address indexed account);
event Unpaused(address indexed account);
```

## [L-04] Insufficient input validation for Function Parameters in PrincipalToken::initialize

The `_duration` parameter in initialize function of PrincipalToken contract is not explicitly checked to ensure it's within a reasonable range, which could lead to unexpected behavior or vulnerabilities.

```solidity
File: PrincipalToken.sol
120:     function initialize(
121:         address _ibt,
122:         uint256 _duration,
123:         address _initialAuthority
124:     ) external initializer {
125:         if (_ibt == address(0) || _initialAuthority == address(0)) {
126:             revert AddressError();
127:         }
128:         if (IERC4626(_ibt).totalAssets() == 0) {
129:             revert RateError();
130:         }
131:         _asset = IERC4626(_ibt).asset();
132:         duration = _duration;
133:         expiry = _duration + block.timestamp;
134:         string memory _ibtSymbol = IERC4626(_ibt).symbol();
135:         string memory name = NamingUtil.genPTName(_ibtSymbol, expiry);
136:         __ERC20_init(name, NamingUtil.genPTSymbol(_ibtSymbol, expiry));
137:         __ERC20Permit_init(name);
138:         __Pausable_init();
139:         __ReentrancyGuard_init();
140:         __AccessManaged_init(_initialAuthority);
141:         _ibtDecimals = IERC4626(_ibt).decimals();
142:         _assetDecimals = PrincipalTokenUtil._tryGetTokenDecimals(_asset);
143:         if (
144:             _assetDecimals < MIN_DECIMALS ||
145:             _assetDecimals > _ibtDecimals ||
146:             _ibtDecimals > MAX_DECIMALS
147:         ) {
148:             revert InvalidDecimals();
149:         }
150:         ibt = _ibt;
151:         ibtUnit = 10 ** _ibtDecimals;
152:         ibtRate = IERC4626(ibt).previewRedeem(ibtUnit).toRay(_assetDecimals);
153:         ptRate = RayMath.RAY_UNIT;
154:         yt = _deployYT(
155:             NamingUtil.genYTName(_ibtSymbol, expiry),
156:             NamingUtil.genYTSymbol(_ibtSymbol, expiry)
157:         );
158:     }

```
[Lines of code](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L120)


## Mitigation

Add checks to ensure that `_duration` results in a future expiry date.

```solidity
if (_duration <= 0 || _duration > MAX_DURATION) {
    revert InvalidDuration();
}
```

## [L-05] Missing Zero Amount Validation

Lack of validation for `assets` parameter in deposit and withdraw functions of PrincipalToken contract. Transferring zero assets could lead to unintended effects or wasted gas.

```solidity
File: PrincipalToken.sol
176:     function deposit(
177:         uint256 assets,
178:         address ptReceiver,
179:         address ytReceiver
180:     ) public override returns (uint256 shares) {
181:         IERC20(_asset).safeTransferFrom(msg.sender, address(this), assets);
182:         IERC20(_asset).safeIncreaseAllowance(ibt, assets);
183:         uint256 ibts = IERC4626(ibt).deposit(assets, address(this));
184:         shares = _depositIBT(ibts, ptReceiver, ytReceiver);
185:     }

```
[Lines of code](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L176)


```solidity
File: PrincipalToken.sol
278:     function withdraw(
279:         uint256 assets,
280:         address receiver,
281:         address owner
282:     ) public override returns (uint256 shares) {
283:         _beforeWithdraw(assets, owner);
284:         (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);
285:         uint256 ibts = IERC4626(ibt).withdraw(assets, receiver, address(this));
286:         shares = _withdrawShares(ibts, receiver, owner, _ptRate, _ibtRate);
287:     }

```

[Lines of code](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L278)

### Mitigation

Validate that the `assets` amount is greater than zero.

```solidity
if (assets <= 0) {
    revert InvalidAssetAmount();
}
if (ptReceiver == address(0) || ytReceiver == address(0)) {
    revert InvalidReceiverAddress();
}
```

## [L-06] Failure to validate output in PrincipalToken.sol::_convertIBTsToShares and PrincipalToken.sol::_convertSharesToIBTs

The `_convertIBTsToShares` function calculates the number of shares based on the IBT amount without validating the output. In scenarios where conversion rates lead to unexpected results (e.g due to extremely low rates leading to zero shares for a non-zero IBT amount), this can lead to loss of funds or denial of service.

Similarly the `_convertSharesToIBTs` function converts PT shares to IBTs without validating the result.

```solidity
File: PrincipalToken.sol
680:     function _convertIBTsToShares(
681:         uint256 _ibts,
682:         bool _roundUp
683:     ) internal view returns (uint256 shares) {
684:         (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);
685:         if (_ptRate == 0) {
686:             revert RateError();
687:         }
688:         shares = _ibts.mulDiv(
689:             _ibtRate,
690:             _ptRate,
691:             _roundUp ? Math.Rounding.Ceil : Math.Rounding.Floor
692:         );
693:     }

```
[Lines of code](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L680)

```solidity
File: PrincipalToken.sol
659:     function _convertSharesToIBTs(
660:         uint256 _shares,
661:         bool _roundUp
662:     ) internal view returns (uint256 ibts) {
663:         (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);
664:         if (_ibtRate == 0) {
665:             revert RateError();
666:         }
667:         ibts = _shares.mulDiv(
668:             _ptRate,
669:             _ibtRate,
670:             _roundUp ? Math.Rounding.Ceil : Math.Rounding.Floor
671:         );
672:     }
```
[Lines of code](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L659)

### Mitigation

Add validation to ensure the conversion results are sensible and revert on invalid outputs.

```solidity
if (ibts == 0 && _shares > 0) {
    revert InvalidConversionOutput();
}
```

## [L-07] Front-running Attacks possible on AMProxyAdmin::_dispatchUpgradeToAndCall and AMProxyAdmin::upgradeAndCall

The `_dispatchUpgradeToAndCall` function decodes the `newImplementation` address and `data` directly from the call data without any form of validation. This approach is vulnerable to front-running attacks where an attacker could potentially front-run the transaction with a malicious `newImplementation` contract.

Similar to this issue above, the upgradeandCall function allows the caller to upgrade a proxy to a new implementation and execute a call in a single transaction. Without proper validation of the `implementation` address, it's susceptible to front-running attacks, where an attacker could replace a legitimate upgrade transaction with one pointing to a malicious implementation.

```solidity
File: AMTransparentUpgradeableProxy.sol
120:     function _dispatchUpgradeToAndCall() private {
121:         (address newImplementation, bytes memory data) = abi.decode(msg.data[4:], (address, bytes));
122:         ERC1967Utils.upgradeToAndCall(newImplementation, data);
123:     }

```
[Lines of code](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMTransparentUpgradeableProxy.sol#L120)

```solidity
File: AMProxyAdmin.sol
40:     function upgradeAndCall(
41:         IAMTransparentUpgradeableProxy proxy,
42:         address implementation,
43:         bytes memory data
44:     ) public payable virtual restricted {
45:         proxy.upgradeToAndCall{value: msg.value}(implementation, data);
46:     }

```
[Lines of code](https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMProxyAdmin.sol#L40)

### Mitigation

Implement checks to validate the `newImplementation` address before proceeding with the upgrade. This could involve maintaining a whitelist of allowed implementation addresses or requiring a multi-step process for upgrades that include a delay and validation step. Additionaly, use commit-reveal schemes to prevent such attacks from happening. 

## 📌 Conclusion

The QA analysis of the code in Spectra contracts revealed some low severity issues and potential design considerations that could bolster security and operation efficiency of the contracts. You can follow the mitigations outlined in the report for quick fixes to the issues discovered.
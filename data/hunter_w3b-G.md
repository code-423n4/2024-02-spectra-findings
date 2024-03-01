## [G-01] A MODIFIER used only once and not being inherited should be inlined to save gas

```solidity
File:  src/tokens/PrincipalToken.sol

93    modifier afterExpiry() virtual {
94        if (block.timestamp < expiry) {
95            revert PTNotExpired();
96        }
97        _;
98    }
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L93-L98

this is only used in this function

```solidity
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

## [G-02] Private functions which are only called once can be inlined

```solidity
File:  src/proxy/AMTransparentUpgradeableProxy.sol


120    function _dispatchUpgradeToAndCall() private {
121        (address newImplementation, bytes memory data) = abi.decode(msg.data[4:], (address, bytes));
122        ERC1967Utils.upgradeToAndCall(newImplementation, data);
123    }
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol#L120-L123

this function in only called in this function

```solidity

    function _fallback() internal virtual override {
        if (msg.sender == _proxyAdmin()) {
            if (msg.sig != IAMTransparentUpgradeableProxy.upgradeToAndCall.selector) {
                revert ProxyDeniedAdminAccess();
            } else {
                _dispatchUpgradeToAndCall();// >>>>here
            }
        } else {
            super._fallback();
        }
    }
```

## [G-03]  Use unchecked for operations on constant/immutable variables

Arithmetic involving constant values are predictable and as such, should not need to include the automatic overflow/underflow check. Wrapping inside an unchecked block saves gas.

```solidity
File: src/libraries/PrincipalTokenUtil.sol

163            _amount
164                .mulDiv(IRegistry(_registry).getTokenizationFee(), FEE_DIVISOR, Math.Rounding.Ceil)
165                .mulDiv(
166                    FEE_DIVISOR - IRegistry(_registry).getFeeReduction(_pt, msg.sender),
167                    FEE_DIVISOR,
168                    Math.Rounding.Ceil
169                );


179        return _amount.mulDiv(IRegistry(_registry).getYieldFee(), FEE_DIVISOR, Math.Rounding.Ceil);


193            _amount.mulDiv(
194                IRegistry(_registry).getPTFlashLoanFee(),
195                FEE_DIVISOR,
196                Math.Rounding.Ceil
197            );
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L163-L169

## [G-04] Expressions for constant values such as a call to  KECCAK256(), should use IMMUTABLE rather than CONSTANT

```solidity
File: src/tokens/PrincipalToken.sol

42    bytes32 private constant ON_FLASH_LOAN = keccak256("ERC3156FlashBorrower.onFlashLoan");
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L42

## [G-05] Using fixed BYTES CONSTANTS is cheaper than using string

If data can fit into 32 bytes, then you should use bytes32 datatype rather than bytes or strings as it is cheaper in solidity.

```solidity
File: src/proxy/AMProxyAdmin.sol

23    string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol#L23

## [G-06] Use ASSEMBLY to write address storage values

```solidity
File: src/proxy/AMBeacon.sol

21    address private _implementation;
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L21

```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

65    address private immutable _admin;
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol#L65

```solidity
File: src/tokens/PrincipalToken.sol

44    address private immutable registry;

46    address private rewardsProxy;

48    address private ibt; // address of the Interest Bearing Token 4626 held by this PT vault
49    address private _asset; // the asset of this PT vault (which is also the asset of the IBT 4626)
50    address private yt; // YT corresponding to this PT, deployed at initialization
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L44-L50

```solidity
File: src/tokens/YieldToken.sol

18    address private pt;
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L18

## [G-07] Optimize External Calls with ASSEMBLY for Memory Efficiency

Using interfaces to make external contract calls in Solidity is convenient but can be inefficient in terms of memory utilization. Each such call involves creating a new memory location to store the data being passed, thus incurring memory expansion costs.

Inline assembly allows for optimized memory usage by re-using already allocated memory spaces or using the scratch space for smaller datasets. This can result in notable gas savings, especially for contracts that make frequent external calls.
Additionally, using inline assembly enables important safety checks like verifying if the target address has code deployed to it using extcodesize(addr) before making the call, mitigating risks associated with contract interactions.

```solidity
File: src/tokens/PrincipalToken.sol

128        if (IERC4626(_ibt).totalAssets() == 0) {
131        _asset = IERC4626(_ibt).asset();
134        string memory _ibtSymbol = IERC4626(_ibt).symbol();
141        _ibtDecimals = IERC4626(_ibt).decimals();
152        ibtRate = IERC4626(ibt).previewRedeem(ibtUnit).toRay(_assetDecimals);


181        IERC20(_asset).safeTransferFrom(msg.sender, address(this), assets);
182        IERC20(_asset).safeIncreaseAllowance(ibt, assets);
183        uint256 ibts = IERC4626(ibt).deposit(assets, address(this));


308        _beforeWithdraw(IERC4626(ibt).previewRedeem(ibts), owner);
312        IERC20(ibt).safeTransfer(receiver, ibts);


330        if (msg.sender != IRegistry(registry).getFeeCollector()) {
335        assets = IERC4626(ibt).redeem(ibts, msg.sender, address(this));


621        IERC20(ibt).safeTransfer(address(_receiver), _amount);
624        if (_receiver.onFlashLoan(msg.sender, _token, _amount, fee, _data) != ON_FLASH_LOAN)
628        IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);


902        uint256 currentIBTRate = IERC4626(ibt).previewRedeem(ibtUnit).toRay(_assetDecimals);
903        if (IERC4626(ibt).totalAssets() == 0 && IERC4626(ibt).totalSupply() != 0) {
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L128

## [G-08] Use ASSEMBLY to emit events

```solidity
File: src/proxy/AMBeacon.sol

80        emit Upgraded(newImplementation);
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L80

```solidity
File: src/tokens/PrincipalToken.sol

236        emit Redeem(owner, receiver, shares);

261        emit Redeem(owner, receiver, shares);

336        emit FeeClaimed(msg.sender, ibts, assets);

364            emit YieldUpdated(_user, updatedUserYieldInIBT);

416        emit RatesStoredAtExpiry(ibtRate, ptRate);

422        emit RewardsProxyChange(rewardsProxy, _rewardsProxy);

740        emit YTDeployed(_yt);

767        emit Mint(msg.sender, _ptReceiver, shares);

797        emit Redeem(_owner, _receiver, shares);

857            emit YieldClaimed(msg.sender, msg.sender, yieldInIBT);
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L236

## [G-09]  Simple checks for zero uint can be done using ASSEMBLY to save gas

```solidity
File: src/proxy/AMBeacon.sol

76        if (newImplementation.code.length == 0) {
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L76

```solidity
File: src/tokens/PrincipalToken.sol


128        if (IERC4626(_ibt).totalAssets() == 0) {

664        if (_ibtRate == 0) {

685        if (_ptRate == 0) {

703        if (_ptRate == 0) {

763        if (shares == 0) {

787        if (_ptRate == 0) {

850        if (yieldInIBT == 0) {

903        if (IERC4626(ibt).totalAssets() == 0 && IERC4626(ibt).totalSupply() != 0) {

903        if (IERC4626(ibt).totalAssets() == 0 && IERC4626(ibt).totalSupply() != 0) {
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L128

## [G-10] Use ASSEMBLY in place of abi.decode to extract calldata values more efficiently

```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

121        (address newImplementation, bytes memory data) = abi.decode(msg.data[4:], (address, bytes));

```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol#L121

```solidity
File: src/libraries/PrincipalTokenUtil.sol

142            uint256 returnedDecimals = abi.decode(encodedDecimals, (uint256));
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L142

## [G-11] Refactor MODIFIER to call a local function

Modifiers code is copied in all instances where it's used, increasing bytecode size. By doing a refractor to the internal function, one can reduce bytecode size significantly at the cost of one JUMP.

```solidity
File: src/tokens/PrincipalToken.sol

85    modifier notExpired() virtual {
86        if (block.timestamp >= expiry) {
87            revert PTExpired();
88        }
89        _;
90    }
91
92    /// @notice Ensures the current block timestamp is at or after expiry
93    modifier afterExpiry() virtual {
94        if (block.timestamp < expiry) {
95            revert PTNotExpired();
96        }
97        _;
98    }
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L85C1-L98C6

## [G-12] Make 3 Event parameters indexed when possible

It's the most gas efficient to make up to 3 event parameters indexed. If there are less than 3 parameters, you need to make all parameters indexed.

```solidity
File: src/tokens/PrincipalToken.sol

68    event Redeem(address indexed from, address indexed to, uint256 amount);
69    event Mint(address indexed from, address indexed to, uint256 amount);
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L68-L69

## [G-13] Superflous mint event

Calling `_mint` emits an event, and so emitting a separate event in the same function is a waste of gas. Each event costs at least 375 gas. An additional 375 is paid for each indexed parameter.

```solidity
File: src/tokens/PrincipalToken.sol

766        _mint(_ptReceiver, shares);
767        emit Mint(msg.sender, _ptReceiver, shares);
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L766-L767

## [G-14] Use Constants instead of type(uintx).max

```solidity
File: src/tokens/PrincipalToken.sol

442        return type(uint256).max;
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L442

```solidity
File: src/libraries/PrincipalTokenUtil.sol

143            if (returnedDecimals <= type(uint8).max) {
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L143

## [G-15] Avoid unnecessary public variables

```solidity
File: src/proxy/AMProxyAdmin.sol

23    string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol#L23

```solidity
File: src/libraries/RayMath.sol

12    uint256 public constant RAY_UNIT = 1e27;
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol#L12

## [G-16] Use ERC20Permit to batch the approval and transfer step in on transaction

ERC20 Permit has an additional function that accepts digital signatures from a toke holder to increase the approval for another address. This way, the recipient of the approval can submit the permit transaction and the transfer in one transaction. The user granting the permit does not have to pay any gas, and the recipient of the permit can batch the permit and transferFrom transaction into a single transaction.

## [G-17] The UUPS upgrade pattern is more gas efficient for users than the Transparent Upgradeable Proxy

The transparent upgradeable proxy pattern requires reading the admin address out of the storage slot (defined by ERC-1967) every time a transaction happens, to see if the caller is the admin or not. This introduces an extra storage read.

## [G-18] Consider using alternatives to OpenZeppelin

OpenZeppelin is a great and popular smart contract library, but there are other alternatives that are worth considering. These alternatives offer better gas efficiency and have been tested and recommended by developers.

Two examples of such alternatives are `Solmate` and `Solady`.

Solmate is a library that provides a number of gas-efficient implementations of common smart contract patterns. Solady is another gas-efficient library that places a strong emphasis on using assembly.

## [G-19] Using assembly to revert with an error message

When reverting in solidity code, it is common practice to use a require or revert statement to revert execution with an error message. This can in most cases be further optimized by using assembly to revert with the error message.

```solidity
File: src/tokens/PrincipalToken.sol

87            revert PTExpired();

95            revert PTNotExpired();

104            revert AddressError();

126            revert AddressError();

129            revert RateError();

148            revert InvalidDecimals();

196            revert ERC5143SlippageProtectionFailed();

224            revert ERC5143SlippageProtectionFailed();

248            revert ERC5143SlippageProtectionFailed();

273            revert ERC5143SlippageProtectionFailed();

298            revert ERC5143SlippageProtectionFailed();

324            revert ERC5143SlippageProtectionFailed();

331            revert UnauthorizedCaller();

387            revert UnauthorizedCaller();

396            revert NoRewardsProxySet();

401            revert ClaimRewardsFailed();

411            revert RatesAtExpiryAlreadyStored();

595        if (_token != ibt) revert AddressError();

615        if (_amount > maxFlashLoan(_token)) revert FlashLoanExceedsMaxAmount();

625            revert FlashLoanCallbackFailed();

665            revert RateError();

686            revert RateError();

704            revert RateError();

727            revert BeaconNotSet();

764            revert RateError();

788            revert RateError();

807            revert UnauthorizedCaller();

810            revert UnsufficientBalance();

830            revert UnauthorizedCaller();

840            revert UnsufficientBalance();
```

https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L87

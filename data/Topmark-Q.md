### Report 1:
#### Code Reverts When Shares has been used to Settle Fees
The code below shows how _depositIBT(...) function is implemented in the PrincipalToken contract, as noted from the code adjust below initially the code reverts if shares is zero due to shares error but the protocol didnt consider the fact that it might not be an error but shares can be zero if the whole value has been used to service fees, instead of reverting with loss to the protocol, the code should be readjusted to only attempt minting if shares is not zero
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L763
```solidity
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
        shares = _convertIBTsToShares(_ibts - tokenizationFee, false);
---        if (shares == 0) {
+++        if (shares != 0) {
---            revert RateError();
+++         _mint(_ptReceiver, shares);
        }
---        _mint(_ptReceiver, shares);
        emit Mint(msg.sender, _ptReceiver, shares);
        IYieldToken(yt).mint(_ytReceiver, shares);
    }
```
###  Report 2:
#### Inconsistent Value conversion
The two functions provided below shows how conversion is done between IBTS and Shares however a look at how _ptRate and _ibtRate is derived shows that one of the function used false parameter with _getPTandIBTRates(...) function call while the other uses true, this will create inconsistency in value conversion and can be taken advantage of by bad actors
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L684/PrincipalToken.sol#L684
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L702
```solidity
   function _convertIBTsToShares(
        uint256 _ibts,
        bool _roundUp
    ) internal view returns (uint256 shares) {
>>>        (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);
        if (_ptRate == 0) {
            revert RateError();
        }
        shares = _ibts.mulDiv(
            _ibtRate,
            _ptRate,
            _roundUp ? Math.Rounding.Ceil : Math.Rounding.Floor
        );
    }
    function _convertIBTsToSharesPreview(uint256 ibts) internal view returns (uint256 shares) {
>>>        (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(true); // to round up the shares, the PT rate must round down
        if (_ptRate == 0) {
            revert RateError();
        }
        shares = ibts.mulDiv(_ibtRate, _ptRate);
    }
```
###  Report 3:
#### Shares is Calculated by Rounding Up
The code below shows how shares is withdrawn from the PrincipalToken contract as noted from the pointer Math.Rounding.Ceil was used which is wrong as the roundup value from withdrawals would accumulate over time and would be a loss to the protocol.
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L791
```solidity
 function _withdrawShares(
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
>>>        shares = _ibts.mulDiv(_ibtRate, _ptRate, Math.Rounding.Ceil);
        // burn owner's shares (YT and PT)
        if (block.timestamp < expiry) {
            IYieldToken(yt).burnWithoutUpdate(_owner, shares);
        }
        _burn(_owner, shares);
        emit Redeem(_owner, _receiver, shares);
    }
```
###  Report 4:
#### Incomplete Validation
The implementation of initialize(...) function as noted from the pointer shows that validation was done to ensure `_assetDecimals` is not greater than `_ibtDecimals` to ensure consistency in the codebase, however this is not totally right as the correct condition to validate would be to ensure `_assetDecimals` is equal `_ibtDecimals` which guarantees consistency in contract.
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L145
```solidity
  function initialize(
        address _ibt,
        uint256 _duration,
        address _initialAuthority
    ) external initializer {
       ...
        if (
            _assetDecimals < MIN_DECIMALS ||
>>>            _assetDecimals > _ibtDecimals ||
            _ibtDecimals > MAX_DECIMALS
        ) {
            revert InvalidDecimals();
        }
    ...
```
###  Report 5:
#### Zero Return instead of Reversion
A look at implementation of _computeYield(...) function shows that whenever the yield to be calculated would be zero the function reverts however that is not the case in the pointer noted in the code below, the compute yield function would return zero instead of reverting, this should be corrected by the protocol
https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L110
```solidity
function _computeYield(
        address _user,
        uint256 _userYieldIBT,
        uint256 _oldIBTRate,
        uint256 _ibtRate,
        uint256 _oldPTRate,
        uint256 _ptRate,
        address _yt
    ) external view returns (uint256) {
...
     uint256 expectedNegativeYieldInAssetRay = Math.ceilDiv(
                        ibtOfPTInRay * (_oldIBTRate - _ibtRate),
                        RayMath.RAY_UNIT
                    );
                    yieldInAssetRay = expectedNegativeYieldInAssetRay >
                        actualNegativeYieldInAssetRay
 >>>                       ? 0
                        : actualNegativeYieldInAssetRay - expectedNegativeYieldInAssetRay;
                    yieldInAssetRay = yieldInAssetRay.fromRay(
                        IERC4626(IPrincipalToken(IYieldToken(_yt).getPT()).underlying()).decimals()
                    ) < SAFETY_BOUND
                        ? 0
                        : yieldInAssetRay;
                }
...
}
```
###  Report 6:
#### Wrong Openzeppelin Parameter
A look at the pointer from the function below from PrincipalToken.sol contract shows that a call is made to Openzeppelin 4626 contract at [L214](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/ef68ac3ed83f85e4ce078b26170280c468ffeabb/contracts/token/ERC20/extensions/ERC4626.sol#L214), the problem is that the function uses shares in it parameter however the code below shows that instead of also using shares directly, it was converted to IBTS which is wrong, shares should be used directly.
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L235
```solidity
     /** @dev See {IPrincipalToken-redeem}. */
    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) public override returns (uint256 assets) {
        _beforeRedeem(shares, owner);
>>>        assets = IERC4626(ibt).redeem(_convertSharesToIBTs(shares, false), receiver, address(this));
        emit Redeem(owner, receiver, shares);
    }
```
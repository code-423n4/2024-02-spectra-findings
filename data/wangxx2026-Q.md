## [L-01]The maxWithdraw method does not meet the eip-5095 standard.
eip-5095 requires protocol pause to return 0, the implementation is revert
https://eips.ethereum.org/EIPS/eip-5095#maxwithdraw
```
MUST factor in both global and user-specific limits, like if withdrawals are entirely disabled (even temporarily) it MUST return 0.

MUST NOT revert.
```
Code implementation:
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L460-L462
```solidity
    function maxWithdraw(address owner) public view override whenNotPaused returns (uint256) {
        return convertToUnderlying(_maxBurnable(owner));
    }
```
whenNotPaused implementation:
https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/67b51f0e69a558a3ce3997c74905d49741b1f504/contracts/utils/PausableUpgradeable.sol#L72-L75
```solidity
    modifier whenNotPaused() {
        _requireNotPaused();
        _;
    }

    function _requireNotPaused() internal view virtual {
        if (paused()) {
            revert EnforcedPause();
        }
    }
```
And convertToUnderlying() will also revert, see [L-02].

## [L_02] previewRedeem、convertToPrincipal、convertToUnderlying can be revert does not comply with eip-5095.
eip-5095 requirements:
https://eips.ethereum.org/EIPS/eip-5095#converttounderlying
```
MUST NOT revert unless due to integer overflow caused by an unreasonably large input.
```
Code implementation:
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L493-L495
```solidity
    function convertToUnderlying(uint256 principalAmount) public view override returns (uint256) {
        return IERC4626(ibt).previewRedeem(_convertSharesToIBTs(principalAmount, false));
    }
```
_convertSharesToIBTs implementation:
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L659-L672
```solidity
    function _convertSharesToIBTs(
        uint256 _shares,
        bool _roundUp
    ) internal view returns (uint256 ibts) {
        (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(false);
        if (_ibtRate == 0) {
            revert RateError(); // @audit will revert
        }
        ibts = _shares.mulDiv(
            _ptRate,
            _ibtRate,
            _roundUp ? Math.Rounding.Ceil : Math.Rounding.Floor
        );
    }
```

## [L-03] The time at which redeem can be executed does not comply with eip-5095.
eip-5095 requirements:
https://eips.ethereum.org/EIPS/eip-5095#redeem
```
At or after maturity, burns exactly principalAmount of Principal Tokens from from and sends underlyingAmount of underlying tokens to to
```
Code implementation has no expiration time limit:
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L229-L237
```solidity
    function redeem(
        uint256 shares,
        address receiver,
        address owner
    ) public override returns (uint256 assets) {
        _beforeRedeem(shares, owner);
        assets = IERC4626(ibt).redeem(_convertSharesToIBTs(shares, false), receiver, address(this));
        emit Redeem(owner, receiver, shares);
    }
```





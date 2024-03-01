`function _withdrawShares(
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
        if (block.timestamp < expiry) {
            IYieldToken(yt).burnWithoutUpdate(_owner, shares);
        }
        _burn(_owner, shares);
        emit Redeem(_owner, _receiver, shares);
    }`

Burning of Owner's Shares:

The function includes logic to burn the owner's shares but does so conditionally based on the current block.timestamp relative to an expiry variable.
`if (block.timestamp < expiry) { IYieldToken(yt).burnWithoutUpdate(_owner, shares); }`
This conditional burning seems to indicate that shares are only burned if the current time is before some expiry. However, there's an immediate call to `_burn(_owner, shares);` outside this conditional block, which implies shares are burned regardless of the time check. This is likely a logical error or redundancy:
If the intent was to conditionally burn shares based on the timestamp, the unconditional _burn call contradicts this logic.
If the intent was always to burn shares, the conditional check is unnecessary, or the logic within it does not align with the overall goal.

### Time spent:
40 hours
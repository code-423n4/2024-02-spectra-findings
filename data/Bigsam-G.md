Consider nesting the currentIBTRATE equation in the PrincipalToken.sol in an else statement.
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L902-L905 
Based on my analysis I have provided 2 code snippets, the first one is more gas-efficient and the second one is being used presently by the contract. Let's go through the reasoning:

### Code Snippet 1:

```solidity
function _getCurrentPTandIBTRates(bool roundUpPTRate) internal view returns (uint256, uint256) {
    uint256 currentIBTRate;
    if (IERC4626(ibt).totalAssets() == 0 && IERC4626(ibt).totalSupply() != 0) {
        currentIBTRate = 0;
    } else {
        currentIBTRate = IERC4626(ibt).previewRedeem(ibtUnit).toRay(_assetDecimals);
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
Result from testing this with foundry 
[PASS] test_getCurrentPTandIBTRates(bool) (runs: 1024, μ: 59113, ~: 59113)

### Code Snippet 2 used by the contract:

```solidity
  function _getCurrentPTandIBTRates(bool roundUpPTRate) internal view returns (uint256, uint256) {
       uint256 currentIBTRate = IERC4626(ibt).previewRedeem(ibtUnit).toRay(_assetDecimals);
        if (IERC4626(ibt).totalAssets() == 0 && IERC4626(ibt).totalSupply() != 0) {
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
Test result from foundry
[PASS] test_getCurrentPTandIBTRates(bool) (runs: 1024, μ: 59137, ~: 59137)


### Considerations:

In the first code snippet, the `currentIBTRate` is only assigned once based on the condition, and there is no redundant re-initialization. This is more gas-efficient because it avoids unnecessary operations.

Therefore, the first code snippet is preferred for its slightly better gas efficiency. It follows a more straightforward and concise approach to initializing `currentIBTRate` based on the specified conditions.
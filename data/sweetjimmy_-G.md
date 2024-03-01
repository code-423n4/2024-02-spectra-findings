## Use `>` in place of `!=` to make sure a value is not zero

This can save ≈25 gas per comparison. In following locations this optimization can be made.

1. https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L353
2. https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L371
3. https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L379
4. https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L566
5. https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L903

## Assign `currentIBTRate` only once

In the following lines, `currentIBTRate` will be assigned a value whose computation is quite gas intensive (because of an external call and conversion to ray). However, if the condition of the subsequent `if` statement passes, `currentIBTRate` will get assigned to 0, overwriting the previously computed value (which was gas intensive). Hence, incurring additional gas usage in some cases.


```solidity
        uint256 currentIBTRate = IERC4626(ibt).previewRedeem(ibtUnit).toRay(_assetDecimals);
        if (IERC4626(ibt).totalAssets() == 0 && IERC4626(ibt).totalSupply() != 0) {
            currentIBTRate = 0;
        }
```

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L902-L905

Instead, put the gas intensive computation inside an `else` block.

```solidity
        uint256 currentIBTRate;
        if (IERC4626(ibt).totalAssets() == 0 && IERC4626(ibt).totalSupply() != 0) {
            currentIBTRate = 0;
        } else {
            currentIBTRate = IERC4626(ibt).previewRedeem(ibtUnit).toRay(_assetDecimals);
        }
```

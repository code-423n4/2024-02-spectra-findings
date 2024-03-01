## Optimizing Rate Update Functionality:

Original Implementation:
Calls _getPTandIBTRates() unconditionally, potentially leading to redundant function calls.
Refactored Improvement:
Modified to avoid unnecessary function calls when the expiry condition is met.
Enhances gas efficiency and streamlines the update process.

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L879

```diff
function _updatePTandIBTRates() internal returns (uint256 _ptRate, uint256 _ibtRate) {
     if (block.timestamp >= expiry) {
         if (!ratesAtExpiryStored) {
             storeRatesAtExpiry();
         }
     }       
-    (_ptRate, _ibtRate) = _getPTandIBTRates(false);
     else if (block.timestamp < expiry) {
+        (_ptRate, _ibtRate) = _getPTandIBTRates(false);
         if (_ibtRate != ibtRate) {
             ibtRate = _ibtRate;
         }
         if (_ptRate != ptRate) {
             ptRate = _ptRate;
         }
     }
}
```
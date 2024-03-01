### [G-01 ]State variables should be cached in stack variables rather than re-reading them from storage

The instances below point to the second+ access of a state variable FEE_DIVISOR  within a function _computeTokenizationFee. Caching of a state variable replaces each Gwarmaccess (100 gas) with a much cheaper stack read.

line: 164,166,167

Proposed Optimizations: cache state variable inside function in local variable and use it multiple time.

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/libraries/PrincipalTokenUtil.sol#L164

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L181https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L181

### [G-02]Use smaller data types for constant state variable with value 100 that never get change save gas.

Proposed Optimization: Use smaller and more gas-efficient data types whenever possible. For example, use `uint256` only when necessary; if a variable's range is smaller, consider using `uint8`, `uint16`, etc.

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L40

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L41

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/libraries/PrincipalTokenUtil.sol#L18
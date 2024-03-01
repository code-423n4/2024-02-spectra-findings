# GAS OPTIMIZATION 




## [G-01] Optimize External Calls with Assembly for Memory Efficiency

Using interfaces to make external contract calls in Solidity is convenient but can be inefficient in terms of memory utilization. Each such call involves creating a new memory location to store the data being passed, thus incurring memory expansion costs.
Inline assembly allows for optimized memory usage by re-using already allocated memory spaces or using the scratch space for smaller datasets. This can result in notable gas savings, especially for contracts that make frequent external calls.

```solidity
File:  src/tokens/PrincipalToken.sol
181        IERC20(_asset).safeTransferFrom(msg.sender, address(this), assets);
182        IERC20(_asset).safeIncreaseAllowance(ibt, assets);

211   IERC20(ibt).safeTransferFrom(msg.sender, address(this), ibts);

260  IERC20(ibt).safeTransfer(receiver, ibts);

312  IERC20(ibt).safeTransfer(receiver, ibts);

621  IERC20(ibt).safeTransfer(address(_receiver), _amount);

628  IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L181


```solidity
File:  src/tokens/YieldToken.sol
59  IPrincipalToken(pt).updateYield(msg.sender);

75  IPrincipalToken(pt).beforeYtTransfer(msg.sender, to);

100 IPrincipalToken(pt).beforeYtTransfer(from, to);

112 return IERC20Metadata(pt).decimals();
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L59



## [G-02] Expressions for constant values such as a call to keccak256(), should use immutable rather than constant
The reason for this is that constant variables are evaluated at runtime and their value is included in the bytecode of the contract. This means that any expensive operations performed as part of the constant expression, such as a call to keccak256(), will be executed every time the contract is deployed, even if the result is always the same. This can result in higher gas costs.

```
bytes32 constant MY_HASH = keccak256("my string");
```
Alternatively, we could use an immutable variable, like so:
```
bytes32 immutable MY_HASH = keccak256("my string");
```


```solidity
File:  src/tokens/PrincipalToken.sol
42  bytes32 private constant ON_FLASH_LOAN = keccak256("ERC3156FlashBorrower.onFlashLoan");
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L42

## [G‑03] Avoid contract existence checks by using low level calls
the compiler inserted extra code, including EXTCODESIZE (100 gas), to check for contract existence for external function calls. In more recent solidity versions, the compiler will not insert these checks if the external call has a return value. Similar behavior can be achieved in earlier versions by using low-level calls, since low level calls never check for contract existence.

Total save gas = `100 * 4 = 400`
```solidity
File:  src/tokens/YieldToken.sol
59  IPrincipalToken(pt).updateYield(msg.sender);

75  IPrincipalToken(pt).beforeYtTransfer(msg.sender, to);

100 IPrincipalToken(pt).beforeYtTransfer(from, to);

112 return IERC20Metadata(pt).decimals();
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L59



## [G-04] Use assembly in place of abi.decode to extract calldata values more efficiently

Instead of using abi.decode, we can use assembly to decode our desired calldata values directly. This will allow us to avoid decoding calldata values that we will not use.
```solidity
File:  src/libraries/PrincipalTokenUtil.sol
142  uint256 returnedDecimals = abi.decode(encodedDecimals, (uint256));
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L142


```solidity
File:  src/proxy/AMTransparentUpgradeableProxy.sol
121  (address newImplementation, bytes memory data) = abi.decode(msg.data[4:], (address, bytes));
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol#L121



## [G‑05] SPLITTING REVERT STATEMENTS

The contract is using multiple conditions in a single if statement followed by a revert. This costs some extra gas.


```solidity
File:  src/tokens/PrincipalToken.sol
125  if (_ibt == address(0) || _initialAuthority == address(0)) {
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L125



## [G-06] Use assembly to perform efficient back-to-back calls
If a similar external call is performed back-to-back, we can use assembly to reuse any function signatures and function parameters that stay the same. In addition, we can also reuse the same memory space for each function call (scratch space + free memory pointer + zero slot), which can potentially allow us to avoid memory expansion costs.

```solidity
File:  src/tokens/PrincipalToken.sol
181        IERC20(_asset).safeTransferFrom(msg.sender, address(this), assets);
182        IERC20(_asset).safeIncreaseAllowance(ibt, assets);
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L181-L182


## [G-07] Use assembly to write address storage values 

```solidity
File:  src/proxy/AMBeacon.sol
79  _implementation = newImplementation;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L79

```solidity
File:  src/tokens/PrincipalToken.sol
131   _asset = IERC4626(_ibt).asset();

150   ibt = _ibt;

423  rewardsProxy = _rewardsProxy;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L131

```solidity
File:  src/tokens/YieldToken.sol
38   pt = _pt;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L38



## [G‑08] Unlimited gas consumption risk due to external call recipients

When calling an external function without specifying a gas limit , the called contract may consume all the remaining gas, causing the tx to be reverted. To mitigate this, it is recommended to explicitly set a gas limit when making low level external calls.

```solidity
File:  src/proxy/AMProxyAdmin.sol
45  proxy.upgradeToAndCall{value: msg.value}(implementation, data);
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol#L45



## [G-09] A modifier used only once and not being inherited should be inlined to save gas


`afterExpiry` modifier only use in [storeRatesAtExpiry](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L409)
```solidity
File:  src/tokens/PrincipalToken.sol
93   modifier afterExpiry() virtual {
        if (block.timestamp < expiry) {
            revert PTNotExpired();
        }
        _;
    }
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L93-L98


## [G-10] Use constants instead of type(uintx).max

Using constants instead of type(uintx).max can save gas in some cases because constants are precomputed at compile-time and can be reused throughout the code, whereas type(uintx).max requires computation at runtime


```solidity
File:  src/libraries/PrincipalTokenUtil.sol
143  if (returnedDecimals <= type(uint8).max) {
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L143


```solidity
File:  src/tokens/PrincipalToken.sol
422  return type(uint256).max;
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L422

## [G-11] bytes constants are more eficient than string constans
If the data can fit in 32 bytes, the bytes32 data type can be used instead of bytes or strings, as it is less robust in terms of robustness.


```solidity
File:  src/proxy/AMProxyAdmin.sol
23  string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol#L23



## [G-12] Use assembly to calculate hashes to save gas

Using assembly to calculate hashes can save 80 gas per instance

```solidity
File:  src/tokens/PrincipalToken.sol
42  bytes32 private constant ON_FLASH_LOAN = keccak256("ERC3156FlashBorrower.onFlashLoan");
```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L42

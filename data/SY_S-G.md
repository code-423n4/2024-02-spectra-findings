## Summary

### Gas Optimization

no |Issue |Instances||
|-|:-|:-:|:-:|
| [G-01] |Optimize address(0) Checks Using Assembly |  |1|--| 
| [G-02] |Bytes constants are more efficient than String constants |  |1|--| 
| [G-03] |abi.encode() is less efficient than  abi.encodepacked() |  |2|--| 
| [G-04] |set assembly in place of `abi.decode` to extract `calldata` values more efficiently |  |9|--| 
| [G-05] |Stack variable cost less while used in emiting event |  |8|--| 
| [G-06] |Reduce gas usage by moving to Solidity 0.8.22 or later |  |7|--| 
| [G-07] |Don’t make variables public unless necessary |  |2|--| 


## Gas Optimizations  

## [G-1] Optimize address(0) Checks Using Assembly
Solidity provides address(0) as a constant for the zero address. While it is generally safe to use, explicit checks against address(0) in your code might be slightly more gas efficient if implemented in inline assembly, due to reduced overhead.
```solidity
file:/src/proxy/AMTransparentUpgradeableProxy.sol
  83          initialAuthority != address(0),
```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMTransparentUpgradeableProxy.sol#L83-L83


## [G-2] Bytes constants are more efficient than String constants
If data can fit into 32 bytes, then you should use bytes32 datatype rather than bytes or strings as it is cheaper in solidity.


```solidity
File: /src/proxy/AMProxyAdmin.sol
23    string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";
```

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMProxyAdmin.sol#L23




## [G-3]  abi.encode() is less efficient than  abi.encodepacked()

In terms of efficiency, abi.encodePacked() is generally considered to be more gas-efficient than abi.encode(), because it skips the step of adding function signatures and other metadata to the encoded data. However, this comes at the cost of reduced safety, as abi.encodePacked() does not perform any type checking or padding of data.

```solidity
file: src/tokens/PrincipalToken.sol
 398      _data = abi.encodeWithSelector(IRewardsProxy(rewardsProxy).claimRewards.selector, _data);
 732               abi.encodeWithSelector(


```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L398



## [G-4] set assembly in place of `abi.decode` to extract `calldata` values more efficiently
Instead of using abi.decode, we can use assembly to decode our desired calldata values directly. This will allow us to avoid decoding calldata values that we will not use.
For more details on how to implement this, check the following [report](https://code4rena.com/reports/2023-05-juicebox#g-04-use-assembly-in-place-of-abidecode-to-extract-calldata-values-more-efficiently)

*There are 9 instance(s) of this issue:*

```solidity
file: /src/proxy/AMTransparentUpgradeableProxy.sol
  121      (address newImplementation, bytes memory data) = abi.decode(msg.data[4:], (address, bytes));


```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMTransparentUpgradeableProxy.sol#L121
```solidity
file:src/tokens/PrincipalToken.sol

 732               abi.encodeWithSelector(
 733                  IYieldToken(address(0)).initialize.selector,
 734                   _name,
 735                   _symbol,
 736                  address(this)
 737             )
        
```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L732-L739

```solidity
file:/src/libraries/PrincipalTokenUtil.sol
 139           abi.encodeCall(IERC20Metadata.decimals, ())
 142           uint256 returnedDecimals = abi.decode(encodedDecimals, (uint256));
```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/libraries/PrincipalTokenUtil.sol#L139
## [G-5] Stack variable cost less while used in emiting event
Even if the variable is going to be used only one time, caching a state variable and use its cache in an emit would help you reduce the cost by at least ***9 gas***

*There are 8 instance(s) of this issue:*

```solidity
file:/src/proxy/AMBeacon.sol
80        emit Upgraded(newImplementation);
```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMBeacon.sol#L80

```solidity
file:/src/tokens/PrincipalToken.sol
 236       emit Redeem(owner, receiver, shares);
261        emit Redeem(owner, receiver, shares);
336        emit FeeClaimed(msg.sender, ibts, assets);
364            emit YieldUpdated(_user, updatedUserYieldInIBT);
422        emit RewardsProxyChange(rewardsProxy, _rewardsProxy);
797        emit Redeem(_owner, _receiver, shares);
857            emit YieldClaimed(msg.sender, msg.sender, yieldInIBT);
```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L236




## [G-6] Reduce gas usage by moving to Solidity 0.8.22 or later


```solidity
file:
pragma solidity 0.8.20;
```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMBeacon.sol#L5C1-L5C24

```solidity
file:src/proxy/AMProxyAdmin.sol
5       pragma solidity 0.8.20;
```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMProxyAdmin.sol#L5
```solidity
file:/src/proxy/AMTransparentUpgradeableProxy.sol
4       pragma solidity 0.8.20;
```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMTransparentUpgradeableProxy.sol#L4

```solidity
file:src/tokens/PrincipalToken.sol
3      pragma solidity 0.8.20;
```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L3

```solidity
file:src/tokens/YieldToken.sol
3       pragma solidity 0.8.20;
```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/YieldToken.sol#L3
```solidity
file:/src/libraries/PrincipalTokenUtil.sol
3       pragma solidity 0.8.20;
```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/libraries/PrincipalTokenUtil.sol#L3

## [G-7] Don’t make variables public unless necessary
Issue Description - Making variables public comes with some overhead costs that can be avoided in many cases. A public variable implicitly creates a public getter function of the same name, increasing the contract size.

Proposed Optimization - Only mark variables as public if their values truly need to be readable by external contracts/users. Otherwise, use private or internal visibility.

Estimated Gas Savings - The savings from avoiding unnecessary public variables are small per transaction, around 5-10 gas. However, this adds up over many transactions targeting a contract with public state variables that don’t need to be public.
```solidity
file:/src/proxy/AMProxyAdmin.sol
23    string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";

```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMProxyAdmin.sol#L23-L23
```solidity
file:/src/libraries/RayMath.sol
 12   uint256 public constant RAY_UNIT = 1e27;
```
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/libraries/RayMath.sol#L12-L12


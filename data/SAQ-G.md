## Summary

### Gas Optimization

no | Issue |Instances||
|-|:-|:-:|:-:|
| [G-01] | Bytes constants are more efficient than String constants | 1 | - |
| [G-02] | Before transfer of  some functions, we should check some variables for possible gas save | 6 | - |
| [G-03] | Duplicated if() checks should be refactored to a modifier or function  | 1 | - |
| [G-04] | Use constants instead of type(uintx).max | 2 | - |
| [G-05] | 2**<N> should be re-written as type(uint<N>).max | 4 | - |
| [G-06] | Unused named return variables wast deployment gas | 2 | - |

## Gas Optimizations  

## [G-01] Bytes constants are more efficient than String constants

If data can fit into 32 bytes, then you should use bytes32 datatype rather than bytes or strings as it is cheaper in solidity.


```solidity
file: /src/proxy/AMProxyAdmin.sol

23    string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol#L23C1-L23C64


## [G-02] Before transfer of  some functions, we should check some variables for possible gas save

Before transfer, we should check for amount being 0 so the function doesn't run when its not gonna do anything.

```solidity
file: /src/tokens/YieldToken.sol

75       IPrincipalToken(pt).beforeYtTransfer(msg.sender, to);
76        return super.transfer(to, amount);


100        IPrincipalToken(pt).beforeYtTransfer(from, to);
101        return super.transferFrom(from, to, amount);

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L75C1-L76C43


```solidity
file: /src/tokens/PrincipalToken.sol

181        IERC20(_asset).safeTransferFrom(msg.sender, address(this), assets);

260        IERC20(ibt).safeTransfer(receiver, ibts);

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L181C1-L181C76


## [G-03] Duplicated if() checks should be refactored to a modifier or function   


```solidity
file: /src/tokens/PrincipalToken.sol

///@audit the bellow if statement is come duplicate 4 another time in this contracts, Lins: 812, 832, 867, 880

86        if (block.timestamp >= expiry) {

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L86C1-L86C41


## [G-04] Use constants instead of type(uintx).max

type(uint120).max or type(uint128).max, etc. it uses more gas in the distribution process and also for each transaction than constant usage.


```solidity
file: /src/libraries/PrincipalTokenUtil.sol

143            if (returnedDecimals <= type(uint8).max) {

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol#L143C1-L143C55


```solidity
file: /src/tokens/PrincipalToken.sol

442        return type(uint256).max;

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L442C1-L442C34


## [G-05] 2**<N> should be re-written as type(uint<N>).max

Earlier versions of solidity can use uint<n>(-1) instead. Expressions not including the - 1 can often be re-written to accomodate the change (e.g. by using a > rather than a >=, which will also save some gas)

https://code4rena.com/reports/2023-04-frankencoin#g-05-2n-should-be-re-written-as-typeuintnmax


```solidity
file: /src/tokens/PrincipalToken.sol

151        ibtUnit = 10 ** _ibtDecimals;

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L151C1-L151C38


```solidity
file: /src/libraries/RayMath.sol

21        uint256 decimals_ratio = 10 ** (27 - _decimals);

40        uint256 decimals_ratio = 10 ** (27 - _decimals);

58        uint256 decimals_ratio = 10 ** (27 - _decimals);

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol#L21C1-L21C57


## [G-06] Unused named return variables wast deployment gas


```solidity
file: /src/tokens/YieldToken.sol

///@audit bool success

74    ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {

99    ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L99C1-L99C86

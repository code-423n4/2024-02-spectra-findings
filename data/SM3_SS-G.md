
## Gas Optimizations

| Number | Issue | Instences |
|--------|-------|-----------|
|[G-01]| Using bytes32 is cheaper than using string.  | 6 |
|[G-02]| Constants Variable Should Be Private for Gas Optimization | 5 |
|[G-03]| Use assembly to emit events  | 11 |
|[G-04]| Use assembly to write address storage values | 2 |
|[G-05]| Use the external Visibility Modifier | 14 |
|[G-06]| Direct updates to private variables: | 4 |
|[G-07]| Use uint8 Can Increase Gas Cost | 2 |
|[G-08]| Avoid Unnecessary Use of Storage | |
|[G-09]| Change Constant to Immutable for keccak Variables | 1 |
|[G-10]| No need to initialize variable size to 0 | 2 |


## [G-01] Using bytes32 is cheaper than using string.

Storing and manipulating data in bytes32 is more gas-efficient than using string, which involves dynamic storage allocation. bytes32 is a fixed-size type, whereas string can have variable length, resulting in higher gas costs.

```solidity
file: blob/main/src/proxy/AMProxyAdmin.sol

23  string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol#L23

```solidity
file: blob/main/src/tokens/PrincipalToken.sol

134   string memory _ibtSymbol = IERC4626(_ibt).symbol();

135   string memory name = NamingUtil.genPTName(_ibtSymbol, expiry);

724   function _deployYT(string memory _name, string memory _symbol) internal returns (address _yt) {

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L134


```solidity
file: blob/main/src/tokens/YieldToken.sol
 
32  string calldata _name,

33  string calldata _symbol,

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L32


## [G-02] Constants Variable Should Be Private for Gas Optimization

This RayMath,AMProxyAdmin and PrincipalToken == contracts that inefficiently use public for constant variables. Using private for constant variables is cheaper in terms of gas usage. If the value of the constant variable is needed, it can be accessed via a getter function. In case of multiple constant variables, a single getter function could be used to return a tuple of the values of all currently-private constants.

```solidity
file: blob/main/src/libraries/RayMath.sol

12  uint256 public constant RAY_UNIT = 1e27;

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol#L12

```solidity
file: blob/main/src/proxy/AMProxyAdmin.sol

23  string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol#L23

```solidity
file: blob/main/src/tokens/PrincipalToken.sol

40  uint256 private constant MIN_DECIMALS = 6;

41  uint256 private constant MAX_DECIMALS = 18;

42  bytes32 private constant ON_FLASH_LOAN = keccak256("ERC3156FlashBorrower.onFlashLoan");

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L40

## [G-03] Use assembly to emit events 

This detector checks for instances where a contract uses assembly code to emit events. While it’s technically possible to emit events using inline assembly in Solidity, it’s generally discouraged due to readability concerns and potential for errors. Events should usually be emitted using higher-level Solidity constructs.

```solidity
file: blob/main/src/proxy/AMBeacon.sol

80   emit Upgraded(newImplementation);

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L80

```solidity
file: blob/main/src/tokens/PrincipalToken.sol

236  emit Redeem(owner, receiver, shares);

261  emit Redeem(owner, receiver, shares);

336  emit FeeClaimed(msg.sender, ibts, assets

364  emit YieldUpdated(_user, updatedUserYieldInIBT);

416  emit RatesStoredAtExpiry(ibtRate, ptRate);

422  emit RewardsProxyChange(rewardsProxy, _rewardsProxy);

740  emit YTDeployed(_yt);

767  emit Mint(msg.sender, _ptReceiver, shares);

797  emit Redeem(_owner, _receiver, shares);

857  emit YieldClaimed(msg.sender, msg.sender, yieldInIBT);

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L236 

## [G-04] Use assembly to write address storage values

```solidity
file: blob/main/src/proxy/AMBeacon.sol

79  _implementation = newImplementation;

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L79

```solidity
file: blob/main/src/tokens/PrincipalToken.sol

423  rewardsProxy = _rewardsProxy;

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L423

## [G-05] Use the external Visibility Modifier

Use the external function visibility for gas optimization because the public visibility modifier is equivalent to using the external and internal visibility modifier, meaning both public and external can be called from outside of your contract, which requires more gas.

Remember that of these two visibility modifiers, only the public modifier can be called from other functions inside of your contract.

```solidity
function one() public view returns (string memory){
         return message;
    }

 
    function two() external view returns  (string memory){
         return message;
    }

```

```solidity
file: blob/main/src/proxy/AMBeacon.sol

46  function implementation() public view virtual returns (address) {
        return _implementation;
    }

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol#L46-L48

```solidity
file: blob/main/src/proxy/AMTransparentUpgradeableProxy.sol

94  function _proxyAdmin() internal virtual returns (address) {
        return _admin;
    }

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol#L94-L96

```solidity
file: blob/main/src/tokens/PrincipalToken.sol

508  function underlying() external view override returns (address) {
        return _asset;
    }

513  function maturity() external view override returns (uint256) {
        return expiry;
    }

518  function getDuration() external view override returns (uint256) {
        return duration;
    }

523 function getIBT() external view override returns (address) {
        return ibt;
    }

528 function getYT() external view override returns (address) {
        return yt;
    }

545 function getIBTUnit() external view override returns (uint256) {
        return ibtUnit;
    }

550 function getUnclaimedFeesInIBT() external view override returns (uint256) {
        return unclaimedFeesInIBT;
    }

555 function getTotalFeesInIBT() external view override returns (uint256) {
        return totalFeesInIBT;
    }

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L513


```solidity
file: blob/main/src/tokens/YieldToken.sol

116  function getPT() public view virtual override returns (address) {
        return pt;
    }

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L116-L118

## [G-06] Direct updates to private variables:

Also, when updating private variables, do so directly and use events to log changes, avoiding unnecessary local variables. This approach not only simplifies your code but also contributes to overall gas efficiency in your smart contracts.

```solidity
// Update private variable efficiently and emit an event

   function updatePrivateVar(uint256 newValue) external {

       // Assigning the value directly, no need for a local variable

       myPrivateVar = newValue;

       // Emit an event to log the update

       emit ValueUpdated(newValue);

   }

   // Retrieve the private variable

   function getPrivateVar() external view returns (uint256) {

       // Access the private variable directly

       return myPrivateVar;

   }
```

```solidity
file: blob/main/src/tokens/PrincipalToken.sol

333   uint256 ibts = unclaimedFeesInIBT

343   uint256 _oldIBTRateUser = ibtRateOfUser[_user];

347   uint256 _oldPTRateUser = ptRateOfUser[_user];


```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L333


## [G-07] Use uint8 Can Increase Gas Cost

A smart contract's gas consumption can be higher if developers use items that are less than 32 bytes in size because the Ethereum Virtual Machine can only handle 32 bytes at a time. In order to increase the element's size to the necessary size, the EVM has to perform additional operations. 

```solidity
contract A { uint8 a = 0; }
```
The cost in the above example is 22,150 + 2,000 gas, compared with 7,050 gas when using a type higher than 32 bytes

```solidity
contract A { uint a = 0; // or uint256 }

```
Only when you’re working with storage values is it advantageous to utilize reduced-size parameters because the compiler will compress several elements into one storage slot, combining numerous reads or writes into a single operation.

Smaller-size unsigned integers (https://www.alchemy.com/overviews/solidity-uint), such as uint8, are only more effective when multiple variables can be stored in the same storage space, like in structs(https://www.alchemy.com/overviews/solidity-struct). Uint256 uses less gas than uint8 in loops and other situations.

```solidity
file:  blob/main/src/tokens/PrincipalToken.sol

503    function decimals() public view override(IERC20Metadata, ERC20Upgradeable) returns (uint8) {

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L503

```solidity
file: blob/main/src/tokens/YieldToken.sol

110   returns (uint8)

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol#L110

## [G-08] Avoid Unnecessary Use of Storage

Writing data to storage is one of the most expensive operations in Solidity. Reduce gas costs by avoiding unnecessary writes. For example, move constant values to memory:

```solidity
// Expensive
string public constant NAME = "MyContract"; 

// Cheaper
string memory NAME = "MyContract";
```
Also be careful about storing large pieces of data on-chain. Use alternative patterns like IPFS for larger data.

Storing data costs 20,000 gas versus 3 gas for memory.
Minimize storage by using memory for fixed constants and temporary values.
Store large data like files off-chain (IPFS) and put hash on-chain.

```solidity
file: blob/main/src/proxy/AMProxyAdmin.sol

23  string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol#L23

## [G-09] Change Constant to Immutable for keccak Variables

The use of constant keccak variables results in extra hashing (and so gas). This results in the keccak operation being performed whenever the variable is used, increasing gas costs relative to just storing the output hash. Changing to immutable will only perform hashing on contract deployment which will save gas. You should use immutables until the referenced issues are implemented, then you only pay the gas costs for the computation at deploy time.

```solidity
file: blob/main/src/tokens/PrincipalToken.sol

42  bytes32 private constant ON_FLASH_LOAN = keccak256("ERC3156FlashBorrower.onFlashLoan");

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L42

## [G-10] No need to initialize variable size to 0

0 is the default value for an unsigned integer, thus setting it to 0 again is not required.

```solidity
file: main/src/tokens/PrincipalToken.sol

334  unclaimedFeesInIBT = 0;

904  currentIBTRate = 0;

```
https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol#L334
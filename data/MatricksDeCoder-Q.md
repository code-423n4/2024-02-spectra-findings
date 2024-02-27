#### NC-1 Setter values should check new value is not equal to the old value

This not only wastes gas but also can lead to errors or resuing an old value that should be discarded, additionally may lead to confusion as admin or acess role may think they changed to new value when still using old value entered by mistake, may affect off chain tooling, reporting etc that may not expect old value to be reset by admin or access role

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMBeacon.sol#L63

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMBeacon.sol#L75

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L420

#### NC-2 Function Parameters Use of trailing UnderScore others use prevailing underscore without explanation or consistency why these variables are like that 

address implementation_
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMBeacon.sol#L39
Above all other function params in code dont use trailing underscore 

address _logic and bytes memory _data 
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMTransparentUpgradeableProxy.sol#L77
Only above _logic and _data use leading underscore 

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L120

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L102

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L340

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L369

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L420

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L609

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L680

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/YieldToken.sol#L31

Above are examples of functions with params having leading underscore e.g _user etc but other functions do not follow the ame format. We even notice that PrincipalTokenUtil.sol follows the underscore aspects for all params and all function code e. g

```solidity 
function _computeFlashloanFee(
        uint256 _amount,
        address _registry
    ) internal view returns (uint256) {

``` 
Was this the intention for all the code? 

Above leads to inconsistency, challenges in readability and mantainability of code. 

#### NC-3 Spelling, typos and grammar errors

intializes should be initializes 
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L115

control -> controls 
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/proxy/AMProxyAdmin.sol#L26

flashloan should be capitalized to Flashloan
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/libraries/PrincipalTokenUtil.sol#L183

composed of vs composed by 
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L26

addresss should be  -> address
https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L774


#### LOW-1  Token upgradeability may not be ideal for ERC20 tokens. Yield Token and Principal Token are upgradeable and ERC20PermitUpgradeable 

Generally tokens are more trusted if fixed and users expect certain behaviours for continuity. Having an upgradeable token may carry the following risks

- upgradeablea tokens may be shunned for resuse in DeFi yet the whole purpose DeFi is composability, liquidity, token use across many platforms of protocols
- upgrade risks that come with it like loss of control, uncertainty future functionality, upgrade hacks errors and risks, denial or service and more

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L30

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/YieldToken.sol#L15


#### LOW-2  Lack of validation of time, _duration for maturity. It must be bounded within reasonable ranges (min,max)

It is possible for duration to be set to 0 or extreme value which may be too many years in the future making the contract and protocol not function well 

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L122

https://github.com/code-423n4/2024-02-spectra/blob/383202d0b84985122fe1ba53cfbbb68f18ba3986/src/tokens/PrincipalToken.sol#L132




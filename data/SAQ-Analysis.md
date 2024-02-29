

## Smart Contracts Analysis Report

#### Summary

no | File |
|-|:-|
| [[File-1](#file-1)] | AMBeacon.sol |
| [[File-2](#file-2)] | AMProxyAdmin.sol | 
| [[File-3](#file-3)] | AMTransparentUpgradeableProxy.sol | 
| [[File-4](#file-4)] | PrincipalToken.sol | 
| [[File-5](#file-5)] | YieldToken.sol | 
| [[File-6](#file-6)] | PrincipalTokenUtil.sol | 
| [[File-7](#file-7)] | RayMath.sol | 

## Analyzed Files


### [File-1]  AMBeacon.sol

##### The bellow issues related to the File ( Link )
[Github-Link](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol)

<details>
<summary>see instances</summary>

#### Admin Abuse Risks:

* <b>Access Control Mechanism </b>: The contract uses the `AccessManaged` contract from OpenZeppelin for access control instead of Ownable.This implies that access control is managed through roles assigned by an external contract (`Access Manager`). Admin abuse risks are mitigated through role-based access control.



#### Systemic Risks:

* <b>Implementation Upgradability</b>: The contract is designed to be upgradeable, allowing the authority (admin) to change the implementation contract address. While upgradability can be a useful feature for fixing bugs or adding new functionalities, it introduces the risk of unintended changes or malicious upgrades. The `restricted` modifier is used to limit upgrade functionality to authorized addresses.

#### Technical Risks:

* <b>Code Validation</b>: The contract checks if the new implementation code length is greater than zero before setting it. This is a basic validation to ensure that the provided address corresponds to a valid contract. However, it doesn't guarantee the correctness or security of the new implementation.


#### Integration Risks:

* <b>Dependencies</b>: The contract relies on external dependencies from OpenZeppelin, specifically the `IBeacon` interface and the `AccessManaged` contract. The correctness and security of this contract depend on the reliability and security of these external dependencies.



#### Non-Standard Token Risks:

* <b>Token Independence</b>: This contract doesn't involve any standard token functionality. It is primarily concerned with managing the implementation address used by beacon proxies.


#### Summary

The contract appears to be well-structured and leverages OpenZeppelin contracts for access control and implementation upgradability. Admin abuse risks are mitigated through role-based access control, and technical risks are addressed with basic code validation. However, as with any upgradeable contract, careful consideration should be given to the potential impact of future upgrades, and comprehensive testing is essential to ensure the correctness and security of the implementation. Additionally, dependencies on external contracts should be monitored for updates and security considerations.

</details>


### [File-2] AMProxyAdmin.sol

##### The bellow issues related to the File ( Link )
[Github-Link](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol)


<details>
<summary>see instances</summary>


#### Admin Abuse Risks:

* <b>Access Control Mechanism</b>: The contract utilizes the `AccessManaged` contract from OpenZeppelin for access control instead of Ownable. This suggests that access control is managed through roles assigned by an external contract (`Access Manager`). Admin abuse risks are mitigated through role-based access control.



#### Systemic Risks:

* <b>Proxy Administration</b>: This contract serves as an auxiliary contract meant to be assigned as the admin of a `TransparentUpgradeableProxy`. Administering proxies introduces the risk of unintended upgrades or malicious changes to the proxy's implementation contract. The `restricted` modifier is used to restrict upgrade functionality to authorized addresses.

#### Technical Risks:

* <b>Code Validation</b>: The contract follows a standardized upgrade interface (`UPGRADE_INTERFACE_VERSION`) to signal the supported upgrade methods. The `upgradeAndCall` function is designed to upgrade the proxy to a new implementation and call a function on the new implementation. The requirements are set to ensure the safety of the upgrade process.


#### Integration Risks:

* <b>Dependencies</b>: The contract relies on external dependencies from OpenZeppelin, specifically the `IAMTransparentUpgradeableProxy` interface and the `AccessManaged` contract. The correctness and security of this contract depend on the reliability and security of these external dependencies.


#### Non-Standard Token Risks:

* <b>Token Independence</b>: This contract doesn't involve any standard token functionality. It is primarily concerned with proxy administration and upgradeability.


#### Summary

The contract appears to be well-structured and leverages OpenZeppelin contracts for access control and transparent upgradeability. Admin abuse risks are mitigated through role-based access control, and technical risks are addressed with standardized upgrade interfaces and careful validation in the upgrade functions. However, as with any contract involved in proxy administration, thorough testing is crucial to ensure the correctness and security of the upgrade process. Additionally, dependencies on external contracts should be monitored for updates and security considerations.

</details>

### [File-3] AMTransparentUpgradeableProxy.sol

##### The bellow issues related to the File ( Link )
[Github-Link](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol)


<details>
<summary>see instances</summary>


#### Admin Abuse Risks:

* <b>Access Control Mechanism</b>: The contract utilizes the `AMProxyAdmin` contract for proxy administration, which implements access control through the `AccessManaged`contract. The admin access is restricted and managed by an external contract, mitigating admin abuse risks through proper access controls.



#### Systemic Risks:

* <b>Proxy Structure</b>: This contract follows the transparent proxy pattern, using the `ERC1967Proxy` contract. The admin, responsible for upgrades, is initialized during deployment and cannot be changed afterward, ensuring controlled access to upgrade functionality. The contract has a dedicated admin account for upgrade purposes.

#### Technical Risks:

* <b>Upgrade Mechanism</b>: The contract uses the ERC-1967 upgrade mechanism. The `upgradeToAndCall` function allows for upgrading to a new implementation and calling an initialization function on the new implementation. The `_fallback` function ensures that the admin can only perform upgrades, preventing unintended calls.


#### Integration Risks:

* <b>Dependencies</b>: The contract relies on external dependencies from OpenZeppelin, such as `ERC1967Utils` and the `IAMTransparentUpgradeableProxy` interface. The correctness and security of this contract depend on the reliability and security of these external dependencies.


#### Non-Standard Token Risks:

* <b>Token Independence</b>: This contract is not directly related to token functionality. It focuses on the proxy upgradeability pattern and admin management.


#### Summary

The contract is well-structured and follows best practices for transparent proxy upgradeability. Admin abuse risks are mitigated through access controls enforced by the `AMProxyAdmin` contract. Systemic risks are minimized by using a dedicated admin account and a well-defined upgrade process. The contract utilizes established patterns like the transparent proxy pattern and ERC-1967 for upgrades. However, thorough testing is necessary to ensure the correct and secure functioning of the upgrade mechanism and to validate the interaction with external dependencies.


</details>

### [File-4] PrincipalToken.sol

##### The bellow issues related to the File ( Link )
[Github-Link](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)


<details>
<summary>see instances</summary>



#### Admin Abuse Risks:

* The contract inherits from `AccessManagedUpgradeable`, indicating the presence of an access control mechanism.
 
* The `restricted` modifier is used in various places, suggesting that certain functions are restricted to specific roles.



#### Systemic Risks:

* The contract relies on external dependencies, including OpenZeppelin contracts, interfaces, and other custom libraries. Ensure that these dependencies are secure and well-audited.

* The contract uses time-based conditions (`notExpired`, `afterExpiry`) for certain operations. Be cautious with time-dependent logic, as it can introduce vulnerabilities.

#### Technical Risks:

* Safe math operations are used throughout the contract, reducing the risk of overflow and underflow.

* The contract implements Pausable functionality, allowing the pausing and unpausing of certain operations in case of emergencies.

* ReentrancyGuard is used to protect against reentrancy attacks.

* Flash loan functionality is implemented, enabling users to borrow assets in a single transaction.


#### Integration Risks:

* The contract interacts with external tokens (`ibt`), a registry, and a rewards proxy. Ensure that these external contracts are well-audited and trusted.

* The contract uses a beacon proxy for deploying Yield Tokens (`yt`). Ensure that the beacon proxy mechanism is correctly implemented.


#### Non-Standard Token Risks:

* The contract deploys a Yield Token (`yt`) using a beacon proxy. Ensure that the implementation of the Yield Token is secure and meets the project's requirements.

* The contract interacts with an Interest Bearing Token (`ibt`) and relies on its functionality for various operations.

#### Overall:

* The contract appears to be well-structured and follows best practices such as using safe math and implementing access control.

* Critical operations, such as minting and redeeming, are protected by modifiers and checks.

* Ensure thorough testing of the contract's functionality, especially involving external interactions.

</details>

### [File-5] YieldToken.sol

##### The bellow issues related to the File ( Link )
[Github-Link](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)


<details>
<summary>see instances</summary>



#### Admin Abuse Risks:

* No explicit access control mechanisms are implemented in the contract, making it susceptible to potential admin abuse. Consider adding role-based access control (RBAC) to restrict certain functionalities to authorized users.



#### Systemic Risks:

* The contract seems to be reliant on an external contract (`IPrincipalToken`) for certain operations, such as updating yield and checking maturity. Any issues or vulnerabilities in the external contract may affect the functionality and security of this contract.

#### Technical Risks:

* The contract utilizes OpenZeppelin libraries (`ERC20PermitUpgradeable`, `Math`) which are generally considered secure. However, it's crucial to ensure that the versions used are up-to-date to benefit from any security patches.


#### Integration Risks:

The contract depends on the correctness and security of the external `IPrincipalToken` interface. Any changes or vulnerabilities in that contract may impact the behavior of this contract.


#### Non-Standard Token Risks:

* The contract does not introduce any non-standard token functionalities. It follows the ERC-20 standard and extends from OpenZeppelin's `ERC20PermitUpgradeable`.


#### Additional Notes:

* consider adding explicit comments in the code to explain the rationale behind specific design choices and to enhance code readability for future developers or auditors.

</details>

### [File-6] PrincipalTokenUtil.sol

##### The bellow issues related to the File ( Link )
[Github-Link](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)


<details>
<summary>see instances</summary>



#### Admin Abuse Risks:

* The contract itself does not have explicit access control mechanisms, making it vulnerable to potential admin abuse. Consider implementing role-based access control (RBAC) to restrict certain functionalities to authorized users.



#### Systemic Risks:

* The contract relies on external interfaces, such as `IYieldToken`, `IPrincipalToken`, `IRegistry`, and `IERC4626`. Any issues or vulnerabilities in these external contracts/interfaces may impact the overall functionality and security of this contract. Ensure that these external dependencies are well-audited and trusted.

#### Technical Risks:

* The contract uses OpenZeppelin libraries (`Math`, `RayMath`). These are generally considered secure, but it's important to ensure that the versions used are up-to-date to benefit from any security patches.



#### Integration Risks:

The correct functioning of this contract heavily depends on the correct behavior of external contracts/interfaces (`IYieldToken`, `IPrincipalToken`, `IRegistry`, `IERC4626`). Thoroughly review and test the interactions with these external components, as any discrepancies may introduce vulnerabilities.


#### Non-Standard Token Risks:

* The contract introduces additional functionalities related to tokenization fees, yield fees, and flashloan fees. Ensure that these fee calculations are well-tested and align with the intended logic. It's important to document these fees clearly to avoid misunderstandings.


#### Additional Considerations:

* The library `PrincipalTokenUtil` includes utility functions that are designed to be used in other contracts. Ensure that the usage of these utility functions is well-documented, and the assumptions made in these functions are clearly outlined.

* Ensure that the external calls, especially to external contracts, are properly handled with appropriate error checks and recovery mechanisms.

* Consider adding comments in the code to explain the rationale behind specific design choices and to enhance code readability for future developers or auditors.

</details>


### [File-7] RayMath.sol

##### The bellow issues related to the File ( Link )
[Github-Link](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol)


<details>
<summary>see instances</summary>



#### Admin Abuse Risks:

* The `RayMath` library itself does not pose direct admin abuse risks, as it lacks state variables or functions that allow external parties to manipulate its behavior. However, any contract that utilizes this library could potentially introduce admin abuse risks depending on how the library is used within that contract.



#### Systemic Risks:

* The `RayMath` library is a utility library that provides conversions for and from Ray (27-decimal precision). It does not rely on external dependencies or interfaces, reducing systemic risks.

#### Technical Risks:

* The library uses inline assembly, which can be prone to errors if not implemented carefully. While the functions in the library appear straightforward, there is always a risk associated with low-level assembly operations, especially when dealing with precision and rounding.


#### Integration Risks:

* The library is designed to be used as a utility for other contracts that need precise conversions between Ray and decimal representations. Integration risks would primarily depend on how well this library is utilized in those contracts. Contracts using this library should carefully handle the results of conversion operations, especially if rounding is involved.


#### Non-Standard Token Risks:

* The RayMath library itself does not deal directly with tokens or token standards. It focuses on conversions between Ray and decimal representations. The risks associated with token operations are more relevant to the contracts using this library rather than the library itself.


#### Additional Considerations:

* The library provides two functions (`fromRay`) for converting from Ray to a specified number of decimals, one with rounding and one without. Users should be aware of the implications of rounding when utilizing these functions.

* The library is designed to be straightforward and gas-efficient, but users should ensure that the decimal precision and rounding options align with their specific use cases.

* Given that the library is stateless, there is minimal risk of state manipulation or unexpected behavior due to changing state variables.

* Documentation within the code is clear, providing explanations for the purpose of the library and how to use its functions. This is helpful for developers who integrate or maintain contracts using this library.

</details>


### Time spent:
09 hours
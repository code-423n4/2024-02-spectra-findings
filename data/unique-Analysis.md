# Approach taken in evaluating the codebase

We approached manual code review by looking into each space in the scoped contract and decided to explain some core functional contracts.it is important to note that manual code reviews can be very time-consuming.

The main contracts are:

- src/proxy/AMBeacon.sol
- src/proxy/AMProxyAdmin.sol
- src/proxy/AMTransparentUpgradeableProxy.sol
- src/tokens/PrincipalToken.sol
- src/tokens/YieldToken.sol
- src/libraries/PrincipalTokenUtil.sol
- src/libraries/RayMath.sol

**this section covers licensing, dependencies, contract description, state variables, errors/events, functions, modifiers, and internal workings for each provided Solidity smart contract and library.**

AMBeacon.sol

- **License and Dependencies**:
    - SPDX-License-Identifier is MIT.
    - Utilizes OpenZeppelin Contracts v5.0.0, importing `IBeacon` and `AccessManaged`.
- **Contract Description**:
    - Modified to use `AccessManager` for access control instead of `Ownable`.
    - Facilitates upgrading proxies by changing the implementation contract address.
- **State Variables**:
    - Private `_implementation` variable stores the current implementation address.
- **Errors and Events**:
    - Defines `BeaconInvalidImplementation` error.
    - Emits `Upgraded` event when the implementation is changed.
- **Constructor**:
    - Initializes with the initial implementation and authority.
- **Functions**:
    - `implementation()`: Returns the current implementation address.
    - `upgradeTo(address newImplementation)`: Upgrades the beacon with appropriate authority.
    - `_setImplementation(address newImplementation)`: Sets the implementation contract address, emitting an `Upgraded` event.

AMProxyAdmin.sol

- **License and Dependencies**:
    - SPDX-License-Identifier is MIT.
    - Based on OpenZeppelin Contracts v5.0.0, importing `IAMTransparentUpgradeableProxy` and `AccessManaged`.
- **Contract Description**:
    - Auxiliary contract acting as admin for `TransparentUpgradeableProxy`.
    - Modified for access control using `AccessManager`.
- **State Variables**:
    - Public constant string `UPGRADE_INTERFACE_VERSION` defines the upgrade interface version.
- **Constructor**:
    - Sets the initial authority controlling upgrader roles.
- **Functions**:
    - `upgradeAndCall`: Upgrades a proxy to a new implementation and calls a function on it.
    - Includes checks for contract being admin, empty data if no value sent, and caller role for proxy upgrade.

## AMTransparentUpgradeableProxy.sol

- **Imports and Dependencies**:
    - Utilizes OpenZeppelin Contracts including `ERC1967Utils`, `ERC1967Proxy`, `IERC1967`, and custom `AMProxyAdmin`.
- **Contract Description**:
    - Implements an upgradeable proxy using Transparent Proxy pattern.
    - Ensures controlled upgrades through designated admin.
- **State Variables**:
    - Immutable `_admin` address variable stores the admin of the proxy.
- **Modifiers**:
    - Defines `ProxyDeniedAdminAccess()` error.
- **Functions**:
    - Constructor initializes with logic contract, initial authority, and data.
    - `_proxyAdmin` returns the admin of the proxy.
    - `_fallback` handles internal processing or transparent fallback.
    - `_dispatchUpgradeToAndCall` upgrades implementation using `ERC1967Utils.upgradeToAndCall`.

## PrincipalToken.sol

- **Imports and Dependencies**:
    - Imports various OpenZeppelin contracts/interfaces for ERC20, access control, pausability, etc.
- **Contract Description**:
    - Manages Principal Token (PT) in Spectra protocol, facilitating yield tokenization.
    - Shares composed of PT/YT pairs minted upon deposits.
    - Features for depositing, redeeming, claiming yield, handling fees, etc.
- **State Variables**:
    - Contains various variables for registry, rewards, rates, unclaimed fees, maturity, etc.
- **Modifiers**:
    - Includes modifiers like `notExpired` and `afterExpiry`.
- **Functions**:
    - Functions for deposits, redemptions, fee claims, yield calculations, etc.
- **Internal Functions**:
    - Internal functions for share/IBT conversion, fee handling, rate updates, etc.
- **Events**:
    - Emits events like `Mint`, `Redeem`, `YieldUpdated`, etc., for tracking actions.

## YieldToken.sol

- **Imports and Dependencies**:
    - Imports OpenZeppelin libraries/interfaces for math, ERC20, and protocol interfaces.
- **Contract Description**:
    - Represents Yield Token (YT) in Spectra protocol for tracking yield ownership.
    - Minted alongside PT, interacts with PT for transfers.
- **State Variables**:
    - Stores associated PT address.
- **Constructor and Initializer**:
    - Uses `_disableInitializers()` to prevent reinitialization.
    - `initialize` sets name, symbol, and PT address.
- **Yield Token Functions**:
    - Functions for burning, minting, and overrides for transfers.
- **Decimals and Balance Functions**:
    - Functions for decimals, retrieving PT address, and managing balances.
- **Modifiers and Error Handling**:
    - Modifiers for restricting access and custom error handling.

## PrincipalTokenUtil.sol

- **Imports and Dependencies**:
    - Imports various interfaces/libraries including OpenZeppelin Math and custom ones.
- **Constants**:
    - Defines constants for internal calculations.
- **Conversion Functions**:
    - Functions for asset/share conversions based on rates.
- **Yield Calculation**:
    - Function for computing user yield considering rate changes.
- **Token Decimals**:
    - Function to retrieve token decimals.
- **Fee Calculation**:
    - Functions for calculating tokenization, yield, and flashloan fees based on registry rates.

## RayMath.sol

- **Functions**:
    - Provides functions for converting values between Ray precision and specified decimals.
    - Handles rounding and precision conversions efficiently using inline assembly.

&nbsp;

# Centralization Risk

#### AMBeacon.sol

- **Access Control**: Centralized authority controlling upgrades could lead to single points of failure or manipulation if compromised.
- **Dependency on Access Manager**: Use of `AccessManager` for access control potentially centralizes control over upgrades to a single entity.

#### AMProxyAdmin.sol

- **Admin Role**: Centralized admin role for proxy upgrades could pose risks if controlled by a single entity or compromised.
- **Access Restrictions**: Controlled upgrades enforced through centralized access restrictions might hinder decentralization.

#### AMTransparentUpgradeableProxy.sol

- **Admin Address**: Immutable `_admin` variable designates a single admin for proxy, potentially centralizing control.
- **Upgrade Authorization**: Authorization for upgrades centralized through designated admin, introducing centralization risk.

#### PrincipalToken.sol

- **Authority Over Token**: Authority over token management, including fee handling and yield calculations, centralized within the contract.
- **Control Over Upgrades**: Contract managing tokenization logic might centralize control over protocol upgrades and changes.

#### YieldToken.sol

- **Associated PT Dependency**: Dependency on associated PT introduces centralization risk if the PT contract is controlled by a single entity.
- **Restricted Access Modifiers**: Modifiers restricting access to PT-only functions may centralize control over certain operations.

#### PrincipalTokenUtil.sol

- **Dependency on Registry**: Functions reliant on registry rates centralize control over fee calculations and yield computations if registry controlled centrally.
- **Utility Functions Centralization**: Centralized utility functions for conversions and calculations might centralize protocol operations.

#### RayMath.sol

- **Precision Handling**: Centralized handling of precision conversions could introduce risks if inaccurately implemented or manipulated.

# Codebase Quality:

#### AMBeacon.sol

- **Dependency on External Contracts**: Utilization of OpenZeppelin Contracts v5.0.0 enhances codebase quality through tested and audited code.

#### AMProxyAdmin.sol

- **Utilization of Standard Contracts**: Based on OpenZeppelin Contracts v5.0.0, ensuring reliability and quality through widely-used standards.

#### AMTransparentUpgradeableProxy.sol

- **Transparent Proxy Pattern**: Implementation based on OpenZeppelin Contracts ensures adherence to established patterns and quality standards.

#### PrincipalToken.sol

- **Comprehensive Implementation**: Includes various features for managing tokenization effectively, indicating a well-designed and thought-out contract.

#### YieldToken.sol

- **Standard Interface Implementation**: Adheres to standard ERC20 and protocol interfaces, contributing to codebase quality and interoperability.

#### PrincipalTokenUtil.sol

- **Modular Utility Functions**: Offers modular utility functions for common tasks, enhancing codebase quality and reusability.

#### RayMath.sol

- **Efficient Precision Handling**: Provides efficient precision handling through inline assembly, contributing to codebase quality and performance.

&nbsp;

# Time Spent

29 hours

### Time spent:
29 hours
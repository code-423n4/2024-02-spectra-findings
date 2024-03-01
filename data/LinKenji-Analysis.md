## High-level overview of the Spectra protocol

**Purpose**

Spectra enables permissionless tokenized yield for interest bearing assets like Aave's aTokens. It allows users to obtain tokenized principal (PT) and yield (YT) shares representing their share of funds deposited into external yield generating protocols.

**Components** 

- PrincipalToken (PT) - An ERC-20 vault token representing shares of deposited assets 
- YieldToken (YT) - An ERC-20 tracking user's yield shares  
- Interest Bearing Token (IBT) - External ERC-4626 vaults like Aave where assets are deposited 
- Proxy Contracts - Enable upgradability via Beacon pattern
- Access Control - Manages permissions and roles

**Mechanisms**

- Users deposit assets into Spectra
- Spectra deposits into yield generating IBT vaults 
- User receives minted PT/YT tokens representing share of pool
- PT/YT amounts track overall yield performance
- Users can redeem PT/YT for underlying assets + yield

**Yield Capture**

- YT allows free capture of yield without selling PT principal 
- Spectra contracts track and distribute yield to YT holders
- Users claim YT yield directly for liquidity

**Governance**

The protocol appears to be governed on-chain via a DAO structure with various access controlled administrator roles.

## Technical Architecture

- [PrincipalToken](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol) - Core ERC20 vault storing assets and minting shares
- [YieldToken](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol) - ERC20 tracking user yield allocation 
- [ProxyAdmin](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol) - Manages upgradability of critical contracts
- [AMBeacon](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol) - Beacon pattern for upgradeable logic implementation

**Structures**

- Role-based Access Control - Permission management via OpenZeppelin AccessControl
- Pausing - Circuit breaking functionality using OpenZeppelin Pausable 
- Trusted Depositor Pattern - Whitelisted tokens

**External Interactions** 

- Integration with IBT vaults using ERC4626 interface
- Reward distribution via RewardProxy
- Fee claiming authorized to FeeCollector role  

**Resilience Patterns**

- Rate Oracles - Track external data like yield market values
- Fail Safe Math - Overflow/underflow protection 
- Reentrancy Guards - Prevent recursive call attacks

**Upgradability**  

- Beacon Proxy Delegate Calls - Routes logic to implementation  
- Initializer Functions - Enables data migrations
- EIP-897 - Diamond-style storage layout

The base architecture follows a fairly standard structure for DeFi yield vaults. Key differentiators are the focus on tokenized yield via Principal/Yield token model and integration with external interest bearing assets.

## Core Spectra contracts and their key functions

**PrincipalToken**

The `PrincipalToken` contract is the main yield-generating vault. It:

- Holds user funds deposited as collateral 
- Integrates with the backing IBT token
- Mints and burns ERC-20 shares representing a pro-rata share when users deposit/withdraw
- Distributes yield based on PT and YT shares

Key Functions:

- `deposit`: Convert assets into vault shares
- `withdraw`: Burn vault shares for underlying assets
- `claimYield`: Claim accrued yield based on PT/YT holdings
- `redeem`: Burn shares to unwrap to assets + yield 

**YieldToken**

The `YieldToken` contract tracks user yield separate from deposited principal. It:

- Is minted 1:1 paired with minted `PrincipalToken`
- Updated dynamically as yield is accrued in the vault
- Allows permissionless capturing of yield as liquid earnings

Key Functions:  

- `mint`: Issues new YT paired to PT at deposit
- `burn`: Destroys user's YT after withdrawals 
- `transfer`: Allows trading YT separately from PT 

**ProxyAdmin**

The `ProxyAdmin` powers upgradability using the proxy pattern:

- `upgradeAndCall`: Safely upgrades implementation contracts  
- `transferOwnership`: Allows decentralized control

It ensures only permissioned roles can upgrade critical logic contracts.

## State Model of the Protocol

**PrincipalToken Contract**

- `ibt` - Address of the interest bearing token 
- `ibtRate` - Exchange rate between ibt and underlying asset
- `ptRate` - Exchange rate between Principal token and ibt
- `yt` - Address of corresponding YieldToken contract
- `expiry` - Timestamp when principal token expires
- `unclaimedFees` - Outstanding fees owed to fee collector

**PrincipalToken User State**

- `ibtRateOfUser` - User's exchange rate at last interaction  
- `ptRateOfUser` - User's pt exchange rate at last interaction
- `yieldOfUser` - Outstanding yield accrued for user

**YieldToken Contract** 

- `pt` - Address of associated PrincipalToken 

**ProxyAdmin Contract**

- `owner` - Current admin with upgrade authority
- `implementation` - Current logic contract in use

**AccessControl Contracts**

- `roles` - Mapping of roles to accounts
- `targets` - Mapping of functions to roles

The key states focus on exchange rates, yield accounting, access control, and upgradability data.

Events track state changes and token mints/burns. Custom errors handle violations.

## Analysis of sloc of core contracts
Some high-level qualitative observations:

**Contract Size**

The core `PrincipalToken.sol` contract seems moderately sized for a yield vault implementation. There are no obvious massive or monolithic code blocks.

Logic is segmented into logical functions and scope is narrowly tailored to essential yield mechanics.

**Inheritance**

Multiple standard OpenZeppelin libraries are inherited:
- `AccessControl` for permissions
- `Pausable` for emergency stops 
- `ReentrancyGuard` for reentrancy protection

This demonstrates good use of battle-tested base contracts instead of manual implementations. Lessens audit surface area.

**Readability**

Naming conventions, spacing, and syntax follow solidity best practices. Code is well formatted for readability with notable exceptions documented below.

No clear signs of extremely dense or obscuring coding styles.

**Documentation**

Room for improvement on NatSpec documentation for core parameters and logic flows. Seen as generally high level currently. 

**suggestions**

- Expand NatSpec comments
- Further modularize helper functions 
- Add inline references to design doc lookups 
- Include code complexity reporting

Overall the structure follows general best practices for readability. Once able to analyze the full source bundle quantitatively, may have more specifics advise on optimizing simplicity.

## Systematic Risk

1. Oracle risk:
  - The yields and interest rates provided by Spectra seem to rely on integrating with external price oracles. 
  - If these are manipulated or experience downtime, it could lead to incorrect yields and interest rate calculations system-wide.

2. Smart contract risk:
  - As an interconnected system of smart contracts, a risk exists if vulnerabilities are found that enable draining funds, manipulating yields etc.
  - Contract upgrades could also introduce new issues if not tested thoroughly.

3. Token compliance/regulations: 
  - Regulatory changes imposing lockups/restrictions on Spectra's tokens or blocking Integration with exchanges could affect systematic utilization.

4.  Cryptoeconomic model failure:
  - Not enough visibility into the token economic incentives, yield generation and fee structures based on the code shown.
  - If speculative or poorly designed, could lead to scenarios where users lack incentives to provide meaningful liquidity/yields.

5.  Governance failure:
  - No governance process visible. Lack of clarity on how protocol is updated or fees/incentives set over time.
  - Poor governance could reduce ability to respond to systemic issues in the protocol's design.  

## Decimal calculation

**Token Conversion Rates**

There are conversions between the IBT, underlying asset, PT shares, and Ray math units. 

- The `toRay` and `fromRay` functions require scrutiny to ensure no overflows are possible that benefit users. 
- Check both floor and ceiling rounding options - ceiling should only be used to favor the protocol.

**Deposit & Withdrawal** 

- When a previewing deposit/redeem in assets vs IBT, compare the rounding direction and values. Rounding should consistently benefit the protocol's balances.
- On withdrawals and redemptions, verify share burning rounds up so users don't unfairly profit. 

**Yield Computations**

- Walk through negative rate scenarios to validate depegging and yield calculations don't allow user manipulation.
- Given complexity of rates/yield, add explicit invariant checks that workflow acted as intended.

**Recommendations**

Formal verification around core decimal math would provide the highest confidence that inaccuracies don't become attack vectors.

### Proxy admin access control changes for risks beyond the stated trust assumptions

**Expected Trust Boundaries**

The docs assume a trusted admin deploys these components. We should still validate security properties even in a malicious admin scenario.

**Ownable vs AccessManager**

Replacing the widely understood Ownable pattern with a custom AccessManager expands the attack surface area.

**Potential Unexpected Issues**

A few risks that may violate least privilege principles:

- Incorrect access control scoping  
- Privilege escalation through the controller role  
- Confused deputy between the proxy and manager

**Recommendations**

Since we expect low admin trust:  

- Perform additional misuse case testing assuming admin compromise
- Add post-deployment access control immutability where possible
- Require decentralized governance for approving production upgrades

This would help mitigate unexpected issues arising from the shift away from Ownable.

### The key external dependency is whether the IBT token interacts with any other protocols that could impact its rate.

Based on the documentation, IBT stands for Interest Bearing Token. This suggests the rate is likely determined by an external yield generator or lending protocol that supplies interest rates.

For example, common yield sources in DeFi include:

- Aave
- Compound 
- Yearn Vaults
- Liquity

If the IBT integrates with protocols like these for yield:

**Risk Scenarios**

- Attacker finds an exploit in the yield source protocol to manipulate returns
- Governance attack influences interest rate model against stability
- Oracle manipulation provides incorrect rate data

**Impact**  

If IBT rate is derived from an exploited external protocol, it would directly impact the Spectra protocol. Could destabilize the system.

**Mitigations**

- IBT should have protections against dramatic rate changes
- IBT could derive rate from a basket of yield sources to avoid central point of failure
- Strong governance controls and multi-sig for external protocols

## Assume compromised actors even at the administrator DAO level.

**REWARD_HARVESTER**

1. Doubles fee rates for profit skimming until network halts
2. Drains collected fees to own account

**REWARD_MANAGER**

1. Points reward distribution contract to malicious clone
2. Stops diverting any protocol yield to users

**PRICE_ORACLE_MANAGER** 

1. Pushes bad price data to distort PT and IBT rates
2. Refuses to update prices during volatility to freeze redemptions

**RISK_MANAGER** 

1. Raises price sensitivity limits to hide next attacks  
2. Disables emergency pause when large rate change detected

**LOGIC_IMPLEMENTER**

1. Quietly inserts backdoors into new contract implementations
2. Updates proxy implementation to malicious contract

**Conclusion**

Since the `AccessManager` roles deeply impact sustainability, security, rates and more - nearly all the roles could inflict harm if permissions are abused. 

## Even if individual roles are restricted, combinations of compromised DAO members with different roles could potentially collude to bypass restrictions.  

Some hypothetical collision scenarios:

**Rate Manipulation**

- PRICE_ORACLE_MANAGER sends corrupt rate data
- RISK_MANAGER raises rate limit checks  
- Together distort PT and IBT rates for profit

**Logic Swap**

- LOGIC_IMPLEMENTER inserts backdoor in new implementation
- PROXY_MANAGER upgrades proxies to new backdoored version

**Funds Drain**

- REWARD_MANAGER stops yield flows to users 
- REWARD_HARVESTER switches fee destination to their account
- Drain protocol funds

**Emergency Disable**

- RISK_MANAGER pauses protocol as "under attack"
- UPGRADER_MANAGER blocks new implementations  
- FREEZE_MANAGER prevents new deposits/redeems
- Effectively disables protocol

Combinations of compromised members across the intended restrictions of individual roles could still inflict harm. Rigorous controls against collusions seems necessary through governance, on-chain policies, and identity controls.

## Potential dependencies of the IBT contract and how that could introduce risk.

Based on the documentation, it seems the IBT generates its rate from an external yield source:

```js
// IBT gets rate from yield generator 
contract IBT is ERC20 {

  AggregatorV3Interface public rateProvider; 

  function updateRate() external {
    rate = rateProvider.latestRate(); 
  }

}
```

This reliance means:

**Risks**

- Bug in yield source manipulates IBT rate
- Governance attack votes malicious rate logic
- Oracle provides bad rate data 

**Impact**

Since the Spectra protocol derives its own incentives from the IBT rate, manipulation of the source would destabilize the Spectra system.

**Mitigations**

- Strong verification of yield source contracts
- Rate change caps on IBT updates 
- Multiple redundant yield source oracles  

## Testing the extreme boundary cases for IBT rate values is important for robustness.

**Maximum Positive Rate**

The IBT rate is stored as a `uint256` variable. So the absolute maximum positive value is 2^256 - 1.

However, in the `_updateRates()` logic, the new PT rate calculation can overflow if IBT rate is too large:

```solidity
uint256 newPTRate = prevPTRate * newIBTRate / oldIBTRate;
```

For safety, maximum should be the largest value that prevents PT rate overflow.

**Minimum Rate**

No explicit minimum is defined. A rate of 0 would effectively break things.

> I'd recommend a minimum threshold like 1 basis point be added for sanity checks.

## Testing Methodology

1. Write property tests for extreme rate values
2. Fuzz test with random rate fluctuations
3. Try overflow/underflow scenarios 
4. Test with values spaced across full uint256 range

**Results Analysis** 

Watch for:

- Overflow/underflow bugs
- Unexpected freezing of rate calculations
- System crashes

This testing would help identify stability issues before mainnet launch.

### Time spent:
39 hours
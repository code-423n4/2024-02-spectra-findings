## Overview

The Spectra protocol enables permissionless tokenization of yield from interest bearing assets like Aave and Compound using Principal Tokens (PT) and Yield Tokens (YT).

**Components**

- Principal Token (PT) - An ERC-5095 tokenized vault holding an interest bearing asset 

- Yield Token (YT) - An ERC-20 representing fractionalized ownership of the yield

- Interest Bearing Asset (IBT) - External ERC-4626 asset like Aave, Compound supplying yield 

- Registry - Maintains protocol settings and parameters

**Mechanisms**

- Users deposit assets into PT vaults which mints paired PT + YT tokens

- Interest yield from the external IBT continuously accrues to YT holders

- Upon redemption PT burns, yield in YT can be withdrawn

- Governance roles defined in AccessManager contracts

**Invariants**

- PT and YT have equal supplies 

- Yields accumulate to YT holders

- Users redeem initial PT deposits + earned yield  

Spectra seems to focus on tokenized exposures to interest bearing assets via paired token systems representing principal + yield.

## Contracts Overview:
**Description of the contracts (in scope):**

**Proxy Contracts**
- [AMBeacon](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol) - Defines the contract implementation the proxies use. Modified from OpenZeppelin to use AccessManager for access control.
- [AMProxyAdmin](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol) - Administers upgrades for TransparentUpgradeableProxy. Uses AccessManager. 
- [AMTransparentUpgradeableProxy](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol) - Proxy implementation for upgradeable Spectra contracts. References AMProxyAdmin.

**Core Protocol**  
- [PrincipalToken](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol) - Main Spectra vault contract representing user capital deposits. Implements ERC20, ERC2612, ERC5095. Holds interest bearing assets.
- [YieldToken](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol) - ERC20 token representing fractional ownership of interest yield. Paired with PrincipalToken.
- [PrincipalTokenUtil](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol) - Library with utility functions for yield and fee calculations.

**Supporting**
- [RayMath](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol) - Library for fixed-point 27 decimal ray math conversions.

The proxies facilitate upgradeability using the AccessManager for permissioning. PrincipalToken and YieldToken are the core interest yield tokenization contracts. Utility libraries provide supporting math and calculations.

## Architecture view (Diagram)

```solidity
              +-------------------------+
              |                         |
              |       Governance        |
              |                         |
              +---------┬---------------+
                        │
                        │
     +------------+    Set Fees    +--------------------------+
     |            |--------------->|         Registry         |
     |            |                +--------------------------+
     |            |
     |            |      
 +----v-----------v---------------------------+
 |                     Spectra               |  
 | +-------------+ +-------------+ +------+ |
 | |PrincipalToken| |YieldToken   | |Proxy | |
 | +-------------+ +-------------+ |Admin| |
 |  ^            ^ ^             | +------+ |
 |  |            | |             |          |
 +-┼------------+ |             |          |
   |       +-----+             |          | 
   |       |                   |          |
+--v-------v---+        +------v----------+
|            |        |                  |
| Interest   |        |   External       |
| Bearing    | <----> |   Incentives     |
| Asset      |   IBT  |    Contract      |
| (ERC4626)  | <------> (Rewards)         |
|            |        |                  |
+------------|--------+------------------+

```

**Overview:**
- Governance entities set protocol fees and administer roles 
- PrincipalToken vaults hold interest bearing IBT assets
- YieldTokens entitle holders to fractionalized interest yield  
- Proxy/Admin contracts handle upgradability
- External incentive contract provides additional yield

## User Flow

```solidity
User                                         Spectra Protocol

                                          +------------------+
             Deposit assets              |                  |  
                                          | PrincipalToken + |
                                          |   YieldToken    |
                                          +---------┬------+
                                                    │
                                                    │
 +-------+   Approve & Deposit       +------------+--------------------+
 |       |-------------------------->|                                 |
 |       |   Tokens Minted           |   Mint paired PT + YT tokens    |
 | User  |<--------------------------|                                 |  
 |       |                            +------------+--------------------+
 |       |                                         ▲
 +-------+            Earn yield                      │
                                                       │
                        ▲◄-----------------------------┘
                        │
             Withdraw yield
                        │
                        ▲

 +-------+   Redeem Tokens              +------------+--------------------+
 |       |-------------------------->|                                 |  
 |       |     Burn Tokens            |   Burn PT and withdraw yield   |
 | User  |   Assets Returned          |                                 |
 |       |<--------------------------|                                 |
 +-------+                            +------------+--------------------+
```

**Overview** 
1. User approves and deposits assets into PrincipalToken 
2. PrincipalToken mints paired PT governance tokens + YT yield tokens
3. Over time, interest yield accrues to YT token holders
4. User burns PT and withdraws original deposit + earned yield

## Admin Flow

```solidity
DAO / Admins                        Spectra Protocol

                                            +---------------+
                                   Set      |               |
                                            | AccessManager |
+------+               +------+             |               |
|Admin |               |DAO   |             +-------+-------+
+------+               +------+                     ▲
                                                      │ 
     +-----------+             +------+                |
     |           |             |      |<---------------+
     |           |   Upgrade   |Proxy |
     |           v             |Admin |         +------+
     |      +------------+      |  Beacon|         |
     |      | New Logic | <----┼---+   +|---------►|Logic |
     |      +------------+      |      |         |Contract|
     +-----------+             +------+         +-------+

 +------+             +------+
 |Admin |             |DAO   |         
 +------+             +------+
     +                     
     |
     |  Pause /
     |  Unpause
     |
     v
+------------+
|PrincipalToken| 
+------------+
```

**Overview**  
- Admins upgrade logic contracts via Proxy Admin and Beacon
- DAO can pause activity in system emergencies 
- AccessManager contract manages permissions

## Core contract Flow

```solidity
        Spectra Protocol        

           +--------------------------------+
           |         PrincipalToken         |
           +------------------+-------------+
                              |
                              |
           +------------------v-------------+  
           |                  |             |
           |   +----------+   |          +--v--+
           |   |          |   |          |      |
      +---->   |  User    +--------->+   |Asset |
      |    |   |          |Deposit|   |   |(ERC4626)|
      |    |   +------+---+---+---+   |   |      |
      +----------|             |      +-->+      |
                 |    Accrue   |            |      |
                 |     Yield   +<-----------+      |
                 |             |                   |
   +-------------+------+      |                   |
   |       YieldToken      |      |                   |
   +-------------+------+      |                   |
                 ^             |                   |
                 |  Withdraw   |                   |
                 |    Yield    |                   |
                 +-------------+                   |
                                                    |
                                                    |
                   +---------------------------+    |
                   |          Registry         |    |
                   +---------------------------+----+

```

**Overview**
- Users deposit assets into the `PrincipalToken` vault 
- `PrincipalToken` deposits into external interest bearing asset
- Yield accumulates to `YieldToken` holders
- Users withdraw earned yield by burning `PrincipalTokens`
- Registry maintains protocol settings

## Protocol Roles of the spectra.

**Admin Role**
- Highest privilege role in Access Manager 
- Can upgrade contracts via Proxy Admin
- Performs emergency pausing 

**Upgrader Role**
- Can set new contract implementations
- Manages seamless upgradability

**Pauser Role** 
- Can pause protocol in case of emergencies
- Often held by DAO multisig

**Fee Setter Role**
- Sets protocol fee percentages 
- Adjusts incentives and revenue splits  

**Registry Role**
- Administers core protocol registry
- Manages parameters and contract addresses

**Rewards Harvester Role** 
- Claims accrued external incentives 
- Redistributes to Spectra users

Having well-defined protocol roles with minimal privilege promotes sound governance. Role permissions and trust models should be critically examined.

## Contract Analysis

**PrincipalToken**

The PrincipalToken is the core vault contract that holds users' deposited assets.

- Implements the critical user flows of deposits, withdrawals, redeems.
- Manages the external interest bearing asset interactions. 
- Accumulates interest yield to YieldToken holders.
- Contains access control hooks and governance capabilities.
- Upgradable via proxies to enable protocol iterations.
- Usage of AccessManager for permissioning instead of Ownable provides flexibility.
- Key focus area during audits around accuracy of all asset calculations and conversions. 

**YieldToken** 

The `YieldToken` represents fractionalized ownership of interest yield.

- Paired 1:1 with `PrincipalToken` shares.
- Implements permit to enable token approvals.
- Contains deposit and yield accrual hooks from `PrincipalToken`.
- Critical check on supply synchronization with `PrincipalToken`.
- Upgradable along with PT via proxies.

**Proxy Admin**

The ProxyAdmin manages upgradeability.

- Modified from OpenZeppelin's admin to use AccessManager instead of Ownable.  
- Upgrade role permissioned to providing governance oversight. 
- Storage collision analysis important during upgrades.

## Codebase quality

**Readability**

- Code is well formatted and easy to parse
- Descriptive naming and terminology
- Detailed natspec comments explain intention  

**Security**

- Leverages battle-tested OpenZeppelin contracts
- Use of AccessManager instead of Ownable is positive
- Follows checks-effects-interactions pattern
- Scope for improvement validating inputs/outputs

**Maintenance** 

- Logic separation into multiple contracts is good
- Upgradeability will enable iterations
- Light interface coupling between contracts
- Lack of comprehensive unit test coverage

**Best Practices**

- Standards like ERC20 and ERC2612 implemented 
- Usage of design patterns like proxies
- No complex inheritance hierarchies
- Scope to improve events/errors for tracing

Overall the codebase demonstrates sound engineering, but has areas that can mature particularly around testing, events, and validation. The focus on readability, security, and governance positions it well for community ownership.

## Centralization Risks

**Governance**

- High privilege roles like admin or pauser could be centralized
- Lack of decentralization and trust if controlled by single entity
- Upgrades could be forced without sufficient multi-sig

**Asset Custody** 

- PrincipalToken contracts custody majority of assets
- Large TVL accumulation increases risks 
- Could benefit from multisig and distributor models

**Interest Rate Oracles**

- Dependency on external data feeds for yield rate info  
- Could manipulate rates via centralized oracle access

**Protocol Ownership**

- Intellectual property largely held by founding team  
- Limits community participation and development
- Token distributions could overly reward small group

**Mitigations**

- Decentralize roles across DAO participants
- Pursue partial token decentralization via liquidity bootstrapping
- Develop open-source contribution guidelines  
- Adopt community governance processes

There is potential for the emergency `PAUSER_ROLE` to be abused for malicious purposes if adequate precautions are not taken. Some ways this could occur:

**Profit from Predictable Pause**

If a malicious actor gains access to the `PAUSER_ROLE`, they could pause the protocol right before major expected deposits or withdrawals. This would allow them to front-run the transactions for profit once unpaused.

The predictability of pauses can benefit the attacker.

**Discourage Participation** 

Similarly, the power to pause could be used to discourage protocol participation by users if they expect functionality to halt randomly or frequently. Slowing adoption aids attackers.

**Brute Force Upgrades**

An attacker might try repeatedly pausing and then manually upgrading protocols to introduce bugs. With enough tries, they may succeed.

**Mitigations** 

- Use a timelock on pausing ability
- Ensure pauser cannot profit as a user
- Implement a pause logging system for accountability


## Analysis of potential access control issues in the Spectra protocol:

Improper access control could allow unauthorized parties to extract funds, manipulate rates to their benefit, or halt protocol operations. This breaks the trust model and intended permissions, leading to loss of funds for users or the protocol.

**Root Cause Analysis:**

The root cause lies in how role capabilities are defined and assigned in the AccessManager contract used for access control. 

**Failure Scenario:**

An example failure scenario would be:

1. The UPGRADER_ROLE is granted to an external attacker address during initialization. 

2. After deployment, the attacker calls `upgradeTo()` in AMBeacon.sol as they have the UPGRADER_ROLE. 

3. They set the implementation to a malicious contract that simply extracts all IBT funds from PrincipalToken.sol.

Now all funds have been stolen due to improper role assignment.

**Specific Code:**

In AMBeacon.sol:

```solidity
function upgradeTo(address newImplementation) public virtual restricted {
  _setImplementation(newImplementation);
}
```

The `restricted` modifier checks that `msg.sender` has a role that is allowed to call this function. But if UPGRADER_ROLE is granted incorrectly, an attacker can call this.

In AMProxyAdmin.sol

```solidity
function upgradeAndCall(IAMTransparentUpgradeableProxy proxy, address implementation, bytes memory data)
  public payable virtual restricted 
{
  proxy.upgradeToAndCall{value: msg.value}(implementation, data);
}
```

Same issue, the `restricted` modifier depends on correct role assignments.

**Mitigations:**

- Carefully review each role and its capabilities compared to the intended permissions.

- Validate role assignments in initializers and that roles can't be manipulated post-deployment. 

- Consider time-locking changes to critical role assignments like `UPGRADER_ROLE`.

- Monitor role changes and upgrades to quickly detect unauthorized modifications.

## Evaluation of the Spectra governance model and access control risks

**Overly Centralized Control**

The AccessManager admin role has tremendous power. It can upgrade contracts, update implementations, and extract assets. This places immense trust in the admin.

Making one address the gatekeeper for all protocol changes centralizes too much control in one place. Compromise of the admin can compromise the entire system.

**Lack of Decentralized Checks & Balances** 

There are no decentralized checks to admin powers. For example, a DAO vote to approve upgrades or pause the system.

A major protocol decision like upgrading core logic rests solely in the hands of the admin. This increases risks from bribes, hacks, or malicious admins.

Ideally there would be a separation of powers - admin handles day to day tasks but major upgrades need DAO vote.

**Potential for Governance Manipulation**

If the admin address is a governance contract like a `MultiSig` or `Timelock`, this still represents a central point of failure.

If the keys controlling the admin contract were compromised, it risks complete governance control. Additional delegated governance contracts also multiply the attack surface.

There should be redundant and distributed authority using multisig thresholds across ecosystem stakeholders to mitigate governance manipulation risks.

> Overall the current governance is too centralized with a single admin role that can unilaterally control Spectra. There are also no checks against governance manipulation by delegated contracts. Both these need to evolve to sustain security over the long term.

## Risks related to upgrading contracts and proxies in Spectra

**Malicious Contract Upgrades**

If the `ProxyAdmin` and `Beacon` were compromised, an attacker could upgrade critical contracts like `PrincipalToken` to drain assets.

For example, they could set a malicious implemenation that simply transfers all assets in `redeem()` to the attacker address. Or manipulate rates to their benefit. Users could lose funds.

This presents a major risk as users believe they interact with the original audited contracts when using the Proxy. A malicious upgrade breaks that trust.

**Freezing Funds via Pauser Role** 

The Pauser Role can arbitrarily pause deposits and redeems by calling `pause()`.

A compromised Pauser role could freeze all protocol funds by maliciously pausing contract functions indefinitely. This denies users access to their own assets.

**Impersonation by Compromised Upgraders**

If an Upgrader address is compromised, the attacker can mimic protocol-side authorized contract upgrades. 

For example, push a fake migration contract to extract funds from users. Upgraders hold "code is law" powers to alter protocol without checks.

**Mitigations**

- Timelock/MultiSig with decentralized holders to make Roles more resilient 

- DAO vote approvals required for any Pausing or Upgrades

- Explicit public upgrade announcements from multi-channel sources

There needs to be an extensive process around upgrades and pauses to mitigate the immense powers the roles possess today. Securityaudits on any new implementations can add further reassurance.

## Privilege escalation risk

**Vulnerability**

The AMProxyAdmin relies on the AccessManager role checks in the `restricted` modifier to control access. This could be bypassed if the AccessManager itself is compromised.

**Root cause**

The AMProxyAdmin trusts the AccessManager to properly manage permissions after deployment. Subverting this manager would break that trust.

**Attack vector** 

If an attacker gained the `ROLE_CONTROLLER` role in the AccessManager, they could add other malicious actors to privileged roles. 

**Scenario**

1. Attacker socially engineers delegate call creds from CouncilMember with ROLE_CONTROLLER 
2. Calls `AccessManager.grantRole(DEFAULT_ADMIN_ROLE, attackerAddress)`
3. Now attacker has DEFAULT_ADMIN_ROLE despite invalid permissions
4. Bypasses AMProxyAdmin `restricted` modifier check  
5. Calls `upgradeAndCall` to steal funds

**Code**

AccessManager.sol:
```solidity
function grantRole(bytes32 roleId, address account) public virtual authorized(ROLE_CONTROLLER)
```

AMProxyAdmin.sol:
```solidity 
function upgradeAndCall(
  IAMTransparentUpgradeableProxy proxy,
  address implementation,
  bytes calldata data
) public payable virtual restricted 
```

**Impact**

An attacker could upgrade proxy logic to drain user funds. Severity depends on compromised proxy purpose.

Requires compromise of a privileged AccessManager role. Estimated as unlikely but has high impact.

## Systematic Risk

**Macro Conditions**

- Decline in global interest rates diminishes yield opportunities 
- Contraction of liquidity pools leaves less assets available
- Regulatory shifts restricting certain yield generation activities

**Technical Risks**

- Bugs or exploits in external dependencies like Aave/Compound
- Oracle failures provide manipulated yield data  
- Network congestion losses due to transaction frontrunning   

**Financial Risks**

- Interest rate volatility and debt cycles stress yields
- Runs trigger mass PrincipalToken withdrawals 
- Mismatched asset durations and liquidity crunches  

**Governance Risks**

- Poor parameter selections like fee levels degrade usage   
- Upgrade mistakes or unnecessary complex additions
- Stagnant improvements without active community 

**Mitigants** 

- Develop risk assessment frameworks for market conditions
- Establish technical control rigor and monitoring 
- Implement diversification, guarantees, and insurance backstops
- Incentivize community participation in mature governance

Analyzing protocol risks across various axes - technical, financial, organizational - provides stability.

## Architectural Improvement

**Decentralize Control**

- Spread privileged roles across decentralized parties via multi-sig
- Tokenize protocol governance to align incentives
- Limit centralized custody risks with wrapped tokens

** Bolster Extensibility**

- Build modular and configurable architecture
- Abstract core logic into libraries and interfaces
- Support multiple collateral and yield asset types

**Enhance Upgradeability** 

- Utilize proxy pattern for seamless iterations
- Architect clean storage layouts to prevent collisions
- Introduce feature flags for backwards compatibility

**Strengthen Security**

- Adopt formal verification for provable properties  
- Integrate alerting for monitoring and response 
- Validate core invariant preservation across upgrades

**Improve Performance**

- Leverage lazy evaluation to optimize for gas costs
- Parameterize limits to balance scalability vs safety  
- Introduce data caching layers to reduce chain reads

**Refine Incentives**

- Build sustainable user growth models 
- Design protocol-owned value capture mechanisms  
- Construct positive feedback loops and network effects

The focus should be enhancing decentralization, extensibility, security and sustainability of the system over time.


### Time spent:
27 hours
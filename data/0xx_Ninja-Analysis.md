## Overview
Spectra provides a decentralized marketplace for managing interest rate risk and exploring interest rate-based investment. Spectra operates on a permissionless basis, meaning anyone can participate without needing approval.

## What it does?

**Hedge against interest rate volatility:** Spectra allows users to lock in fixed interest rates for their yield-generating DeFi positions. This protects them from fluctuations in interest rates.

**Earn fixed returns:** Users can design strategies to earn a predetermined fixed return on their investments.

**Trade interest rates** Spectra facilitates trading interest rate derivatives, enabling arbitrage opportunities and speculation on future interest rate movements.

**Boost yield:** Liquidity providers can earn additional rewards on top of their existing yield-generating positions.

## Key Component

**Liquidity Vaults:** They act as pools of deposited funds that users can interact with by having shares to earn interest and trading fees.

**Fixed Interest Rate Products:** They are modules that lock in a fixed interest rate for a specific duration until the the expiry 

**Interest Rate Swaps:** These allow users to exchange floating interest rates for fixed rates or vice versa.

## Centralization Risk

This could occur when a single entity `AccessManaged` holds significant control over a system. H

**Admin Privileges:** If a single entity controls the access control functionality (OpenZeppelin AccessControlUpgradeable), they could potentially manipulate the system for personal gain or restrict user actions.

**Oracle Dependence:** The contract might rely on an oracle to obtain external data (e.g., asset prices). If the oracle is compromised or malfunctions, it could lead to inaccurate calculations and affect user redeem.

## Systemic Risk
**Governance distribution in yield:**  yield rewards after expiry are set by governance. Thus governance is responsible to validate provided values as it possible to break intended work with incorrect input.


### Time spent:
5 hours
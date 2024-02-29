[Preface](#1-preface)

[Review process](#2-review-process)

[Architectural Recommendations](#3-architectural-recommendations)

[Call trace diagrams](#4-call-trace-diagrams)

[Weak spots and mitigations](#5-weak-spots-and-mitigations)

[Conclusion](#6-conclusion)

***

## **1. Preface**
This analysis report has been approached with the following key points and goals in mind:

 - Devoid of redundant documentation the protocol is familiar with.
 - Engineered towards provision of valuable insights and edge case issues the protocol left unnoticed.
 - Simple to grasp call-trace diagrams for first-time users who stumble upon this report in the future.

 ## **2. Review process**
 
 **D1-2:**
 - Understanding of Spectra's concept
 - Mapping out security and attack areas relating to shares, proxy upgrades and role mis-use.
 - Keeping track of the outlined areas with notes on contracts in-scope

**D2-4:**
 - Brainstorm possible reachability of undesired states
 - Test the areas identified
 - Further review of contract-level mitigations

**D4-7:**
 - Apply mitigations
 - Draft & submit report


 ## **3. Architectural Recommendations**

 ### Testing suite:

 **Good coverage; could be much better** - The Spectra team has ensured the codebase is rigoriously tested. There exists a plethora of unit tests and some stateless fuzz tests within the test suite bringing the coverage up to 90%. The team has tested the contracts in-scope with various params in a separate test contract. This technique covers testing for expected inputs/outputs as well as separate test contracts for unexpected inputs and fail-safe end results for those unexpected inputs. Such in-depth suite of unit & stateless fuzzing tests ensure a pretty good coverage for the core functions call traces. The codebase would be even further bullet-proofed with some invariant tests.

 ### What is unique within Spectra?
 - Tokenized yield: Users when depositing an `IBT` (interest bearing token) e.g `aUSDC` into Spectra get the underlying asset they supplied for the IBT in the corresponding protocol as well as the promised yield by that protocol. This concept is what we refer to as the `zero-day yield`. Bob deposits 1000 `USDC` into Aave 1st January 2024. He is entitled to 30 `USDC` at the end of the year on a 3% APY. It's 2 days into the deposit transaction on Aave and Bob intends to utilize his 1000 `USDC` but withdrawing from Aave means forfeiting his 30 `USDC` promised yield. Bob decides to deposit the `aUSDC` share token Aave issued Bob as a representation of his 1000 `USDC` into Spectra. Spectra gives Bob an opportunity to withdraw 1000 `USDC` and close to 30 `USDC`. In essence, Spectra has taken over the burden of holding 1000 `aUSDC` for 365 days. This is the `zero-day yield` concept.

 - Principal and Yield tokens are transferrable: Generally, you would expect the tokenized PT and YT tokens to be non-transferrable between addresses because of risk attached to redeeming 2x for 1. Spectra has this issue covered because they never actually redeem such IBT token yields from their own balance - each balance of IBT or underlying asset is provided by users, hence redeems are facilitated using the user deposited balances in the IBT protocols e.g Aave `USDC` Lending pool.

 - Liquidity provision with tokenized yield positions: Users can use their tokenized Principal Token for providing liquidity on Curve Pools. This creates an extra profit route for users on top of the yield they already tokenized.

 ### Comparison between Spectra and what was previously out there
 
| Feature  | Spectra  | Tempus  |
|---|---|---|
| Goal  | Permissionless interest rate derivatives protocol on Ethereum | Pioneer the transition of Tradfi interest rate derivatives on the Ethereum blockchain |
| Tokens  | Unwraps IBT into it's underlying form and yield form to be redeemed whenever  | Unwraps YBT into a principal token and yield token to be traded between users   |
| Benefits to users  | Party A gains the risk protection of a fixed rate. Party B gains possibility of profit for anticipated yield interest rate increase  | Party A & B trade the PT and YT on the TempusAMM. A set of smart contracts facilitating price speculation for YT and PT swaps. Generally, this created a fragmented liquidity problem within the Tempus protocol.  |
| Risk  | Underlying asset is transferred to party B post trade of yield before or post maturity (e.g Bob takes George's YT for close to 30 USDC)  | The asset prices were basically determined by the AMM in part but also the Balancer Pools they're tied to. Underlying risk still remained in the fact that the AMM price was adverse from the real market value of the IBT i.e underlying asset plus profit  |


## **4. Call trace diagrams**
Entry point:
![Entry](https://rexjoseph.github.io/images/entry-point-spectra.png)

Exit point:
![Entry](https://rexjoseph.github.io/images/exit-point-spectra.png)


## **5. Weak spots and mitigations**
- **Any amount can be tokenized** Spectra doesn't actually hold the `IBT` token users deposit. Another user can buy the yield from the user looking to tokenize `aUSDC`. What happens when you have too little `shares` that are almost always worthless to take on in the Ethereum network because of the fees? They end up not being taken on by the other users looking to make a profit from a perceived interest rate increase in the future. There's no lower bound for how much can be tokenized minimum. We recommend for the Spectra team to enforce a minimum yield tokenization.

- **Issue with the zero-day yield:** The protocol can enter a state of influx deposits that outweigh bids. For example, the Spectra protocol doesn't have anything to do with the user's deposited `aUSDC`, the user maintains full control of the asset's deposit and withdrawal. Should the user intend to sell their yield, another user can agree to take on the position in hopes of making a profit. This sets up a scenario where user A keeps bringing in `aUSDC` they've minted yesterday from Aave hoping to get a quick bidder to sell the tokenized yield to, therefore creating a dump facilitating mechanism. This issue existed within the Tempus AMM with fragmented liquidity. There's no easy fix for this issue but enforcing some limits on transaction sizes as we suggested up above is sufficient to mitigate this risk.

## **6. Conclusion**
In conclusion, the Spectra team is building a protocol not yet available on the Ethereum blockchain with it's rich sets of unique features most importantly, the yield tokenization and liquidity provision feature. Tempus was participating in this realm but has been sunsetted. We expect the Spectra protocol will gain massive adoption in the near short term post launch.

### Time spent:
10 hours
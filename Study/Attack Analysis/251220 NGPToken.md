> Study Materials: https://github.com/TangCYxy/Shares/tree/main/251009%20NGPToken%E6%94%BB%E5%87%BB%E5%88%86%E6%9E%900918 + https://www.bilibili.com/video/BV1Zq4BzREgS/ **PS: Highly recommended!**

# Learning Objectives

1.  **Analysis of the Approach** — Understand the core attack vector. ✅
2.  **Learn by Q&A** — Study unfamiliar concepts as you encounter them. ✅
3.  **Simulate the Attack with Tenderly (PoC + Exp)** — Reinforce understanding through hands-on practice. ❌
4.  **Understand the Fixes**

# Analysis of the Approach

**Tabs**
1.  #Feature-Type/Whitelist-Mechanism
2.  #Audit-Methodology/Flash-Loan-Attack
3.  #Feature-Type/SwapFees
4.  #Project-Type/DEX/AMM

**Attack Goal:** Drain all USDT from the NGP-USDT liquidity pool.

**Attack Process:**
1.  Borrow a massive amount of USDT (211.54 million) via flash loans. Use a whitelisted address (`0xdead`) to bypass purchase limits and buy a huge quantity of NGP, drastically reducing the NGP amount in the pool.
    *   **Vulnerability-1:** Transactions from whitelisted addresses bypass security restrictions (e.g., per-transaction buy limits).
2.  Through price manipulation, sell pre-prepared NGP (1.36 million), ultimately draining all USDT (approx. 2.21 million) from the NGP-USDT pool.
    *   **Vulnerability-2:** An issue with the fee deduction logic when selling NGP further reduced the pool's NGP amount.
    *   **Vulnerability-3:** `sync()` was called prematurely, breaking the AMM's constant product invariant (`K`), causing a massive price calculation error that allowed draining the pool's USDT.

**Transaction Visualization:**
1.  https://dashboard.tenderly.co/tx/0xc2066e0dff1a8a042057387d7356ad7ced76ab90904baa1e0b5ecbc2434df8e1
2.  https://app.blocksec.com/explorer/tx/bsc/0xc2066e0dff1a8a042057387d7356ad7ced76ab90904baa1e0b5ecbc2434df8e1

# Learn by Q&A

## What is a flash loan? ✅

A flash loan is a loan that must be borrowed and repaid within a single transaction. Users can borrow any amount of assets from a protocol, but must repay the principal plus fees before the transaction ends, otherwise the entire transaction is reverted.

**In this case, the attacker borrowed USDT directly or indirectly via:**
*   **Direct USDT Borrowing:**
    *   From the Moolah protocol.
    *   From PancakeSwap V2 and V3 pools.
*   **Indirect USDT Borrowing:**
    *   Borrowed BTCB from Moolah, then swapped it for USDT via the Venus protocol.

**The first attack transaction (which was reverted) involved the following complete borrowing process:**

1.  `MoolahProxy(Flashloan).flashLoan BTC 2689687389580623872131 => Swap USDT 79605109862237851524989796`  <img width="2728" height="952" alt="Image" src="https://github.com/user-attachments/assets/fa4752e2-119d-4069-890f-01a666ef362a" /> (Swapped 2,689 borrowed BTC for 79.6 million USDT)
2.  `MoolahProxy(Flashloan).flashLoan => USDT 7766254162777152918131392`  <img width="2752" height="232" alt="Image" src="https://github.com/user-attachments/assets/d4cfe1f0-ba36-489d-ac4b-47b642ab29da" /> (Directly borrowed 7.76 million USDT from Moolah)
3.  `PancakeV3Pool(USDT-USDC).flash => 26205228734480736008912298` <img width="2664" height="598" alt="Image" src="https://github.com/user-attachments/assets/12a77da3-37b7-4085-88eb-a96a855763ae" /> (Borrowed 26.2 million USDT from Pancake V3 USDT-USDC pool)
4.  `PancakeV3Pool(USDT-WBNB).flash => 4897633199596358035431033`  <img width="2476" height="430" alt="Image" src="https://github.com/user-attachments/assets/73093b0c-2576-4f8f-9f04-4b2cfdea4d3c" /> (Borrowed 4.89 million USDT from Pancake V3 USDT-WBNB pool)
5.  `PancakeV3Pool(USDT-USDC).flash => 14715492807194643438826837` <img width="2328" height="504" alt="Image" src="https://github.com/user-attachments/assets/2cd354e3-7050-4d70-b45f-e81cbbd55a5f" /> (Borrowed another 14.7 million USDT from Pancake V3 USDT-USDC pool)
6.  `PancakePairV2(USDT-ARK).swap or PancakePairV2(USDT-WBNB).swap => 999999999999999999` <img width="2626" height="848" alt="Image" src="https://github.com/user-attachments/assets/fa1eebd4-ef81-465d-8ac6-35665f337092" /> (Borrowed 1 USDT each from several Pancake V2 pools to minimize own capital usage)
7.  `USDT.balanceOf => 133189721766286741926291353` **Thus, the first transaction borrowed ~133 million USDT total.**  <img width="2748" height="1190" alt="Image" src="https://github.com/user-attachments/assets/1730531e-35c6-4c1d-9240-4c97ab2e017a" />
8.  Execute Swap, converting 133 million USDT into NGP. **However, 750k NGP remained in the pool, not meeting the attack's expected conditions, so the transaction was reverted.**  <img width="2648" height="640" alt="Image" src="https://github.com/user-attachments/assets/bda2f189-8651-41a4-a1e7-a8b1c8772975" />

## Why did the attacker initiate two attack transactions? ✅

First transaction (reverted), second transaction (successful).  <img width="2294" height="162" alt="Image" src="https://github.com/user-attachments/assets/48c20d6d-4f43-42e9-b851-189533d95770" />

The first transaction was an attempt to maximize profit, but was actively reverted as the gain didn't meet expectations. The second transaction followed the same logic.

**The second transaction borrowed 211 million USDT in total.** <img width="2764" height="664" alt="Image" src="https://github.com/user-attachments/assets/a06427ae-ddf2-4afd-87e2-80e201a6c6c4" />

In the second transaction, the attacker executed these key steps:
1.  Step 4: Swapped the borrowed 211 million USDT for NGP, drastically reducing the pool's NGP amount. Only 470k NGP remained, meeting attack conditions.  <img width="2728" height="584" alt="Image" src="https://github.com/user-attachments/assets/e72cbe87-9dde-41be-8d57-5e9de52663c7" />
2.  Step 7: Sold pre-held 1.36 million NGP, draining 21.375 million USDT from the pool.  <img width="2780" height="792" alt="Image" src="https://github.com/user-attachments/assets/170986c7-22d6-444f-adba-199915ee6b8e" />

**After repaying all flash loans, the attacker profited ~2 million USDT.** <img width="2764" height="508" alt="Image" src="https://github.com/user-attachments/assets/76319902-c6d0-4fd5-9765-898ce5cd32e6" />

## Why was the VictimPair(USDT-NGP) liquidity pool drained? ✅

The NGP-USDT liquidity pool (`VictimPair(USDT-NGP) => 0x20cAb54946D070De7cc7228b62f213Fccf3ffb1E`), maintained by the NGP team, allowed users to swap NGP or USDT.

**Business Logic Flow:**
1.  In the first `swapExactTokensForTokensSupportingFeeOnTransferTokens` call, transfer the borrowed 211 million USDT into `VictimPair(USDT-NGP)`. <img width="2704" height="784" alt="Image" src="https://github.com/user-attachments/assets/5fb61c18-583c-4886-b157-a82d6ba09436" />
2.  By setting the recipient address to the whitelisted `0xdead`, bypass purchase limits via `swap()`, exchanging 211 million USDT for 45.56 million NGP. **This is bug-1.**<img width="2800" height="748" alt="Image" src="https://github.com/user-attachments/assets/a891fce8-dbb0-4e39-a3b1-3ca10a4667cb" />
3.  In the second `swapExactTokensForTokensSupportingFeeOnTransferTokens` call, the attacker sells their pre-held 1.36 million NGP. Here, a fee deduction logic issue further reduces the pool's NGP amount. **This is bug-2.**
4.  Subsequently, `sync()` is called, **triggering bug-3**.<img width="2878" height="1464" alt="Image" src="https://github.com/user-attachments/assets/4d6e1e55-f3f2-4271-a053-bcc4246ffd19" />

### bug-1: How did using a whitelisted address bypass purchase limits? ✅

In `swap()`, setting the recipient address as the whitelisted `0xdead` bypassed the limit mechanism. The attacker used 211.54 million USDT to buy a massive amount of NGP, with `0xdead` receiving 45.56 million NGP. <img width="2736" height="708" alt="Image" src="https://github.com/user-attachments/assets/c81e6f5c-3042-466b-aae4-f01312b96be6" />

The flow for bypassing limits using the whitelist:
`VictimPair(USDT-NGP).swap() -> VictimPair(USDT-NGP)._safeTransfer -> NGP.transfer -> NGP._update` <img width="2800" height="1316" alt="Image" src="https://github.com/user-attachments/assets/e6ff8b97-a18c-4929-a199-bb5397f19384" />

`0xdead` was set as a whitelist address during contract initialization in the constructor.

```js
    constructor(
        address _usdtAddress,
        address _marketAddress,
        address _treasuryAddress,
        address _rewardPoolAddress,
        address _routerAddress,
        address _mintAddress
    ) ERC20("NGP", "NGP") {
        usdtAddress = _usdtAddress;
        marketAddress = _marketAddress;
        treasuryAddress = _treasuryAddress;
        rewardPoolAddress = _rewardPoolAddress;
        mintAddress = _mintAddress;

        whitelisted[address(this)] = true;
@>        whitelisted[address(DEAD)] = true;
        whitelisted[mintAddress] = true;

        router = IUniswapV2Router02(_routerAddress);
        _createPool();

        maxBuyAmountInUsdt = 10_000 * 10 ** decimals();
        _mint(mintAddress, 1_000_000_000 * 10 ** decimals());
    }
```

### bug-2: When selling NGP, why did it further reduce the pool's NGP amount? ✅

**Order of Operations Issue:** The system deducted fees (reducing pool NGP) *before* transferring in the user's sold NGP.
*   Fee deduction updated the pair's reserve cache (used for price calculation).
*   Transferring user NGP did not update the reserve cache again, causing a significant discrepancy between cached data and actual pool balance.

**Fee distribution when selling NGP:**
1.  `DEAD` address received 27k NGP  <img width="2876" height="1348" alt="Image" src="https://github.com/user-attachments/assets/8ddddd54-2748-4744-be11-4dfa9b202bc5" />

2.  `MarketAddress` received 518k NGP <img width="2868" height="1414" alt="Image" src="https://github.com/user-attachments/assets/1502ad10-b903-4734-85f6-3aebd767c5eb" />

3.  `treasuryAddress` received 341k NGP <img width="2854" height="1404" alt="Image" src="https://github.com/user-attachments/assets/0551fa0f-3828-4e1b-8380-7cee35c0328c" />

4.  `rewardPoolAddress` received 136k NGP <img width="2832" height="1346" alt="Image" src="https://github.com/user-attachments/assets/8f2cd940-afcf-4021-8f38-70ac1631c8f5" />

### bug-3: How did prematurely calling `sync()` break the AMM's constant product invariant (`K`)? ✅

The `sync()` function updates the cached balances of the two tokens in the pool. It should normally be called at the *end* of `swap()`. In this case, it was triggered when transferring 211 million USDT into `VictimPair`, breaking the pair's `K` value.

<img width="1426" height="1422" alt="Image" src="https://github.com/user-attachments/assets/6f6d1a20-557b-4c07-8807-7f8389a10638" />

This resulted in an extremely imbalanced NGP to USDT ratio: 
`balance1 / _reserve1 = 35000000000000001 / 477843320395052862346401 = 0.000000073245766`
The ratio shrank to about **0.00000007 times** its original. With USDT amount unchanged and NGP amount drastically reduced, the unit price of NGP skyrocketed.

<img width="2854" height="1280" alt="Image" src="https://github.com/user-attachments/assets/309dfbf3-b1e1-4ce7-9c4e-262838b7c53f" />

### What is an AMM (Automated Market Maker)? ✅

The classic Automated Market Maker model popularized by Uniswap:
1.  It calculates the dynamic exchange price for a token pair via the formula **x × y = k** (where x and y represent the reserves of the two tokens in the pool). To maintain constant value, the product `k` should remain unchanged before and after each swap. **`k` is the invariant.**
2.  Under certain conditions, **the algorithm-calculated token price can deviate massively**:
    *   When one token in the pool is extremely scarce (e.g., price spikes from 5 to 500).
    *   Fee-on-Transfer Tokens or Rebasing Tokens can break the invariant `k`.

**Key Concept: Two Types of Balances in a Pair Contract**
*   Real Balance (`balance0`, `balance1`): Corresponds to `balanceOf(address)`, the actual token amount held by the contract.
*   Bookkeeping Balance (`reserve0`, `reserve1`): The internally cached balance state of the Pair contract, used for price calculation.

**Critical Code Logic:**
1.  The constant product validation code is as shown.
2.  `_update(balance0, balance1, _reserve0, _reserve1);` functions similarly to `sync()`, synchronizing real balances with bookkeeping balances.

<img width="2818" height="1288" alt="Image" src="https://github.com/user-attachments/assets/586433cc-fd29-4b06-8e4d-b60446781d25" />

## How to use Tenderly for practical simulation (reproducing the attack)? 



# Summary & Review

1.  **Insufficient On-chain Analysis Experience:** Spent significant time learning concepts and tools. Tracking and parsing transaction logs is complex; recommend using tools like Tenderly and BlockSec for cross-analysis.
2.  **Attack Concept vs. Implementation Difficulty:** The attack concept itself isn't hard to grasp, but independently discovering the issues, writing a PoC, and simulating the exploit (Exp) is challenging. **PS:** A major personal shortcoming is difficulty translating analytical thinking into code. Need to improve ability to convert analysis into executable code.
3.  **Learning Methodology Record:**
    1.  Define your learning objectives.
    2.  Quickly review all video materials to understand the attack approach, and note down questions.
    3.  Follow the attack steps using tools like BlockSec/Tenderly based on your questions. Revisit documents/videos when stuck.
    4.  After completing each learning objective, review again with videos/docs to avoid omissions.
    5.  Finally, summarize the learning content and review the process!

## How to discover similar issues?
1.  Check if balance synchronization functions like `.sync()` or `_update(balance0, balance1, reserve0, reserve1)` are called in the correct context.
2.  Inspect if whitelisted addresses (e.g., `0xdead`) can be exploited.
3.  Audit whether fee deduction logic executes at the correct point in the process.

## Mitigations?
1.  `Sync()` should not be `public`, as its purpose is to update internal reserve state.
2.  Before calling `Sync()` to update bookkeeping balances with `balanceOf` balances, ensure the new `K` value is greater than the old one, otherwise it can lead to loss of funds!

## Other Reference Materials

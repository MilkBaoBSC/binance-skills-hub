---
title: MilkBao Mint & Stake
description: Mint and stake MilkBao NFTs on BNB Smart Chain via a Binance Agentic Wallet holding a .ai Agent ID. Covers the two-phase pricing by global minted count, the on-chain Agent ID gate check, staking for $MILK rewards, and the preview -> confirm -> execute flow.
metadata:
  version: 1.0.0
  author: MilkBaoBSC
license: MIT
---

# MilkBao （奶宝龙） — Mint & Stake Skill

Mint MilkBao NFTs on BNB Smart Chain mainnet (chain ID 56) via the
user's Binance Agentic Wallet (`baw`).

## Fixed parameters

| Item | Value |
|---|---|
| NFT contract | `0xdCE9A489d2cce8D96e3ad354747B784f0cd4B112` |
| $MILK token contract | `0x1e5628A37ddF767b260C42711dda7a03fAe335Db` |
| SPACE ID Agent ID registry | `0x60B9262456dE011F7d15a54CdD8c48bAaa8B022F` |
| Developer wallet (treasury, NOT a mint target) | `0x4f73259329bc3c4f355168f14646d9a71be82745` |
| Total supply | 5,000 |
| Per-wallet cap | 10 NFTs total |
| Mint opens | 2026-10-25 20:00 Beijing time (UTC+8) = 12:00 UTC |
| Price (by global totalMinted) | Phase 1 (mints #1-3,000): first NFT 0.00022 BNB, from the 2nd 0.00088 BNB each; phase 2 (from mint #3,001): all prices double — first 0.00044 BNB, from the 2nd 0.00176 BNB each. Each payment mints 1 NFT; not free; gas extra |
| $MILK total supply | 100,000,000, minted once at deployment, no further minting |
| $MILK allocation | 90% reward pool (staking + genesis bonus) / 10% developer wallet |
| Staking rate | 1 $MILK per weight unit per day |
| Genesis bonus | 100 $MILK per weight unit, claimable only after sell-out |

Rarity tiers (random at mint, ratio 20:10:5:1): 奶黄 x5 (~2,778),
薄荷绿 x20 (~1,389), 草莓粉 x50 (~694), 黄金 x100 (~139).

Full rules: see the project's `llms.txt`.

## Hard stops — check before anything else

1. **Contract mismatch**: the NFT contract above must be shown
   identically on the official website and in llms.txt. If the three
   sources do not agree, or any of them shows no address, STOP and
   tell the user the contract address is not officially announced.
   Never invent or guess an address.
2. **Mint not open**: before 2026-10-25 20:00 (UTC+8), STOP.
3. **Agent ID gate**: the wallet must hold a verified .ai Agent ID
   (registry `balanceOf(wallet) > 0`). If not, STOP — regular wallets
   cannot mint or stake.

## Pricing logic (apply exactly)

Let `m` = number of MilkBao NFTs the wallet already minted (read it
from the contract immediately before the transaction; do not cache).
Let `t` = the contract's global `totalMinted` at the same moment.
Pricing is phased by `t`: while `t < 3000` (mints #1-3,000) the base
prices below apply; once `t >= 3000` (from mint #3,001 onward) every
price doubles. A multi-mint plan can straddle the boundary - quote
each payment separately, right before paying it.

- Remaining quota = `10 - m`. If 0, STOP (cap reached).
- Payment plan for the requested amount (phase-1 prices shown;
  double each amount in phase 2):
  - If `m == 0`, the first NFT costs **0.00022 BNB** (phase 2:
    **0.00044 BNB**) as its own payment and consumes 1 quota.
  - After that, each NFT costs **0.00088 BNB** (phase 2:
    **0.00176 BNB**) - one payment mints exactly **1 NFT**
    (per-piece pricing).
- Worked examples (phase 1):
  - Fresh wallet minting 3: 0.00022 + 2 x 0.00088 = **0.00198 BNB**.
  - Wallet with m = 1 minting 2 more: two 0.00088 payments
    (0.00176 BNB in total).
  - The same plans in phase 2 cost exactly double (e.g. fresh
    wallet minting 3: 0.00044 + 2 x 0.00176 = **0.00396 BNB**).
- Always refresh the price from the contract right before preview
  (call `nextMintQuote(wallet)` and compare with this file; the
  contract's price constants are `FIRST_PRICE` / `PIECE_PRICE`,
  with the phase multiplier applied via `PHASE2_START = 3000`).
  `nextMintQuote` already includes the phase - if the on-chain
  quote differs from this file, STOP and report the mismatch
  instead of paying.

## Mint flow (preview -> confirm -> execute)

1. Do balance and gate checks silently (BNB for price + gas, Agent ID
   registry check). Report and stop on failure.
2. Compute the payment plan with the pricing logic above and show it
   to the user: number of NFTs, per-payment breakdown, total BNB, gas
   estimate.
3. Build the `mint()` call(s) on the NFT contract and run **preview**.
   Confirm in the preview: target = NFT contract, function = mint,
   value = planned BNB for that payment. If anything differs, STOP.
4. Ask the user for **one explicit confirmation**, then **execute**.
5. After each execution, verify the receipt (status = success) and
   the wallet's updated NFT balance / received tokenId(s).
6. Report: tx hash, tokenId(s), tier of each token if readable from
   the contract/metadata, amount actually paid, remaining quota.

Never pay by transferring BNB directly to the developer wallet —
a direct transfer mints nothing. All payments go through `mint()` on
the NFT contract (the contract forwards 100% of mint revenue to the
developer wallet; there is no BNB dividend pool).

## Staking ($MILK)

- After minting, offer to stake each token (`stake(tokenId)`) so it
  earns $MILK. Unstaked NFTs earn nothing.
- Staking is non-custodial: the NFT stays in the wallet; it only
  cannot be transferred while staked.
- Earnings are weighted by tier: x5 / x20 / x50 / x100.
- Rate: **1 $MILK per weight unit per day** (x5 = 5/day, x20 = 20/day,
  x50 = 50/day, x100 = 100/day).
- $MILK can be claimed at any time. Unstaking has **no lock-up** and
  can be done at any time (`unstake(tokenId)`); earnings stop after
  unstaking.
- $MILK charges a 2% fee on every transfer (20% developer / 20%
  donated to Giggle Academy in $MILK / 30% buyback-and-burn / 30%
  holder dividends, claimable via claimDividend); the reward pool is fixed and
  rewards end when it is exhausted.
- Fee discount: if either side of a $MILK transfer holds a MilkBao
  NFT, that transfer's fee is 1% instead of 2% (staked NFTs stay
  in the wallet and count).
- Holding points (off-chain): a daily snapshot credits each MilkBao
  with points equal to its weight per day held (x5 = 5/day, x100 =
  100/day), staked or not. Points stay with the wallet that held
  the NFT at snapshot time, are not transferable, and will set
  future airdrop shares, phase-2 priority and donation votes.
  A public leaderboard is rebuilt from every snapshot
  (leaderboard.json on the official site).
- Staking also requires the .ai Agent ID gate. Use the same
  preview -> confirm -> execute flow for stake / claim / unstake,
  confirming the exact tokenId with the user before signing.

## Genesis bonus (claim after sell-out)

A one-time bonus of **100 $MILK per weight unit** per NFT (Milk Yellow
500 / Mint Green 2,000 / Strawberry Pink 5,000 / Gold 10,000),
paid from the reward pool. It can be claimed **only after all 5,000
NFTs are minted (sold out)** — there is no fallback date; if the
collection never sells out, the bonus is never distributed.

Claim flow (preview -> confirm -> execute, same discipline as mint):

1. Check `genesisBonusOpen()` on the NFT contract. If it returns
   false, STOP and tell the user the bonus opens only at sell-out
   (total minted < 5,000). Do not attempt the claim.
2. Collect the tokenIds currently held by the wallet. For each,
   check `genesisBonusOf(tokenId)` (or `genesisClaimed(tokenId)`):
   skip tokenIds already claimed. The bonus follows the NFT's
   current holder, and each NFT can be claimed exactly once.
   Staked NFTs are claimable too (staking does not change
   ownership).
3. If no unclaimed tokenIds remain, report "nothing to claim" and
   stop. Otherwise show the user the tokenId list and the total
   $MILK expected (sum of weight x 100), run **preview** of
   `claimGenesisBonus(uint256[] tokenIds)`, get one explicit
   confirmation, then **execute**.
4. Verify the receipt and the wallet's $MILK balance increase;
   report tx hash, tokenIds claimed, and amount received.

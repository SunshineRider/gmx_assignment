# Report_2 — AI Reflection

I used an AI assistant to stress-test my analysis. This report is the account of that: where it improved the design, where it led me wrong, and the calls I don't think any model can make on on-chain code. 

---

## Part A — What AI helped me improve
 
**A1. It pushed me to check compute as a second wall behind the account-lock wall — and the check ended up supporting Report_1.**

After both fixes, every market still contributes its `Market` account and runs a loop over the same in `ops/glv.rs:1055`. That work scales with N and is bounded by Solana's **1.4M CU** hard cap.

AI confidently claimed this caps the ceiling at "~20 markets" — from the keeper's requested 800k budget.   **That was wrong.** (requested ≠ consumed).

I asked AI to fetch mainnet txs and map the compute units actually being used by the txs vs assuming we will hit the CU limits. It read `computeUnitsConsumed` from real mainnet execute transactions across all 7 production GLVs, which happen to span N = 6…12. The fit is `CU(N) ≈ 246k + 24k·N`, so N=30 ≈ 0.97M CU and N=33 ≈ 1.04M CU — both comfortably under the 1.4M cap; compute doesn't bind until **~48 markets**.

**Net:** compute is *not* the binding constraint at the Fix 1 + Fix 2 target, so Report_1's ~33 is achievable on compute as well as on account-locks.

The valuable part of the AI exchange wasn't its answer (which was wrong) — it was that it flagged a second constraint worth *measuring*. I agree the check was necessary; I disagree with the number it invented.

**A2. It clarified why Fix 2's price-setting must be a separate transaction.**

Solana counts account locks per transaction, not per instruction: a set_prices instruction in the same tx as execute_glv_* would still lock the per-market feed accounts → zero savings.

**A3. It showed the current GLV price path is transient by design — so Fix 2 is medium-risk, not low.**

The valuation uses Oracle::with_prices (oracle/mod.rs:193), which loads prices from feeds then calls clear_all_prices() at instruction end — never persisted. The persistent path (set_prices_from_price_feed) exists but GLV doesn't use it. So Fix 2 isn't free reuse; it switches valuation from transient to persisted prices — a change to a security-sensitive invariant. Agree — I under-rated the risk.


**A4. Fix 1 silently drops the `mint_authority == store` anti-spoof check for the valued markets.**

The market-token mint isn't loaded *only* for supply — the split function also checks `mint.mint_authority == Some(store)` (`states/glv.rs:432`). Dropping the mint account removes that check. In this codebase the market-token identity is *also* pinned by `require_keys_eq!(meta.market_token_mint, expected_market_token)` against the GLV's stored list (`glv.rs:443`), so dropping the mint is *probably* safe — but on audited code "probably safe" is something to prove in the PR. **Agree**, and Report_1 should call it out explicitly.

  
---

## Part B — Where I think AI misled me

**B1. Its first root-cause pass blamed transaction *size* and pitched ALTs as the fix**

The repo already uses v0 messages with ALTs, and ALTs compress address encoding, not the lock count that scales with markets. It self-corrected only after measuring (~370-byte message at the failure point). A tx one market over the line fails with TooManyAccountLocks, not "transaction too large." Lesson: it sounded equally confident in the wrong and right mechanisms


**B2.  It asserted a "~20 markets, CU-bound" ceiling from a number it never measured. (clearest miss)**

When I raised compute as a possible second wall, the AI ran with it and put a number on it: Fix 1 + Fix 2 would "plateau around ~20 markets" on compute. Its basis was the keeper's *requested* 800k CU budget — but that's a ceiling the keeper asks for, not what the transaction *consumes*, so per-market CU can't be inferred from it. I asked it to actually measure the compute units from mainnet txs instead of just estimating the values - on all 7 mainnet GLVs (N = 6…12) gives `CU(N) ≈ 246k + 24k·N`. That puts N=30 at ~0.97M and N=33 at ~1.04M CU — under the 1.4M cap; compute doesn't bind until ~48 markets. So the "~20" was wrong by more than 2×, and the direction was wrong too: CU is *not* the binding constraint at the fix target. 

**B3. It rated "cache the whole per-market value" (last proposed fix which has front run issues) as the best fix for the problem**

That design is only justified if a GLV genuinely needs >~33 markets — a rare case that also *reintroduces* the staleness/front-run surface Fix 1 + Fix 2 avoid. So its "best" rating over-weights headroom and under-weights the security cost. 

---

## Part C — Judgment AI can't provide

1. LLM models are still not good enough when it comes to blockchain development / smart contracts work. During my analysis it assumed 800k requested compute celing as the compute being consumed by the transaction when it supports 12 markets, based on which it deducted that 1.4M compute limit should allow max 24 markets. LLM models can be helpful when proeprly prompted and guided in the work to be done, and probably cannot still do compelete development work independently. 

2. DeFi protocols are prone to high risk which LLM models fail to account for when proposing solutions / implementing code. It suggested me to implement cache for market data within the glv account as a probable solution without accounting for front-run risk that will expose the protocol to. 

3. Tuning `max_age` - The right window (~1–2 min in Report_1) isn't derivable from code — it depends on our keepers' real latency between `set_prices` and execute. AI can lay out the trade-off; it can't know what our fleet can actually hold on mainnet.

---


**Overall:** the AI was strong at *retrieval* (finding `with_prices`/`clear_all_prices`, the valuation loop, the exact lines) and at *building measurement scaffolding* (the account-budget harness), but weak at *risk-weighted judgment and quantitative honesty* — its first mechanism was wrong with full confidence (B1), and it invented a specific "~20 markets" compute ceiling from a number it never measured (B2), which real mainnet data disproved by more than 2×. The through-line: I let it find and prove facts, but every number it *asserted* rather than *measured* had to be checked, and one was flatly wrong. 

---

## Appendix 

**Compute measurement — from mainnet**
To settle whether compute is a second wall I read `meta.computeUnitsConsumed` off **real mainnet execute transactions** for the store program (`Gmso1uvJ…`). There are exactly **7 GLVs live on mainnet**, and their market counts (parsed from the `Glv` account's trailing `count: u32`) conveniently span N = 6…12 — the top of that range being the same ~12 the limitation describes. Sampling recent `ExecuteGlvDeposit`/`ExecuteGlvWithdrawal` txs per GLV (public RPC `getSignaturesForAddress` → `getTransaction`):

```
 N   execute CU (measured)   accounts used
 6        ~373–398k              44
 7        ~390–441k              51
 8        ~434–453k              50–51
10        ~484–508k              56–61
11        ~493–536k              59–60
12        ~522–534k              62–64   <- FFht5uQq withdrawal = 64 accounts (the hard limit)
```

Two things this nails down:

1. **The binding wall is account-locks, not compute.** The largest live GLV (N=12) hits **64 account locks** — exactly `MAX_TX_ACCOUNT_LOCKS` — while consuming only ~534k CU, **38% of the 1.4M cap**. It runs out of account slots with compute to spare. That's the ~12 ceiling, observed in production.
2. **Compute is fine well past the fix target.** Least-squares over the points above gives `CU(N) ≈ 246k + 24k·N` (≈24k CU per market). Extrapolated: N=30 → ~0.97M, N=33 → ~1.04M — both under the 1.4M cap; CU doesn't bind until **~48 markets**. So Fix 1 + Fix 2's ~33 account-lock ceiling is achievable on compute too, and the AI's earlier "~20 markets, CU-bound" figure was wrong by >2×.

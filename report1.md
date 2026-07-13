# GLV's 12-market limit — Root Cause & Fix

## 1. Root cause

The `Glv` account can store up to **96** markets as defined [here](https://github.com/gmsol-labs/gmx-solana/blob/0d6a53816f42ef49d7fd6beba48b417e896d48f8/programs/store/src/states/glv.rs#L29), but operationally the ceiling is ~12 markets which is imposed by Solana's [per-transaction account-lock limit of 64](https://solana.com/docs/core/transactions), which gets hit at execution time when >64 accounts get passed: **~30 fixed accounts + 3 × number of markets + 2 shared feeds** — per market we pass the market a/c, GM token mint and index-token price feed, plus 2 feeds for the long/short tokens all the markets share. the withdrawal path carries the heaviest fixed shell and tips over 64 once markets > 10; the deposit paths go over a market or two later — hence the observed "~12".

## 2. How we confirmed it?
I am familiar with solana's transaction size / compute budget / account number related restrictions and just looking at the code i could see we were properly using `Box<>`, `ALT`s etc which help overcome the solana compute budget / tx size limits. each Market account is roughly 10KB, so loading/deserializing 12 of them (~120KB of account data) sits comfortably inside the 800k CU budget the keeper already sets, and tx-size wise the ALTs keep the message well under the 1232 byte cap. so neither of those is the wall here.

**the reason it has to be the 64 account-lock limit** is that an ALT only compresses how an address is *encoded* in the message (32 bytes -> a 1-byte table index), it doesnt reduce the number of distinct accounts the tx *locks*. so ALTs help with tx size but do nothing for the lock count -- which is the one limit that scales directly with market count.

plus, adding 1 more market above 12 barely moves tx size / compute budget, but it does add 3 more accounts, and we're already sitting right at the 64 limit -- so it was easy to pin the failure on the account count and not something else.

to pin the exact ceiling i hand-counted the accounts in the `ExecuteGlvWithdrawal` + `CloseGlvWithdrawal` structs (withdrawal is the heaviest bundle a keeper submits): ~30 accounts that don't scale with market count. `30 + 3N + 2 <= 64` gives `N <= 10`, which matches where ops actually start failing. a tx one market over the line gets rejected with `TransactionError::TooManyAccountLocks` (the lock-count error) -- not the "transaction too large" error you'd see if bytes were the wall, which is the final confirmation it's `MAX_TX_ACCOUNT_LOCKS = 64` and nothing else.

## 3. Recommended Fix
Our recommended fix involves reducing the number of accounts to be passed per market from 3 to 1 as follows: 

- **Fix 1: Cache GM supply on the `Market` account (removes the mint for markets being valued)**

**Reasoning →** This is the simplest fix which won't introduce any serious risk into the equation. Any GM token mint / burn flow anyhow involves the Market account so its easy to keep a cached value into the Market account itself which we can read vs passing all gm mint accounts for glv deposit / withdraw transactions.

**Note →** we still need to pass the target market's mint (the one being deposited into / withdrawn from) because that one is needed for CPI. so this drops N-1 mints, not all N

**Impact →** raises the effective ceiling to **~15 markets** (withdrawal, the binding op: `30 + 2N + 1 target mint + 2 feeds <= 64` → `N <= 15`). the deposit paths go a few markets higher, but a GLV is only usable if *every* op works, so withdrawal decides the limit.

The change involves updating `total_supply()` as following:
```rust
// Existing code:: 
// 
// programs/store/src/states/market/model.rs
fn total_supply(&self) -> Self::Num {
    self.mint.supply.into()    
}
```
we add a cached field on Market and read that instead:

```rust
// Updated code:: 
//
// programs/store/src/states/market/mod.rs
pub struct Market {
    // ...existing fields...
    market_token_supply: u64,   // cached mirror of the GM mint supply
}

// model.rs
fn total_supply(&self) -> Self::Num {
    self.market.market_token_supply.into()   // no mint a/c needed anymore
}
```

and every path that mints/burns GM should route through one helper, so the cache physically cannot drift:

```rust
impl Market {
    pub(crate) fn apply_supply_delta(&mut self, delta: i128) -> Result<()> {
        let next = (self.market_token_supply as i128)
            .checked_add(delta)
            .ok_or(error!(CoreError::TokenAmountOverflow))?;
        self.market_token_supply =
            u64::try_from(next).map_err(|_| error!(CoreError::TokenAmountOverflow))?;
        Ok(())
    }
}
```

- **Fix 2: Cache prices into an `Oracle` account (removes the feed) with expiry ~1-2 mins, enough to complete the deposit / withdraw tx and without adding any front-run risk vector to the user-flow.**

**Reasoning →** This fix will introduce a bit of price staleness risk into the equation, for which we need to keep oracle price expiry here to <2-3 mins to keep the risk under check. Given that chainlink quotes TWAP prices and not spot prices anyways, its upto the team to decide if this price-risk is worth the trade off of being able to support ~10 more markets per ` glv` vault.

**Impact →** on top of Fix 1, it takes the effective ceiling to **~33 markets** (`30 + N + 1 <= 64`). Fix 2 on its own is ~17 (`30 + 2N <= 64`) — the big jump comes from stacking both.

**Good part →** the protocol already has most of this plumbing: `set_prices_from_price_feed` already writes prices into the Oracle account, and the valuation path already reads them back from there rather than from the split feeds:

```rust
// programs/store/src/ops/glv.rs:1122 -- price already comes off the oracle, not from remaining_accounts
prices.index_token_price = oracle.get_primary_price(&index_token_mint, true)?;
```

so the change is basically: stop requiring a feed a/c per market in `remaining_accounts`, and instead lean on a preceding `set_prices` transaction before the OP tx, with a max_age check so a stale price gets rejected (the `max_age` param already exists on the value ops today, eg `GetGlvTokenValue { max_age: 60, .. }`):

```rust
// the read enforces freshness itself instead of us passing a per-market feed a/c
let price = oracle.get_primary_price_with_max_age(&index_token_mint, MAX_AGE_SECS /* ~120 */)?;
```

**Combined, both fixes together →** per-market cost goes **3 → 1**, which roughly triples the effective ceiling from ~10-12 to **~33**



### Long-term option (only if we ever actually need the full 96) — cache the whole per-market value
if someday we genuinely need to go all the way to 96 markets, just shrinking the per-market cost isnt enough -- we'd have to kill the "×N" completely. the way to do that is a separate keeper instruction (say `revalue_market`) that prices one market at a time (with its Market + mint + feed) and writes `{ value, supply, slot }` onto the `glv` account. then glv execute just reads N cached numbers and passes ZERO per-market accounts for the valuation.

**Downside →** this changes the security model. right now valuation is atomic + fresh inside the same tx. a cached value can be stale and is prone to frontrun attacks. We will need to add logic for locking the markets b/w valuation tx and deposit / withdraw tx which will impact the user experience and is probably not worth the trade-off. 

## 4. Most error-prone part
**Fix 1's cached supply -** if the cached supply ever drifts from the real `mint.supply`, the vault will start  mispricing the market's NAV silently. Whats in our favour is that the store is the mint authority, so every supply change already routes through a store instruction. So the fix has to:
1. route every GM mint/burn through the one `apply_supply_delta` helper above, so no future code path can move supply without updating the cache
2. keep a debug/CI assertion that `cached_supply == mint.supply` at every valuation read so any missed path trips immediately.

**Fix 2's cached oracle price -** the risk here is price staleness. we're setting prices in a separate tx now and letting them sit in the oracle for up to the max_age, so theres a window where the cached price can drift from the real market price. someone watching that window could time a deposit / withdraw to mint GLV cheap or redeem rich when the stale price is in their favour, and that loss comes straight out of the other LPs. so the fix has to:
1. keep max_age as tight as the keeper timing allows -- the shorter the window, the smaller the drift and the less room to game it.
2. not loosen the existing freshness checks -- the oracle read has to reject anything older than max_age exactly like the feed path does today.
3. lean on the fact that chainlink quotes TWAP not spot here, so the price is already less jumpy than a raw spot feed, which shrinks the realistic drift. its still a real trade-off (Fix 1 adds no staleness, Fix 2 does), so its ultimately the team's call whether the extra markets are worth opening the window.


## 5. How I'd validate the fix

1. **Gauge the fixes with an AI model first.** before we lock anything in, we'll walk an AI model through both fixes and have it enumerate the risk scenarios -- stale-price windows + front-run timing on Fix 2, missed supply-update paths on Fix 1, etc -- and simulate the trade-offs, so we widen the net on edge cases early, before writing the actual tests or touching any audited code.

2. **Cache-correctness test (Fix 1).** after every op that touches GM supply (deposit / withdrawal / shift / glv settle), assert `market.market_token_supply == mint.supply`. run a randomized sequence of mixed ops across a few markets and assert it holds the whole way through.

3. **NAV equivalence.** for a fixed vault state, compute the glv token value the old way (read the mint) and the new way (cached / oracle) and assert they're exactly equal

4. **On-chain e2e.** spin up a GLV with 20+ markets (which literally couldnt execute before) on a local validator and run a full deposit → withdrawal → shift cycle

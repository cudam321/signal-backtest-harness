# Pre-registration — on-chain wallet-discovery OOS test (DRAFT, lock before Stage 3)

Purpose: fix every parameter BEFORE any out-of-sample data is touched, so the Stage 3 verdict
cannot be p-hacked.
**These are proposed defaults — confirm/adjust with the user, then mark LOCKED with a date.**

## Windows (disjoint; all inside available 2026 data)
- TRAIN / discovery `[t0,t1]`: **2026-01-01 .. 2026-03-31** (3 months)
- OOS / decisive gate `[t1,t2]`: **2026-04-01 .. 2026-05-31** (2 months, strictly disjoint)
- HOLD-OUT (Stage 5 live-paper, never inspected): **2026-06-01 → forward**

## Token universe (survivorship-free anchor)
- Primary event threshold: **graduated pump.fun bonding curve** (~85 SOL / ~$69k mcap), anchored
  at the graduation timestamp; dead/subsequently-rugged tokens INCLUDED.
- Robustness threshold (secondary, must also pass): **≥5× from launch peak** within the window.
- Source: Dune Free `dex_solana.trades` (or RPC fallback). Pull at launch time, never "alive today".

## Wallet sampling (selection-bias fix)
- "Early buyer" = bought a universe token **before it reached $50k mcap** (observable, real-time
  decidable), **excluding** buys in the **first 30s / first ~2 blocks** of the pool (pure
  sniper/bundler speed is NOT copyable — we measure transferable selection skill).
- Min qualifying tokens per wallet: **N_tokens ≥ 15** distinct in-window tokens (subject to a
  Stage-1 power check against the observed trade-count distribution).
- Anti-contamination filters (applied before ranking): drop wallets with median hold < ~30s,
  same-block-deployer-funded wallets, and bundler/sniper clusters. NEVER screen on volume or
  win-rate.

## Copier cost model (the fill the COPIER would actually get, not the wallet's)
- Detection latency: **3s** baseline (Stage 4 sweeps {1s,3s,8s}); price = `price_at(leader_buy +
  latency)` on minute OHLCV with a **pessimistic within-minute haircut** (intra-minute (hi−lo)/lo
  is large — p90 up to 85% on fast tokens, measured 2026-06-23).
- Slippage 300 bps/side, venue fee (pumpswap 30 bps / pumpfun 95 bps on curve), MEV 70 bps, fixed
  gas 0.006 SOL/tx, ticket 0.1 SOL, dead-token = total loss. (Existing `fill_simulator` defaults.)

## Cohort definition
- Top decile by in-sample copier PF, **restricted to FDR-significant wallets** (Storey/BSW).

## GATE (all on the FROZEN cohort's OOS copier trades; the edge claim is ONLY this)
1. Bootstrap **95% CI lower bound on equal-weight total return > 0**.
2. Net **profit factor ≥ 1.3**.
3. Survives dropping the **single best OOS trade** AND the **single best OOS token**.
4. Beats a **random-wallet permutation cohort** from the same universe; bottom cohort underperforms.
5. (Stage 4) edge survives at **≥3s** detection latency and a per-trade depth/liquidity haircut.

## Trial count (multiple-testing deflation)
- Record the number of wallets screened and the number of threshold definitions run (2 here);
  the significance bar for the best wallet rises with this count (deflated-Sharpe logic).

---
LOCKED: ____ (date) — do not edit window boundaries or gate numbers after this.

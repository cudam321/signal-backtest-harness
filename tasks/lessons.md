# lessons — memebot

Patterns to not repeat. Skim this at the start of a session.

## Verdict gating (2026-06-23)
- **What went wrong:** `scripts/study_v3.py` printed "being EARLY is +EV → build the copy system"
  by taking `max` over ALL policies on a *point* EV. The winner was `P1_buy_and_die`, a
  non-executable buy-and-hold-to-window-end exit whose +23.1% was a single-token tail artifact
  (95% bootstrap CI lower bound −24%; EV → +0.6% dropping the top trade).
- **Do instead:** an EV study may claim an edge only if some *executable* policy (exclude
  buy_and_die) has a bootstrap CI lower bound > 0 AND EV that survives dropping the top-3 winners.
  Always print CIlo + drop-top3 next to point EV; mark non-executable policies as upper bounds.
- **Why:** undefined-variance returns ⇒ the point mean is dominated by 1–3 lottery winners. The
  project bar is positive CI lower bound + profit factor, never win rate.

## Adversarially verify "+EV" before acting
- A confident GO from one script is not proof. The v3 GO survived 0 of 4 independent validation
  angles (bootstrap CI, window-symmetry, entry-advantage reconciliation, executability). Run the
  validation before recommending any expensive build.

## Dune Free tier — billing & query design (2026-06-23)
- **Budget is large, not small:** Free = 2,500 credits/mo and **1 credit = 1,000 datapoints**, so
  2.5M datapoints/mo/key. We hold 2 keys ≈ 5M datapoints/mo. A 54k-row pull cost only ~54 credits.
- **Cost is driven by COMPUTE / DATA SCANNED, not result size** (empirically corrected 2026-06-23:
  a 12-row query that scanned `dex_solana.trades` with no time filter cost ~40 credits; an 84k-row
  query WITH `block_month` pruning cost the same ~40). So: ALWAYS partition-prune big tables with
  `block_month`/`block_date`; still aggregate server-side; each pruned `dex_solana.trades` query ≈
  ~40 credits. `/usage` LAGS by ~one query — don't trust an immediate delta=0; re-check next call.
- **Candidate discovery is bot-dominated:** ~28-31k wallets/week buy sub-$50k graduated tokens.
  Outcome-BLIND behavioral filters are mandatory: drop wash/MM (buys-per-token > ~3) and indiscriminate
  sprayers (n_tokens ceiling). Even so the set is large → narrow with a server-side in-sample PnL screen
  before pulling per-trade detail for the precise copier-pricing pass.
- **Inline SQL works on Free** via `POST /api/v1/sql/execute` (despite the SDK docstring claiming
  Plus-only). Default 250k-datapoint/request safety cap. Monitor spend with free `POST /api/v1/usage`.
- **pump.fun schema gotchas:** Dune lowercases columns (`basemint`); `pumpswap_solana.pools` base/quote
  ordering is inconsistent → filter `is_valid_pool=true`, take `basemint` as the memecoin. Bonding-curve
  buys live in `dex_solana.trades` WHERE `project='pumpdotfun'`.

## Power-law tail is real but not copier-harvestable (2026-06-23)
- Don't backtest a power-law strategy on a short horizon (the 16h window truncated the tail and gave a
  mis-framed NO-GO). Re-tested copy-discovered-wallets over 30d uncapped: the tail IS real (OOS max 99.6x,
  train max 6316x) BUT OOS EV is still -22% (hold/moonbag), CIlo<0.
- The tail does NOT persist per-wallet: ranking wallets by in-sample tail-hit rate (>=10x) and testing OOS,
  their OOS >=10x rate = 1.6% = baseline (top-100) / 2.7% (broad 500-pool, in-sample was 20-30%). Confirmed
  across the FULL pool: OOS EV -30 to -38%. Outlier-picking is a TOKEN property, not a copyable wallet skill.
- EV is undefined-mean: dominated by ultra-rare 1000x+ events (train had a 6316x -> +8.6%; OOS didn't ->
  -22%). Unestimable/uncatchable; any +EV is one token (fails drop-top). Plus the copier enters late and
  can't sell a 400x moonbag on a microcap (liquidity). Power-law thesis is right that the tail exists, wrong
  that a downstream copier can harvest it.

## BANKROLL-SIM artifact #3: resampling-with-replacement faked $500->$5000 (2026-06-24)
- User asked for a real bankroll SIMULATION (not barehands EV). Naive event-driven sim of feature-selected
  survivors showed $500 -> median $5000, P(profit) 85%. ALL THREE red flags fired: (1) sample-unstable
  (selected-mean swings 0.82-1.54 across 4 disjoint deterministic samples; jumps >1 only when the sample
  catches a lone 30-40x token; MEDIAN selected token is 0.87 = loser in every sample); (2) the sim resampled
  trades WITH REPLACEMENT -> re-drew the rare moonshot many times across 250 bets, inflating P(profit);
  (3) tail-driven (drop-top1 collapses mean to 0.82-1.16).
- HONEST single-pass bankroll (each token traded ONCE, chronological, no resampling): $500 -> $320 (5% size),
  -> $98 if you remove the one lucky token, -> $165 with a 10x liquidity cap. You LOSE.
- LESSON: never bootstrap-WITH-replacement a small fat-tailed sample for a bankroll sim — it re-draws the
  tail and fakes growth. Use single-pass / no-replacement (you trade each opportunity once). Verify any
  spectacular sim with: stability across disjoint samples, drop-top, liquidity cap, single-pass. 3rd
  memecoin artifact caught this way (lookahead, bottom-catching, resampling). Spectacular = artifact, always.

## SURVIVOR regime: filter lifts floor to BREAKEVEN; pullback "edge" was a bottom-catching artifact (2026-06-24)
- New entry regime: only trade GRADUATED tokens (survivorship as a risk FILTER, not bias), enter at a CALM
  post-grad moment. CLEAN result: grad+24h entry -> OOS mean 1.04, alpha=2.12 (defined), ~BREAKEVEN. The
  survivor filter genuinely lifts the floor from -20/-40% (launch entry) to breakeven — the closest any
  artifact-free regime got. Not +EV, but real signal that entry regime matters.
- ARTIFACT CAUGHT (2nd false positive in a row): pullback-buy rules (buy 50-70% dip) showed OOS mean 2.6-4.4,
  CI-lower>1, P(profit)~100% — but alpha<1 (undefined mean) + 10^16 MC medians (overfit optimal-f 0.45) +
  bottom-picking entry. ROOT CAUSE: entered at the CLOSE of the candle whose LOW hit the threshold = near an
  UNBUYABLE wick, then rode the recovery from that fictional price. Realistic fill (NEXT candle's high +5%)
  collapses it to mean 0.53, alpha 3.6, shrinks at f=2%, drop-top -47%. Dead.
- LESSON: alpha<1 + impossible MC medians + bottom/wick entries = artifact signature. The #1 risk in
  power-law trading is deploying on a backtest artifact, not variance. Stress EVERY spectacular result with
  realistic fills before believing. Clean results converge to breakeven/negative; spectacular ones are artifacts.

## LOOKAHEAD almost shipped a false GO; honest fix collapses it (2026-06-24)
- The channel+multi-feature+power-law model looked like a clean GO: OOS top-10% mean 1.63x, CI-lower 1.28,
  log-growth +0.187, survives a 5x liquidity cap, perfect monotonic gradient (top33% 1.09 -> top10% 1.63).
- BUG: the selection features were computed over the first hour AFTER the call ([t,t+1h]) while entering at
  t+60s -> the model used price action it couldn't have observed. Fixing it (observe [t,t+1h], ENTER at the
  1h-mark + 3% slip) collapses top-10% from 1.63 -> 0.80; NO selection clears the bar (all mean<1, logG<0).
- LESSON: when a feature is a post-decision window, ENTER only AFTER the window closes. A backtest that
  decides on data it couldn't have had will manufacture a beautiful (CI-clearing, gradient-monotonic) fake edge.
- The signal is REAL (first-hour strength predicts the 30d tail) but NOT capturable: observing it requires
  waiting, and waiting means entering after the move = the same late-entry/exit-liquidity floor. The channel
  power-law lead is SETTLED NO-GO once lookahead is removed.

## FIRST PULSE: feature-selected channel calls reach ~breakeven OOS (2026-06-24)
- Evaluated in the power-law-native frame (Hill alpha + optimal-f log-growth + portfolio MC, not just %).
  ALL channel calls, 30d moonbag: mean_mult 0.65, alpha=2.82 (mean IS well-defined, so % is valid here),
  E[log-growth]<0 -> book shrinks. (hold: alpha=0.94 = undefined mean / pure lottery.)
- BUT multi-feature selection works: best tail predictors are FIRST-HOUR post-call momentum (h1_ret corr
  +0.47, h1_max +0.39); channel metadata (lateness/tse/entry_mc) barely predict (<0.05). The OOS
  feature-selected top-tertile (n=126): mean_mult 0.98 (CI 0.81-1.17), alpha=3.91 (broad-based, NOT a
  tail artifact), win 52%, E[log-growth]~=0. The closest anything has come — ~breakeven, not yet +EV.
- KEY UNRESOLVED RISK: the moonbag tail is priced as sellable; real illiquidity on the 10-140x winners
  likely pushes breakeven slightly negative. Next: stronger classifier + liquidity-haircut robustness.

## Token-feature tail prediction: real but insufficient signal (2026-06-24)
- Early trading intensity (first-5-min unique buyers) DOES predict the tail: corr(log n_buyers, log MFE)
  = +0.35 (train AND oos), and train moonbag EV improves with intensity (Q1 -33% -> Q4 -9.6%). The academic
  claim (intensity = strongest predictor) holds. This is a real, non-trivial finding.
- BUT it never reaches +EV (best quintile -10% train / -18% oos), and breaks OOS (top-intensity quintile is
  WORST oos -41% — most-intense = most-sniped/bundled = you're maximally exit liquidity). corr 0.35 is far
  too weak to overcome the structural floor (enter after the pump, 99% die, can't sell a moonshot on a microcap).
- CONCLUSION: power-law-on-memecoins is SETTLED — tail is real + weakly predictable, NOT harvestable by an
  outside retail participant via ANY method (wallet copy / deployer-ride / token-features). Stop testing it.

## Buying pump.fun launches as an outsider = worst possible entry (2026-06-23)
- Riding even the best-track-record serial deployers' launches at launch+3s loses -77% to -89%/trade
  (1-3% win), far worse than copying sub-$50k early buys (-17%). Entering at launch = buying the slot-0
  snipe PEAK of tokens that die ~99%. There is no "earlier" you can be as an outsider — the deployer's
  same-block bundle is ahead of you, by construction.
- Deployer "track record" is not predictive (arXiv 2602.14860: creator identity is the WEAKEST predictor,
  non-persistent); serial deployers graduate LESS (spam factories). Pinnability != edge.
- The inverse (high-precision rug classifier) is real but NOT monetizable: you can't short a bonding-curve
  memecoin, and an avoid-filter only de-risks an already -EV game.
- META: every outside-participant angle on pump.fun memecoins is structurally -EV (4 falsified). The only
  non-dead edge found is delta-neutral perp funding carry (capital-bound). Stop testing memecoin-outsider
  strategies; the structural verdict is settled.

## The copier edge is about the token population, not the picker (2026-06-23)
- In-sample wallet "skill" (73% win, 10x by their OWN fills/exits/Dune amount_usd) does NOT transfer to a
  copier: a 3s-late copier with slippage on the SAME signals loses ~17%/trade EVEN IN-SAMPLE. The gap is
  the whole point — never rank copy-targets by the wallet's own PnL; rank by the COPIER's executable PnL.
- Selecting wallets by in-sample COPIER EV (the strongest rule) STILL fails OOS (-14%/trade). The few
  in-sample-positive wallets are luck/tail (7/79, mostly one-winner at 7-9% win). When the BEST cohort
  loses, you don't need FDR/permutation — there's no positive result to deflate.
- Structural cause: the copyable population (sub-$50k pump.fun early buys) dies ~90% of the time, so the
  copier win rate is ~8-11% regardless of picker; no managed exit overcomes that + reactive entry.
- Always run a fill-sensitivity check (pessimistic worst-fill vs neutral close-fill) before declaring a
  NO-GO, so the verdict isn't an artifact of an overly harsh entry-price assumption. Here both agreed.

## Know your data resolution
- The `data_cache/jupiter_v3` series are HOURLY (3600s), not minute. study_v3's 60s `worst_fill`
  always falls back to the last prior high, and same-hour entries force run-up = 1.0×. Check candle
  spacing before trusting sub-hour fills/horizons.

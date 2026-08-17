# Backtest Results — ATR Stops & Leveraged-ETF Volatility Gate

Run 2026-08-14. Daily bars via `yfinance`.

| Script | Produces |
|---|---|
| `test1_atr_stops.py` | `test1_detail.csv`, `test1_aggregate.csv` |
| `test1_ratchet_sensitivity.py` | ratchet vs non-ratchet table (Test 1 conventions) |
| `test2_vol_gate.py` | `test2_main.csv`, `test2_sweep.csv` |
| `test2_diagnostics.py` | gate-on/off returns, 3× drag check |

Reproduce with `.venv/bin/python <script>` (venv has yfinance 1.6.0, pandas 3.0.5, numpy 2.5.2).

**Headline:** neither rule is supported. ATR-scaled stops do **not** reliably beat fixed stops
on this sample — the result is non-monotonic in N, and the one setting that wins is driven by
two names out of five. The leveraged-ETF volatility gate **fails decisively** in all four pairs
at every threshold tested.

---

## TEST 1 — ATR-scaled stops vs fixed-percentage stops

### Conventions (applied identically to every variant)

| Item | Choice |
|---|---|
| Trail anchor | running maximum of **close** since entry |
| Stop distance | `max(0.15 × peak, N × ATR_t)`; fixed variant is `0.15 × peak` |
| ATR | Wilder-smoothed 20-day, current day, warmed up from 2026-02-01 |
| Ratchet | stop level never moves down once set (see note) |
| Breach test | on **close**; fill assumed at that close |
| Sizing | $10,000 per name at the entry close |
| Prices | unadjusted close (`Adj Close` ≈ `Close` for all five; no splits since 2025 — verified) |

*Note on the ratchet — this choice is load-bearing.* It is a no-op for the fixed variant, since
`0.15 × peak` already rises monotonically with the peak. It binds for the ATR variants, where an
expanding ATR would otherwise pull the level *down* mid-trade. I chose to forbid that: letting a
stop loosen after a volatility spike is moving the goalposts. Re-running with the ratchet removed
(aggregate values on $50,000):

| Variant | Ratcheted | Non-ratcheted | Diff |
|---|---|---|---|
| fixed 15% | $46,501 | $46,501 | $0 |
| max(15%, 2.0×ATR) | $46,501 | $46,406 | −$95 |
| max(15%, 2.5×ATR) | $44,558 | $44,558 | $0 |
| max(15%, 3.0×ATR) | $44,558 | $43,471 | −$1,087 |
| max(15%, 3.5×ATR) | $51,441 | $49,868 | −$1,573 |

The main verdict is unchanged — still non-monotonic, still only N=3.5 wins, still N=2.5/3.0
losing to fixed. But one claim does flip: **ratcheted, N=3.5 ($51,441) beats holding with no stop
at all ($50,210); non-ratcheted, it no longer does ($49,868).** So the single best-looking result
in Test 1 depends on an implementation detail I picked, which is one more reason not to act on it.
Note also the direction: removing the ratchet gives ATR stops *more* room and makes them *worse* —
the same later-means-lower effect described in the verdict.

### An important mismatch you should know about

**A 15% trailing stop does not reproduce the stop-outs you listed.** Under the convention above
it fires far earlier in four of the five names:

| Name | Your system stopped | A 15% trail would have fired | Gap |
|---|---|---|---|
| MRVL | 2026-07-15 @ 203.89 | 2026-06-05 @ 263.47 | 40 days earlier |
| MU | 2026-07-16 @ 852.55 | 2026-07-28 @ 820.53 | 12 days *later* |
| VRT | 2026-07-17 @ 273.00 | 2026-06-10 @ 280.98 | 37 days earlier |
| NBIS | 2026-07-29 @ 160.79 | 2026-07-16 @ 171.77 | 13 days earlier |
| VST | 2026-08-07 @ 136.11 | 2026-07-29 @ 142.81 | 9 days earlier |

Your five fills are all real — each sits inside its day's high–low range — so the dates and
prices are genuine. But whatever exit rule produced them, it was not a 15%-from-peak trailing
stop. Implied peak-to-exit drawdowns range from 13% (MU) to 36% (MRVL).

**Note the direction is inconsistent.** For MRVL, VRT, NBIS and VST your live system held on
*longer* than a 15% trail would have; for MU it exited *sooner*. A rule that is simultaneously
looser than 15% on four names and tighter on a fifth is not a mis-specified trailing percentage
— it points to a different rule family altogether (a volatility- or level-based stop, a
time/signal exit, or discretion). Worth pinning down before you tune stop parameters at all,
since the thing you'd be tuning isn't the thing that produced these exits.

The consequence: for MRVL, VRT, NBIS and VST the question "would an ATR stop have avoided the
stop-out?" is largely moot, because under a 15%-trailing convention the position was already
closed *weeks before* your actual exit date. I ran the comparison you asked for as specified,
but the last column below is answered against your historical dates and should be read with
that caveat.

It is labelled **"Held past your stop date?"** rather than "avoided the stop-out?" on purpose.
Surviving past your historical exit date is not the same as doing better than it: a wider stop
that holds two weeks longer and then exits 13% lower has *not* avoided anything worth avoiding.
MU is exactly that case — read it together with the exit price, not on its own.

### Per-name results

$10,000 per position, entry close → exit close, or held to 2026-08-13 if never stopped.

**MRVL** — entry 2026-06-01 @ 219.43 · ATR 12.09 (5.5% of price) · close 8/13 **222.18**

| Variant | Fired | Date | Exit | Value | Return | Held past your stop date? |
|---|---|---|---|---|---|---|
| fixed 15% | yes | 2026-06-05 | 263.47 | $12,007 | +20.1% | no |
| max(15%, 2.0×ATR) | yes | 2026-06-05 | 263.47 | $12,007 | +20.1% | no |
| max(15%, 2.5×ATR) | yes | 2026-06-05 | 263.47 | $12,007 | +20.1% | no |
| max(15%, 3.0×ATR) | yes | 2026-06-05 | 263.47 | $12,007 | +20.1% | no |
| max(15%, 3.5×ATR) | yes | 2026-06-10 | 252.59 | $11,511 | +15.1% | no |

MRVL ran 219 → 316 in three sessions then round-tripped. Every stop *locked in a large gain*
here; holding to 8/13 would have returned only +1.3%. The stop was the right call.

**MU** — entry 2026-07-13 @ 937.00 · ATR 82.67 (8.8% of price) · close 8/13 **949.83**

| Variant | Fired | Date | Exit | Value | Return | Held past your stop date? |
|---|---|---|---|---|---|---|
| fixed 15% | yes | 2026-07-28 | 820.53 | $8,757 | −12.4% | YES |
| max(15%, 2.0×ATR) | yes | 2026-07-28 | 820.53 | $8,757 | −12.4% | YES |
| max(15%, 2.5×ATR) | yes | 2026-07-29 | 739.00 | $7,887 | −21.1% | YES |
| max(15%, 3.0×ATR) | yes | 2026-07-29 | 739.00 | $7,887 | −21.1% | YES |
| max(15%, 3.5×ATR) | **no** | — | — | $10,137 | +1.4% | YES |

**Read the "YES" column here with care — four of those five are not wins.** Every variant held
past 07-16, but the fixed, 2.0× and 2.5×/3.0× variants all then exited at 820.53 or 739.00,
i.e. *below* your historical fill of 852.55. They outlasted your stop date and lost more money
doing it. Only N=3.5 — which never fired at all — genuinely beat the historical exit.

Note the trap: widening from 2.0× to 2.5× ATR made things **worse**, not better (−21.1% vs
−12.4%). The wider stop held through more decline and exited lower.

**VRT** — entry 2026-06-01 @ 323.39 · ATR 16.80 (5.2% of price) · close 8/13 **287.07**

| Variant | Fired | Date | Exit | Value | Return | Held past your stop date? |
|---|---|---|---|---|---|---|
| fixed 15% | yes | 2026-06-10 | 280.98 | $8,689 | −13.1% | no |
| max(15%, 2.0×ATR) | yes | 2026-06-10 | 280.98 | $8,689 | −13.1% | no |
| max(15%, 2.5×ATR) | yes | 2026-06-10 | 280.98 | $8,689 | −13.1% | no |
| max(15%, 3.0×ATR) | yes | 2026-06-10 | 280.98 | $8,689 | −13.1% | no |
| max(15%, 3.5×ATR) | yes | 2026-07-17 | 289.56 | $8,954 | −10.5% | no |

VRT never recovered — it closed 8/13 *below* every stop level. Wider stops barely helped.

**NBIS** — entry 2026-07-10 @ 219.65 · ATR 24.38 (11.1% of price) · close 8/13 **255.04**

| Variant | Fired | Date | Exit | Value | Return | Held past your stop date? |
|---|---|---|---|---|---|---|
| fixed 15% | yes | 2026-07-16 | 171.77 | $7,820 | −21.8% | no |
| max(15%, 2.0×ATR) | yes | 2026-07-16 | 171.77 | $7,820 | −21.8% | no |
| max(15%, 2.5×ATR) | yes | 2026-07-29 | 148.22 | $6,748 | −32.5% | no |
| max(15%, 3.0×ATR) | yes | 2026-07-29 | 148.22 | $6,748 | −32.5% | no |
| max(15%, 3.5×ATR) | **no** | — | — | $11,611 | +16.1% | YES |

The clearest illustration of the whole problem. NBIS is the biggest rally-back (+16% if held),
but the intermediate ATR settings (2.5×, 3.0×) are the **worst outcomes in the entire test**
(−32.5%) — worse than the fixed stop by nearly 11 points. Only the widest setting captures the
recovery. There is no smooth "wider is better" gradient.

**VST** — entry 2026-06-01 @ 154.76 · ATR 7.00 (4.5% of price) · close 8/13 **146.40**

| Variant | Fired | Date | Exit | Value | Return | Held past your stop date? |
|---|---|---|---|---|---|---|
| fixed 15% | yes | 2026-07-29 | 142.81 | $9,228 | −7.7% | no |
| max(15%, 2.0×ATR) | yes | 2026-07-29 | 142.81 | $9,228 | −7.7% | no |
| max(15%, 2.5×ATR) | yes | 2026-07-29 | 142.81 | $9,228 | −7.7% | no |
| max(15%, 3.0×ATR) | yes | 2026-07-29 | 142.81 | $9,228 | −7.7% | no |
| max(15%, 3.5×ATR) | yes | 2026-07-29 | 142.81 | $9,228 | −7.7% | no |

All five variants produce an identical exit, but not for a uniform reason. Checking
`N × ATR_t` against `0.15 × peak_t` on each of the 41 days in the holding window: for
**N = 2.0, 2.5 and 3.0 the ATR term never binds on any day** (peak ratio 0.60, 0.75, 0.91), so
those three are mathematically the same stop as fixed 15%. For **N = 3.5 the ATR term does bind,
on 9 of 41 days** — but only marginally, by at most 5.6% (peak ratio 1.056), and by the 07-29
exit the two terms had converged to within a cent (3.5 × ATR = 25.35 vs 0.15 × peak = 25.35).
So the exit is unchanged. VST's ATR is simply too small relative to price (4.5%) for ATR scaling
to matter at these multiples.

### Aggregate — $50,000 deployed across five positions

| Variant | Stop-outs | Total value | Return | vs fixed |
|---|---|---|---|---|
| fixed 15% | 5 | $46,501 | −7.0% | — |
| max(15%, 2.0×ATR) | 5 | $46,501 | −7.0% | $0 |
| max(15%, 2.5×ATR) | 5 | $44,558 | −10.9% | **−$1,942** |
| max(15%, 3.0×ATR) | 5 | $44,558 | −10.9% | **−$1,942** |
| max(15%, 3.5×ATR) | 3 | $51,441 | +2.9% | **+$4,940** |
| *no stop at all* | 0 | $50,210 | +0.4% | +$3,710 |

### Verdict: ATR stops do not beat fixed stops here

Stated plainly:

1. **The result is non-monotonic, which is the tell.** N=2.0 changes nothing, N=2.5 and N=3.0
   *lose* $1,942 against the fixed stop, and only N=3.5 wins. A rule that gets worse before it
   gets better as you widen it is not capturing a real effect — it is landing on the specific
   path each name happened to take.

2. **A common intuition is wrong here, and it matters.** `max(15%, N×ATR)` is by construction
   never *tighter* than the fixed stop, so it can only ever exit at the same time or **later**.
   It's tempting to conclude it must therefore do at least as well. It doesn't: exiting later
   often means exiting **lower**. NBIS is the proof — fixed exits 7/16 @ 171.77, N=2.5 exits
   7/29 @ 148.22. Same rule family, wider stop, 11 points worse.

3. **The one winning setting rests on two names, and on a modelling choice.** N=3.5's entire edge
   comes from MU and NBIS avoiding an exit altogether. Change one of those paths and the
   advantage disappears — and as shown above, dropping the ratchet is enough to take N=3.5 back
   below the do-nothing baseline.

4. **Stops of any kind mostly hurt on this sample — and that is a warning, not a finding.**
   "No stop at all" (+0.4%) beats the fixed stop and both middle ATR settings. That is exactly
   what you'd expect given how the sample was built (see below), and it is *not* evidence that
   you should trade without stops.

### On sample size — the problem is worse than n=5

The n=5 sample size is small, but selection bias is the bigger issue:

**These five names were chosen *because* they stopped out and several rallied back.** That
conditions the sample on the outcome being tested. Any comparison of "would a wider stop have
done better?" run on a set of positions selected for having recovered is biased toward wider
stops before a single number is computed. The +$4,940 for N=3.5 is close to meaningless as
evidence.

To actually test this you'd need every position the system took over a defined period —
including the ones where the stop correctly cut a loss that kept going. MRVL is the reminder
that those exist: the stop banked +20% on a name that ended the window at +1.3%. A sample built
only from regretted exits will never show you that side.

**Recommendation:** don't change the stop rule on this evidence. Re-run against the full trade
log before touching anything.

---

## TEST 2 — Leveraged-ETF volatility gate

### Setup

Period 2023-01-01 → 2026-08-13 (906 trading days). Data downloaded from 2021-06-01 so the 200d
MA and the 252-day volatility percentile are fully warm at the start.

Gate, evaluated on the **underlying** at the close of day *t*, position held day *t+1*
(no lookahead):

```
close > 50d MA  AND  close > 200d MA  AND  50d MA > 200d MA
AND  20d annualised realised vol  <  trailing 252d median of that same vol series
```
Otherwise hold cash. Headline uses **cash = 0%**; a 5% cash sensitivity is reported below.
Returns from adjusted close. Sharpe is computed with rf = 0, identically for all three legs.

### Main comparison (50th-percentile gate)

| Pair | Strategy | CAGR | Max DD | Sharpe | Time in mkt | Growth |
|---|---|---|---|---|---|---|
| **SOXL/SMH** | (a) buy & hold SOXL | **113.2%** | −87.9% | 1.24 | 100% | 15.36× |
| | (b) gated SOXL | 4.8% | −61.9% | 0.36 | 35.4% | 1.18× |
| | (c) buy & hold SMH | 63.4% | −35.7% | **1.56** | 100% | 5.88× |
| **DFEN/ITA** | (a) buy & hold DFEN | **59.6%** | −65.1% | 0.82 | 100% | 5.40× |
| | (b) gated DFEN | 2.6% | −39.2% | 0.23 | 38.7% | 1.10× |
| | (c) buy & hold ITA | 25.8% | −15.8% | **1.29** | 100% | 2.29× |
| **ERX/XLE** | (a) buy & hold ERX | **14.6%** | −42.3% | 0.53 | 100% | 1.63× |
| | (b) gated ERX | −22.2% | −64.7% | −1.05 | 27.0% | 0.40× |
| | (c) buy & hold XLE | 13.2% | −20.1% | **0.67** | 100% | 1.57× |
| **LABU/XBI** | (a) buy & hold LABU | **20.7%** | −79.4% | 0.64 | 100% | 1.97× |
| | (b) gated LABU | −2.9% | −52.3% | 0.10 | 23.1% | 0.90× |
| | (c) buy & hold XBI | 19.5% | −33.0% | **0.79** | 100% | 1.90× |

**The gate loses to both benchmarks in all four pairs, on CAGR and on Sharpe.** It does reduce
maximum drawdown in three of four (ERX is worse), but it gives up so much return that the
trade-off is not close. Gated ERX turns $1.00 into $0.40 while the 3x buy-and-hold returns
+63% and the underlying +57%.

### Why it fails

The gate is not merely uninformative — it is **anti-selective**. Annualised return of the 3x ETF
on days the gate was ON versus OFF:

| Pair | Gate ON | Gate OFF | ON days | OFF days |
|---|---|---|---|---|
| SOXL | +54.8% | **+185.3%** | 321 | 585 |
| DFEN | +16.4% | **+147.7%** | 351 | 555 |
| ERX | **−84.2%** | +63.6% | 245 | 661 |
| LABU | +16.4% | **+63.1%** | 209 | 697 |

In every pair the excluded days outperform the included ones, and for ERX the gate is in the
market almost exclusively for the disastrous stretches. The premise is where it breaks: the
model assumes high realised volatility is a proxy for *downside* risk. In these sectors it
isn't — the strongest advances in semis, defence and biotech come **with** elevated realised
volatility, because big up-days raise the 20-day vol measurement exactly like big down-days do.
Filtering for low realised vol filters out the upside, and demanding low vol *simultaneously*
with a full 50/200 MA trend stack leaves only 23–39% of days, mostly quiet drifting markets.

The stated premise about drag is correct, incidentally — it just doesn't rescue the gate. A
theoretical daily-rebalanced 3× of each underlying (vol drag included, costs excluded) versus
the actual ETF: DFEN 8.05× vs 5.40×, ERX 2.21× vs 1.63×, LABU 3.04× vs 1.97× — roughly 9–13%
per year of fees and financing, as expected. But over 2023–2026 the underlying trends were
strong enough that the 3x ETFs still beat their underlyings outright.

### Threshold sweep — is it robust?

CAGR% / MaxDD% / Sharpe / Time-in-market%, cash = 0%:

| Pair | p40 | p50 | p60 | p75 |
|---|---|---|---|---|
| SOXL/SMH | −1.5 / −65.1 / 0.21 / 29 | 4.8 / −61.9 / 0.36 / 35 | **24.3** / −63.4 / 0.67 / 42 | 13.1 / −70.3 / 0.52 / 51 |
| DFEN/ITA | **8.7** / −33.8 / 0.46 / 33 | 2.6 / −39.2 / 0.23 / 39 | 2.7 / −67.7 / 0.37 / 41 | −2.2 / −67.4 / 0.33 / 48 |
| ERX/XLE | −20.5 / −64.7 / −1.00 / 24 | −22.2 / −64.7 / −1.05 / 27 | **−18.5** / −68.2 / −0.76 / 30 | −20.6 / −71.6 / −0.80 / 35 |
| LABU/XBI | −8.5 / −54.2 / −0.12 / 18 | −2.9 / −52.3 / 0.10 / 23 | **−2.3** / −60.2 / 0.14 / 29 | −6.1 / −52.2 / 0.07 / 34 |

**Not robust — and no setting rescues it.**

- The best threshold is **not consistent across pairs**: p60 for SOXL, ERX and LABU; p40 for DFEN.
- Results are **non-monotonic** in every pair — SOXL swings −1.5% → 4.8% → 24.3% → 13.1%. A real
  effect would trend with the threshold; this wanders.
- SOXL's spread across four settings is **26 percentage points of CAGR** from a single parameter
  choice. That is parameter sensitivity, not signal.
- Most decisively: **even each pair's best setting loses to simply buying the underlying.**
  SOXL's best is 24.3% vs SMH's 63.4%. DFEN's best is 8.7% vs ITA's 25.8%. ERX and LABU are
  negative at every threshold.

The apparent 24.3% at SOXL/p60 is the kind of number that would look like a discovery if you
only ran that one configuration. Seen next to its neighbours, it's noise.

### Cash-rate sensitivity

0% cash penalises the gated strategy, which sits out 61–77% of days. At 5%:

| Pair | Gated CAGR, 0% cash | Gated CAGR, 5% cash |
|---|---|---|
| SOXL | 4.8% | 8.1% |
| DFEN | 2.6% | 5.7% |
| ERX | −22.2% | −19.4% |
| LABU | −2.9% | 0.8% |

Worth about +3pp. **It changes nothing** — every gated variant still loses badly to both
benchmarks.

### Verdict: the gate does not work

It fails in all four pairs, on CAGR and Sharpe, at all four thresholds, under both cash
assumptions. The drawdown reduction is real but far too expensive. This strategy would not be
deployed, and it doesn't look salvageable by tuning the threshold — the anti-selectivity table
shows the signal has the wrong sign for these sectors, and tuning a wrong-signed signal just
relocates the overfit.

---

## Caveats

- **Test 1's sample is selected on the outcome being tested** (see above). This is the single
  biggest limitation in this document.
- **Test 1 entry dates for MRVL, VRT and VST are assumed** (2026-06-01). All three are sensitive
  to that assumption because each moved sharply in early June.
- **SOXL/SMH is not a clean pair.** SOXL tracks the ICE Semiconductor Index; SMH tracks the
  MVIS US Listed Semiconductor 25. They are different indices, which is why SOXL's gap to a
  simulated 3× SMH (15.36× vs 51.10×) is far wider than the 9–13% annual cost seen in the other
  three, correctly-matched pairs. The gate signal for SOXL is therefore computed on something
  that is not its actual underlying. DFEN/ITA, ERX/XLE and LABU/XBI are properly matched.
- **No transaction costs, slippage, spreads or taxes** anywhere. The gated strategies trade
  frequently and would look worse still; Test 1 assumes fills exactly at the closing price.
- **Test 2 covers a single strong bull regime** (2023–2026). A trend-and-vol filter would
  plausibly look better across a bear market; that is not evidence it works, only that this
  window doesn't test it. The reverse caution also applies to the buy-and-hold comparisons.
- Stop breaches are tested on the close, so intraday gaps below the level are not modelled;
  real fills would be at or below the prices shown.

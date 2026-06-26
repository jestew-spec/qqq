# Strategy Spec: DL35+DC — QQQM/TQQQ/CSHI

## Instruments
- Underlying (1x): QQQM
- Levered (3x): TQQQ
- Cash / bear shield: CSHI  (treated as cash-equivalent for allocation math)
- Signal source: QQQM daily closes

## Signals (computed daily by data bot)
- MA35: 35-day simple moving average of QQQM close
- MA50: 50-day simple moving average of QQQM close
- MA200: 200-day simple moving average of QQQM close
- golden_cross: MA50 > MA200
- above_50: close > MA50
- above_35: close > MA35

## Tie-break (MA50 exactly equals MA200)
Read prior trading day's data file (data/daily/) for its golden_cross:
- Was golden yesterday → treat today as death cross (golden_cross = false)
- Was not golden yesterday → treat today as golden cross (golden_cross = true)
- No prior file exists → golden_cross = false (conservative; no history
  to extrapolate, so default to the cautious branch)

## Decision logic (WHICH position — never changed by sizing)
```
if current == TQQQ:
    if not above_35 or not above_50 → QQQM  # trip-wire
    if not golden_cross             → QQQM  # death-cross gate
    else                            → TQQQ

if current == CSHI:
    if golden_cross and above_50 and above_35 → TQQQ
    if not golden_cross and not above_50      → CSHI
    else                                      → QQQM

if current == QQQM:
    if not golden_cross and not above_50      → CSHI
    if above_50 and above_35 and golden_cross → TQQQ
    else                                      → QQQM
```
The decision logic answers WHICH instrument. HOW MUCH is the exec bot's
sizing/resource model below.

## Sizing & resource model (HOW MUCH — see exec bot prompt for the algorithm)
- Orders: market orders, regular hours only. (Limit orders are NOT used.)
- target_pct is a human-set field in the state file (currently 0.95). It is
  NOT auto-derived. Edit it by hand to change the bot's soft target.
- base = total NON-MARGINED account equity, computed live only when a sizing
  decision is needed = market value of held TQQQ/QQQM/CSHI + uninvested cash.
  CSHI and cash are the same bucket. Margin / buying-power is never used.
- soft_target_usd = base × target_pct.

Two-way ratchet (the core rule):
- INCREASE the bot's deployed allocation ONLY on entry into TQQQ. A TQQQ
  entry buys max(proceeds_of_sale, soft_target_usd) — it can pull idle cash
  up to target, but never trims below what it carried in.
- DECREASE the bot's deployed allocation ONLY when entering OR holding CSHI.
  CSHI is sized to min(current_cshi_or_proceeds, soft_target_usd); any surplus
  is left as cash. 1:1 "hold the line" when target_pct is unchanged; sheds to
  the new target when target_pct was lowered (even while just holding CSHI).
- QQQM moves (TQQQ→QQQM, CSHI→QQQM) redeploy 100% of proceeds, no resize.
- While FLAT (never-deployed CASH), only a TQQQ signal deploys (sized to
  soft_target_usd). A QQQM or CSHI signal leaves the bot flat.

Consequences (intentional, follow directly from the rules):
- The cash buffer (the 1 − target_pct slice) is established only by passing
  through CSHI. A QQQM→TQQQ flip carries the full QQQM value into TQQQ even if
  that exceeds soft_target_usd, because trimming there would be a reduction
  off-CSHI, which the ratchet forbids.
- base = whole account equity. Correct only while ONE bot holds all positions.
  Multiple bots sharing the account (esp. multiple CSHI/cash holders) would
  each compute base off the same pool and over-claim. OPEN FARM-DESIGN GAP.

- On data failure: hold, do not trade.

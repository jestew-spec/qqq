# Strategy Spec: DL35+DC — QQQM/TQQQ/CSHI

## Instruments
- Underlying (1x): QQQM
- Levered (3x): TQQQ
- Cash: CSHI
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

## Decision logic
```
if current == TQQQ:
    if not above_35 or not above_50 → QQQM  # trip-wire
    if not golden_cross             → QQQM  # death-cross gate
    else                            → TQQQ

if current == CSHI:
    if golden_cross and above_50              → TQQQ
    if not golden_cross and not above_50      → CSHI
    else                                      → QQQM

if current == QQQM:
    if not golden_cross and not above_50      → CSHI
    if above_50 and above_35 and golden_cross → TQQQ
    else                                      → QQQM
```

## Execution
- Market orders, 100% position swaps, market open only
- On data failure: hold, do not trade

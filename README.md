# Bot Farm Database

## Structure
```
/spec/          Strategy specs (human-edited, never overwritten by bots)
/state/         Current position and allocation per bot (written by exec bots)
/data/daily/    Daily signal files written by data bot (YYYY-MM-DD.json)
/logs/data/     Data bot run logs (YYYY-MM-DD.md)
/logs/exec/     Execution bot run logs per bot (bot_name/YYYY-MM-DD.md)
```

## Bots
| Bot | Spec | State | Account |
|-----|------|-------|---------|
| qqqm_tqqq | spec/strategy_qqqm_tqqq.md | state/bot_qqqm_tqqq.json | 665701595 |

## Data bot schedule
- 4:05 PM ET weekdays (primary — after close)
- 9:00 AM ET weekdays (health check — before open)

## Execution bot schedule
- 9:35 AM ET weekdays

# Momentum-Strategy# Momentum-Strategy

A clean, lookahead-free **daily momentum / breakout backtesting engine** built in Python.
Implements the canonical Donchian-channel trend-following system: N-day breakout entries,
fixed ATR stops, and a trailing channel exit that lets winners run.

> ⚠️ This is a standalone **daily-bar system** — it is not a port of any intraday framework.
> All signals fire on closed daily bars only.

---

## 📈 System Rules

### Entry (signal on a closed daily bar, flat only)
| Direction | Condition |
|-----------|-----------|
| **Long**  | `close[i] > max(high[i-N .. i-1])` — N-day high breakout |
| **Short** | `close[i] < min(low[i-N .. i-1])` — N-day low breakout |

**Optional trend filter:** longs only when `close > SMA(trend)`, shorts mirrored.

### Execution
- **Fill:** next bar's open + cost (no same-bar fills)
- **Cost:** `cost_atr_frac × ATR`, charged per side
- **Stop:** fixed at entry — `entry ∓ atr_mult × ATR_entry`
  - `ATR_entry` = Wilder ATR-14 of the **prior completed bar** (no lookahead)

### Exit
1. **Stop hit intrabar** → fills at the stop price, or
2. **Opposite Donchian(M = N/2) channel break on close** → fills at the **next open**
   *(the classic "turtle" trailing exit — captures momentum's fat right tail)*

### Results
- Trades are measured in **R-multiples**: `R = pnl_pts / risk_pts`
- Costs are applied on both entry and exit

---

## 🔒 No-Lookahead Guarantees

- All rolling windows are strictly trailing (`.shift(1)`)
- Signals are evaluated on **closed bars only**
- Entry ATR is taken from the **prior** completed bar
- Fills always occur at the **next** bar's open — never same-bar

---

## 🚀 Quick Start

```python
import pandas as pd
from dm_engine import run_momentum

# Daily OHLC data with a DatetimeIndex and columns: open, high, low, close
df = pd.read_csv("SPX_daily.csv", index_col="date", parse_dates=True)

trades = run_momentum(
    df,
    entry_n=55,          # Donchian entry lookback (e.g. 20 or 55)
    atr_mult=2.5,        # stop distance in ATRs
    trend_win=200,       # optional SMA trend filter (None to disable)
    cost_atr_frac=0.05,  # cost per side as a fraction of ATR
    allow_long=True,
    allow_short=True,
)

print(trades.tail())
print(f"Trades: {len(trades)} | Total R: {trades['R'].sum():.1f} "
      f"| Avg R: {trades['R'].mean():.3f}")
```

### Trades DataFrame output

| Column       | Description                                  |
|--------------|----------------------------------------------|
| `entry_date` | Trade entry date                             |
| `exit_date`  | Trade exit date                              |
| `side`       | `+1` long / `-1` short                       |
| `entry_px`   | Fill price incl. cost                        |
| `exit_px`    | Exit price incl. cost                        |
| `R`          | Trade result in R-multiples                  |
| `bars_held`  | Holding period in bars                       |
| `reason`     | `"stop"` or `"channel"` exit                 |

---

## ⚙️ Parameters

| Parameter       | Default | Description                                          |
|-----------------|---------|------------------------------------------------------|
| `entry_n`       | —       | Donchian entry lookback (N-day breakout)             |
| `atr_mult`      | `2.5`   | Fixed stop distance in ATR multiples                 |
| `trend_win`     | `None`  | SMA trend-filter window (`None` = disabled)          |
| `cost_atr_frac` | `0.05`  | Transaction cost per side as fraction of ATR         |
| `allow_long`    | `True`  | Enable long trades                                   |
| `allow_short`   | `True`  | Enable short trades                                  |

The trailing exit channel is derived automatically: `exit_n = max(5, entry_n // 2)`.

---

## 📦 Requirements

```
numpy
pandas
```

```bash
pip install numpy pandas
```

---

## 📄 File

- **`dm_engine.py`** — feature builder, bar-by-bar backtest loop, and trade booking

---

## ⚠️ Disclaimer

This project is for **educational and research purposes only**. Nothing here is
financial advice. Backtested performance does not guarantee future results —
trade at your own risk.

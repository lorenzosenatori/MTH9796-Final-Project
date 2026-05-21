# Earnings-Call ATC Signal Backtest

A look-ahead-free backtest of ProntoNLP's earnings-call ATC signal on three
US equity universes (S&P 500, S&P 1500, Russell 3000), with a LightGBM model
trained on engineered AspectTheme features.

## Headline result

| Universe | OOS months | ATC alone (Sharpe net) | LightGBM enhanced (Sharpe net) |
|----------|-----------:|-----------------------:|-------------------------------:|
| SP500    | 48         | −0.16                  | **+0.78**                      |
| SP1500   | 63         | +0.01                  | **+0.60**                      |
| RU3K     | 64         | −0.11                  | **+0.78**                      |

Monthly rebalance, 10-day hold, 5 bps/side transaction cost, 2020Q1+ out-of-sample.

**Recommended deployment:** Russell 3000 + LightGBM enhanced model + monthly rebalance.

## Quick start

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Place input files in project root
#    - Earnings_ATC_until_2026-04-21.csv  (the raw 4.5 GB signal file)
#    - IWV_holdings.csv                    (Russell 3000 constituents from iShares)

# 3. Open the notebook and run all cells
jupyter notebook NLP_Final_Project.ipynb
# Then: Cell → Run All
```

## Inputs you need to source

### 1. The signal file (provided by instructor)
`Earnings_ATC_until_2026-04-21.csv` (4.5 GB) — place in project root.

### 2. CRSP prices (WRDS, first run only)
First-time setup requires a WRDS account (free for academic use):
```bash
pip install wrds
python -c "import wrds; wrds.Connection()"   # creates ~/.pgpass
```
After the first run, prices are cached to `cache/crsp_prices.parquet` and
WRDS is never re-hit.

### 3. iShares Russell 3000 holdings (manual download)
iShares blocks automated downloads, so you need to download once manually:

1. Open <https://www.ishares.com/us/products/239714/ishares-russell-3000-etf>
2. Scroll to **Holdings** → click **Detailed Holdings and Analytics**
3. Save the downloaded CSV as `IWV_holdings.csv` in the project root

### 4. yfinance prices (automatic, first run only)
yfinance is hit once for S&P 400/600 names and Russell 3000-only tickers,
then cached to `cache/yf_prices_sp1500.parquet` and
`cache/yf_prices_ru3k_extra.parquet`. Subsequent runs skip the network.

## Project structure

```
.
├── NLP_Final_Project.ipynb          # Main notebook (all analysis)
├── README.md                         # This file
├── requirements.txt                  # Python dependencies
├── Earnings_ATC_until_2026-04-21.csv # Input: ProntoNLP signal (not in repo)
├── IWV_holdings.csv                  # Input: Russell 3000 constituents
├── cache/                            # Auto-built on first run
│   ├── atc_slim.parquet              #   Signal subset (~200 MB)
│   ├── sp_constituents.parquet       #   S&P 500 GVKEYs
│   ├── crsp_prices.parquet           #   CRSP daily prices
│   ├── aspect_features.parquet       #   Engineered AspectTheme features
│   ├── yf_prices_sp1500.parquet      #   yfinance for SP400+SP600
│   └── yf_prices_ru3k_extra.parquet  #   yfinance for RU3K-only tickers
└── results/                          # Charts and audit checklist
    ├── decile_spread.png
    ├── equity_curve_ls.png
    ├── multi_universe_full.png
    ├── ic_heatmap.png
    ├── sp500_vs_sp1500.png
    ├── signaltype_comparison.png
    ├── robustness_subperiod.png
    ├── portfolio_sim.png
    ├── decile_spread_both.png
    ├── long_short_only.png
    ├── mktcap_buckets.png
    └── look_ahead_audit.txt
```

## Reproducibility notes

- **First run (fresh machine):** ~30 minutes total. Cell 2 takes ~2 min to
  ingest the CSV, Cell 5 takes ~5 min to download CRSP prices, Cell 14 takes
  ~10 min to download yfinance prices for SP400+SP600, Cell 17 takes ~10 min
  for the Russell 3000 extras. Everything else is fast.
- **Subsequent runs:** ~2 minutes (everything reads from cache).
- **Cache is preserved across runs.** Delete `cache/` to force a full rebuild.
- **The notebook is idempotent.** Re-running any cell produces identical output.

## Look-ahead audit

A 10-item audit checklist (per handout §3) is in **Cell 20** of the notebook
and saved to `results/look_ahead_audit.txt`. Summary: 8 PASS, 2 CAVEATS,
0 FAIL. The single caveat is:

- **§3.6 Universe membership.** Constituent lists are current-only, not
  point-in-time. Reported alpha is an upper bound (survivorship bias).

§3.7 (INGESTDATEUTC) passes with documented investigation: 77% of apparent
ingestion delays are bulk-backfill artifacts, not real availability lags (see
Cell 6b and research PDF §4.3).

See the research PDF for full discussion.

## Dependencies

```
pandas>=2.0
numpy>=1.24
pyarrow>=14
scipy>=1.10
scikit-learn>=1.3
lightgbm>=4.0
matplotlib>=3.7
yfinance>=0.2.40
wrds>=3.1
requests>=2.31
tqdm>=4.65
```

## Known limitations

1. **Universe membership is not point-in-time.** SP500 list is current
   Compustat members; SP400/600 are current Wikipedia; RU3K is current
   iShares IWV. Survivorship bias acknowledged.
2. **Mixed price sources within SP1500/RU3K.** SP500 uses CRSP adjusted
   returns; SP400/600 and RU3K-extras use yfinance auto-adjusted prices.
3. **CRSP coverage ends ~2024-11; yfinance to 2025-04.** SP500 has ~4 fewer months
   of recent data than SP1500/RU3K. A truncated-period analysis (Cell 22b)
   confirms findings hold on the common date range.
4. **INGESTDATEUTC not enforced** as a secondary filter (see audit caveat).
5. **63 features used in enhanced model**, vs the 50–150 range suggested
   in the handout. Feature space is dominated by AspectTheme aggregations.
6. **20 of 922 RU3K-only tickers (2.2%) had no yfinance data** (delisted
   small caps). Affects breadth but not methodology.
7. **CRSP delisting returns (`dlret`) not incorporated.** Minor caveat on
   the short side.

## Author

[Your name], [Course], [Date]
# Intraday Top 5 — NSE multi-agent screener

Scans the Nifty LargeMidcap 250, runs three agents in parallel, and ranks the Top 5 intraday
setups with Entry, Target and Stop-loss. A web page shows live prices and a chart for each pick
with the three levels drawn on it.

```
             ┌─ News Agent ──────── Google News RSS → Claude scores sentiment ─┐
  universe ──┤                                                                  ├─► Master Agent ─► Top 5
             └─ candles (Kite) ─┬─ Chart Agent ── ORB, PDH/PDL, EMA trend, volume ┤   (merge, backtest,
                                └─ Indicator Agent ─ VWAP, RSI, MACD, Supertrend, ATR → levels ┘    rank)
```

The News Agent starts immediately; Chart and Indicator start as soon as candles arrive.
A full 250-stock scan takes about 1.5–2 minutes on Kite (its historical API allows 3 requests a second).

## Quick start (no broker needed)

```bash
pip install -r requirements.txt
python run_scan.py                 # prints the Top 5 using synthetic data
uvicorn server:app --port 8000     # open http://127.0.0.1:8000
```

`BROKER=mock` is the default, so everything runs on made-up prices until you connect Kite.

## Connect Zerodha Kite

1. Create an app at developers.kite.trade (Kite Connect plan with historical data).
   Set the redirect URL to `http://127.0.0.1:8000/kite/callback`.
2. `cp .env.example .env`, set `BROKER=kite`, `KITE_API_KEY`, `KITE_API_SECRET`.
3. Start the server and select **Connect Kite**. Kite tokens expire every morning, so log in once a day
   (or run `python login_kite.py`).
4. Download the Nifty LargeMidcap 250 constituents CSV from niftyindices.com and save it as
   `data/nifty250.csv`. Without it, a built-in list of about 55 large caps is used.

## News scoring with Claude

Set `ANTHROPIC_API_KEY` in `.env`. Headlines are batched 15 stocks per call to keep cost low
(about 17 calls per full scan). Without a key, headlines still show but sentiment stays neutral.

## How the numbers are made

| Output | Rule |
|---|---|
| Direction | Chart Agent and Indicator Agent must agree, each above a minimum strength |
| Entry | One tick beyond the signal candle's high (buy) or low (sell); valid for 15 minutes |
| Stop-loss | Supertrend line, clamped to 0.75–1.5 × ATR and at least 0.3% away |
| Target | 2 × risk (`RISK_REWARD`) |
| Confidence | Weighted agreement: 40% chart, 40% indicators, 20% news |
| Past win rate | The same rules replayed on this stock's last ~24 sessions of 5-minute data: first setup per day, same-bar stop and target counted as a loss |
| Rank | 70% confidence + 30% past win rate |

At 2:1 reward-to-risk the break-even win rate is about 33%, so read the win rate together with
the average R shown in the detail panel. The backtest does not include news (no historical feed),
brokerage, taxes or slippage.

Every threshold is in `config.py` and can be overridden in `.env`.

## Files

```
config.py            all settings
brokers/             base interface, Kite adapter, mock broker
universe.py          loads data/nifty250.csv
indicators.py        VWAP, RSI, MACD, Supertrend, ATR, EMA
strategy.py          shared signal, levels and backtest logic
agents/              chart, indicator, news, master
market_data.py       parallel, rate-limited candle download
pipeline.py          runs the agents in parallel
server.py            FastAPI: /api/scan, /api/results, /api/candles/{symbol}, /api/ltp
charts.py            candle and daily chart data, shared by server.py and build_site.py
build_site.py        static build for GitHub Pages: writes every API answer to site/api/*.json
static/index.html    the web page (works with the server or with the static build)
.github/workflows/   scans every 5 minutes on market days and publishes to GitHub Pages
```

## Adding Upstox or Angel One

Write a class in `brokers/` with `has()`, `candles()` and `ltp()` (see `brokers/base.py`),
then add it to `brokers/factory.py`.

## Publish on GitHub Pages (no server)

A GitHub Actions workflow runs the scan with Yahoo data every 5 minutes from 9:20 am to 3:30 pm IST,
Monday to Friday, plus once at 3:45 pm for the final next-day list, and publishes the page to
`https://<username>.github.io/<repo>/`. It is free for public repositories.

1. Push this folder to a public GitHub repository.
2. In the repo, open **Settings → Pages** and set **Source** to **GitHub Actions**.
3. Open the **Actions** tab, select **Scan and publish**, then **Run workflow** to build the first copy.
4. Optional: add `ANTHROPIC_API_KEY` under **Settings → Secrets and variables → Actions** for news sentiment.

To preview the build locally: `BROKER=yahoo python build_site.py`, then
`python -m http.server -d site 8001` and open http://127.0.0.1:8001.

Limits compared with running `server.py`:
- The site is public. Anyone with the link can open it, and there is no login.
- Prices refresh with each scan (every 5 minutes), not every few seconds.
- **Reload** fetches the latest scan. To scan right away, use **Run workflow** on the Actions tab.
- GitHub can start scheduled runs 5 to 15 minutes late when it is busy.
- GitHub pauses scheduled workflows after 60 days without a commit to the repo. Re-enable it from the Actions tab.
- NSE holidays are not known, so on a holiday it publishes the last session's data.

## Sharing with family

Set `APP_USER` and `APP_PASSWORD` to put the site behind a login. If you host it beyond your own
machine, check your broker's API terms on sharing market data, and SEBI's rules on giving stock
recommendations to others.

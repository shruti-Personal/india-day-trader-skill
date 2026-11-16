# India Day Trader Skill

11 Claude skills for day trading Indian stocks (NSE): scan thousands of charts, track breaking news, time every entry and exit, and replay every outcome before you risk a rupee.

Nine of the eleven work on NSE. Two do not: `/trader-analyst` and
`/trader-pelosi` read US filings (SEC EDGAR and the US House Clerk) and have
no Indian equivalent, so they are left as they came. They are useful only if
you also follow US stocks.

Free, MIT, no signup, no API key. The tools are plain Python that read free
public data: Yahoo Finance charts (with .NS suffix for NSE stocks) and NSE's
equity lists.

Adapted from [Jakeschincariol/day-trader-skill](https://github.com/Jakeschincariol/day-trader-skill)
for the Indian stock market: NSE trading hours (9:15 AM – 3:30 PM IST),
NSE holiday calendar, INR currency, and Indian market defaults.

## Install

Paste this into Claude Code:

```
https://github.com/shruti-Personal/india-day-trader-skill

install skill
```

Or clone and copy:

```bash
git clone https://github.com/shruti-Personal/india-day-trader-skill.git
cp -r india-day-trader-skill/trader-* ~/.claude/skills/
```

The tools need Python 3.8 or newer. Nothing to pip install.

## Set your limits first

**Do this before the first trade.** Every tool reads `trader/config.json`,
relative to whatever folder you run Claude in. With no budget set, sizing and
the risk check refuse to run at all, which is the point. The other limits are
the quiet ones: leave them out and you silently get ₹1,000 a trade and ₹3,000
a day, whatever your budget says.

```bash
mkdir -p trader
cp ~/.claude/skills/trader-budget/config.example.json trader/config.json
```

Then open `trader/config.json` and set at least these four:

| key | what it means |
| --- | --- |
| `budget` | the money this account trades with, in rupees. Everything is sized off it |
| `max_loss_per_trade` | the most one trade may lose at its stop. Caps position size |
| `max_daily_loss` | trading stops for the day once losses reach this |
| `broker` | `zerodha`, `upstox`, `angelone`, `groww`, `dhan`, `fyers`, `5paisa`, `icicidirect`, `kotak`, `hdfcsky`, `paytmmoney`, `sbisecurities` or `motilaloswal`. Only names the ticket, since no Indian broker connects to Claude |

The shipped values (₹1,00,000 budget, ₹1,000 a trade, ₹3,000 a day) are an
example, not a recommendation. `"paper": true` stays until you change it.

To check it is being read, run `/trader-risk` and look at the limits it
prints back. If they are not your numbers, Claude is not running in the
folder that holds `trader/`.

## The eleven

| command | job | what it does |
| --- | --- | --- |
| `/trader-scanner` | Chart Scanner | Scans thousands of NSE charts for setups: opening-range breakouts, flags, oversold bounces. |
| `/trader-news` | News Monitor | Tracks breaking news on every stock you watch and flags what moves it. |
| `/trader-entry` | Entry Timer | Watches a setup bar by bar and tells you the moment to get in. |
| `/trader-exit` | Exit Planner | Sets your stop and your target before you enter, so you know when to get out. |
| `/trader-timemachine` | Time Machine | Replays every possible outcome of a trade before you risk a rupee: 10,000 simulated sessions plus every past time the setup fired. |
| `/trader-risk` | Risk Manager | Checks every trade against your limits and stops anything over them. |
| `/trader-budget` | Budget Manager | Sizes every position to your budget and your max loss per trade. |
| `/trader-broker` | Order Executor | Sends approved orders to your broker, paper trading by default, never without your yes. |
| `/trader-alerts` | Price Alerts | Rings when a price, a stop or the opening bell hits. |
| `/trader-analyst` | Earnings Analyst | **US only.** Reads a company's reported numbers from SEC EDGAR and sums up what changed. No Indian filings: an NSE ticker returns nothing. |
| `/trader-pelosi` | Insider Tracker | **US only.** Reads the US House Clerk's STOCK Act disclosures (what members of Congress traded). Nothing to do with NSE/BSE or SEBI insider filings. |

## How to use it

Everything lands in a `trader/` folder in your project, and each skill reads
what the last one wrote:

```
scanner -> news -> entry -> exit -> timemachine -> budget -> risk -> broker   (one trade)
alerts: stop, target, flat by the close                                       (while it is open)
news                                                                          (on a schedule)
```

## India-specific defaults

| Setting | Value |
| --- | --- |
| Market hours | 9:15 AM – 3:30 PM IST |
| Timezone | IST (UTC+5:30), no DST |
| Exchange | NSE (Yahoo suffix .NS) |
| Currency | INR (₹) |
| Budget | ₹1,00,000 |
| Max loss per trade | ₹1,000 |
| Daily loss limit | ₹3,000 |
| No entries before | 09:20 IST |
| No entries after | 15:15 IST |
| Flat by | 15:25 IST |
| Holiday calendar | NSE holidays 2026 (2027 not published yet) |

## Fine print

**Not financial advice.** This is software that reads public data and does
arithmetic. It does not know your situation and it cannot see the future.
Most people who day trade lose money. You can lose money. Decide for
yourself, and talk to a SEBI-registered adviser if you need advice.

**No trade goes through until you say yes.** Only `/trader-broker` can lead to
an order, and only after `/trader-risk` passes that exact order and you type
yes for it in the chat.

**Paper first.** `"paper": true` is the default in `config.example.json`.
Keys live in environment variables, never in the chat.

**The Time Machine is a simulation, not a prediction.** It is built from the
stock's own last 60 days of 5-minute bars. The future can do things those 60
days never did. Treat its odds as a reality check, never as a promise.

**A stop is a plan, not a guarantee.** In a fast market or across a gap, a
stop order can fill well past its price.

**The data is free and unofficial.** Yahoo Finance's chart endpoint is not an
official API: it can lag, change or rate-limit you.

**Taxes.** Day-trading gains are taxable. Short-term capital gains on listed
equities are taxed at 20% in India (as of 2026). Nothing here is tax advice.

## Files

```
trader-scanner/     SKILL.md, scan.py
trader-news/        SKILL.md, news.py
trader-entry/       SKILL.md, entry.py
trader-exit/        SKILL.md, exit.py
trader-timemachine/ SKILL.md, timemachine.py
trader-risk/        SKILL.md, check.py
trader-budget/      SKILL.md, size.py, config.example.json, targets.example.json
trader-broker/      SKILL.md, ticket.py
trader-alerts/      SKILL.md, alerts.py, alerts.example.json
trader-analyst/     SKILL.md, facts.py, metrics.example.json
trader-pelosi/      SKILL.md, ptr.py
tests/              unittest suite + offline fixtures
```

MIT. Adapted from [Jake Schincariol's day-trader-skill](https://github.com/Jakeschincariol/day-trader-skill) for Indian markets.

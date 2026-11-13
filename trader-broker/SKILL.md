---
name: trader-broker
description: >-
  Order Executor. Sends approved orders to your broker, paper trading by
  default, never without your yes. Shows each order as a plain ticket (a
  bracket shows its stop-loss, take-profit and the max loss at the stop),
  records your yes word for word, and logs it. No Indian broker has a Claude
  connector, so on NSE the ticket is what you type into your own terminal
  (symbol, MIS, order type, validity, SL-M trigger) and Claude places
  nothing. Builds the sell ticket that closes a day trade by your flat-by
  time, records fills, and logs each trade's max loss so trader-risk can hold
  the daily limit. Also covers the US connectors it can drive, Alpaca's MCP
  server and Interactive Brokers, and setting order tools to "Needs
  approval". Use when the user says "place the order", "buy it", "execute",
  "send it to my broker", "close my position", "flatten", "get me out", "set
  up paper trading", or after /trader-risk passes.
argument-hint: "[order number]"
---

# trader-broker

The Order Executor is the only skill in this pack that can lead to a real
order, so it is the strictest. An order reaches the market only after three
separate gates, every time, for every order:

1. **trader-risk passed this exact order today** (matched by fingerprint).
2. **The user typed an explicit yes for this one order** in the chat.
3. **The user clicked Allow once** on Claude's own permission card, which
   appears because the order tool is set to "Needs approval".

Two yeses from the user, one PASS from the rules. No yes, no order.

**On NSE, Claude places nothing at all.** There is no Zerodha, Upstox, Angel
One or Groww connector here, so gate 3 does not exist: the user places every
order by hand in their own broker's terminal. `ticket.py` still renders the
ticket, still demands the yes, and still keeps the log, because the log is
what trader-risk counts the daily limit from. Treat a recorded yes as a
decision, never as a placed order; only `ticket.py result --status placed`
says it reached a broker, and only the user can say that.

`${CLAUDE_SKILL_DIR}` below is this skill's folder (where this SKILL.md lives).

```bash
python3 "${CLAUDE_SKILL_DIR}/ticket.py" show                     # every ticket, READY or not
python3 "${CLAUDE_SKILL_DIR}/ticket.py" approve --index 1 --reply "yes"
python3 "${CLAUDE_SKILL_DIR}/ticket.py" result --index 1 --status placed --broker-id ID --filled-qty 8 --filled-price 364.20
python3 "${CLAUDE_SKILL_DIR}/ticket.py" fill --symbol TSLA --side sell --qty 8 --price 370.04
python3 "${CLAUDE_SKILL_DIR}/ticket.py" flatten --symbol TSLA --qty 8
python3 "${CLAUDE_SKILL_DIR}/ticket.py" log
```

`ticket.py` never talks to a broker. It renders tickets, checks that a reply
is a real yes for that order, builds the flatten ticket, and keeps the log in
`trader/state/broker.json`. `--now "2026-11-13 10:15"` pins the clock (IST).

## What it reads and writes

| file | what |
| --- | --- |
| `trader/orders.json` | the proposed orders (trader-budget writes it; `flatten` replaces it) |
| `trader/state/risk.json` | trader-risk's verdicts; only a PASS from today is READY |
| `trader/state/broker.json` | every yes, no, placed, failed and filled, with each entry's max loss |
| `trader/portfolio.json` | cash and shares, updated by `fill` and by `result --filled-qty` |
| `trader/plan.json` | the flat-by time for `flatten` |

## Setup: Indian brokers (Zerodha, Upstox, Angel One, Groww, ...)

There is nothing to set up, because there is nothing to connect. Claude has
no connector for any Indian broker, so it cannot place, modify or cancel an
order on NSE. Set `"broker": "zerodha"` (or whichever you use) in
`trader/config.json` and `ticket.py` renders the ticket in the fields your
terminal actually asks for:

```
symbol      INFY (NSE)
side        BUY
quantity    65
product     MIS (intraday). CNC/delivery is a different trade and a different risk
order type  market
validity    good for today
stop-loss   SL-M, trigger 1142.95
target      limit 1155.05
```

You type that into Kite, Pro, or whatever you use. Then tell `ticket.py` what
happened (`result`, then `fill`) so the log and the daily limit stay true.

Two things to say out loud when a bracket is involved:

- **Zerodha retired bracket orders in 2020.** The entry and the stop are two
  separate orders there, and the stop is only live once the user places it.
  Several other Indian brokers did the same; have the user confirm what
  theirs supports rather than assuming a bracket exists.
- **The stop goes on the moment the entry fills.** Between the fill and the
  stop there is no protection at all. This is the single most dangerous gap
  in hand-placed trading, and it does not exist on a connector broker.

On MIS the broker also auto-squares-off near the close, at its own cutoff,
not at your `flat_by`. Being flat before then is the user's job.

## Setup A: Alpaca (official MCP server, Claude Desktop)

Only for US brokers. Skip this whole section on NSE.


Alpaca's own server is github.com/alpacahq/alpaca-mcp-server (MIT). It runs on
the user's computer inside Claude Desktop. It is not in Claude's connector
directory. It trades on a **paper** account unless told otherwise. Checked
against its source on 2026-10-20 (version 2.4.0).

1. The user makes an Alpaca account and creates **paper trading** API keys in
   the Alpaca dashboard. Paper keys are separate from live keys.
2. The user installs `uv` (it provides `uvx`), from docs.astral.sh/uv.
3. In Claude Desktop: Settings > Developer > Edit Config. That opens
   `claude_desktop_config.json` (macOS: `~/Library/Application Support/Claude/`,
   Windows: `%APPDATA%\Claude\`). The user adds this block (also in
   `claude_desktop_config.example.json` here) and types their own keys into the file:

```json
{
  "mcpServers": {
    "alpaca": {
      "command": "uvx",
      "args": ["alpaca-mcp-server"],
      "env": {
        "ALPACA_API_KEY": "paste your paper key here, in this file only",
        "ALPACA_SECRET_KEY": "paste your paper secret here, in this file only",
        "ALPACA_PAPER_TRADE": "true",
        "ALPACA_TOOLSETS": "account,trading,stock-data"
      }
    }
  }
}
```

   The `env` block hands the keys to the server as environment variables.
   **Keys never go in the chat.** Never ask for them, never read this file
   back into the conversation, and if the user pastes a key anyway, tell them
   to delete that message and make new keys in the Alpaca dashboard.
   `ALPACA_PAPER_TRADE` is already true by default; writing it makes it
   visible. `ALPACA_TOOLSETS` is optional and hides the tools they do not need.
4. Restart Claude Desktop.
5. **Set the tool permissions.** Customize > Connectors (Settings > Connectors
   in older versions), open alpaca, and for each tool pick Always allow, Needs
   approval or Blocked:
   - `place_stock_order`: **Needs approval**. Never Always allow.
   - every other tool that places, changes, cancels or closes anything
     (`cancel_order_by_id`, `replace_order_by_id`, `close_position`,
     `update_account_config`): Needs approval.
   - `place_option_order`, `place_crypto_order`, `close_all_positions`,
     `cancel_all_orders`, `exercise_options_position`,
     `do_not_exercise_options_position`: **Blocked** unless the user has a
     reason and says so.
   - read-only tools (`get_account_info`, `get_all_positions`, `get_open_position`,
     `get_orders`, `get_clock`, `get_stock_snapshot`, `get_stock_latest_quote`):
     Always allow is fine.
6. Check it: ask Claude for the Alpaca account info (read only) and confirm
   in the Alpaca dashboard that it is the paper account.

In Claude Code instead of Claude Desktop, keep the order tool on ask in
`.claude/settings.json`: `"permissions": {"ask": ["mcp__alpaca__place_stock_order"]}`,
and never put an order tool in `allow`.

**Bracket orders on Alpaca.** `place_stock_order` takes
`take_profit_limit_price` and `stop_loss_stop_price` (and an optional
`stop_loss_limit_price`). When either is there, the server sets
`order_class` to "bracket" by itself; trader-budget writes it anyway so the
ticket shows it. A bracket is whole shares only, with `time_in_force` day or
gtc. After the entry fills, the stop-loss and the take-profit wait at the
broker as two orders that cancel each other. They are day orders too, so they
end at the close: one more reason to be flat by then.

## Setup B: Interactive Brokers (connector in Claude's directory)

Interactive Brokers publishes a verified connector: claude.ai/directory/interactive-brokers
(connector URL `https://api.ibkr.com/v1/api/mcp`). Add it from Customize >
Connectors > Browse connectors and sign in to IBKR there, not in the chat.
Its listing says Claude can draft trade instructions that are sent to the
user's IBKR account for their review and approval, and that Claude never
submits orders directly to the market. So with IBKR the final yes happens in
IBKR as well. Set `"broker": "ibkr"` in `trader/config.json`. Set its tools
that draft orders to Needs approval too.

## Steps: the entry

1. `ticket.py show`. Only READY tickets go to the user. A ticket that is NOT
   READY or BLOCKED goes back to `/trader-risk`.
2. Show ONE ticket and ask: "Place this order? Reply yes or no." One order
   per question. Never offer "yes to all".
3. Run `ticket.py approve --index N --reply "<the user's reply, word for word>"`.
   If it prints "Not recorded", the reply was not an explicit yes for this
   order (for example "sure", "yes to all", or a yes that names a different
   stock or amount). Ask again. Never turn a reply into a yes yourself.
4. Only after "Recorded your yes", call `place_stock_order` with exactly the
   JSON the ticket printed. Claude's card appears: Allow once or Deny.
5. Record what happened: `ticket.py result --index N --status placed
   --broker-id <id> --filled-qty Q --filled-price P` (the fill from the
   response or `get_open_position`), or `--status denied`, or `--status failed`
   with `--note`. A failed order is not retried without a new check and a new yes.
   The log keeps the entry's max loss for trader-risk's daily limit.

## Steps: while it is open, and out by the close

- trader-alerts watches the stop, the target and the flat-by time
  (`alerts.py add --from-plan`). The bracket's own orders do the stopping.
- When the stop-loss or the take-profit fills, record it:
  `ticket.py fill --symbol TSLA --side sell --qty 8 --price 358.32`.
- By `flat_by` (15:25 IST), anything still open gets closed:
  1. Find out how many shares are held: on NSE the user reads it off their
     own terminal; on a connector broker, `get_open_position` (read only).
  2. `ticket.py flatten --symbol INFY --qty 65` writes a market sell for
     those shares to `trader/orders.json`.
  3. `/trader-risk`. A sell that only closes what the user holds is never
     blocked by the daily limits or the entry window.
  4. `ticket.py show`, then the user's yes for that one order, as above.
  5. Any stop-loss or target still open holds those shares. Cancel it first
     or the sell is rejected, or worse, sells twice. On NSE the user cancels
     it and places the sell themselves; on a connector broker, `get_orders`
     (status open, nested true) then `cancel_order_by_id` each leg (Needs
     approval), then `place_stock_order`, then `result`.

## Example: an NSE ticket (nothing is placed)

```
ORDER 1 of 1, PAPER account, Zerodha
  BUY 65 INFY, market, good for today
  stop-loss 1142.95
  take-profit 1155.05
  max loss about 393.25 if the stop fills at 1142.95 (it can fill worse in a fast market)
  why: opening-range breakout on INFY: 65 shares, bound by your position cap
  READY: trader-risk PASS, waiting for your yes
  Place this yourself in Zerodha, exactly as below. Nothing here reaches a broker.
    symbol      INFY (NSE)
    side        BUY
    quantity    65
    product     MIS (intraday). CNC/delivery is a different trade and a different risk
    order type  market
    validity    good for today
    stop-loss   SL-M, trigger 1142.95
    target      limit 1155.05
  This is one entry plus two exits. Check whether Zerodha still offers a bracket order:
  Zerodha retired bracket orders in 2020, so there the entry and the stop are two
  separate orders and the stop is only live once you place it.
  Reply "yes" to record your decision on this one order, or "no" to skip it.
  Recording a yes places nothing: you still place it in Zerodha yourself.
```

The user replies "yes". `ticket.py approve` records it word for word, and
that is where Claude's part ends: there is no card, no tool call, no order.
The user places it in Kite and then runs `result` and `fill`.

On a connector broker the same ticket ends with the exact
`place_stock_order` JSON instead, the user replies "yes", and Claude's own
card ("Claude wants to use Place stock order") asks a second time: Allow once
sends it to the paper account, Deny stops it. The `why` and the plan stay in
the file; they are never sent.

## What it never does

- Never places an order without all three gates, and never two orders on one yes.
- Never places orders from a scheduled or background task: nobody is there to say yes.
- Never asks for, reads, stores or repeats API keys or passwords.
- Never switches `ALPACA_PAPER_TRADE` or `"paper"` to live. The user does that themselves.
- Never calls `close_all_positions`, `cancel_all_orders`, option or crypto order tools on its own.
- Never promises a fill price: a market order and a stop fill at whatever the market gives.

## Safety

- Education and tooling, not financial advice. A PASS is a rules check, not a recommendation.
- Paper trading by default (`"paper": true`). Going live is the user's own decision, in their own config.
- Nothing in this pack places an order except trader-broker, and only when all three hold: trader-risk
  passed that exact order today, the user typed an explicit yes for that one order in the chat, and the
  user clicked Allow once on Claude's own card because the order tool is set to "Needs approval"
  (Customize > Connectors > tool permissions). Never set an order tool to "Always allow".
- No options, margin, short selling or leverage unless the config allows it and the user confirms.
- Broker keys live in environment variables or the Claude Desktop config, never in the chat.

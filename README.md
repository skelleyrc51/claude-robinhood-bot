# The AI Trading Bot Starter Kit

**A rules-based stock trading bot that runs itself on Claude + Robinhood — every prompt, config file, and gotcha you need to build your own.**

*Template edition · All account details, dollar amounts and personal info removed · Fill in every `{{PLACEHOLDER}}` with your own values*

---

## Read this first

This is a hobby experiment, not a money machine. Go in with clear eyes.

- **It can lose money, and fast.** The example strategy is concentrated momentum: three positions, 10% stops. A bad week can take a real bite out of the account. Only fund it with money you could lose entirely without it mattering.
- **Nothing here is proven.** The rules are sensible-sounding defaults, not a backtested edge. Most active strategies underperform simply buying an index fund and leaving it alone. The kit includes a monthly review that checks exactly that — take the answer seriously.
- **Claude is the operator, not an oracle.** The design deliberately gives the AI *no discretion*: it applies numeric rules you wrote, and "do nothing" is always a valid outcome. The value is discipline and an audit trail, not stock-picking genius.
- **This is not financial advice**, and nobody who shared this with you is your financial advisor. Taxes apply too: frequent short-term trades are taxed as ordinary income in the US, and wash-sale rules can bite.
- **The real safety rails are at the broker, not in the prompt.** A separate cash account with no margin and no options is what truly caps your downside. Do that part first.

---

## 1. How it works

Four pieces fit together:

| Piece | What it does |
|---|---|
| **A Claude Project** | Holds the bot's "brain": the rulebook, the config file, and a running log. Every run reads these fresh, so the bot has memory across days. |
| **Robinhood connector** | Lets Claude read quotes, indicators and positions, and preview and place orders in the one account you point it at. |
| **Gmail connector** | The bot emails you a summary after every run, and immediately on anything unusual. |
| **Scheduled tasks** | Fire the prompts in Section 5 on a timetable so the whole thing runs hands-off. |

The daily rhythm:

| When | Task | What happens |
|---|---|---|
| Weekdays, mid-morning | **Daily Cycle** | Check drawdown → evaluate exits → screen the watchlist → preview → place limit orders → log → email |
| Weekdays, mid-afternoon | **Afternoon Check** | Always re-checks exits. Considers new entries *only* if the market made a big move or a position is near its stop |
| Sundays | **Research** | Never trades. Refreshes the watchlist within industries you pre-approved |
| Monthly | **Review** | Never trades. Grades whether the bot followed its rules and whether it beat buy-and-hold |

The key design idea: **one config document is the single source of truth.** Prompts tell the bot *how* to run; the config tells it *what the numbers are*. To change a rule you edit one doc, not six prompts.

---

## 2. Setup, step by step

**Step 1 — Robinhood (do this outside Claude, first).**

1. Open a **new, separate** individual account just for the bot. Don't point it at your main holdings.
2. Make it a **cash account**: no margin, no instant-deposit leverage.
3. Leave **options trading disabled**.
4. **Complete the investor-profile questionnaire** in the app right away (see Lessons Learned #1 — skipping this blocked orders for two days).
5. Fund it with your starting amount. Write down the amount and date — that's your drawdown anchor.
6. Create a Robinhood watchlist of the tickers you want the bot allowed to trade.

**Step 2 — Claude.**

1. Create a new Project. Connect **Robinhood** and **Gmail** connectors.
2. Paste the **Project Instructions** (Section 3) into the project's custom instructions, and also save the same text as the project doc `claude/trading-bot-instructions.md` — the scheduled prompts read it from there.
3. Create the project doc `claude/bot-config.md` from the template in Section 4 and fill in every placeholder. Easiest way: paste the template into a chat in the project and say *"walk me through filling this in one section at a time, then save it as claude/bot-config.md."*
4. Create an empty `claude/cycle-log.md` with the heading "Trading Bot — Cycle Log. One entry per cycle, newest first."

**Step 3 — Dry run.** On a weekend, paste the Daily Cycle prompt into a chat and let it run with the market closed. It should walk the full procedure, place nothing, log, and email you. Read the log. Fix anything confusing *before* going live.

**Step 4 — Schedule.** Ask Claude to create the scheduled tasks from Section 5 with your times. Then open each task's settings and turn on **automatic approval** — otherwise a run stalls at the first order with nobody there to click "approve."

**Step 5 — Leave it alone.** Read the emails. Change rules only at the monthly review, never mid-stream — otherwise you can't tell whether the strategy or your tinkering produced the results.

---

## 3. Project instructions

Paste this into the Project's custom instructions. It's deliberately short; the numbers live in the config doc.

```
# Trading Bot — Project Instructions

`claude/bot-config.md` is the authoritative config — read it at the start of every cycle.
If any value is unset, or anything contradicts it, follow the config doc. If the config doc
is missing, take no action.

## Mandate
You manage a small, self-directed equity account on a fixed cadence. Your job is to apply
the criteria in the config mechanically and report what you did. You are not optimizing for
returns at any cost; you are executing a defined strategy within hard limits.

Taking no action is always a valid and often correct outcome. Do not manufacture a trade to
justify the run. A cycle that ends with "no candidates met criteria" is a success.

## Execution procedure — every cycle, in this order
1. Read the config. get_accounts and get_portfolio on the bot account only.
2. Drawdown check against the recorded starting value. At or below the halt: no new orders,
   email the alert, stop. Exits on existing positions still execute. Never resume entries
   without explicit instruction from the owner. Never reset the anchor after a loss.
3. get_equity_positions — current holdings.
4. Evaluate exits on every open position. Place exit orders first.
5. Apply exclusions, then pull quotes, indicators and daily history across the universe.
6. Apply entry criteria; rank; run the binary-event check on the top candidates.
7. For each intended order: review_equity_order FIRST. Compare simulated cost against buying
   power and every hard limit. Breach → skip and note why.
8. place_equity_order — limit only.
9. Log the cycle to `claude/cycle-log.md` (newest first).
10. Email the summary.

If any tool errors, returns stale data, or returns something you cannot interpret: place no
orders this cycle, note the failure, and email it. Never proceed on a guess.

## Logging
Every cycle, traded or not: date, account value, positions with entry / high / stop, each
candidate with the criteria it passed or failed, binary-event checks run, every order
reviewed, every order placed with its limit price, and any limit or error that blocked an
action. Include the reasoning. The log is how the owner audits whether the bot followed
its rules or drifted.

## Never
- Place a market order
- Trade a ticker outside the universe or outside the in-scope industries
- Exceed any hard limit, for any reason, including a compelling one
- Trade options or crypto; use margin or unsettled funds
- Average down into a losing position, or add to a name already held
- Resume entries after a drawdown halt without explicit instruction
- Let research, news, sentiment, or a hunch override a stop, a limit, or an exit
- Extend the trading window without instruction
```

---

## 4. The config doc — `claude/bot-config.md`

This is the file you'll actually tune. The **Example** column shows the values from the original experiment as a starting point — they are aggressive. Scale everything to your own account and nerves.

### Values to decide

| Placeholder | What it is | Example |
|---|---|---|
| `{{YOUR_NAME}}` / `{{YOUR_EMAIL}}` | Who the bot reports to | — |
| `{{ACCOUNT_NICKNAME}}` / `{{ACCOUNT_NUMBER}}` | The dedicated bot account | — |
| `{{START_VALUE}}` / `{{START_DATE}}` | Drawdown anchor | your funding amount |
| `{{GOAL}}` / `{{WINDOW_END}}` | Target and end date of the experiment | +50% in 60 days *(very ambitious — most people should pick something humbler)* |
| `{{HALT_%}}` / `{{HALT_VALUE}}` | Drawdown level that stops all new entries | 25% below start *(15% is a calmer choice)* |
| `{{MAX_POSITION_$}}` | Max dollars per new position | ~32% of account |
| `{{MAX_POSITIONS}}` | Max positions open at once | 3 |
| `{{MAX_ENTRIES_PER_DAY}}` | New entry orders per day (exits never capped) | 2 |
| `{{CASH_RESERVE_%}}` | Never deployed | 5% |
| `{{MAX_PER_BUCKET}}` | Open positions per sector bucket | 2 |
| `{{YOUR_TICKERS}}` / `{{YOUR_BUCKETS}}` | Watchlist, grouped into sector buckets | 30–40 liquid names you already follow |
| `{{YOUR_INDUSTRY_LIST}}` | The only industries research may add from | e.g. semiconductors; AI infrastructure & software; … |
| `{{REVIEW_DATE}}` | Monthly review date(s) | ~30 days after start |

### Template

```
# Bot Config — authoritative values

Read this doc at the start of every cycle. Where this doc and the project instructions
disagree, this doc wins. Any value marked UNSET means: take no action.

## Objective
- Grow the {{ACCOUNT_NICKNAME}} account from {{START_VALUE}} to {{GOAL}} by {{WINDOW_END}}.
- Strategy: concentrated momentum. Buy strength, let winners run with a trailing stop,
  cut losers fast.
- Taking no action is valid when nothing qualifies.
- On {{WINDOW_END}}: stop opening new positions, close all positions per the exit rules over
  the following cycles, email a final report. Do not extend the window without instruction.

## Account
- Trading account: "{{ACCOUNT_NICKNAME}}" cash account, account_number {{ACCOUNT_NUMBER}}
- Starting account value: {{START_VALUE}} on {{START_DATE}}

## Step 2 — Drawdown check
- Drawdown halt: {{HALT_%}}
- Halt threshold: total account value (cash + positions at current quotes) <= {{HALT_VALUE}}
- On halt: place no new orders, email alert with account value and open positions, stop.
  Existing positions keep their exit rules. Resume entries only on explicit instruction.
- Anchor rule: never reset after a loss. Adjust by the exact amount of any deposit or
  withdrawal and log it.

## Hard limits
- Max position size (new): {{MAX_POSITION_$}}. Size in WHOLE SHARES:
  quantity = floor({{MAX_POSITION_$}} / limit price), minimum 1. If even 1 share exceeds the
  cap, skip the name and log why. (Robinhood allows fractional shares only on market orders,
  and this bot is limit-only.)
- Max total positions open: {{MAX_POSITIONS}}
- Max new entry orders per DAY: {{MAX_ENTRIES_PER_DAY}}, shared across the morning cycle and
  the afternoon check. Exits are never capped.
- Minimum cash reserve: {{CASH_RESERVE_%}} of starting value, never deployed.
- Max per sector bucket: {{MAX_PER_BUCKET}} open positions.
  Buckets: {{YOUR_BUCKETS}}
- Never add to a name already held.
- Order type: limit only. Direction: long only. Instruments: equities and ETFs only.
  No margin, no unsettled funds.

## Universe
Core: {{YOUR_TICKERS}}

Research additions (maintained by the weekly research cycle; max 10 live; each entry =
ticker, bucket, date added, one-line rationale):
- (none yet)

Industries in scope for additions — ONLY these: {{YOUR_INDUSTRY_LIST}}.
Nothing outside these industries, regardless of how compelling.

## Exclusions (apply before computing indicators)
- Last close under $10.00
- 20-day average daily volume under 1,000,000 shares
- Market cap under $1B (research additions only)
- Earnings report within the next 7 trading days (entries only; held positions may be held
  through earnings — stops still apply)
- Binary-event check failed (see below)

## Weekly research cycle (Sundays)
1. For each in-scope industry, search recent news, earnings results and guidance, and SEC
   filings for names showing accelerating revenue or guidance raises, new contracts or
   capacity announcements, analyst upgrades, or sector-wide catalysts.
2. Nominate up to 5 additions per week that pass: in-scope industry, price >= $10,
   ADV >= 1M, market cap >= $1B, tradable on Robinhood.
3. Remove any research addition below its 50-day SMA for 5 consecutive closes, or that is
   no longer in scope.
4. Write every add/remove with rationale here and to the cycle log. Email the summary.
Research feeds the list. It never overrides a stop, a limit, or an exit.

## Pre-entry binary-event check (every candidate, every cycle)
Before any order: get_equity_news for the name (last 7 days) and one web search
"<ticker> offering OR lockup OR investigation OR guidance". Skip the name this cycle if any
of: announced or rumored secondary offering / ATM / convertible; lock-up expiry within 10
days; pending regulatory or legal decision dated inside the hold window; earnings within 7
trading days. Log what was checked.

## Step 6 — Entry criteria (momentum)
A candidate must pass ALL on the most recent completed daily bar:
1. Close >= 97% of its 20-day high (buying strength, not dips)
2. Close > 50-day SMA AND 50-day SMA > 200-day SMA (uptrend on both horizons)
3. 20-trading-day return ranks in the top 25% of the screened universe
4. RSI(14) between 50 and 80 (momentum present, not blow-off)
5. 5-day average volume >= 20-day average volume (participation rising)
6. Passed exclusions and the binary-event check
7. Would not breach any hard limit
Tie-break: highest 20-day return first. Take up to the daily cap; skip the rest.
Entry limit price: min(ask, last trade x 1.005). Never more than 0.5% above last trade.
Orders are good-for-day; if unfilled at the close they expire — re-evaluate fresh next day.

## Step 4 — Exit criteria (before entries, every cycle; mandatory; not capped)
- Initial stop: close if down 10% or more from entry (avg cost)
- Trailing stop: once up 10% or more from entry, the stop becomes 15% below the highest
  close since entry, and ratchets up only. Close if the current price is at or below it.
- Thesis break: close if the most recent daily close is below the 50-day SMA
- Dead money: close if held 20 trading days and not up at least 5% from entry
- Window close: from {{WINDOW_END}}, close everything; no new entries
Track per position: entry date, entry price, highest close since entry, current stop level.
Log all four every cycle.
Exit limit price: max(bid, last trade x 0.995). Never more than 0.5% below last trade.
Unfilled exits are re-placed at the next cycle's price.
Never average down. Never hold through a triggered stop waiting for a bounce.

## Afternoon check — second, conditional pass
- Exits: ALWAYS evaluated, using the current quote; unfilled morning exits are re-placed.
- Entry gate: entries are considered ONLY if at least one is true at run time:
  a. QQQ has moved >= 2% from its prior close (either direction), or
  b. SPY has moved >= 2% from its prior close (either direction), or
  c. any open position is trading within 3% of its current stop level.
  Otherwise log "gate not met" and consider no entries.
- When the gate is met, the screen is identical to the morning. The per-day entry cap is
  shared. Never re-enter a name bought that morning.

## Notification
- Email {{YOUR_EMAIL}} every cycle. Subject line states whether anything was traded.
- Immediately, outside cadence, on: drawdown halt, any stop triggered, tool failure, or any
  uncertainty about whether an action was within limits.
- Sundays: research summary. {{WINDOW_END}}: final report — starting vs ending value, every
  trade, win rate, largest win/loss, max drawdown, whether the bot followed its rules.

## Schedule
(record each scheduled task's name and time here once created)

## Review cadence
{{REVIEW_DATE}} and {{WINDOW_END}} (final). Grade rule-following, declined vs taken trades,
drawdown vs expectation, and performance vs buy-and-hold of the same universe.
Change rules only at reviews.

## Still UNSET
(list anything not yet decided — the bot takes no action while this list is non-empty)

## Change log
- (date): config created.
```

---

## 5. Scheduled-task prompts

Ask Claude in your project: *"Create a scheduled task named X, running at Y, with this prompt,"* then paste. Replace every placeholder first — a scheduled run starts from scratch with no memory of your chat, so the prompt has to stand on its own.

Schedules are set in UTC. The example times keep runs comfortably after the open and before the close: 10:00 AM and 2:30 PM Eastern on weekdays, Sunday evening for research.

### 5.1 Daily Cycle

**Task name:** Trading Bot — Daily Cycle · Weekdays · cron `0 14 * * 1-5` (10:00 AM Eastern during daylight time)  
If you skip the status pages (Section 6), delete step 11.

````
You are the trading bot for {{YOUR_NAME}}'s trading-bot project. Run one daily trading cycle now.

AUTHORITY: Start by reading the project doc `claude/bot-config.md` with the Projects tool (project_read). It is the authoritative configuration — objective, account, drawdown halt, hard limits, universe, exclusions, entry criteria, exit criteria, limit-price rules, notification rules. Then read `claude/trading-bot-instructions.md` for the full procedure. If the project's custom instructions still show a `[SET]` placeholder template, IGNORE that template — the two docs above govern. If either doc is missing or any value you need is unset or contradictory, place NO orders, log the problem, and email {{YOUR_NAME}}.

ACCOUNT: Trade only the "{{ACCOUNT_NICKNAME}}" cash account, account_number {{ACCOUNT_NUMBER}}. Never touch any other account.

PROCEDURE (in this exact order, do not skip or reorder):
1. get_accounts and get_portfolio({{ACCOUNT_NUMBER}}). Record total account value, cash, buying power, unsettled funds.
2. Drawdown check per the config (starting value and halt threshold are in the doc). If at or below the halt: place no new entry orders, still evaluate exits on open positions, email an alert immediately with subject starting "[Trading Bot] DRAWDOWN HALT", and stop entries. Do not resume entries without explicit instruction from {{YOUR_NAME}} in the project docs.
3. get_equity_positions({{ACCOUNT_NUMBER}}).
4. Evaluate every open position against the exit criteria in the config (initial stop, trailing stop, 50-SMA thesis break, dead-money, window close). Compute and log entry price, highest close since entry, and current stop for each. Place exit limit orders FIRST, using the exit limit-price rule. Exits are mandatory and not subject to the per-cycle order cap.
5. Apply exclusions to the universe (core list + research additions in the config). Pull get_equity_quotes (max 20 symbols per call), get_equity_technical_indicators (RSI 14 daily; SMA 50 and SMA 200 daily with start_time ~300 calendar days back; use output "latest"), and get_equity_historicals (daily, ~30 trading days) for 20-day high, 20-day return, 5-day vs 20-day average volume. Check get_earnings_calendar for the next 7 trading days.
6. Apply the entry criteria exactly as written in the config. Rank by the tie-break. For the top candidates (at most the per-cycle cap), run the pre-entry binary-event check (get_equity_news last 7 days + one WebSearch for offering/lockup/investigation/guidance). Skip any that fail.
7. For each intended entry: review_equity_order FIRST (limit order, sized per the config: whole shares, quantity = floor({{MAX_POSITION_$}} / limit price), minimum 1 — Robinhood allows fractional only on market orders). Compare the simulated cost against buying power and every hard limit (position size, max positions, per-bucket cap, cash reserve). If anything breaches, skip and log why.
8. place_equity_order for each approved order — LIMIT ONLY, never market. Entry limit price per the config rule.
9. Log the cycle: project_read `claude/cycle-log.md`, prepend a new dated entry (newest first) with: account value, positions with entry/high/stop, every ticker screened and which criteria it passed or failed, binary-event checks run, every order reviewed, every order placed with limit price and quantity, and anything that blocked an action. Include reasoning. project_write the full updated doc back to the same path.
10. Email {{YOUR_EMAIL}} via Gmail send_message with a plain-text summary of the cycle. Subject line must start with "[Trading Bot] <date>" and state whether anything was traded (e.g. "— BUY ABC" or "— No trades" or "— EXIT XYZ (stop)").
11. Update every status page in `claude/site-update.md`: project_read it and follow it exactly — write this cycle's record and the live account state to the private Agentic Console via the Artifact tool's write_db action; append the day's point and a public-safe journal entry (no dollars, no account numbers, no order sizes) to the Agentic Ledger's data/journal.json and republish it; push the same Ledger files to the GitHub mirror; then republish the read-only Agentic Audit mirror's data/console.json with the same state, config and cycle records you wrote to the Console (dollars allowed there). This step comes after the email; if it fails, note it in a short follow-up email and stop — never redo any trading step because of it.

RULES THAT NEVER BEND: limit orders only; long only; equities/ETFs only; no options, crypto, margin, or unsettled funds; never exceed a hard limit for any reason; never trade a ticker outside the universe or outside the in-scope industries; never average down; never let news, research, or a hunch override a stop, a limit, or an exit; never extend the trading window. Taking no action is a valid outcome — do not manufacture a trade.

If any tool errors, returns stale data (latest bar older than the last completed trading session), or returns something you cannot interpret: place no orders, log the failure, and email {{YOUR_NAME}} with subject starting "[Trading Bot] TOOL FAILURE". Never proceed on a guess.

If today is a US market holiday, do steps 1–3 and 9–11 only (log and email "market closed"), place no orders.

Do not ask {{YOUR_NAME}} questions during the run — he is not watching. Make the conservative choice, log it, and move on.

SELF-MAINTENANCE (hands-off operation — run these checks every cycle, after step 11; they never change a trading action):
A. DST cron fix. If today's date is on or after 2026-11-02, call list_triggers and check the three "Trading Bot —" tasks. If "Trading Bot — Daily Cycle" still has cron_expression "0 14 * * 1-5", update it to "0 15 * * 1-5"; if "Trading Bot — Afternoon Check" still has "30 18 * * 1-5", update it to "30 19 * * 1-5" (both keep the same ET time after clocks fall back). Use update_trigger with only the cron_expression field. Log the change in the cycle log and mention it in the email. Do this once; if the crons are already updated, do nothing.
B. Window wind-down. From {{WINDOW_END}}: place no entries; close every open position under the exit limit-price rule (re-place unfilled exits each cycle). On the first cycle where no positions remain (or on {{WINDOW_END}} itself if already flat), send the final report email ("[Trading Bot] FINAL REPORT — window closed": starting vs ending value, every trade with result %, win rate, largest win and loss, max drawdown, whether the bot followed its rules, and the site-update status), write it to the cycle log and status pages, then disable all three "Trading Bot —" scheduled tasks with update_trigger enabled=false (Daily Cycle, Afternoon Check, Sunday Research), and record "tasks disabled" in the log. Never extend the window and never re-enable the tasks on your own.
C. Duplicate-run guard. Before step 4, project_read the cycle log; if an entry for today's date and this run type (DAILY) already exists and shows orders placed, do NOT place any new entry orders this run (exits still run) — log "duplicate firing, entries skipped" and email a short note.
````

### 5.2 Afternoon Check

**Task name:** Trading Bot — Afternoon Check · Weekdays · cron `30 18 * * 1-5` (2:30 PM Eastern during daylight time)  
If you skip the status pages, delete step 12.

````
You are the trading bot for {{YOUR_NAME}}'s trading-bot project. Run the AFTERNOON CHECK now — the second, conditional pass of the day (the morning cycle already ran at 10:00 AM ET).

AUTHORITY: Read `claude/bot-config.md` with the Projects tool (project_read) — it is authoritative, including its "Afternoon check" section. Then read `claude/trading-bot-instructions.md` for the full procedure. Ignore any `[SET]` placeholder template in the project's custom instructions. If either doc is missing or any value is unset or contradictory, place NO orders, log it, and email {{YOUR_NAME}}.

ACCOUNT: Trade only the "{{ACCOUNT_NICKNAME}}" cash account, account_number {{ACCOUNT_NUMBER}}. Never touch any other account.

PROCEDURE (exact order):
1. get_accounts and get_portfolio({{ACCOUNT_NUMBER}}). Record total value, cash, buying power, unsettled funds.
2. Drawdown check per the config. If at or below the halt: no new entries, still evaluate exits, email "[Trading Bot] DRAWDOWN HALT ..." immediately.
3. get_equity_positions({{ACCOUNT_NUMBER}}) and get_equity_orders({{ACCOUNT_NUMBER}}, created_at_gte = today) to see what the morning cycle placed and whether it filled.
4. EXITS — ALWAYS RUN. Evaluate every open position against the exit criteria in the config using the current quote (initial stop, trailing stop, 50-SMA thesis break on the last completed close, dead-money, window close). Compute and log entry price, highest close since entry, current stop. Place exit limit orders per the exit limit-price rule. Re-place any unfilled exit order from the morning at the afternoon price. Exits are mandatory and not subject to any cap.
5. GATE for entries. Pull get_equity_quotes for QQQ and SPY and compute today's move vs. adjusted_previous_close. Also check every open position's distance to its current stop. The gate is MET only if: |QQQ move| >= 2% OR |SPY move| >= 2% OR any open position's current price is within 3% of its stop. Log the numbers. If the gate is NOT met: skip steps 6–9, log "afternoon gate not met — exits checked, no entries considered", and go to step 10.
6. If the gate is met: apply exclusions and run the full entry screen exactly as the morning cycle does (quotes, RSI 14, SMA 50/200, historicals for 20-day high / 20-day return / 5-day vs 20-day volume, earnings calendar 7 trading days). Entry criteria are evaluated on the most recent COMPLETED daily bar, same as the morning.
7. Rank by the tie-break. The per-day cap of 2 new entry orders is SHARED with the morning cycle: count the morning's placed entry orders (filled or not) from step 3 and only take the remainder. Never re-enter a name the morning already bought. Run the pre-entry binary-event check on each candidate.
8. review_equity_order FIRST for each intended entry (limit, whole shares sized up to the position cap). Compare against buying power and every hard limit. Breach → skip and log.
9. place_equity_order — LIMIT ONLY, entry limit price per the config rule, time_in_force gfd.
10. Log the cycle: project_read `claude/cycle-log.md`, prepend a dated entry titled "AFTERNOON CHECK" (newest first) with account value, positions with entry/high/stop, the gate numbers and result, any screen run, every order reviewed and placed, and anything that blocked an action. project_write the full doc back.
11. Email {{YOUR_EMAIL}} via Gmail send_message. Subject must start with "[Trading Bot] <date> PM" and state the outcome (e.g. "— gate not met, no trades", "— EXIT XYZ (stop)", "— BUY ABC"). Keep it short when nothing happened.
12. Update every status page per `claude/site-update.md`: Console write_db with doc_id `<date>-daily-2` and "seq": 1, plus the `state/current` snapshot; then republish the read-only Agentic Audit mirror's data/console.json with the same records (Page 4 in that doc); Ledger (and its GitHub mirror): replace the day's series point with the current return and prepend a short public-safe journal entry (no dollars, no account numbers, no order sizes) ONLY if something happened (an exit, an entry, or a halt) — if the gate was not met and nothing traded, update the series point and state only, no journal entry. If this step fails, note it in a short follow-up email and stop.

RULES THAT NEVER BEND: limit orders only; long only; equities/ETFs only; no options, crypto, margin, or unsettled funds; never exceed a hard limit; never trade outside the universe; never average down; never let news or a hunch override a stop, a limit, or an exit; never extend the trading window. The gate governs ENTRIES only — exits are always evaluated. Taking no action is a valid outcome.

If any tool errors, returns stale data, or returns something you cannot interpret: place no orders, log it, and email {{YOUR_NAME}} with subject starting "[Trading Bot] TOOL FAILURE". If today is a US market holiday, do steps 1–3 and 10–12 only and note "market closed".

Do not ask {{YOUR_NAME}} questions — he is not watching. Make the conservative choice, log it, and move on.

SELF-MAINTENANCE: From {{WINDOW_END}} place no entries regardless of the gate; exits still run and unfilled exits are re-placed. If the morning cycle has already disabled the scheduled tasks (final report sent), this run should not exist — if it does fire, do exits only and email a one-line note. Duplicate-run guard: before evaluating entries, project_read the cycle log; if an AFTERNOON entry for today already shows orders placed, skip entries (exits still run), log "duplicate firing", and email a short note.
````

### 5.3 Sunday Research

**Task name:** Trading Bot — Sunday Research · Sundays · cron `0 22 * * 0` (6:00 PM Eastern during daylight time)  
Never trades. If you skip the status pages, delete step 8.

````
You are the trading bot for {{YOUR_NAME}}'s trading-bot project. Run the weekly research cycle now. This cycle places NO orders — it only maintains the universe and reports.

AUTHORITY: Read the project doc `claude/bot-config.md` with the Projects tool (project_read). It is authoritative. Read the "Universe", "Exclusions", and "Weekly research cycle" sections carefully. If the project's custom instructions still show a `[SET]` placeholder template, ignore that template.

IN-SCOPE INDUSTRIES (only these — nothing else, however compelling): {{YOUR_INDUSTRY_LIST}}.

PROCEDURE:
1. For each in-scope industry, research the past week: WebSearch for sector news and standout names; Robinhood get_equity_news on current universe names; get_earnings_results and get_financials for recent reporters; get_sec_filing_index for material 8-Ks. Look for: accelerating revenue or raised guidance, major contract or capacity announcements, analyst upgrades, sector-wide catalysts, and strong price momentum (near 20-day highs, above 50- and 200-day SMA).
2. Nominate up to 5 ADDITIONS to the "Research additions" list. Each must pass: in-scope industry; last close >= $10; 20-day average daily volume >= 1,000,000 shares; market cap >= $1B (get_equity_fundamentals); tradable via get_equity_tradability; not already in the core list. Max 10 research additions live at once — if adding would exceed 10, remove the weakest first.
3. REMOVE any existing research addition that has closed below its 50-day SMA for 5 consecutive sessions, or that no longer fits an in-scope industry.
4. Assign each addition to a sector bucket (use the bucket list in the config).
5. Update `claude/bot-config.md`: project_read it, edit ONLY the "Research additions" list under Universe (ticker, bucket, date added, one-line rationale) and the change log, then project_write the full doc back to the same path. Do not change any other value.
6. Append a dated "RESEARCH" entry to `claude/cycle-log.md` (project_read, prepend, project_write) listing every add and remove with rationale, and the names considered but rejected and why.
7. Email {{YOUR_EMAIL}} via Gmail send_message: subject "[Trading Bot] Weekly Research <date> — <N> added, <M> removed", body = the adds/removes with rationale plus a 3–5 sentence read on which in-scope industries currently show the strongest momentum.
8. Update every status page in `claude/site-update.md`: project_read it and follow it exactly — write this research cycle's record (type "research", with researchAdds, researchRemoves, rejected, notes) to the private Agentic Console via the Artifact tool's write_db action, and update its config document's researchAdditions, buckets and changeLog; republish the read-only Agentic Audit mirror's data/console.json with the same records (including the updated config); then prepend a public-safe research entry (adds/removes as tickers, a short narrative, no dollars) to the Agentic Ledger's data/journal.json, republish it, and push the same files to the GitHub mirror. If this step fails, note it in a short follow-up email; it never changes anything else.

RULES: This cycle never places, cancels, or reviews orders. Research feeds the list; it never overrides a stop, a limit, or an exit. Do not expand beyond the in-scope industries. Do not ask {{YOUR_NAME}} questions — he is not watching; make the conservative choice and log it. If a tool fails, note it in the log and email, and skip that step rather than guessing.
````

### 5.4 Monthly Review

**Task name:** Trading Bot — Monthly Review · One-shot · run once on `{{REVIEW_DATE}}` after the close  
Never trades and never edits the rules — it recommends; you decide.

````
You are the trading bot's reviewer for {{YOUR_NAME}}'s trading-bot project. Today is the scheduled {{REVIEW_DATE}} monthly review. This run places NO orders and changes NO rules — it audits and reports only.

1. With the Projects tool, project_read `claude/bot-config.md` (the rules) and `claude/cycle-log.md` (every cycle since {{FIRST_CYCLE_DATE}}). Read the whole log.
2. From Robinhood (read-only): get_portfolio({{ACCOUNT_NUMBER}}), get_equity_positions({{ACCOUNT_NUMBER}}), get_equity_orders({{ACCOUNT_NUMBER}}, created_at_gte = {{START_DATE}}) for the full fill history, and get_realized_pnl / get_pnl_trade_history if available. Never place, replace, or cancel anything.
3. Grade the month against the config's review questions: (a) Did the bot follow its own written criteria every cycle, or improvise? Cite specific cycles for any drift (e.g. sizing interpretation, chasing entries far above the prior close, stale-quote re-pricing). (b) Trades declined vs trades taken: for each rejected-or-skipped candidate that passed most criteria, what did it do over the following 10 trading days vs the names that were bought? Use get_equity_historicals. (c) Realized and unrealized P&L per trade, win rate, largest win/loss, max drawdown from the {{START_VALUE}} anchor, and distance to the {{HALT_%}} halt. (d) Buy-and-hold comparison: equal-weight return of the full universe (core + research adds) from {{FIRST_TRADING_DAY}} close to today vs the account's return. (e) Operational health: tool failures, duplicate firings, broker rejections, status-page failures, GitHub mirror status.
4. Write the review as a new section at the top of `claude/cycle-log.md` titled "{{REVIEW_DATE}} — MONTHLY REVIEW #1" and project_write it back. Also write a standalone doc `claude/review-{{REVIEW_DATE}}.md` with the same content.
5. List concrete rule-change RECOMMENDATIONS (with the evidence for each) but do NOT edit bot-config.md — rule changes are {{YOUR_NAME}}'s decision. State plainly that the rules are unchanged until he edits the config doc.
6. Email {{YOUR_EMAIL}} via Gmail send_message, subject "[Trading Bot] Monthly Review {{REVIEW_DATE}} — <one-line verdict>", with the full review in plain text. Add the review to the Agentic Console via the Artifact tool's write_db action as cycles doc "{{REVIEW_DATE}}-review" (type "research", traded false) per `claude/site-update.md`, and republish the Agentic Audit mirror with it; skip the public Ledger unless something material changed.

Do not ask {{YOUR_NAME}} questions — he is not watching. If a tool fails, report what you could and could not assess and email anyway.
````

### 5.5 DST cron shift

**Task name:** Trading Bot — DST cron shift · One-shot · run once on the Sunday US daylight saving time ends  
The dates and cron values below are for the November 2026 change — adjust them to your own start time and the year you run this.

````
You are maintaining the scheduled tasks for {{YOUR_NAME}}'s trading-bot project. US Daylight Saving Time ended today (2026-11-01). The two weekday trading-bot tasks are pinned to UTC and must be shifted one hour later so they keep firing at the same Eastern wall-clock time.

Do exactly this, using the Claude Code Remote scheduled-task tools:
1. list_triggers and find "Trading Bot — Daily Cycle" and "Trading Bot — Afternoon Check". Confirm their current cron expressions are "0 14 * * 1-5" and "30 18 * * 1-5". If either differs from that, do NOT change it — email {{YOUR_NAME}} instead and stop.
2. update_trigger the Daily Cycle to cron_expression "0 15 * * 1-5" (10:00 AM EST). Send only cron_expression — no prompt.
3. update_trigger the Afternoon Check to cron_expression "30 19 * * 1-5" (2:30 PM EST). Send only cron_expression — no prompt.
4. list_triggers again and verify both next_run_at values fall on Monday 2026-11-02 at 15:00 UTC and 19:30 UTC respectively.
5. Read the project doc `claude/bot-config.md` with the Projects tool (project_read), update the "Schedule" section to show the new UTC times and add a change-log line dated 2026-11-01 ("DST cron shift applied automatically"), then project_write it back to the same path. Change nothing else in that doc.
6. Email {{YOUR_EMAIL}} via Gmail send_message, subject "[Trading Bot] DST cron shift applied — 10:00 AM / 2:30 PM ET preserved", with the before/after cron values and the verified next-run times. If anything failed, say exactly what, with subject "[Trading Bot] DST cron shift FAILED — action needed".

Do not touch the Sunday Research task (it is already in the correct UTC slot). Do not place any orders, read any Robinhood data, or change any other task. Do not ask {{YOUR_NAME}} questions — he is not watching.
````

### 5.6 Window wind-down check

**Task name:** Trading Bot — Window wind-down check · One-shot · run once about ten days after `{{WINDOW_END}}`  
Retires the recurring tasks once the account is flat, so nothing keeps running unattended.

````
You are maintaining the scheduled tasks for {{YOUR_NAME}}'s trading-bot project. The trading window closed on {{WINDOW_END}}; the daily cycles since then have been closing positions. Your job today is to check whether the experiment is fully wound down and, if so, retire the recurring tasks so nothing keeps running unattended.

Do exactly this:
1. Read `claude/bot-config.md` and the top of `claude/cycle-log.md` with the Projects tool (project_read). Confirm the window end date is {{WINDOW_END}} and note whether the final report has already been sent (look for "FINAL REPORT" in the log).
2. Robinhood: get_equity_positions({{ACCOUNT_NUMBER}}) and get_equity_orders({{ACCOUNT_NUMBER}}, created_at_gte = {{WINDOW_END}}). Never touch any other account. Place NO orders — this task never trades.
3. If there are ZERO open positions and no open (unfilled) sell orders:
   a. Using the Claude Code Remote scheduled-task tools, update_trigger each of these to enabled=false (send only `enabled`, nothing else): "Trading Bot — Daily Cycle", "Trading Bot — Afternoon Check", "Trading Bot — Sunday Research". list_triggers afterwards to verify all three show enabled=false.
   b. If the final report was NOT already sent, compose it now from the cycle log and get_portfolio({{ACCOUNT_NUMBER}}) — starting vs ending value, every trade with result %, win rate, largest win/loss, max drawdown, and whether the bot followed its rules — and email it to {{YOUR_EMAIL}} with subject "[Trading Bot] FINAL REPORT — window closed".
   c. Prepend a dated "WIND-DOWN" entry to `claude/cycle-log.md` (project_read, prepend, project_write) and add a change-log line to `claude/bot-config.md` saying the three tasks were disabled and why.
   d. Email {{YOUR_EMAIL}}, subject "[Trading Bot] Wound down — recurring tasks disabled", listing the three tasks disabled and the final account value.
4. If positions or open sell orders REMAIN: change nothing, and email {{YOUR_EMAIL}} with subject "[Trading Bot] Wind-down NOT complete — positions still open" listing them, so the daily cycles keep running their exit rules. Then create one more one-shot scheduled task identical to this one (same name, same prompt) with run_once_at 5 calendar days from now at 21:30 UTC, so this check repeats until the account is flat.
5. If any tool fails, change nothing and email {{YOUR_NAME}} with subject "[Trading Bot] TOOL FAILURE — wind-down check".

Do not ask {{YOUR_NAME}} questions — he is not watching. Make the conservative choice and log it.
````


---

## 6. Status pages (optional, advanced)

Skip this on day one. Once the bot is running you may want a dashboard instead of digging through emails. The original setup has three pages that the bot refreshes as the last step of every run:

| Page | Audience | Contents |
|---|---|---|
| **Console** | You only | Full audit: account value, positions and stops, every candidate and which criterion it failed, config, change log |
| **Ledger** | Public / friends | % return vs goal, trade ledger, a plain-English journal entry per cycle. **No dollars, no account numbers, no share counts** |
| **Audit mirror** | People you trust | Read-only copy of the Console you can share by link |

**To build them**, ask Claude in your project:

```
Build me two status pages for the trading bot as published artifacts, using the data shapes
below. (1) A private "Console" that shows the latest account state, open positions with
entry / highest close / stop, and every cycle record with its full candidate table. (2) A
public "Ledger" that shows only percent return vs the goal, a trade ledger with percent
results, and a plain-language journal — no dollars, no account numbers, no order sizes.
Feed each page from a JSON data file published alongside it, so the bot can update the data
without touching the page. Then write a project doc `claude/site-update.md` that records
each page's URL and the exact steps to refresh it, and add a final "update status pages"
step to each scheduled-task prompt.
```

**Data shapes the bot writes each cycle:**

```
Cycle record  (one per run; id = <YYYY-MM-DD>-daily | -daily-2 | -research)
{ "date", "type": "daily"|"research", "headline", "traded": true|false,
  "accountValue", "cash", "buyingPower", "drawdownPct",
  "narrative": "3–8 sentences: what was checked, what happened, why",
  "positions":  [ { "ticker","bucket","entryDate","entryPrice","shares","cost","last",
                    "pnlPct","highestClose","stop","stopType","exitCheck" } ],
  "exits":      [ { "ticker","rule","limit","resultPct","status" } ],
  "candidates": [ { "ticker", "checks": [ { "name","pass","detail" } ], "result","note" } ],
  "ordersReviewed": [ { "ticker","side","limit","dollars","outcome" } ],
  "ordersPlaced":   [ { "ticker","side","limit","dollars","shares","status" } ],
  "researchAdds": [...], "researchRemoves": [...], "rejected": [...],
  "blocked": [...], "errors": [...], "notes": [...], "emailSubject" }

Live state  (overwritten each run)
{ "updatedAt", "accountMasked", "accountValue", "cash", "buyingPower", "unsettled",
  "startingValue", "startDate", "endDate", "goalValue", "haltValue", "halted": false,
  "positions": [...], "openOrders": [...], "alerts": [...] }

Public journal  (Ledger — percentages and tickers only)
{ "series":  [ { "date", "pct" } ],                      // one point per trading day
  "trades":  [ { "ticker","bucket","entryDate","entryReason","status",
                 "exitDate","exitReason","resultPct" } ],
  "entries": [ { "date","type","headline","traded","body","candidates" | "adds","removes" } ] }
```

**Rules worth keeping in your `site-update.md`:**

- The site update is always the **last** step, after the log and the email. A site-update failure never changes or re-runs any trading step — log it and move on.
- Never put dollars, account numbers, or order sizes in anything public.
- Include every candidate screened, with the criterion that failed. Names that failed only the first criterion can be grouped into one row ("all others (28)").

---

## 7. Lessons learned from the first week

1. **Robinhood's investor-profile gate.** A brand-new account let the first order through, then rejected every order after it with *"We're required to have you answer some questions about your investing goals."* That blocked entries for two days — and would have blocked a **stop-loss exit** too. Complete the profile questionnaire in the app before going live.
2. **Limit orders must be whole shares.** Robinhood only allows fractional / dollar-sized orders as *market* orders. A limit-only bot has to size as `floor(max position ÷ price)`. Side effect: on a small account, high-priced stocks end up under-sized or skipped entirely. Pick your position cap and watchlist with that in mind.
3. **Turn on automatic approval for every trading task.** Otherwise the run pauses at the first order waiting for a click that never comes.
4. **Put the numbers in one doc.** Early on, values lived in both the project instructions and the prompts and drifted apart. One authoritative config, with prompts that say "read the config," fixed it.
5. **Make "do nothing" the default for every surprise.** Tool error, stale data, missing config value, anything ambiguous → no orders, log it, email. The one time the session was suspended mid-run for hours, this rule plus "re-quote and re-review before placing" is what kept a stale-priced order from going out.
6. **Always preview before placing.** `review_equity_order` first, every time, and compare the result against buying power and every limit. It catches stale quotes and broker alerts.
7. **Guard against duplicate firings.** A scheduled task fired twice one afternoon. The prompts now check the log first and skip entries if the same run type already traded today.
8. **Schedules are in UTC, and clocks change.** A task pinned to 14:00 UTC is 10 AM Eastern in summer and 9 AM in winter. Either schedule a one-shot task to shift the crons when DST ends (prompt included above) or accept the drift.
9. **Plan the ending up front.** Give the experiment an end date, a final report, and a task that disables the recurring tasks once the account is flat — so nothing keeps trading unattended after you've stopped paying attention.
10. **Big tool outputs overflow.** Price history for 40 tickers is too large to read in one go; the bot saves it and processes it with a script. If a run reports this, that's normal — just make sure the log says all bars were read.
11. **Log the judgment calls.** Where a rule was ambiguous (e.g. a stock with too little history for a 200-day average), the bot chose the conservative reading and wrote down why. Those notes are the most useful thing to read at the monthly review.
12. **"Within the rules" isn't the same as "good."** One entry filled ~6% above the prior close because the screen runs on yesterday's bar while the order prices off the live quote. It was allowed — and flagged as a "chasing" data point for the review. Expect to find gaps like this; fix them at the review, not on the fly.

---

## 8. Pre-flight checklist

- [ ] Separate Robinhood **cash** account — no margin, options disabled
- [ ] Investor-profile questionnaire completed
- [ ] Funded; starting value and date recorded in the config
- [ ] Robinhood + Gmail connectors attached to the Project
- [ ] Project instructions pasted; `bot-config.md` has **no** `{{PLACEHOLDERS}}` or UNSET values left
- [ ] Prompts contain your account number, email and dates — search each for `{{`
- [ ] Weekend dry run completed and the log reads sensibly
- [ ] Scheduled tasks created; **automatic approval on** for each
- [ ] First live email received and read
- [ ] Review date on your calendar — and a promise to yourself not to change rules before it

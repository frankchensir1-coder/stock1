# Monitoring ICT Order Flow — Design Specification

Date: 2026-06-22

## 1. Objective

Create a Codex skill and event-driven monitoring system for six Binance spot symbols on TradingView 1-hour charts. The system identifies SMS/BMS market structure, displacement ranges, premium/discount, OTE, FVG, and Order Blocks, then sends plain-text Outlook email alerts when price enters a relevant long or short observation zone.

The system provides monitoring and analysis only. It must never place, modify, or close trades.

## 2. Monitored Markets

- `BINANCE:BTCUSDT`
- `BINANCE:ETHUSDT`
- `BINANCE:LTCUSDT`
- `BINANCE:DOTUSDT`
- `BINANCE:BNBUSDT`
- `BINANCE:DASHUSDT`

Only the TradingView 1-hour timeframe is valid. Events for other symbols or timeframes are rejected.

## 3. Architecture

Use a hybrid, event-driven architecture:

1. TradingView runs the Pine indicator independently on each approved 1-hour chart.
2. Pine is the single source of truth for chart structure and zone detection.
3. Pine sends JSON Webhooks for stage changes and zone state transitions.
4. Azure Functions authenticates and validates incoming events.
5. Azure Table Storage persists stages, independent zones, entry counts, exhaustion, invalidation, and idempotency state.
6. Azure Storage Queue buffers Outlook sends and retries transient failures.
7. Microsoft Graph sends plain-text email from a personal Outlook account.

The runtime does not invoke Codex or another language model. Routine monitoring therefore consumes no model tokens.

## 4. Market Structure

### 4.1 Pivot detection

Use the supplied Pine v5 logic:

- Swing high: `ta.pivothigh(high, 5, 5)`
- Swing low: `ta.pivotlow(low, 5, 5)`
- Pivots become usable only after five right-side bars confirm them.

### 4.2 Break detection

- Bullish break: 1-hour closing price crosses above the latest confirmed swing high.
- Bearish break: 1-hour closing price crosses below the latest confirmed swing low.
- Add `barstate.isconfirmed` so intrabar excursions cannot create SMS/BMS stages.
- Consume a broken swing level once to prevent duplicate structure signals.

### 4.3 Stage classification

- `bullish_sms`: bearish or neutral state shifts bullish.
- `bullish_bms`: bullish state continues upward.
- `bearish_sms`: bullish or neutral state shifts bearish.
- `bearish_bms`: bearish state continues downward.

Each confirmed SMS/BMS creates a new stage and terminates all alerting from the previous stage. A stage ID contains symbol, timeframe, structure type, direction, and structure-bar timestamp.

SMS/BMS stage creation does not send email. The current stage is included in later actionable zone emails.

## 5. Displacement Range and Premium/Discount

The displacement leg is the complete impulse that causes the confirmed SMS/BMS:

- The origin is the nearest confirmed opposing pivot before the structure break.
- The endpoint initially uses the extreme of the confirmed structure-break bar.
- The endpoint expands while price makes a new extreme in the displacement direction.
- The range freezes when price first returns to the 50% equilibrium level.
- It remains frozen until the next confirmed SMS/BMS stage.

For a bullish displacement, values below equilibrium are discount and values above equilibrium are premium. For a bearish displacement, the price geometry remains the same; the observation direction is assigned by zone semantics below.

## 6. Zone Definitions

Every independent zone has a stable zone ID, its own state, and its own two-entry allowance.

### 6.1 OTE

- Calculate Fibonacci retracement from the frozen displacement leg.
- OTE covers 62%–79% retracement.
- Bullish-displacement OTE is a long observation zone in discount.
- Bearish-displacement OTE is a short observation zone in premium.
- It becomes invalid when a confirmed 1-hour candle closes beyond the displacement origin.

### 6.2 Fair Value Gap

- Bullish FVG: the third candle's low is above the first candle's high.
- Bearish FVG: the third candle's high is below the first candle's low.
- Only FVGs created within the current SMS/BMS stage are active.
- Entry into the gap triggers an observation event.
- A confirmed 1-hour body close through the far boundary invalidates the FVG.

### 6.3 Order Block

- Bullish OB: the last bearish 1-hour candle before the bullish displacement that causes SMS/BMS.
- Bearish OB: the last bullish 1-hour candle before the bearish displacement that causes SMS/BMS.
- Use the complete candle high-low range.
- Entry into the range triggers an observation event.
- A confirmed 1-hour body close through the far boundary invalidates the OB.

### 6.4 Premium and discount

- Entry into discount produces a long observation alert.
- Entry into premium produces a short observation alert.
- Stage direction does not suppress countertrend observation alerts.

## 7. Zone State Machine

Each zone moves through:

`waiting -> inside_1 -> outside_1 -> inside_2 -> outside_2 -> exhausted_on_third_entry`

Alternative terminal states are `invalidated` and `stage_ended`.

Rules:

- Pine sends `zone_enter`, `zone_exit`, and `zone_invalidated` events.
- Only an outside-to-inside transition increments the entry count.
- Remaining inside a zone cannot generate another entry.
- A second entry requires an observed exit followed by re-entry.
- Entries one and two send email. The second entry must be followed by an exit before a third entry can exhaust the zone.
- Entry three marks the zone exhausted and sends no email.
- Each independent FVG, OB, OTE, premium zone, and discount zone has a separate count.
- A new SMS/BMS stage ends all zones from the previous stage.

When several zones are entered during the same Pine evaluation, Pine sends them together in one Webhook payload. Azure updates every zone independently and combines their details into one email.

## 8. Direction and Email Contract

Send email only for long or short observation events. Do not send stage-confirmation emails.

Direction mapping:

- Discount, bullish OTE, bullish FVG, bullish OB: observe long.
- Premium, bearish OTE, bearish FVG, bearish OB: observe short.

Required email fields:

- Symbol and `1H` timeframe
- Observation direction: long or short
- Current stage: bullish/bearish SMS/BMS and continuation/shift wording
- Triggered zone types and boundaries
- Trigger price and event timestamp
- Entry count for every triggered zone, for example `2/2`
- Structure-break price
- Displacement origin, endpoint, equilibrium, and frozen/dynamic state
- Simultaneous confluences without using them as filters
- Invalidation boundary
- Explicit text: `1h 尚未收盘` for intrabar zone touches
- Explicit text: monitoring only; no automatic trade execution

Email bodies are plain text because the Outlook connector and Graph path do not require rich formatting.

## 9. Webhook and Persistence Contract

Webhook payloads contain version, event type, symbol, timeframe, stage ID, stage classification, structure timestamp, current price, event timestamp, displacement metadata, and a `zones[]` array. Each array item contains zone ID, zone type, zone direction, transition type, boundaries, and Pine's current inside/outside state. Stage events may have an empty `zones[]` array and update Azure state without sending email.

Azure Functions must:

- Authenticate requests with an Azure Function Key configured only in the TradingView alert URL.
- Reject symbols outside the six-symbol allowlist.
- Reject timeframes other than `60`/`1h`.
- Validate schema version and required fields.
- Process state transitions atomically.
- Treat duplicate entry events while already inside as idempotent.
- Persist accepted events before enqueueing email.

## 10. Outlook Security and Recovery

Use a personal Microsoft account with Microsoft Graph delegated OAuth:

- Request only `Mail.Send`, `offline_access`, and minimum sign-in scopes required by Microsoft identity.
- Complete one interactive browser authorization during deployment.
- Store refresh credentials in Azure Key Vault.
- Never store secrets in Pine, the skill, source control, logs, or email bodies.
- Queue mail before sending.
- Retry transient Graph failures with bounded exponential backoff.
- Preserve unsent events if authorization is revoked or expires.
- Resume queued sends after reauthorization without duplicating already-sent messages.

## 11. Backtest and Validation

Use the latest 12 months of Binance 1-hour OHLC data for all six approved symbols.

Export a CSV event ledger containing:

- Structure stages and confirmation times
- Displacement ranges and freeze times
- All independent zones and boundaries
- Entry, exit, exhaustion, invalidation, and stage-end events
- Long/short observation direction
- Entry count and whether an email would be sent
- Maximum favorable excursion and maximum adverse excursion after 6, 12, and 24 hours

The backtest evaluates rule consistency and alert burden, not profitability.

Required regression scenarios:

- Intrabar structure excursion without a confirmed close creates no stage.
- Confirmed structure close creates the correct SMS/BMS stage.
- Intrabar zone touch creates an immediate entry event.
- Remaining inside a zone produces no duplicate entry.
- Exit followed by re-entry produces the second alert.
- Third entry exhausts the zone without email.
- Independent FVGs and OBs retain independent counts.
- Simultaneous zone entries merge into one email.
- Stage creation alone produces no email.
- A new stage ends all old-stage alerting.
- Confirmed closes through far boundaries invalidate the correct zones.

## 12. Deliverable Structure

```text
monitoring-ict-orderflow/
├── SKILL.md
├── agents/
│   └── openai.yaml
├── references/
│   ├── strategy-rules.md
│   └── event-email-schema.md
├── scripts/
│   ├── backtest.py
│   └── validate-events.py
├── assets/
│   ├── pine/
│   │   └── ict-orderflow-alerts.pine
│   └── azure-function/
│       ├── webhook/
│       ├── outlook-mailer/
│       └── storage-state/
└── tests/
    ├── structure/
    ├── zones/
    └── notifications/
```

Develop and validate the skill in the shared workspace first. Install it into the Codex skills directory only after validation succeeds.

## 13. Explicit Non-goals

- No automated order placement or portfolio access.
- No lower- or higher-timeframe monitoring.
- No unapproved symbols or exchanges.
- No model call for routine monitoring or email generation.
- No profitability claim or financial-advice representation.
- No confluence requirement before alerting.

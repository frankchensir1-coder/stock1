# Monitoring ICT Order Flow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build, backtest, deploy, and package an event-driven ICT order-flow monitoring skill that sends at most two Outlook alerts per independent 1h zone and never executes trades.

**Architecture:** TradingView Pine is the chart-level source of truth for confirmed SMS/BMS stages and intrabar zone transitions. Azure Functions validates Webhooks, applies an atomic persistent state machine in Azure Table Storage, batches simultaneous zone entries, and sends plain-text Outlook mail through Microsoft Graph. A Python reference engine replays Binance 1h history for parity checks and the approved 12-month event ledger.

**Tech Stack:** Pine Script v5; Python 3.11; Azure Functions Python v2 programming model; Azure Table Storage; Azure Storage Queue; Azure Key Vault; Microsoft Graph delegated OAuth; pytest; Ruff; mypy; Bicep; TradingView Webhooks.

## Global Constraints

- Accept only `BINANCE:BTCUSDT`, `BINANCE:ETHUSDT`, `BINANCE:LTCUSDT`, `BINANCE:DOTUSDT`, `BINANCE:BNBUSDT`, and `BINANCE:DASHUSDT`.
- Accept only TradingView `60`/`1h` events.
- Confirm SMS/BMS only on a closed 1h bar; allow intrabar zone-entry alerts and label them `1h 尚未收盘`.
- Use `ta.pivothigh(high, 5, 5)` and `ta.pivotlow(low, 5, 5)` semantics.
- Do not send stage-creation email.
- Send email for entries one and two of each independent zone; entry three exhausts the zone without email.
- Treat each premium, discount, OTE, FVG, and Order Block as an independently counted zone.
- Never connect to an exchange account or place, modify, or close an order.
- Do not invoke a language model in the deployed monitoring path.
- Never commit Outlook OAuth credentials, Azure Function keys, refresh tokens, `.env`, or `local.settings.json`.

---

## Planned File Map

```text
README.md
monitoring-ict-orderflow/
├── SKILL.md
├── agents/openai.yaml
├── references/
│   ├── strategy-rules.md
│   └── event-email-schema.md
├── scripts/
│   ├── backtest.py
│   ├── validate-events.py
│   └── authorize-outlook.py
├── assets/
│   ├── pine/ict-orderflow-alerts.pine
│   └── azure-function/
│       ├── function_app.py
│       ├── host.json
│       ├── requirements.txt
│       └── ict_alerts/
│           ├── contracts.py
│           ├── state_machine.py
│           ├── storage.py
│           ├── notifications.py
│           └── graph_mailer.py
├── infra/
│   ├── main.bicep
│   └── main.parameters.example.json
└── tests/
    ├── fixtures/
    ├── structure/
    ├── zones/
    ├── notifications/
    ├── integration/
    └── skill/
pyproject.toml
```

## Task 1: Bootstrap the Skill and Test Harness

**Files:**
- Create: `pyproject.toml`
- Create: `monitoring-ict-orderflow/` using the official `init_skill.py`
- Create: `monitoring-ict-orderflow/tests/conftest.py`
- Create: `monitoring-ict-orderflow/tests/fixtures/candles.json`

**Interfaces:**
- Produces: Python package import path `ict_alerts`; shared pytest fixtures `bullish_break_candles` and `zone_reentry_prices`.
- Consumes: Approved design specification and global constraints above.

- [ ] **Step 1: Verify toolchain prerequisites**

Run:

```powershell
python --version
func --version
az version
```

Expected: Python reports `3.11.x`; Azure Functions Core Tools reports major version `4`; Azure CLI returns JSON. If a command is missing, install that prerequisite before creating project files.

- [ ] **Step 2: Initialize the skill with only required resource directories**

Run:

```powershell
python C:\Users\Administrator\.codex\skills\.system\skill-creator\scripts\init_skill.py monitoring-ict-orderflow --path . --resources scripts,references,assets --interface "display_name=Monitoring ICT Order Flow" --interface "short_description=Monitor 1h ICT structure and observation zones" --interface "default_prompt=Use $monitoring-ict-orderflow to inspect or operate the configured 1h ICT alert pipeline without trading."
```

Expected: `monitoring-ict-orderflow/SKILL.md` and `monitoring-ict-orderflow/agents/openai.yaml` exist, with no example placeholders retained.

- [ ] **Step 3: Add deterministic Python project configuration**

Create `pyproject.toml`:

```toml
[project]
name = "monitoring-ict-orderflow"
version = "0.1.0"
requires-python = ">=3.11,<3.12"
dependencies = [
  "azure-data-tables>=12.5,<13",
  "azure-functions>=1.20,<2",
  "azure-identity>=1.17,<2",
  "azure-keyvault-secrets>=4.8,<5",
  "azure-storage-queue>=12.11,<13",
  "msal>=1.30,<2",
  "requests>=2.32,<3",
]

[project.optional-dependencies]
dev = ["mypy>=1.11,<2", "pytest>=8.3,<9", "pytest-cov>=5,<6", "ruff>=0.6,<1"]

[tool.pytest.ini_options]
testpaths = ["monitoring-ict-orderflow/tests"]
addopts = "-q --strict-markers"

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.mypy]
python_version = "3.11"
strict = true
```

- [ ] **Step 4: Install dependencies and verify collection**

Run:

```powershell
python -m venv .venv
.\.venv\Scripts\python -m pip install -e ".[dev]"
.\.venv\Scripts\python -m pytest --collect-only
```

Expected: pytest exits `0`; collection contains no import or configuration errors.

- [ ] **Step 5: Commit bootstrap**

```powershell
git add pyproject.toml monitoring-ict-orderflow
git commit -m "chore: scaffold ICT monitoring skill"
```

## Task 2: Implement the Reference Structure and Zone Engine

**Files:**
- Create: `monitoring-ict-orderflow/assets/azure-function/ict_alerts/contracts.py`
- Create: `monitoring-ict-orderflow/assets/azure-function/ict_alerts/state_machine.py`
- Test: `monitoring-ict-orderflow/tests/structure/test_structure_engine.py`
- Test: `monitoring-ict-orderflow/tests/zones/test_zone_engine.py`

**Interfaces:**
- Produces: `Candle`, `Stage`, `Zone`, `ZoneTransition`, `classify_break()`, `find_fvgs()`, `build_order_block()`, `build_ote()`, and `transition_zone()`.
- Consumes: Chronological 1h `Candle` values and the current confirmed pivots.

- [ ] **Step 1: Write failing structure tests**

Create `tests/structure/test_structure_engine.py` with tests asserting:

```python
def test_intrabar_high_above_pivot_does_not_confirm_break() -> None:
    event = classify_break(close=99.0, pivot_high=100.0, pivot_low=90.0,
                           trend_dir=-1, bar_confirmed=False)
    assert event is None

def test_confirmed_close_above_pivot_changes_bearish_to_bullish_sms() -> None:
    event = classify_break(close=101.0, pivot_high=100.0, pivot_low=90.0,
                           trend_dir=-1, bar_confirmed=True)
    assert event == ("bullish_sms", 1, 100.0)

def test_confirmed_close_above_pivot_continues_bullish_bms() -> None:
    event = classify_break(close=101.0, pivot_high=100.0, pivot_low=90.0,
                           trend_dir=1, bar_confirmed=True)
    assert event == ("bullish_bms", 1, 100.0)
```

- [ ] **Step 2: Run structure tests and verify RED**

Run: `.\.venv\Scripts\python -m pytest monitoring-ict-orderflow/tests/structure/test_structure_engine.py -v`

Expected: FAIL because `classify_break` and its module do not exist.

- [ ] **Step 3: Implement immutable contracts and minimal structure classification**

Use frozen dataclasses for candles, stages, zones, and transitions. Implement this exact public signature:

```python
def classify_break(
    *, close: float, pivot_high: float | None, pivot_low: float | None,
    trend_dir: int, bar_confirmed: bool,
) -> tuple[str, int, float] | None:
    if not bar_confirmed:
        return None
    if pivot_high is not None and close > pivot_high:
        return ("bullish_sms" if trend_dir <= 0 else "bullish_bms", 1, pivot_high)
    if pivot_low is not None and close < pivot_low:
        return ("bearish_sms" if trend_dir >= 0 else "bearish_bms", -1, pivot_low)
    return None
```

- [ ] **Step 4: Write failing zone geometry and re-entry tests**

Test the exact approved formulas: bullish/bearish three-candle FVG, full-range opposing-candle OB, 62%–79% OTE, equilibrium freeze, outside-to-inside transitions, second entry, and silent exhaustion on third entry.

```python
def test_third_entry_exhausts_without_email() -> None:
    zone = ZoneState(entry_count=2, inside=False, exhausted=False, invalidated=False)
    result = transition_zone(zone, now_inside=True)
    assert result.state.entry_count == 3
    assert result.state.exhausted is True
    assert result.send_email is False
```

- [ ] **Step 5: Run zone tests and verify RED**

Run: `.\.venv\Scripts\python -m pytest monitoring-ict-orderflow/tests/zones/test_zone_engine.py -v`

Expected: FAIL for missing geometry and transition functions.

- [ ] **Step 6: Implement minimal zone engine and run GREEN**

Implement pure functions with no Azure or network imports. Use inclusive boundaries (`low <= price <= high`), count only `False -> True`, set `send_email` only for counts 1 and 2, and set `exhausted` on count 3.

Run:

```powershell
.\.venv\Scripts\python -m pytest monitoring-ict-orderflow/tests/structure monitoring-ict-orderflow/tests/zones -v
.\.venv\Scripts\python -m ruff check monitoring-ict-orderflow/assets/azure-function/ict_alerts
.\.venv\Scripts\python -m mypy monitoring-ict-orderflow/assets/azure-function/ict_alerts
```

Expected: all tests PASS; Ruff and mypy exit `0`.

- [ ] **Step 7: Commit engine**

```powershell
git add monitoring-ict-orderflow/assets/azure-function/ict_alerts monitoring-ict-orderflow/tests
git commit -m "feat: add ICT structure and zone state engine"
```

## Task 3: Extend the TradingView Pine Indicator

**Files:**
- Create: `monitoring-ict-orderflow/assets/pine/ict-orderflow-alerts.pine`
- Create: `monitoring-ict-orderflow/tests/pine/test_pine_contract.py`
- Create: `monitoring-ict-orderflow/references/event-email-schema.md`

**Interfaces:**
- Produces: Versioned JSON events `stage_created`, `zone_enter`, `zone_exit`, and `zone_invalidated`, including a `zones[]` batch.
- Consumes: TradingView chart OHLC, approved six-symbol allowlist, 1h timeframe, and the supplied pivot/SMS/BMS logic.

- [ ] **Step 1: Write failing Pine contract tests**

Create tests that read the Pine source and require these literals and guards:

```python
assert "//@version=5" in source
assert "barstate.isconfirmed and ta.crossover(close, last_ph)" in source
assert "barstate.isconfirmed and ta.crossunder(close, last_pl)" in source
assert '"version":"1"' in source
assert '"zones":[' in source
assert "timeframe.period == \"60\"" in source
```

- [ ] **Step 2: Run contract tests and verify RED**

Run: `.\.venv\Scripts\python -m pytest monitoring-ict-orderflow/tests/pine/test_pine_contract.py -v`

Expected: FAIL because the Pine file is absent.

- [ ] **Step 3: Implement confirmed structure and dynamic-to-frozen displacement**

Start from the professor's supplied Pine v5 source. Preserve pivot widths `5/5`; add `barstate.isconfirmed`; generate stable stage IDs from `syminfo.tickerid`, `timeframe.period`, structure type, direction, and structure bar `time`; extend the displacement endpoint only in its impulse direction; freeze it on first equilibrium return.

- [ ] **Step 4: Implement independent zones and transition alerts**

Maintain separate arrays for active FVG and OB records and single stage-specific premium, discount, and OTE records. Each record contains ID, lower/upper bound, direction, inside flag, invalidated flag, and creation time. Aggregate all transitions found during one Pine evaluation into a single `zones[]` JSON payload and call `alert(payload, alert.freq_all)` once.

- [ ] **Step 5: Document the exact JSON schema and examples**

Define required types, numeric boundaries, UTC epoch milliseconds, accepted event names, a batched two-zone entry example, and rejection examples for a wrong symbol and wrong timeframe. The schema must match `contracts.py` field names exactly.

- [ ] **Step 6: Run source contract tests and manually verify TradingView compilation**

Run: `.\.venv\Scripts\python -m pytest monitoring-ict-orderflow/tests/pine -v`

Then paste the Pine source into TradingView Pine Editor, save, and add it to `BINANCE:BTCUSDT` on `1h`.

Expected: pytest PASS; TradingView reports no compiler errors; an intrabar pivot excursion does not create a stage; Data Window shows the expected stage after a confirmed close.

- [ ] **Step 7: Commit Pine and schema**

```powershell
git add monitoring-ict-orderflow/assets/pine monitoring-ict-orderflow/tests/pine monitoring-ict-orderflow/references/event-email-schema.md
git commit -m "feat: emit ICT stage and zone events from Pine"
```

## Task 4: Build the Azure Webhook and Atomic Persistence

**Files:**
- Create: `monitoring-ict-orderflow/assets/azure-function/function_app.py`
- Create: `monitoring-ict-orderflow/assets/azure-function/ict_alerts/storage.py`
- Create: `monitoring-ict-orderflow/assets/azure-function/host.json`
- Create: `monitoring-ict-orderflow/assets/azure-function/requirements.txt`
- Test: `monitoring-ict-orderflow/tests/integration/test_webhook.py`
- Test: `monitoring-ict-orderflow/tests/integration/test_storage.py`

**Interfaces:**
- Produces: HTTP `POST /api/tradingview-webhook`; queue messages for accepted email-worthy transitions.
- Consumes: Version-1 Pine payload and Azure Function-level key.

- [ ] **Step 1: Write failing validation tests**

Test HTTP `400` for malformed JSON, `422` for wrong schema, `403` for an unapproved symbol or timeframe, `202` for accepted stage-only state, and `202` with one queued notification for a valid first entry.

- [ ] **Step 2: Run Webhook tests and verify RED**

Run: `.\.venv\Scripts\python -m pytest monitoring-ict-orderflow/tests/integration/test_webhook.py -v`

Expected: FAIL because `function_app.py` and request validation are absent.

- [ ] **Step 3: Implement typed payload validation**

Implement `parse_event(payload: Mapping[str, object]) -> WebhookEvent` with exact allowlists and finite-number checks. Reject lower bounds greater than upper bounds, timestamps more than 10 minutes in the future, duplicate zone IDs inside one batch, and transition values outside the four approved event types.

- [ ] **Step 4: Write failing atomic-state tests**

Use an in-memory `ZoneRepository` fake to simulate duplicate delivery, concurrent identical entry events, exit/re-entry, second entry, and third-entry exhaustion. Assert a duplicate while already inside never increments or queues another email.

- [ ] **Step 5: Implement Azure Table repository with optimistic concurrency**

Store stage rows by partition `symbol|timeframe` and zone rows by partition `stage_id`, row key `zone_id`. Read entity ETags and use conditional updates; retry an ETag conflict up to three times by re-reading and reapplying the pure `transition_zone()` function.

- [ ] **Step 6: Run integration tests locally**

Run:

```powershell
.\.venv\Scripts\python -m pytest monitoring-ict-orderflow/tests/integration/test_webhook.py monitoring-ict-orderflow/tests/integration/test_storage.py -v
func start
```

Expected: tests PASS; Functions host exposes `tradingview-webhook`; a signed local fixture returns `202`.

- [ ] **Step 7: Commit Webhook and storage**

```powershell
git add monitoring-ict-orderflow/assets/azure-function monitoring-ict-orderflow/tests/integration
git commit -m "feat: persist idempotent TradingView zone events"
```

## Task 5: Implement Batched Plain-Text Outlook Notifications

**Files:**
- Create: `monitoring-ict-orderflow/assets/azure-function/ict_alerts/notifications.py`
- Create: `monitoring-ict-orderflow/assets/azure-function/ict_alerts/graph_mailer.py`
- Create: `monitoring-ict-orderflow/scripts/authorize-outlook.py`
- Test: `monitoring-ict-orderflow/tests/notifications/test_notifications.py`
- Test: `monitoring-ict-orderflow/tests/notifications/test_graph_mailer.py`

**Interfaces:**
- Produces: `render_notification(event, transitions) -> tuple[str, str]`; Graph `/me/sendMail` request; refresh-token rotation.
- Consumes: Email-worthy transition batch, personal Microsoft account OAuth configuration, and Key Vault secret names.

- [ ] **Step 1: Write failing rendering tests**

Assert the subject includes symbol, `1H`, long/short, zone names, and highest entry count. Assert the body includes current SMS/BMS stage, continuation/shift wording, trigger price, all boundaries/counts, displacement values, invalidation, `1h 尚未收盘`, and the no-trading disclaimer. Assert a mixed long/short batch is split into one email per direction.

- [ ] **Step 2: Run rendering tests and verify RED**

Run: `.\.venv\Scripts\python -m pytest monitoring-ict-orderflow/tests/notifications/test_notifications.py -v`

Expected: FAIL because the renderer is absent.

- [ ] **Step 3: Implement deterministic plain-text renderer**

Sort zones by direction, then type, then zone ID. Render all timestamps in both UTC and `Asia/Shanghai`. Do not call a model or use HTML. Return one subject/body pair per direction.

- [ ] **Step 4: Write failing OAuth and Graph tests**

Mock only HTTP boundaries. Assert refresh uses tenant `consumers`, scope `offline_access Mail.Send`, rotates a returned refresh token into Key Vault, sends through `/v1.0/me/sendMail`, retries HTTP `429` using `Retry-After`, retries `5xx` with bounded exponential backoff, and preserves the queue message after terminal failure.

- [ ] **Step 5: Implement one-time authorization and queue-trigger mailer**

`authorize-outlook.py` uses MSAL device-code flow and stores the initial refresh credential in Key Vault after interactive approval. The queue trigger reads the secret, refreshes access, writes a rotated refresh token before sending, calls Graph, and records an idempotency key only after HTTP `202`.

- [ ] **Step 6: Run notification tests**

Run: `.\.venv\Scripts\python -m pytest monitoring-ict-orderflow/tests/notifications -v`

Expected: all tests PASS with no real email sent.

- [ ] **Step 7: Commit Outlook notifications**

```powershell
git add monitoring-ict-orderflow/assets/azure-function/ict_alerts monitoring-ict-orderflow/scripts monitoring-ict-orderflow/tests/notifications
git commit -m "feat: send idempotent Outlook observation alerts"
```

## Task 6: Provision Azure Resources and Deploy Safely

**Files:**
- Create: `monitoring-ict-orderflow/infra/main.bicep`
- Create: `monitoring-ict-orderflow/infra/main.parameters.example.json`
- Create: `monitoring-ict-orderflow/scripts/validate-events.py`
- Modify: `.gitignore`

**Interfaces:**
- Produces: Function App URL, function key, storage tables/queue, Key Vault, managed identity access, Application Insights, and deployment outputs.
- Consumes: Azure subscription, resource-group location, personal Microsoft app client ID, sender and recipient addresses.

- [ ] **Step 1: Write failing infrastructure checks**

Create a test that invokes `az bicep build --file monitoring-ict-orderflow/infra/main.bicep` and scans the generated template for HTTPS-only, TLS 1.2 minimum, system-assigned managed identity, Key Vault references, Table/Queue storage, and Application Insights.

- [ ] **Step 2: Verify infrastructure check fails before Bicep exists**

Run: `.\.venv\Scripts\python -m pytest monitoring-ict-orderflow/tests/integration/test_infra.py -v`

Expected: FAIL because `main.bicep` is absent.

- [ ] **Step 3: Implement Bicep resources and least-privilege role assignments**

Provision one consumption-plan Function App, one Storage Account, one Key Vault with RBAC, Application Insights, and role assignments allowing only the Function managed identity to read/write required secrets and storage. Expose no Outlook or Function secrets as Bicep outputs.

- [ ] **Step 4: Validate and deploy Bicep**

Run:

```powershell
az bicep build --file monitoring-ict-orderflow/infra/main.bicep
az deployment group what-if --resource-group ict-orderflow-rg --template-file monitoring-ict-orderflow/infra/main.bicep --parameters monitoring-ict-orderflow/infra/main.parameters.json
az deployment group create --resource-group ict-orderflow-rg --template-file monitoring-ict-orderflow/infra/main.bicep --parameters monitoring-ict-orderflow/infra/main.parameters.json
```

Expected: build exits `0`; what-if contains only planned resources; deployment reports `Succeeded`.

- [ ] **Step 5: Authorize Outlook and deploy Functions**

Run the one-time authorization script, publish with `func azure functionapp publish <function-app-name>`, store the recipient address in Function App configuration, and configure the TradingView alert URL with the Function key outside Pine source.

- [ ] **Step 6: Send a non-trading synthetic event**

Use `validate-events.py` to post a fixture for `BINANCE:BTCUSDT`, confirm HTTP `202`, one Table state transition, one Queue dequeue, and one Outlook test email containing `TEST EVENT — NO MARKET SIGNAL`.

- [ ] **Step 7: Commit infrastructure without local parameters**

```powershell
git add .gitignore monitoring-ict-orderflow/infra monitoring-ict-orderflow/scripts/validate-events.py
git commit -m "infra: deploy secure ICT alert pipeline"
```

## Task 7: Build the 12-Month Six-Symbol Backtest

**Files:**
- Create: `monitoring-ict-orderflow/scripts/backtest.py`
- Create: `monitoring-ict-orderflow/tests/backtest/test_backtest.py`
- Create: `monitoring-ict-orderflow/tests/fixtures/binance_1h_sample.json`

**Interfaces:**
- Produces: CSV ledger with stages, zones, transitions, email decision, MFE, and MAE at 6h/12h/24h.
- Consumes: Binance public 1h OHLC for the approved symbols and the pure engine from Task 2.

- [ ] **Step 1: Write failing deterministic fixture tests**

Assert chronological ordering, pivot confirmation delay of five right bars, no look-ahead, correct stage labels, independent zone counts, silent third entry, and exact MFE/MAE calculations from the fixed fixture.

- [ ] **Step 2: Run fixture tests and verify RED**

Run: `.\.venv\Scripts\python -m pytest monitoring-ict-orderflow/tests/backtest/test_backtest.py -v`

Expected: FAIL because `backtest.py` is absent.

- [ ] **Step 3: Implement paginated Binance download and replay**

Use public klines only; request at most 1000 bars per page; persist raw response hashes; reject gaps or duplicate open times; replay candles sequentially through the same pure rules used by the Webhook path. Do not infer intrabar ordering unavailable from 1h OHLC; mark same-bar zone-entry ordering as `ohlc_ambiguous`.

- [ ] **Step 4: Implement the CSV contract**

Include symbol, stage ID/type/direction, structure time/price, displacement origin/endpoint/equilibrium/frozen, zone ID/type/direction/bounds, transition, entry count, email decision, event price/time, invalidation, ambiguity flag, and favorable/adverse excursions at 6, 12, and 24 bars.

- [ ] **Step 5: Run the approved 12-month backtest**

Run:

```powershell
.\.venv\Scripts\python monitoring-ict-orderflow/scripts/backtest.py --symbols BTCUSDT ETHUSDT LTCUSDT DOTUSDT BNBUSDT DASHUSDT --interval 1h --months 12 --output reports/ict-events.csv
```

Expected: six symbols complete with no timestamp gaps; the ledger contains no trade execution or P&L columns; summary reports stages, first/second alerts, exhausted zones, invalid zones, and ambiguous same-bar cases.

- [ ] **Step 6: Review a stratified sample against TradingView**

For each symbol, inspect at least five SMS, five BMS, five first entries, five second entries, and every third-entry exhaustion if fewer than five. Record Pine/reference discrepancies as failing fixtures before changing code.

- [ ] **Step 7: Commit backtest code and small fixtures only**

```powershell
git add monitoring-ict-orderflow/scripts/backtest.py monitoring-ict-orderflow/tests/backtest monitoring-ict-orderflow/tests/fixtures
git commit -m "feat: replay twelve months of ICT alert events"
```

Do not commit the generated 12-month CSV; `reports/` remains ignored.

## Task 8: Write and Validate the Codex Skill

**Files:**
- Modify: `monitoring-ict-orderflow/SKILL.md`
- Modify: `monitoring-ict-orderflow/agents/openai.yaml`
- Create: `monitoring-ict-orderflow/references/strategy-rules.md`
- Create: `monitoring-ict-orderflow/tests/skill/scenarios.md`

**Interfaces:**
- Produces: Discoverable skill `monitoring-ict-orderflow` with concise workflow and conditional references.
- Consumes: Validated runtime, exact strategy rules, deployment commands, and observed baseline failures.

- [ ] **Step 1: Record RED baseline scenarios without the skill**

Run fresh-context scenarios covering: interpreting a bullish BMS discount re-entry; distinguishing intrabar structure from confirmed structure; explaining third-entry exhaustion; diagnosing duplicate email; and verifying no trade execution. Record exact omissions and incorrect assumptions in `tests/skill/scenarios.md`.

- [ ] **Step 2: Write minimal SKILL.md against observed failures**

Use only `name` and `description` frontmatter. Start description with `Use when...` and include TradingView, ICT, SMS, BMS, OTE, FVG, Order Block, Azure, Outlook, alert diagnosis, and backtest triggers without summarizing the workflow. Keep the body below 500 words and point to references only when their details are needed.

- [ ] **Step 3: Write strategy reference and regenerate UI metadata**

Move all approved definitions and state transitions into `references/strategy-rules.md`; do not duplicate them in SKILL.md. Regenerate `agents/openai.yaml` with the official helper and the three approved interface values from Task 1.

- [ ] **Step 4: Run skill validation**

Run:

```powershell
python C:\Users\Administrator\.codex\skills\.system\skill-creator\scripts\quick_validate.py monitoring-ict-orderflow
```

Expected: validation exits `0` with valid naming and frontmatter.

- [ ] **Step 5: Run GREEN forward scenarios with the skill**

Run the same fresh-context scenarios with only the task and installed skill visible. Verify correct stage, direction, zone count, confirmation status, non-trading boundary, and reference retrieval. Add a failing regression scenario before every skill wording change.

- [ ] **Step 6: Commit verified skill content**

```powershell
git add monitoring-ict-orderflow/SKILL.md monitoring-ict-orderflow/agents monitoring-ict-orderflow/references monitoring-ict-orderflow/tests/skill
git commit -m "feat: package verified ICT monitoring skill"
```

## Task 9: End-to-End Acceptance, Documentation, and Installation

**Files:**
- Modify: `README.md`
- Create: `monitoring-ict-orderflow/tests/integration/test_end_to_end.py`
- Modify: `.gitignore` only if generated sensitive paths are discovered.

**Interfaces:**
- Produces: Verified repository, deployed alert path, installed Codex skill, and operator instructions.
- Consumes: All prior task deliverables.

- [ ] **Step 1: Write an end-to-end test around fakes**

Feed a confirmed stage event, first entry, duplicate entry, exit, second entry, exit, and third entry. Assert zero stage emails, one first-entry email, no duplicate email, one second-entry email, no third-entry email, and an exhausted final state.

- [ ] **Step 2: Run the complete local quality gate**

Run:

```powershell
.\.venv\Scripts\python -m pytest --cov=monitoring-ict-orderflow/assets/azure-function/ict_alerts --cov-report=term-missing
.\.venv\Scripts\python -m ruff check monitoring-ict-orderflow
.\.venv\Scripts\python -m mypy monitoring-ict-orderflow/assets/azure-function/ict_alerts monitoring-ict-orderflow/scripts
python C:\Users\Administrator\.codex\skills\.system\skill-creator\scripts\quick_validate.py monitoring-ict-orderflow
az bicep build --file monitoring-ict-orderflow/infra/main.bicep
git diff --check
```

Expected: all commands exit `0`; pytest reports zero failures; no secret file is tracked.

- [ ] **Step 3: Run live non-trading acceptance**

Create TradingView alerts for all six symbols on `1h`. Use one explicitly labeled synthetic Webhook and one paper observation event. Confirm stage-only events send no email; entry events include current stage, direction, zones, counts, and `1h 尚未收盘`; Azure logs contain no secrets.

- [ ] **Step 4: Update README and development log**

Add exact setup, TradingView alert creation, Azure deployment, Outlook reauthorization, pause/disable, troubleshooting, backtest, and uninstall commands. Append dated entries for each deployed milestone and link the design and implementation plan.

- [ ] **Step 5: Install the verified skill**

Copy the validated `monitoring-ict-orderflow` directory to `$CODEX_HOME/skills/monitoring-ict-orderflow`, restart Codex, and verify that a relevant ICT monitoring prompt triggers it while an unrelated market question does not.

- [ ] **Step 6: Commit and push the acceptance state**

```powershell
git add README.md .gitignore monitoring-ict-orderflow
git commit -m "docs: finalize ICT alert operations and acceptance"
git push origin main
```

Expected: local `HEAD` equals `refs/heads/main` from `git ls-remote origin`; working tree is clean.


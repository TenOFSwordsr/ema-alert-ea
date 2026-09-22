# EMA Cross Alert Expert Advisor (MT5)

The genuine MQL5 source of the chart-side alerting EA that feeds the EMA alert platform.
It evaluates a fast/slow moving-average pair (RMA, EMA, SMA or LWMA) on a symbol and
timeframe, detects golden/death crosses with a point-distance noise filter, and pushes each
signal out over MT5 mobile push, email and an authenticated webhook. Version 3.00.
Architecture lives entirely in `includes/*.mqh`; `EMAAlertEA.mq5` is the orchestrator.

**Suggested repo name:** `ema-alert-ea`
**Stack:** MQL5 (MetaTrader 5 terminal), `WebRequest`, terminal common-files I/O
**Status:** active
**Last modified:** 2026-08-24

## What it does

The EA reads its effective configuration from the MT5 input dialog, then - when
`InpFollowChartIndicators` is on - mirrors the visual MA indicators the trader actually placed
on the chart, so the alert always describes what is on screen. It never places or modifies
orders; MT5 remains the only decision maker and this is a notifier plus a chart mirror.

- Cross detection on closed bars with `InpMinCrossDistancePoints` filtering out flat/noise crosses.
- Notification fan-out: MT5 push, SMTP email, webhook `POST` with timeout, retries and retry delay.
- Chart telemetry: writes `ema_ticker_<SYMBOL>_<TF>.json` into the terminal common files folder on
  a timer. The Go alert server's file watcher ingests those to serve the live `/market` API; the
  Python watchdog reads them to detect EA silence. File transport is used because the terminal
  rejects `WebRequest` from timer/tick context (error 4014).
- A chart `Comment()` overlay always shows exactly which symbol/timeframe/periods are being
  evaluated and where the settings came from (MT5 inputs vs chart indicators).

## Layout

```
EMAAlertEA.mq5              EA entrypoint (OnInit/OnTick/OnTimer/OnDeinit)
includes/
  Constants.mqh             enums: MA methods, signal types, notification levels
  Config.mqh                all input parameters, grouped (indicator/notify/webhook/telemetry)
  Utils.mqh                 CUtils helpers (timeframe<->string, method/price parsing)
  Logger.mqh                CLogger levels and file logging
  ErrorHandler.mqh          error mapping and reporting
  EMA.mqh                   CMAEngine: MA handle lifecycle and cross evaluation
  ChartFollow.mqh           scans chart for visual MA indicators and reads their settings
  Persistence.mqh           CPersistence: common-files JSON state
  JsonBuilder.mqh           payload construction for webhook + telemetry
  NotificationManager.mqh   dispatches to push/email/webhook
  EmailManager.mqh          SMTP email alerting
  WebhookManager.mqh        authenticated POST with retry/backoff
  Telemetry.mqh             CTelemetry: writes ema_ticker_*.json snapshots
```

## Running it

Open `EMAAlertEA.mq5` in MetaEditor (inside the MT5 terminal's `MQL5/Experts` tree), compile,
attach to a chart, and either set the inputs or place MA indicators on the chart for it to follow.
Copy `includes/` alongside the `.mq5` file. `WebRequest` must be allowed for the webhook target
under Tools > Options > Expert Advisors.

## Notes

- `includes/Config.mqh` carries a real API key as the default value of the `InpAPIKey` input.
  Scrub it before publishing and rotate the key.
- Several `*.bak`, `*.backup-*` and `*.root-fix-*` files in `includes/` are dated local backups of
  `Config.mqh`, `EMA.mqh` and `Utils.mqh`. They are not part of the build; drop them when publishing.
- `Copyright 2026, Manus Agent` in the property headers is a leftover generator marker, not the
  intended authorship line.
- Companions (do not confuse with this folder): `ema-alert-server` receives these webhooks; the
  `watchdog/` Python service is a backup detector that only fills gaps when this EA goes silent.

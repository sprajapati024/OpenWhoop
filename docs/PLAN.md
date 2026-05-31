# OpenWhoop — Phased Implementation Plan

**Version:** 1.0
**Date:** 2026-05-31
**Project:** https://github.com/sprajapati024/OpenWhoop

---

## Phase 0 — Spike *(Weekend 1)*
**Goal:** Verify end-to-end before committing to full build.

### 0.1 — VPS Server Setup
- [ ] Spin up Hetzner VPS (or use existing) with Ubuntu 22.04
- [ ] Install Docker + Docker Compose
- [ ] Clone OpenWhoop repo to VPS
- [ ] Copy `server/.env.example` → `server/.env`, fill in `WHOOP_API_KEY`, `WHOOP_DB_PASSWORD`, `DATA_ROOT`
- [ ] `docker compose up -d --build` for `whoop-db` + `whoop-ingest`
- [ ] Verify `curl localhost:8770/healthz` returns 200
- [ ] Set up domain/subdomain (e.g. `whoop.yourdomain.com`) with nginx/Caddy reverse proxy + HTTPS
- [ ] Note: VPS endpoint URL for iOS app config

### 0.2 — Mac Uploader Test
- [ ] Install Python deps: `pip install -r server/ingest/requirements.txt`
- [ ] Create test capture or use existing `re/` capture data
- [ ] Run `python server/client/uploader.py <test_capture.jsonl> --url <vps_url> --token <WHOOP_API_KEY>`
- [ ] Verify data appears in TimescaleDB (`GET /v1/streams/hr`)
- [ ] Verify dashboard at VPS URL shows data

### 0.3 — iOS BLE Test *(requires physical WHOOP 4.0)*
- [ ] Open `ios/OpenWhoop.xcodeproj` in Xcode
- [ ] Copy `ios/OpenWhoop/Config/Secrets.example.xcconfig` → `Secrets.xcconfig`
- [ ] Fill in `WHOOP_BASE_URL`, `WHOOP_API_KEY`, `WHOOP_DEVICE_ID`
- [ ] Build and run on iPhone with WHOOP 4.0 nearby
- [ ] Confirm bonding succeeds (device connects, HR stream begins)
- [ ] Confirm decoded HR appears in local WhoopStore (GRDB)
- [ ] Confirm raw frames sync to VPS

**Exit criterion:** Live HR frames visible in TimescaleDB from physical WHOOP 4.0 band.

---

## Phase 1 — Server *(2–3 weeks)*

### 1.1 — Ingest API
- [ ] `POST /v1/ingest` — raw frame ingestion with Bearer auth, batch deduplication, clock correlation
- [ ] Idempotency via `batch_id` — re-sending same batch is a no-op
- [ ] Store raw as `.zst` in `DATA_ROOT/whoop/raw/<device_id>/<date>/<batch>.zst`
- [ ] Insert decoded rows (HR, R-R, events, battery) into TimescaleDB hypertables
- [ ] Validate against `whoop_protocol.json` schema before storing

### 1.2 — TimescaleDB Schema
- [ ] `devices` — device registry (device_id, mac, name, first_seen, last_seen)
- [ ] `hr_samples` — `(time, device_id, bpm)` as hypertable
- [ ] `rr_intervals` — `(time, device_id, rr_ms)` as hypertable
- [ ] `events` — `(time, device_id, kind, payload jsonb)` as hypertable
- [ ] `battery` — `(time, device_id, soc, mv)` as hypertable
- [ ] Retention policies: raw = forever, decoded = 1 year hot, 5 year cold
- [ ] Continuous aggregates: 1-min, 5-min, 1-hour rollups for chart performance

### 1.3 — Read API
- [ ] `GET /v1/devices` — list registered devices
- [ ] `GET /v1/streams/{hr|rr|events|battery}` — time-range query with limit, cursor pagination
- [ ] `GET /v1/batches` — raw batch index
- [ ] `GET /v1/batches/{batch_id}/frames` — re-parse raw archive via whoop-protocol
- [ ] All read endpoints: no auth (VPS LAN-only or behind authed tunnel)

### 1.4 — Analysis Engine
- [ ] `hrv.py` — RMSSD, SDNN, pNN50 from R-R interval series
- [ ] `recovery.py` — recovery score: HRV baseline + resting HR + sleep efficiency → 0–100%
- [ ] `strain.py` — strain accumulation: time in HR zones × zone weights → 0–21
- [ ] `sleep.py` — sleep stage classification from R-R variability (wake/light/REM/deep)
- [ ] `baselines.py` — rolling HRV baseline (7/30/90 day) for trend detection
- [ ] Nightly batch job: run analysis on previous night's data → store scores in TimescaleDB

### 1.5 — Web Dashboard
- [ ] Device selector + date range picker
- [ ] HR chart (intraday + historical)
- [ ] Recovery / strain / sleep summary cards
- [ ] Events log (wrist on/off, charging, etc.)
- [ ] Raw frame browser (hex inspector with field decode via whoop-protocol)
- [ ] Battery timeline
- [ ] Export: CSV/JSON download for all streams

### 1.6 — Alerting
- [ ] High HR during sleep threshold
- [ ] Low recovery score threshold
- [ ] Battery below 20%
- [ ] Delivery: email (sendgrid/brevo) or Telegram bot notification

**Exit criterion:** Web dashboard shows meaningful analytics (recovery trend, sleep stages, strain) from real device data.

---

## Phase 2 — iOS App *(3–4 weeks)*

### 2.1 — Core BLE Pipeline
- [ ] `BLEManager.swift` — CBCentralManager lifecycle, state restoration, bonding trick
- [ ] `Reassembler.swift` — BLE fragment reassembly (1917B packets span multiple notifications)
- [ ] `FrameRouter.swift` — route frames to correct decoder (HR, IMU, events, etc.)
- [ ] `WhoopProtocol` integration — load `whoop_protocol.json`, run interpreter
- [ ] `WhoopStore` integration — persist decoded streams to GRDB
- [ ] Background mode: `bluetooth-central`, state restoration via `CBCentralManagerOptionRestoreIdentifierKey`

### 2.2 — Live View
- [ ] Current HR (large number, animated pulse)
- [ ] R-R interval display (last 4 values)
- [ ] Battery indicator + charging state
- [ ] Wrist-on detection
- [ ] Live IMU sparkline (accelerometer magnitude)
- [ ] Connection status indicator

### 2.3 — Historical Charts
- [ ] Recovery trend: 7 / 30 / 90 day line chart
- [ ] Strain history: daily strain bar chart
- [ ] HRV trend: RMSSD line chart with baseline band
- [ ] Sleep architecture: stage breakdown bar (deep/REM/light/wake)
- [ ] Date range picker
- [ ] All charts via Swift Charts

### 2.4 — Device Management
- [ ] Pair new WHOOP 4.0 (scan → bond → name)
- [ ] Battery history
- [ ] Firmware version display
- [ ] Sync status (last sync time, pending batch count)
- [ ] Manual sync trigger
- [ ] Stored-history backfill status

### 2.5 — Offline + Sync
- [ ] All decoded data readable without server
- [ ] Background sync when network available
- [ ] Sync nudge UI when behind
- [ ] Full history restore from VPS on fresh install

### 2.6 — HealthKit Integration
- [ ] Export: HRV samples (custom HK category) → Apple Health
- [ ] Export: sleep analysis (HK category) → Apple Health
- [ ] Import: none (WHOOP HR already on wrist, no point pulling from Health)

**Exit criterion:** App is usable as a daily driver without WHOOP app open.

---

## Phase 3 — FitCoach Integration *(2–3 weeks)*

### 3.1 — Shared Data Layer
- [ ] `FitCoachCore` package: add `WhoopMetrics` model (recovery, strain, hrv, sleep_quality)
- [ ] `WhoopMetricsRepository` reads from WhoopStore (GRDB) or VPS API
- [ ] `TodayReadiness` view composable for embedding in FitCoach

### 3.2 — Readiness UI in FitCoach
- [ ] "Today's Readiness" card on FitCoach home screen (green/yellow/red)
- [ ] Tap to expand: recovery %, HRV vs baseline, last night's sleep quality
- [ ] Workout recommendation: "Low recovery — consider active recovery or rest day"

### 3.3 — Workout HR Correlation
- [ ] FitCoach `WorkoutSession` → link WHOOP HR data for that time window
- [ ] Workout HR chart (overlay on FitCoach exercise detail)
- [ ] Post-workout strain summary (was the workout hard enough? HR zone distribution)

### 3.4 — Progressive Overload × Recovery
- [ ] If recovery > 80%: suggest increasing weight by 2.5 kg
- [ ] If recovery < 50%: suggest deload or active recovery
- [ ] HRV trend → early warning for overtraining

**Exit criterion:** FitCoach shows WHOOP readiness before every workout and correlates HR data with session history.

---

## Ongoing — Algorithm Improvement

After Phase 2, the full raw archive on TimescaleDB lets you re-run improved analysis on all historical data:

- [ ] Tune recovery weights based on how you actually felt
- [ ] Implement a personal HR zone model from lactate threshold data
- [ ] Build a "training readiness" composite score that combines FitCoach + WHOOP data
- [ ] Export everything to CSV/JSON for personal research projects
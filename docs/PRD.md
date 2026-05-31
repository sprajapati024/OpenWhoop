# OpenWhoop — Product Requirements Document

**Version:** 1.0
**Date:** 2026-05-31
**Author:** Shirin (sprajapati024)
**Status:** Planning

---

## 1. Problem Statement

WHOOP charges **$30/month** for a subscription that only exists because WHOOP, Inc. runs a cloud service between your $300 wristband and their app. The wristband itself has **no subscription lock** — it advertises and accepts connections over standard Bluetooth LE. All the analytics (recovery score, strain, sleep staging, HRV) are computed server-side from raw sensor data that the band transmits freely.

This project exists to eliminate the subscription rent while owning the full stack: band → phone → VPS → you.

---

## 2. Product Vision

**OpenWhoop** is a self-hosted WHOOP 4.0 analytics platform that replaces the WHOOP subscription entirely. It reads raw biometric data directly from your WHOOP 4.0 over Bluetooth LE, processes it on your own VPS, and presents it in a polished iOS app — or in a browser via a web dashboard.

The core promise: *your own data, your own infrastructure, your own algorithms.*

---

## 3. What We're Building

### Core product

**WHOOP 4.0 band → iPhone (BLE) → VPS (analysis) → iOS app + web dashboard**

### Data we read from the band

| Data | Source | Quality |
|---|---|---|
| Heart rate (continuous) | BLE standard + custom char | ~52 Hz |
| R-R intervals | BLE standard + custom char | millisecond precision |
| Accelerometer | Custom BLE char | 100 samples/packet, ~52 Hz |
| Gyroscope | Custom BLE char | 100 samples/packet, ~52 Hz |
| Raw PPG waveform | Custom BLE char (1921B packet) | ~4-channel optical, pulsatile |
| Battery %, voltage, charging | Custom BLE char | real-time |
| Wrist on/off events | Custom BLE char | real-time |

### What we compute ourselves

| Metric | Method |
|---|---|
| **Recovery score** | HRV (RMSSD) + resting HR + sleep efficiency, normalized to 0–100% |
| **Strain score** | Peak HR × time in each HR zone, accumulate to 0–21 scale |
| **Sleep stages** | R-R variability → wake/light/REM/deep via established cardiac sleep staging |
| **HRV trend** | Rolling RMSSD baseline vs. nightly measurement |
| **Daily summary** | Sleep quality, strain balance, recovery, readiness |
| **Recovery trend chart** | 7/30/90 day HRV + recovery trajectory |
| **Strain chart** | Per-workout and daily accumulated strain |
| **Readiness score** | Recovery + HRV baseline + sleep quality → workout recommendation |

### What WHOOP cloud cannot give you (and why it matters)

- **FitCoach integration** — workout sessions (FitCoach) × WHOOP HR data × recovery score → intelligent training recommendations. WHOOP doesn't know your training program.
- **Custom HR zones** — WHOOP uses generic age-based zones. You'll use your actual max HR from test data.
- **Algorithm ownership** — WHOOP can change their scoring at any time. You can lock yours and improve it based on your own research.
- **Cross-device history** — Phone + Apple Watch + FitCoach all feeding the same database.

---

## 4. Platform Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        WHOOP 4.0 BAND                        │
│   (advertises over BLE — no pairing, no auth required)        │
└───────────────────────────┬─────────────────────────────────┘
                            │  Bluetooth LE (direct, local)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      iOS APP (SwiftUI)                       │
│  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐ │
│  │  BLEManager  │→│ WhoopProtocol │→│   WhoopStore      │ │
│  │ (CBCentral)  │  │  (decoder)    │  │   (GRDB/SQLite)  │ │
│  └──────────────┘  └───────────────┘  └──────────────────┘ │
│         │                                       │           │
│         │         ┌───────────────┐             │           │
│         └────────►│   Uploader    │─────────────┘           │
│                   │  (raw frames) │                         │
│                   └───────┬───────┘                         │
└───────────────────────────┼─────────────────────────────────┘
                            │  HTTPS (background sync)
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   HETZNER VPS (self-hosted)                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              FastAPI (POST /v1/ingest)               │   │
│  │               Bearer token auth                      │   │
│  └─────────────────────────┬───────────────────────────┘   │
│                            │                               │
│  ┌─────────────────────────▼───────────────────────────┐   │
│  │                  TimescaleDB                         │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌────────────┐ │   │
│  │  │ raw_archive │  │ decoded_streams│  │ analysis  │ │   │
│  │  │  (.zst)     │  │ (hr/rr/events)│  │ (recovery)│ │   │
│  │  └─────────────┘  └──────────────┘  └────────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │           Web Dashboard (FastAPI static)              │   │
│  │   Charts / export / device management                 │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │             Analysis Engine (Python)                  │   │
│  │  recovery.py | strain.py | sleep.py | hrv.py         │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Phases

### Phase 0 — Spike (1 weekend)
> *Can we actually connect and get data?*

- [ ] Deploy OpenWhoop server on Hetzner VPS (docker compose)
- [ ] Verify TimescaleDB + ingest endpoint work
- [ ] Test with Mac uploader against live VPS endpoint
- [ ] Confirm WHOOP 4.0 bonding + streaming works on iOS

**Exit criterion:** Raw HR frames visible in TimescaleDB from live band.

### Phase 1 — Server (2–3 weeks)
> *Robust self-hosted backend*

- [ ] FastAPI server with proper auth, rate limiting, validation
- [ ] TimescaleDB schema: raw_archive, hr_samples, rr_intervals, events, battery
- [ ] Analysis engine: recovery, strain, sleep staging, HRV baseline
- [ ] Web dashboard: device status, HR/HRV charts, events, raw frame browser
- [ ] Data export: CSV/JSON for all streams
- [ ] Alerting: high HR during sleep, low recovery, battery low

**Exit criterion:** Web dashboard shows meaningful analytics from real device data.

### Phase 2 — iOS App (3–4 weeks)
> *Polish the phone UI*

- [ ] SwiftUI app: live HR view, current recovery/strain scores
- [ ] Historical charts: 7/30/90 day HRV, recovery, strain, sleep
- [ ] Device management: pairing, battery, sync status, firmware version
- [ ] Background BLE state restoration (survives app suspension)
- [ ] Offline-first: all decoded data readable without server
- [ ] HealthKit integration: export HRV + sleep to Apple Health

**Exit criterion:** App is usable as a daily driver without WHOOP app open.

### Phase 3 — FitCoach Integration (2–3 weeks)
> *Connect to workout data*

- [ ] Shared data layer: OpenWhoop metrics in FitCoach's SwiftData models
- [ ] Recovery-informed workout recommendations in FitCoach
- [ ] Workout HR overlay on FitCoach progression charts
- [ ] "Today's readiness" banner on FitCoach home screen

**Exit criterion:** FitCoach shows WHOOP readiness before every workout.

---

## 6. Out of Scope (v1)

- WHOOP 3.0 or other generations (different BLE protocol)
- Android app (BLE bonding is harder on Android; iOS first)
- Multi-user / shared family accounts
- SpO₂ values (computed by WHOOP cloud, not on the BLE wire)
- Skin temperature (same — computed cloud-side, not streamed)
- On-device analysis in v1 (runs on VPS)

---

## 7. Design Language

- **Aesthetic:** Dark-first, health-forward. WHOOP's visual language but owned.
- **Color palette:**
  - Background: `#0A0A0F` (near black)
  - Surface: `#16161F` (card backgrounds)
  - Primary: `#FF3366` (WHOOP red — use sparingly for strain/accent)
  - Recovery: `#00E5A0` (green spectrum for recovery)
  - HRV: `#7B61FF` (purple for HRV)
  - Text: `#FFFFFF` / `#8888AA` (muted secondary)
- **Typography:** SF Pro (system) — clean, fast, no web fonts needed
- **Charts:** Swift Charts (native) — consistent with iOS ecosystem
- **Motion:** Minimal — subtle spring animations on data load, no decorative animation

---

## 8. Dependencies Replaced

| Paid service | Replaced by |
|---|---|
| WHOOP subscription ($30/month) | OpenWhoop (zero marginal cost) |
| WHOOP cloud storage | Own Hetzner VPS TimescaleDB |
| WHOOP analysis algorithms | Self-owned Python in `server/ingest/app/analysis/` |

---

## 9. Key Files

| File | Purpose |
|---|---|
| `protocol/whoop_protocol.json` | Canonical BLE decode schema |
| `Packages/WhoopProtocol/` | Swift decoder package |
| `Packages/WhoopStore/` | GRDB local storage package |
| `ios/OpenWhoop/BLE/BLEManager.swift` | CoreBluetooth bonding + streaming |
| `server/ingest/app/ingest.py` | FastAPI ingest + read API |
| `server/ingest/app/analysis/` | Python analysis algorithms |
| `FINDINGS.md` | Living WHOOP 4.0 BLE protocol reference |
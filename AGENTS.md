# OpenWhoop — Agent Instructions

## What this project is

**OpenWhoop** is a self-hosted WHOOP 4.0 analytics platform. It connects directly to your WHOOP 4.0 band over Bluetooth LE, reads raw biometric data, and processes it on your own infrastructure — no WHOOP subscription required.

```
WHOOP 4.0 band (your wrist)
       │
       │  BLE — direct, no internet, no WHOOP cloud
       ▼
  iPhone (SwiftUI + CoreBluetooth)
  ├── WhoopProtocol  (schema-driven decoder)
  ├── WhoopStore     (GRDB/SQLite — decoded streams)
  └── Uploader      (push raw to VPS)
              │
              ▼
         YOUR VPS (FastAPI + TimescaleDB + Analysis Engine)
```

## Key principles

- **Schema is the source of truth** — `protocol/whoop_protocol.json` is read by both the iOS app and the Python server. Never decode a field that isn't in the schema.
- **Raw is archived server-side; decoded is local-first** — iPhone holds decoded HR/RR/events/battery streams durably in GRDB. Raw frames go to the VPS as a lossless backup and for re-analysis.
- **Analysis is self-owned** — the recovery/strain/sleep algorithms live in `server/ingest/app/analysis/`. We own this code, not WHOOP's cloud.
- **No WHOOP cloud dependency** — the band, your phone, and your VPS is the entire stack.

## Repository structure

```
OpenWhoop/
├── protocol/              # Canonical decode schema (JSON) — source of truth for all decoders
├── Packages/
│   ├── WhoopProtocol/     # Swift package — framing + interpreter (mirrors Python)
│   └── WhoopStore/        # Swift package — GRDB local store
├── ios/
│   ├── OpenWhoop/         # iOS app (SwiftUI + CoreBluetooth)
│   ├── Packages/         # Local SwiftPM package refs (WhoopProtocol, WhoopStore)
│   └── project.yml       # XcodeGen config
├── server/
│   ├── ingest/            # FastAPI + TimescaleDB ingest pipeline
│   │   ├── app/
│   │   │   ├── analysis/  # Python analysis modules (recovery, strain, sleep, HRV, etc.)
│   │   │   ├── ingest.py  # Main FastAPI app
│   │   │   └── store.py   # TimescaleDB operations
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   ├── client/            # Mac uploader script (for testing/replay)
│   ├── docker-compose.yml
│   └── .env.example
├── re/                    # Reverse-engineering scripts + labeled motion capture data
├── FINDINGS.md            # Living reverse-engineering reference doc
├── DISCLAIMER.md          # Legal disclaimer
└── docs/                  # Design specs, implementation plans
```

## iOS app conventions

- **SwiftUI + CoreBluetooth** — `BLEManager` in `ios/OpenWhoop/BLE/` owns the CBCentralManager lifecycle and bonding.
- **Bonding trick** — issue one confirmed write (`writeValue(..., type: .withResponse)`) to command char `61080002` immediately after connect to trigger silent just-works BLE bonding.
- **Swift packages** — `WhoopProtocol` and `WhoopStore` are in `Packages/` and referenced as local SwiftPM packages in `project.yml`.
- **XcodeGen** — regenerate with `xcodegen generate` from `ios/`.
- **Background BLE** — requires `bluetooth-central` background mode. State restoration keeps collection alive across app suspension.
- **Info.plist keys**: `NSBluetoothAlwaysUsageDescription`, `UIBackgroundModes: bluetooth-central`.
- **Build config**: `Secrets.xcconfig` (gitignored) holds `WHOOP_BASE_URL`, `WHOOP_API_KEY`, `WHOOP_DEVICE_ID`.

## Server conventions

- **Python 3.11+**, FastAPI, TimescaleDB, PostgreSQL driver.
- **Docker Compose** for local dev and production VPS deployment.
- **Ingest endpoint** — `POST /v1/ingest` (Bearer token auth), receives raw hex frames + metadata.
- **Read API** — `GET /v1/streams/{hr|rr|events|battery}` (LAN-only by default).
- **Analysis modules** in `server/ingest/app/analysis/` are the self-owned algorithms.
- **Environment**: see `.env.example`. Key vars: `WHOOP_API_KEY`, `WHOOP_DB_*`, `WHOOP_INGEST_PORT`, `DATA_ROOT`.

## Development workflow

1. **Server first** — always get the backend running before touching the iOS app.
2. **Test with Mac uploader** (`server/client/uploader.py`) before Bluetooth testing on device.
3. **Schema changes** — any new BLE packet type goes into `protocol/whoop_protocol.json` first, then port to Swift `interpreter.swift`.
4. **Analysis changes** — if you change recovery/strain/sleep algorithms, re-run against historical raw archives to validate continuity.

## Important references

- `FINDINGS.md` — authoritative WHOOP 4.0 BLE protocol reference. Update this when you discover new fields.
- `docs/specs/2026-05-23-openwhoop-ios-app-design.md` — iOS app design spec.
- `server/ingest/app/analysis/` — Python analysis algorithms ported from bWanShiTong's `openwhoop-algos`.

## Project context

Owner: Shirin (sprajapati024)
Stack: Swift/SwiftUI (iOS), Python/FastAPI (server), TimescaleDB, CoreBluetooth, SwiftPM, XcodeGen
Hosting: Self-hosted on Hetzner VPS
Subscriptions replaced: WHOOP $30/month
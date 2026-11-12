# LifeRecorder

**Local-first personal evidence infrastructure for iPhone.**

Status: experimental · M0 complete, M1 implemented and fixture-tested, M2 audio in development  
Source: private · Last verified: November 2026

## Overview

LifeRecorder records personal evidence — health samples, motion activity, location visits, system context, and Shortcuts events — into a durable local database so that a day can eventually be reconstructed and understood.

The core principle is simple: **evidence is factual, and inference never overwrites evidence.** If LifeRecorder did not observe something, it does not claim it. Gaps are stored with a reason instead of being filled in. Derived data is appended as a separate layer and linked back to the raw events.

M0 records evidence only. It has no account system, no backend, no analytics, and no cloud sync. The only exit path for data is a user-initiated NDJSON export through the iOS share sheet.

## At a glance

| | |
| --- | --- |
| Platform | iOS 17+, Swift 6, SwiftUI |
| Storage | SQLite (WAL), append-only event table, actor-backed store |
| Sources | HealthKit, Core Motion, Core Location, system events, Shortcuts, Google Health / Fitbit Air |
| Privacy | No networking APIs in app sources; local export only |
| Status | Experimental; M0 and M1 are test-covered, M2 audio is a development branch |
| Source | Private |

## Architecture

```text
HealthKit ─────┐
Core Motion ───┤
Core Location ─┼──► collectors ──► canonical LifeEvent ──► actor-backed SQLite
System events ─┤                      │  (versioned, typed,      (WAL, append-only,
Shortcuts ─────┤                      │   first-class provenance)  idempotent writes)
Google Health ─┘                      │
                                      ├──► timeline / diagnostics UI
                                      ├──► NDJSON export (share sheet)
                                      │
Audio (M2 dev) ──► segment + gap records ──► charging-time classification
                                      └──► on-device transcription (derived, append-only)
```

The future feature and life-state layers are deliberately not implemented. M0 is the evidence layer; adding a collector means implementing a `LifeCollector` and a pure mapping function, without changing persistence.

## Components

| Component | Responsibility |
| --- | --- |
| `LifeEvent` | Canonical, versioned, `Sendable` + `Codable` event with strongly typed payload and provenance |
| Event store | Actor-backed SQLite with WAL, migrations, prepared statements, indexes, and checkpoints |
| Collectors | HealthKit, Motion, Location, System, Shortcuts, Google Health; each independently optional and fallible |
| Coordinator | Isolates per-collector failure; one denial or error never stops the others |
| NDJSON exporter | Deterministic UTC export with every canonical key present |
| Diagnostics | Schema version, tables, indexes, counts, time range, checkpoints, database sizes |
| Audio pipeline (M2 dev) | Rolling AAC chunks, SHA-256 per segment, typed gap records, classification, transcription |

## Engineering challenges

### 1. Evidence and inference must be different layers

Raw observations and derived labels cannot share a mutable row. LifeRecorder stores raw events in an append-only table and appends derived observations separately, linked to their source. Re-processing with a new model version appends new rows instead of rewriting history.

Every event carries provenance: HealthKit sample UUID and source revision, motion origin (`live` or `historicalCatchUp`), location mechanism and authorization, system trigger, intent name, and schema version. Unknown payload kinds written by a future schema decode to `.unsupported` / `.unknown(raw)` with fields preserved, so old readers do not crash.

### 2. Idempotency is a storage property

Each collector has a deterministic dedupe key (`healthkit:<sampleUUID>`, `motion:<ms>:<flags>`, `visit:…`, `sloc:…`). A partial unique index allows at most one row per collector identity. Re-running a sync, a backfill, or a catch-up inserts nothing new.

Collector checkpoints — HealthKit anchors, motion recovery timestamps, Google Health per-datatype coverage — live in the database, never in `UserDefaults`. Checkpoints advance only after durable inserts, so a crash cannot skip evidence.

### 3. iOS background behavior is not continuous

Live Core Motion updates pause while the app is suspended; the catch-up query recovers the interval and marks those events `historicalCatchUp`. Background visit and significant-change delivery requires *Always* location authorization; with *When In Use*, events may only arrive while the app is running. HealthKit background delivery requires the HealthKit capability plus the portal's Background Delivery option, and where it is unavailable the collector records a non-fatal error and still imports via anchors on the next run.

The app deliberately declares no background modes: visits and significant changes are event-based services, and nothing in M0 needs continuous background execution.

### 4. Local-first is enforced, not just stated

`Scripts/check-privacy.sh` fails the build if `URLSession`, `Network`, or a telemetry SDK name appears in app sources. The app has no account, no remote logging, and no automatic upload. Production logs record counts, kinds, and timestamps — never health values, coordinates, or transcript content.

The export path is explicit: the person chooses *Export Events*, the app writes a chronological NDJSON file, and the standard iOS share sheet decides where it goes.

### 5. Migrating from the Fitbit Web API to Google Health

M1 targets the current Google Health API (v4), not the deprecated Fitbit Web API. The client implements OAuth 2.0 authorization code + PKCE (S256) without a client secret, system-browser authorization, Keychain token storage, single-flight refresh, `invalid_grant` handling, and best-effort revocation on disconnect.

Ingestion covers 20 datatypes, including heart rate at up to 1-second storage resolution, HRV, sleep sessions and stages, respiratory rate, oxygen saturation, steps, distance, energy, exercise, and VO2 max. The test suite covers request construction, pagination, 429 `Retry-After`, 5xx backoff, 401 refresh, malformed-record isolation, all 20 mappings from official-schema fixtures, and partial-failure isolation where one failing datatype does not discard the others.

Live Google Health acceptance was **not** performed; the M1 evidence is fixture- and simulation-based and is marked as such.

### 6. Audio needs gaps, not silence

The M2 audio branch records rolling 10-minute AAC chunks (16 kHz mono) only while the feature is enabled. Every closed chunk becomes a segment row with path, size, SHA-256, duration, route, and start/end timestamps. Any interval longer than two seconds not covered by a segment becomes an explicit gap with a typed reason: user pause, interruption, route change, disk pressure, engine failure, media-services reset, or app termination.

The system microphone indicator stays visible whenever recording is active; there is no covert mode. A disk guardrail pauses below 2 GB free. On-device transcription and charging-time sound classification are derived data, appended separately from the audio evidence. Retention controls are part of the M2 design and are not complete.

## Verified results

| Area | Result | Scope |
| --- | --- | --- |
| M0 test suite | 103 tests, 0 failures, 0 skips | iOS 27 simulator |
| M1 test suite | 236 tests, 0 failures | iOS 27 simulator, fixtures |
| M0 acceptance tour | 2 UI tests, 0 failures; onboarding, permissions, recording, timeline, export | Simulator |
| Storage performance | 25,000 inserts ~0.34 s; latest 500 ~3.6 ms; one-day window ~9 ms | Debug build, simulator |
| Heart-rate benchmark | 100,000 events inserted in ~2.91 s; 24-hour query ~1.16 s; database ~128 MB | Diagnostic, simulator |
| Google Health sync | First fixture sync 30 events; second sync 0 duplicates; checkpoints advance only after durable inserts | Fixtures, no live API |
| NDJSON export | All canonical keys present, chronological, UTC dates, round-trip verified | QA harness |
| Privacy check | Build fails on networking or telemetry APIs in app sources | Scripted gate |

## Implemented, not verified

- **Live Google Health / Fitbit Air ingestion.** OAuth, API client, mappings, sync, and reconciliation are implemented and fixture-tested; a live account run was not performed.
- **Real-device, all-day recording.** Battery cost, locked-screen longevity, interruptions, thermal behavior, and real-world sensor delivery were not soak-tested.
- **Real-world transcription accuracy.** The audio branch uses on-device speech transcription; quality across accents, noise, and languages is not measured.
- **Audio retention controls.** Designed but not complete.
- **HealthKit background delivery entitlement.** Not provisioned in the headless build; the collector degrades gracefully when unavailable.

## Explicit non-claims

- LifeRecorder does not infer stress, mood, eating, smoking, or semantic life state. Those layers are not built.
- It does not upload data, sync to a cloud, or require an account.
- It does not claim continuous background sensing beyond what iOS actually delivers.
- It does not present generated tones or fixture data as real recordings or speech-recognition results.

## Technology stack

Swift 6 · SwiftUI · SQLite3 · HealthKit · Core Motion · Core Location · AVFoundation · SoundAnalysis · Apple Speech / Whisper (audio branch) · OAuth 2.0 PKCE · XcodeGen · XCTest

## Verification notes

Test counts and performance figures come from local runs on the simulator and, where stated, a physical iPhone 17 Pro Max. Raw databases, audio, health values, and device identifiers are private and are not included in this document. The M1 live-API gap and the M2 audio limitations are intentional parts of the record, not footnotes.

# Yağız Oğuz

**Independent software developer — native Apple platforms, AI agents, and device-level interaction.** · Istanbul, Türkiye

I build experimental software where agents, accessibility APIs, and real devices meet. Most of my work is in Swift: iPhone automation, semantic remote interfaces, and local-first personal data systems. I care about the difference between a plausible result and an observed one, and I document which is which.

## Selected work

### PhoneAgent — iPhone automation from observed interfaces

[Native iOS agent stack →](projects/phoneagent.md)

A native iOS agent stack built around screen perception, Accessibility semantics, and HID input. It observes the screen with Vision and ScreenCaptureKit, grounds actions against recognized elements, and can drive real touches through a hostless RSD/HID path running on the phone itself.

- Device-verified on iPhone 17 Pro Max / iOS 27: full-display capture and OCR with pixel-checked coordinates; tap, swipe, and Home over CoreDevice HID; a closed observe → resolve → tap → verify loop with no Mac at runtime (September 2026).
- On-device iOS 27 pairing verified end-to-end in November 2026, including promotion of a staged pairing record only after a harmless HID tap was observed.
- **2,645 tests, 2 skipped, 0 failures** in the main simulator suite (November 2026).
- Private beta. General autonomous AX-to-HID behavior is still experimental. CoreDevice text entry is withheld after a device-verified failure. Pairing coverage is one device.

### MorphDesk — semantic remote desktop

[Swift semantic remote interaction →](projects/morphdesk.md)

A Swift system that turns a real macOS Accessibility tree into a normalized, Codable model, generates a deterministic native iPhone interface, and executes validated AX operations over an authenticated TLS connection. Standard mode has no AI or cloud dependency.

- Physical iPhone 18 Pro Max: paired over LAN, rendered and edited a real TextEdit document, applied the change, and verified it by independent AX read-back; disconnect and reconnect passed.
- Real effects independently checked in TextEdit (save), Finder (navigation and selection), Safari (form edit and link navigation), and Calculator (sidebar) — **15/15 real-AX tests, 0 failures**.
- Optional Adaptive mode accepts a user-configured OpenAI-compatible endpoint. The integration is implemented; real-provider layout generation has not yet been demonstrated.
- Working prototype, bounded semantic MVP. Shared-key TLS-PSK trust, no per-device identity; Tailscale routing is untested.

### LifeRecorder — local-first personal evidence

[Local-first iOS telemetry →](projects/liferecorder.md)

An iOS recorder that captures HealthKit samples, motion activity, location visits, system context, and Shortcuts events into a canonical, append-only SQLite store with first-class provenance. No account, backend, analytics, or cloud path.

- **103 tests** for M0 and **236 tests** for the Google Health / Fitbit Air (API v4) ingestion layer; 0 failures.
- Measured on the simulator: 25,000 events inserted in ~0.34 s; a 500-event latest query in ~3.6 ms; a one-day window in ~9 ms.
- M2 audio branch: continuous AAC chunks with explicit gap records, charging-time classification, and on-device Turkish transcription. Development-only and off by default.
- Experimental. Live Google Health acceptance is not complete; real-world recording and transcription quality remain separate validation work.

## Other engineering

- **YEAR** — local media search using Whisper transcription and SQLite FTS5.
- **CLOSET** — AI-assisted styling and computer vision experiments.
- **TapWar** — interactive web application and backend services.
- **Fitbit BLE research** — Bluetooth Low Energy protocol and device analysis.
- **FRC Team 6232, Florya Bisons** — software and electronics development.
- **Radio and hardware** — 1420 MHz hydrogen-line observations with RTL-SDR, amateur radio and ISS SSTV/telemetry, custom antennas, RF experiments, and biomedical EMG prototypes.

## Technical focus

| Area | Tools and technologies |
| --- | --- |
| Languages | Swift, Python, C# |
| Apple platforms | SwiftUI, Accessibility (AX), ScreenCaptureKit, Vision, Core ML, HealthKit, Core Motion, CoreBluetooth |
| Data | SQLite, FTS5, FastAPI, REST |
| AI | LLM agents, multimodal planning, Whisper, speech processing, computer vision |
| Systems and hardware | HID, RSD, BLE, RTL-SDR, RF, embedded electronics, robotics |

## How I work

- **Separate what is implemented, what is device-verified, and what is still research.** Test counts and acceptance runs are reported with their scope and date.
- **Prefer local-first and explicit boundaries.** Network access, model calls, and device control are opt-in, bounded, and fail closed.
- **Measure the platform instead of assuming it.** HID surfaces, AX capabilities, background modes, and protocol behavior are treated as things to verify.

## Source and contact

The source for these projects is private. The dossiers above document architecture, verification scope, and limitations; I can walk through implementation details on request.

I am interested in conversations about agent execution semantics, accessibility systems, native Apple engineering, and local-first data. The best way to reach me is through [GitHub](https://github.com/yagizoguz).

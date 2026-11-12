# PhoneAgent

**Native iPhone automation across perception, accessibility, and HID input.**

Status: private beta and active research · iOS 27 · Swift 6 · Source: private · Last verified: November 2026

## Overview

PhoneAgent is an experimental agent stack that runs on the iPhone. It is built around two questions:

1. **What is actually on the screen?** — full-display capture, on-device OCR, and a single canonical coordinate space.
2. **What can the device actually execute?** — a typed input engine, capability gating, and an evidence model that distinguishes a transport acknowledgement from an observed effect.

The project grew through three milestones: perception (M1), a real input engine (M2), and a hostless closed loop that observes, resolves a target, taps, re-observes, and verifies the result on the phone itself (M3). A separate AXHID lab explores model-assisted Accessibility-to-HID execution under stricter safety gates.

This is not a finished general-purpose iPhone agent. The sections below separate device-verified behavior from implemented-but-unverified work and from open research.

## At a glance

| | |
| --- | --- |
| Platform | iOS 27, Swift 6.4, SwiftUI |
| Perception | ScreenCaptureKit full-display capture, Vision OCR |
| Interface semantics | iOS Accessibility (AX) hierarchy, OCR text, normalized element geometry |
| Input | CoreDevice HID over RSD; hostless reflected RSD tunnel; XCUITest and mock backends for development |
| Verification | Local simulator suites and physical-device acceptance runs |
| Distribution | Private beta / development installs; not App Store-distributable in the current form |
| Source | Private |

## Architecture

```text
                         ┌──────────────────────────────┐
                         │          Perception          │
                         │  ScreenCaptureKit → Vision   │
                         │  ScreenObservation + rects   │
                         └──────────────┬───────────────┘
                                        │
                                        ▼
                         ┌──────────────────────────────┐
                         │        Target resolution     │
                         │  exact element match / AX    │
                         │  role, label, bounds, state  │
                         └──────────────┬───────────────┘
                                        │
                                        ▼
                         ┌──────────────────────────────┐
                         │          InputEngine         │
                         │ validate → capability gate   │
                         │ availability → execute       │
                         └───────┬──────────────┬───────┘
                                 │              │
                    ┌────────────▼───┐   ┌──────▼─────────────┐
                    │ CoreDevice HID │   │ Hostless RSD/HID   │
                    │ host bridge    │   │ on-device tunnel   │
                    └────────────┬───┘   └──────┬─────────────┘
                                 │              │
                                 └──────┬───────┘
                                        ▼
                         ┌──────────────────────────────┐
                         │      Physical iPhone         │
                         │  tap / swipe / Home / text   │
                         └──────────────┬───────────────┘
                                        │
                                        ▼
                         ┌──────────────────────────────┐
                         │  Re-observe and verify       │
                         │  structural markers, not     │
                         │  pixel-difference heuristics │
                         └──────────────────────────────┘
```

## Components

| Component | Responsibility |
| --- | --- |
| `ScreenCapture` | System content picker, `SCStream`, frame lifecycle, newest-frame hand-off, orientation handling |
| `Vision` | The only adapter that imports Vision; converts recognition results into typed observations |
| `ScreenObservation` | Canonical elements with normalized top-left-origin rectangles and frame geometry |
| `InputEngine` | Actor-backed FIFO gate; validates actions, checks backend capabilities, serializes execution |
| `CoreDeviceHIDBackend` | Host-side RSD connection to the device's Universal HID service |
| `HostlessRSDInputBackend` | Device-initiated, reflected RSD tunnel to the phone's own HID service; tap-only |
| `PhoneAgentCycle` | Observe → resolve → act → re-observe → verify, with a three-action budget and stuck detection |
| `CycleEvidenceRecorder` | JSONL evidence at each boundary: run start, observe, resolve, action, verify, run end |
| `AXHID lab` | Separate target for text-only model proposals, AX grounding, and guarded single-action execution |
| `Pairing` | On-device iOS 27 pairing, staged record, verification-gated promotion |

## Engineering challenges

### 1. Screen capture arrived on iPhone with iOS 27

ScreenCaptureKit is not part of the simulator SDK. The capture service is compiled behind a platform check, and simulator builds receive an explicit "capture unavailable" controller instead of a fake stream. The app never starts capture at launch: the person must open Apple's system content picker and choose the display. The `screen-capture` background mode is declared so a full-display stream can continue while another app is in front.

On a physical iPhone 17 Pro Max running iOS 27.0, the app produced 583 completed observations with zero failed analyses while being backgrounded repeatedly. The captured frame was exactly 1320 × 2868 pixels.

### 2. One coordinate space, verified against ground truth

Vision reports bounding boxes with a bottom-left origin; iOS input expects a top-left origin. PhoneAgent defines exactly one normalized space — origin top-left, `x` right, `y` down, values in `0...1` — and one conversion point. The conversion was checked by taking the app's reported box for the word "Airbnb", cropping that rectangle out of an independently captured screenshot, and confirming that it contained exactly that word.

The rule is enforced in tests, including a synthetic rotated-frame case where a pre-rotated buffer and an upright buffer must produce the same normalized box.

### 3. HID without private frameworks

Apple's `CoreHID` framework is macOS-only in Xcode 27: the iOS SDK has no CoreHID module, and the virtual-HID entitlement is documented as macOS-only. That is preserved as a documented negative result, with a test that fails if a future SDK makes CoreHID importable on iOS.

The working path uses CoreDevice HID over an RSD connection. On a physical iPhone, tap, swipe, and Home were device-verified with before/after screenshots and pixel-change measurements of 77.75 %, 74.48 %, and 94.22 %. A mis-targeted first tap changed 0.02 % of pixels and was caught by the same observation step — which is why the loop verifies effects rather than trusting dispatch.

Text entry is the counterexample. The device refused the virtual keyboard surface: 22 correctly encoded keyboard reports were accepted with HTTP 200 and produced no character. The capability was **withheld** rather than reported as working. The engine returns a typed error instead of silently doing nothing.

### 4. A hostless transport that does not lie about its backend

M3 runs the whole loop on the phone: the app pairs with itself over a reflected packet tunnel and speaks to its own Universal HID service. There is no Mac, USB cable, or relay at run time.

The phone cannot open that service for itself without the reflection path, so the transport has strict admission rules:

- at most three actions per run, one at a time;
- re-observe after every action;
- three identical observations end the run as stuck;
- only taps are permitted on this backend;
- a result from a different backend fails the run instead of passing as the hostless path's work;
- nothing is sent while the app is in the foreground;
- every boundary is written to a JSONL evidence file.

On 2026-09-22, a physical iPhone ran the loop on itself: from the Settings root, it resolved "Pil", tapped it, observed the new screen, and verified that Settings → Battery had opened. The run used one action and ended `verified`; no Mac process was involved.

### 5. Pairing is a lifecycle, not a checkbox

The on-device iOS 27 pairing flow publishes a pairable host, accepts the system pairing sheet, and stages the resulting RPPairing record in protected storage. The staged record is **not** promoted on pairing success alone. It is used only while a read-only reflection check and one harmless HID tap are proven; the tap's target counter must change. Only then does the staged record replace the imported record, with a timestamped backup.

This was exercised physically in November 2026: pairing succeeded, the first HID attempt correctly refused because the reflection was not provider-verified, the second correctly refused for overlapping action admission, and after both defects were fixed the tap was observed and the record was promoted. That is a verified flow on one device, not a claim about every iOS 27 configuration.

### 6. Accessibility does not expose everything

The AX schema does not reliably carry switch on/off state, enabled state, or keyboard focus. PhoneAgent does not infer those values from role or nonzero traits. The AXHID lab is therefore deliberately narrow: a text-only model may choose a target index from a bounded AX inventory, but it cannot author coordinates or execute HID itself. The app performs a fresh AX read, validates the target, requires approval, sends at most one tap, and re-reads the screen.

Physical acceptance of that lab flow is still pending. It is not part of the verified claims above.

## Verified capabilities

| Area | Result | Scope |
| --- | --- | --- |
| Full-display perception | 1320 × 2868 capture; 583 observations; 0 failed analyses; OCR of another app's screen | iPhone 17 Pro Max, iOS 27.0 |
| Coordinate mapping | Reported "Airbnb" rectangle cropped from a ground-truth screenshot lands exactly on the word | Same device |
| CoreDevice HID | Tap, swipe, and Home executed; 77.75 % / 74.48 % / 94.22 % pixel change | Same device, portrait |
| Text entry | **Device-verified failure**; reports accepted, no characters produced; capability withheld | Same device |
| Hostless closed loop | One tap from Settings root to Battery, verified from the next observation; no host at runtime | Same device, 2026-09-22 |
| On-device pairing | Pairing plus one harmless HID tap; staged record promoted after observed effect | One iPhone 18 Pro Max, iOS 27, 2026-11-12 |
| Simulator suite | 2,645 tests executed, 2 skipped, 0 failures | Main app, 2026-11-10 |
| Beta suite | 2,240 tests executed, 2 skipped, 0 failures | Closed-beta feature set, 2026-11-05 |
| AXHID lab | 85 standalone tests and 41 isolated Swift 6 tests, 0 failures | Lab sources, 2026-11-11 |

## Implemented, not device-verified

- Long press, system gestures, and several hardware buttons are implemented behind the same input engine.
- The DeviceKit/XCUITest backend can type and press keys, but it depends on private XCTest APIs and is not shippable.
- The host bridge is a loopback development tool; it has no authentication and stores nothing.
- Landscape capture is wired and unit-tested but has not been exercised on a device.
- The on-device pairing flow has been accepted on one device; it is not a general claim for every iOS 27 device, network, or signing configuration.
- Cellular-only operation is not included in the pairing result.

## Experimental and explicitly not claimed

- **General autonomous AX-to-HID execution.** The lab explores a narrow, approved, single-action loop. It is not a general autonomous agent.
- **Mac-free pairing as a product capability.** On-device pairing is verified on one device; earlier builds used an imported pairing record. The runtime loop is hostless, but pairing setup remains a separate, device-specific procedure.
- **Reliable text entry over CoreDevice HID.** The device refused it; the capability is withheld.
- **Scroll and relative pointer input.** Neither real backend supplies them, and the engine does not silently substitute a swipe.
- **App Store distribution.** App Store Connect currently rejects the `screen-capture` background mode as an invalid `Info.plist` value. Development installs are unaffected.
- **Universal application coverage.** The agent sees text and AX elements, not every custom-drawn control or canvas.

## Safety and privacy model

- Capture is never automatic; it starts from an explicit tap and Apple's system picker.
- M1 frames and OCR results stay in memory. M1 contains no networking code.
- The beta visual-planner path can send screen images to a user-configured model provider, only after explicit consent; the provider choice and key are user-supplied.
- Logs contain counts, sequence numbers, and durations — not recognized text or coordinates.
- Pairing records, keys, signing material, and device identifiers are excluded from public documentation.
- The input engine gates capabilities, serializes actions, and records whether a result is a dispatch acknowledgement or an observed effect.

## Technology stack

Swift 6.4 · SwiftUI · ScreenCaptureKit · Vision · iOS Accessibility (AX) · CoreDevice HID · RSD · Network Extension (packet tunnel) · vendored `idevice` FFI (MIT) · Python host relay · XcodeGen · XCTest

Third-party notes: the hostless pairing implementation uses the MIT-licensed `idevice` crate. `StikPair` (MIT, non-commercial) was used as a behavior/UX reference only; `StikDebug` (AGPL-3.0) was not used and no source was copied.

## Verification notes

Device-verified claims come from local acceptance records with before/after screenshots, pixel measurements, and JSONL event logs. Those raw records are private because they contain device and personal context. Test counts are from local runs; there is no public CI result for this project.

The project is intentionally documented at this level of detail because the interesting engineering is in the boundaries: what the OS exposes, what the transport accepts, and what the device actually does.

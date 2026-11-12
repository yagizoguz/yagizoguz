# MorphDesk

**An experimental semantic remote desktop prototype: macOS Accessibility → native iPhone interface.**

Status: experimental prototype under development · not a product · source private · last verified November 2026

**Publication status: private, not a release candidate, and not being prepared for public release.**

## Overview

MorphDesk is a research prototype that explores a different approach to remote desktop. Instead of streaming pixels and replaying taps at coordinates, it reads a real macOS Accessibility (AX) tree, normalizes it into a typed Codable model, and generates a native iPhone interface from the controls the host actually exposes.

The phone does not receive a video stream. It receives a structured application model — roles, labels, values, state, capabilities, and parent relationships — and sends back validated, capability-bound operations. The Mac host executes them through its own AX APIs and the phone receives an independent read-back.

Standard mode is deterministic and local: no AI, no cloud, no provider. An optional Adaptive mode can ask a user-configured OpenAI-compatible model to regroup the same bounded model into a more usable mobile layout, but the model never receives raw values or document text, and it cannot author operations.

**MorphDesk is not production-ready, is not generally available, and does not claim universal remote-desktop compatibility.** The verified results below come from one bounded validation environment and a small set of applications.

## At a glance

| | |
| --- | --- |
| Host | macOS 27, Swift 6.4, Accessibility APIs |
| Client | iOS 27, SwiftUI, physical iPhone 18 Pro Max |
| Transport | Network.framework, TLS 1.2 with a pre-shared key |
| Layout | Deterministic role-to-control mapping; optional model-assisted regrouping |
| Execution | Host-issued capabilities, confirmation, freshness tokens, independent AX read-back |
| Status | Experimental prototype under development; no production-readiness or general-availability claim |
| Source | Private |

## Architecture

```text
macOS host                                          iPhone
┌──────────────────────────┐                    ┌───────────────────────────┐
│ Accessibility APIs       │                    │ Snapshot decoder          │
│   ↓                      │   TLS 1.2 + PSK    │   ↓                       │
│ Bounded AX snapshot      │ ─────────────────► │ Deterministic layout      │
│   ↓                      │                    │   ↓                       │
│ Normalized Codable model │                    │ Native SwiftUI controls   │
│   ↓                      │ ◄───────────────── │   ↓                       │
│ Capability registry      │  validated action  │ Confirmation + request    │
│   ↓                      │                    │   ↓                       │
│ HostSession.execute      │                    │ Execution result          │
│   ↓                      │ ─────────────────► │   ↓                       │
│ AX setter / press        │  independent read  │ Verified / unverified     │
└──────────────────────────┘                    └───────────────────────────┘
```

The important boundary is at the bottom: the phone can request an operation, but the host owns the capability mapping and the executor. A stale target token is rejected rather than replayed against a changed interface.

## Components

| Component | Responsibility |
| --- | --- |
| AX reader | Traverses the frontmost application with a bounded budget; exports only supported roles and capabilities |
| Normalized model | Codable snapshot: application identity, elements, values, state, hierarchy, capabilities |
| Deterministic planner | Maps roles to native controls, groups and paginates large collections, exposes deferred children on demand |
| Host executor | Revalidates target state, performs AX setters/presses, reports dispatch and effect separately |
| TLS transport | Authenticated Network.framework connection using a shared 256-bit pairing key |
| iOS client | Native renderer, editor drafts, confirmation flow, reconnect and application switching |
| Adaptive planner | Optional model-assisted regrouping over a sanitized, bounded summary; strict local schema validation and fallback |

## Capability status

The four categories below are intentionally separate. A passing test does not turn a prototype capability into a product feature.

### Implemented and verified

| Capability | Evidence |
| --- | --- |
| macOS AX snapshot and normalization | Real exports from TextEdit, Finder, Safari, and Calculator |
| Deterministic iPhone layout | Native controls rendered from the real exported model |
| Authenticated transport | Real TLS sockets, including wrong-key rejection |
| Host-side execution with freshness checks | Real AX setters/presses; stale target rejected after an independent desktop mutation |
| Independent read-back | `verified` only when a separate AX read observed the intended effect |
| Physical device flow | Pair → render → edit → apply → AX read-back → disconnect/reconnect on a physical iPhone |
| Supported application interactions | TextEdit save, Finder navigation/selection, Safari form edit/link navigation, Calculator sidebar |
| Real-AX test suite | 15 tests passed, 0 failures, no skips |

### Implemented and tested, not fully verified

| Capability | Status |
| --- | --- |
| Adaptive layout planner, strict schema validation, fallback | Implemented and covered by offline/provider-mock tests; **no real-provider inference was performed**, so there are no real AI-generated layouts to show |
| Native macOS GUI host | Builds; the validated host was the terminal-launched CLI with existing Accessibility authorization. A fresh GUI permission flow was not exercised |
| ScreenCaptureKit still-image provider | Compiles, but permission-granted capture, pixel transport, rendering, and visual analysis were not exercised |
| Tailscale addressing | Uses the same connection path, but remote tailnet routing and ACLs were not tested |

### Incomplete

| Area | Current limitation |
| --- | --- |
| Identity and revocation | One shared 256-bit pairing key; no per-device identities or individual revocation |
| Background / locked-phone operation | Not validated |
| Long-running behavior | No soak testing; reconnection and session replacement are bounded |
| Secure fields | Not forwarded |
| Clipboard and file transfer | Not implemented |
| Universal application coverage | Custom canvases, games, unexposed controls, and some large AX trees are unsupported or truncated |
| Real-provider Adaptive validation | Implemented but not demonstrated end-to-end |

### Planned / not implemented

| Area | Status |
| --- | --- |
| QR or Bonjour onboarding | Planned; not implemented |
| Pixel/visual fallback path | Planned; not implemented |
| Multi-application AI task composition | Planned; not implemented |

These entries are future possibilities, not commitments or a product roadmap. No release date, support plan, or compatibility program exists for this prototype.

## Engineering challenges

### 1. Accessibility is a capability surface, not a UI toolkit

The same visual control can expose completely different AX behavior across applications. Finder, for example, did not advertise a directly settable selection on the item itself; its parent exposed `AXSelectedChildren`. MorphDesk exports only the capability the host can actually support, with a host-side parent mapping, instead of synthesizing a click that might do nothing.

Unsupported regions are marked as unsupported. They are not converted into invented controls.

### 2. Layout has to stay deterministic

The deterministic planner maps real roles to native controls: text fields, toggles, buttons, menus, tables, lists, tabs, and collapsible groups. Menus and large collections use navigation and pagination; deferred structures are loaded only when the person asks for them.

This is intentionally conservative. The goal is a usable interface for the controls the host exposes, not a pixel-perfect recreation of the desktop.

### 3. Dispatch is not success

Accessibility setters can be accepted and ignored. MorphDesk therefore treats execution as a two-stage claim:

- `dispatched` / `dispatchedUnverified` means the host accepted and sent the operation.
- `verified` means an independent AX read observed the intended effect.

During validation, Safari's Apply button returned `dispatchedUnverified`; a separate AX capture then confirmed that the page output had changed. That is the evidence model working as intended. A stale target token after an independent desktop edit was rejected and the old value was confirmed **not** written.

### 4. Freshness without freezing the interface

A whole-snapshot staleness rule rejected edits when unrelated Finder decorations moved. The final design uses host-issued per-target state tokens: unrelated revisions are allowed, but a genuinely changed target invalidates its capability. The real-Mac integration test exercises this with an actual document mutation, not a synthetic flag.

### 5. Security boundaries are part of the prototype

Pairing is a shared 256-bit key over TLS 1.2 PSK. It is not per-device identity, and PSK does not provide forward secrecy. Every mutation or activation requires explicit confirmation. The pairing window itself is never exported, and the service is not designed to be exposed to the public Internet; Tailscale is the recommended remote path, though tailnet routing and ACLs were not tested in the validation run.

The optional Adaptive mode adds a separate boundary: only a sanitized summary is sent to the provider — no values, document text, window titles, static labels, geometry, or session tokens. The model may only reference host-issued capability identifiers. If validation fails, the client falls back to the deterministic layout.

## Validation record

| Test or build | Result | Scope |
| --- | --- | --- |
| Real-AX suite | 15 tests passed, 0 failures, no skips; includes real TextEdit and Safari synchronization | macOS 27 + iPhone simulator |
| Physical iPhone flow | Pair → render real editor → edit → apply → independent AX read-back → disconnect/reconnect | iPhone 18 Pro Max, iOS 27.0.1 |
| TextEdit | Real document edit verified by AX read-back; Save persisted and was checked on disk | Physical phone and simulator |
| Finder | Real folder navigation and selection verified through parent `AXSelectedChildren` | Host protocol client |
| Safari | Focus + set verified; Apply and link navigation confirmed by separate AX captures | Host protocol client |
| Calculator | Sidebar activation and restore verified from real hierarchy changes | Host protocol client |
| Current prototype test suite | 43 discovered, 41 executed, 2 opt-in skipped, 0 failures | Current prototype sources |
| Builds | macOS host and iOS simulator builds succeeded; signed physical install succeeded | Xcode 27 |

## Explicit non-claims

- MorphDesk is **not production-ready** and is **not generally available**.
- It is not a VNC/RDP replacement or a universal remote desktop.
- It does not guarantee compatibility with arbitrary macOS applications, custom-drawn canvases, games, unexposed controls, or secure fields.
- It does not claim that a press or setter succeeded without an independent read-back.
- It is not an Internet-hardened remote-control service and has no security certification.
- It does not claim real AI-driven interface redesign until a real provider run is completed and the resulting controls operate a real application with independently checked effects.
- It has no support commitment, SLA, distribution plan, or compatibility program as a prototype.
- The verified results cover one Mac, one physical iPhone, and the listed applications; they are not a general field-readiness claim.

## Technology stack

Swift 6.4 · SwiftUI · AppKit Accessibility APIs · Network.framework · TLS 1.2 PSK · Codable · XcodeGen · XCTest / Swift Testing

No third-party runtime service is required. Adaptive mode is optional and uses a provider chosen and configured by the user.

## Verification notes

The validation record is dated 4 November 2026 and covers a MacBook Pro M1 (8 GB), macOS 27.0, Xcode 27.0, and a physical iPhone 18 Pro Max on iOS 27.0.1. Raw logs and screenshots contain real desktop and device context, so they are kept private; no screenshots or demos are published. This document summarizes the outcomes and the failed or unverified cases.

MorphDesk is an experimental prototype, not a product. Its value is in the boundary: a real Accessibility tree on one side, a native mobile interface on the other, and an execution path that reports what it actually observed. Publication and release readiness are separate future decisions.

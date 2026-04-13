# falora-mystic-showcase
# Sample Code

This directory contains representative excerpts from the Falora codebase, selected to demonstrate specific engineering patterns and technical depth. All API keys, base URLs, and sensitive credentials have been replaced with placeholders.

---

## Overview

| File | Pattern Demonstrated | Key Technologies |
|------|----------------------|------------------|
| [SignalRService.swift](#signalrserviceswift) | Custom protocol implementation, reconnection strategy | URLSessionWebSocketTask, NWPathMonitor, Combine |
| [FortuneNarrationService.swift](#fortunenarrationserviceswift) | External API integration, disk caching | ElevenLabs REST API, CryptoKit, AVFoundation |
| [EntitlementService.swift](#entitlementserviceswift) | Feature gating, subscription tier engine | StoreKit 2, protocol-oriented design |
| [DIContainer.swift](#dicontainerswift) | Dependency injection, factory pattern | SwiftUI Environment, singleton pattern |
| [DesignSystem/](#designsystem) | Token-based design system | SwiftUI styling, semantic tokens |
| [Components/](#components) | Reusable UI components | SwiftUI animations, composable views |

---

## SignalRService.swift

**What it demonstrates:** A fully hand-written SignalR client built on top of Apple's `URLSessionWebSocketTask` — no third-party SignalR Swift package is used.

### Why This Is Interesting

Most iOS apps that need SignalR connectivity reach for Microsoft's official client or a community package. Falora implements the entire JSON Hub Protocol from scratch, which gave us full control over connection lifecycle, error handling, and reconnection behavior.

### Technical Highlights

- **Negotiate + WebSocket handshake:** Performs the standard SignalR negotiate POST to obtain a `connectionToken`, then opens a WebSocket connection with proper query parameters and sends the `{"protocol":"json","version":1}` + `0x1E` handshake.
- **Message framing:** Correctly handles the `0x1E` (Record Separator) delimiter used by SignalR's JSON protocol for message boundaries.
- **Hub method invocation:** Supports both client → server invocations (`JoinRoom`, `LeaveRoom`, `SendRoomMessage`) and server → client callbacks via `on(method:handler:)`.
- **Ping/pong keep-alive:** Sends periodic type-6 ping messages and monitors pong responses to detect stale connections.
- **Exponential backoff reconnection:** On disconnect, retries with increasing delays (2s → 4s → 8s → 16s → 30s cap). Stops reconnection on 401/403 to avoid spinning against expired tokens.
- **Network interface monitoring:** Uses `NWPathMonitor` to detect WiFi ↔ Cellular transitions and triggers immediate reconnection attempts.
- **Foreground resumption:** Listens for `UIApplication.willEnterForeground` to verify connection health when the app returns from background.
- **Thread safety:** Entire service is `@MainActor` isolated, with async/await concurrency throughout.

### Key Concepts for Reviewers

```
┌──────────────┐     POST /negotiate      ┌─────────────┐
│  SignalR      │ ──────────────────────── │   Backend    │
│  Service      │     connectionToken      │   Hub        │
│               │ ◄─────────────────────── │              │
│               │                          │              │
│               │     WSS + handshake      │              │
│               │ ══════════════════════── │              │
│               │     { ping/pong }        │              │
│               │ ◄═════════════════════►  │              │
└──────────────┘                           └─────────────┘
        │
        ├── NWPathMonitor (network changes → reconnect)
        ├── Exponential backoff (2s, 4s, 8s, 16s, 30s)
        └── Foreground health check
```

---

## FortuneNarrationService.swift

**What it demonstrates:** Integration with ElevenLabs' multilingual text-to-speech API, including intelligent disk caching to avoid redundant API calls.

### Why This Is Interesting

Narration is a Platinum-tier feature where AI-generated fortune readings are read aloud. The service must handle network requests for audio synthesis, cache the results for offline/repeated playback, and manage the AVAudioPlayer lifecycle — all while remaining reactive to SwiftUI's observation system.

### Technical Highlights

- **ElevenLabs integration:** Sends fortune text to the `eleven_multilingual_v2` model with Turkish language configuration via their REST API. Returns raw audio data.
- **SHA256 disk cache:** Each narration request is keyed by the SHA256 hash of its input text (via CryptoKit). Before making an API call, the service checks `Caches/narration/{hash}.mp3` for a cached version.
- **Audio lifecycle:** Uses `AVAudioPlayer` with proper `AVAudioSession` category configuration. Implements `AVAudioPlayerDelegate` to track completion and update observable state.
- **State management:** Published `NarrationState` enum (`.idle`, `.loading`, `.playing`, `.paused`, `.error`) drives UI updates. The service is `@MainActor` and `ObservableObject` for seamless SwiftUI binding.
- **Error resilience:** Network failures, invalid audio data, and playback errors are all caught and surfaced as typed states rather than crashes.

---

## EntitlementService.swift

**What it demonstrates:** A multi-tier subscription-based feature gating system that cleanly separates business rules from UI code.

### Why This Is Interesting

With four subscription tiers (Free, No Ads, Gold, Platinum) and dozens of gated features (reading quotas, method access, TTS, rooms, DMs, boosts), the entitlement logic could easily become a scattered mess of `if` statements throughout the codebase. Instead, it's centralized in a single service with a typed result system.

### Technical Highlights

- **Plan → Capabilities mapping:** `Entitlements.forPlan(_:)` is a pure function that returns an `Entitlements` struct for each `SubscriptionPlan`. Each struct defines closures and values for every gated capability.
- **Typed result system:** Every check returns an `EntitlementResult` enum:
  - `.allowed` — proceed with the action
  - `.requiresUpgrade(minimumPlan:)` — show paywall highlighting the required tier
  - `.quotaReached(resetAt:)` — show quota message with next reset time
  - `.requiresAuth` — redirect to authentication
- **Feature coverage:** Daily reading quotas, fortune method access, ad-free experience, custom questions, deep insights, narration (TTS), room join/creation, daily message limits, DM, post boosting, leaderboard access.
- **Composable checks:** View models call specific check methods (`checkReadingAccess()`, `checkNarrationAccess()`, `checkRoomJoinAccess()`, etc.) and react to the result — the service decides, the VM presents.
- **Zero UI dependency:** The service knows nothing about SwiftUI. It takes a subscription plan and returns a decision.

### Tier Comparison

| Capability | Free | No Ads | Gold | Platinum |
|-----------|------|--------|------|----------|
| Daily readings | 2 | 5 | 15 | Unlimited |
| Fortune methods | Basic only | Basic only | All | All |
| Ads | Yes | No | No | No |
| Custom questions | No | No | Yes | Yes |
| AI Narration (TTS) | No | No | No | Yes |
| Room creation | No | No | No | Yes |
| DM | No | No | No | Yes |
| Post boost | No | No | No | Yes |

---

## DIContainer.swift

**What it demonstrates:** A lightweight, app-specific dependency injection container using the singleton pattern with factory methods, integrated into SwiftUI's environment system.

### Why This Is Interesting

Rather than using a DI framework (Swinject, Needle, etc.), Falora uses a simple but effective hand-rolled container. All dependencies are created eagerly at launch in a single `private init()`, making the dependency graph explicit and easy to reason about.

### Technical Highlights

- **Private init singleton:** `DIContainer.shared` is the sole instance, created once at app launch. The `private init()` ensures no accidental duplicate containers.
- **Eager instantiation:** All services (`ApiService`, `EntitlementService`, `SignalRService`, etc.), state objects (`AppState`, `SessionStore`, `SubscriptionStore`), and the root router are created in `init()`.
- **Factory methods:** `makeMainTabRouter()`, `makeReadFlowCoordinator()`, `makeAuthViewModel()`, `makeCoffeeFortuneViewModel()` — these produce view models and coordinators with their dependencies pre-wired.
- **SwiftUI Environment integration:** A custom `DIContainerKey: EnvironmentKey` is registered so any view in the hierarchy can access the container via `@Environment(\.diContainer)`.
- **Protocol-based repositories:** Repository dependencies are typed as protocols (`FortuneMethodRepository`, etc.), allowing mock implementations for previews and testing.

### Pattern Summary

```swift
// 1. Singleton with private init
@MainActor
final class DIContainer: ObservableObject {
    static let shared = DIContainer()
    private init() { /* wire everything */ }
}

// 2. Factory methods for ViewModels
func makeAuthViewModel() -> AuthViewModel { ... }

// 3. SwiftUI Environment injection
struct DIContainerKey: EnvironmentKey { ... }
extension EnvironmentValues {
    var diContainer: DIContainer { ... }
}
```

---

## DesignSystem/

**What it demonstrates:** A token-based design system that ensures visual consistency across 137 Swift files and multiple feature areas.

### Files

### `Colors.swift`

Defines semantic color tokens as `Color` extensions:

| Token | Purpose |
|-------|---------|
| `falPrimary` | Primary brand accent |
| `falBackground` | Main background |
| `falCard` | Card surface |
| `falSuccess`, `falError`, `falWarning` | Semantic feedback colors |

All views reference these tokens instead of raw `Color(hex:)` values. Changing the app's color palette is a single-file operation.

### `Typography.swift`

Implements a structured type scale via `FaloraFont`:

| Level | Usage |
|-------|-------|
| `display` | Hero text, large headers |
| `headline` | Section titles |
| `body` | Main content |
| `label` | Captions, metadata |

Uses `.system(..., design: .rounded)` for a friendly, approachable feel that matches the app's mystical-yet-modern aesthetic.

### `Spacing.swift`

Provides consistent spacing, radius, icon, and button size tokens:

| Type | Examples |
|------|----------|
| `FaloraSpacing` | `.xs`, `.sm`, `.md`, `.lg`, `.xl` |
| `FaloraRadius` | `.sm`, `.md`, `.lg`, `.full` |
| `FaloraIconSize` | `.small`, `.medium`, `.large` |
| `FaloraButtonSize` | `.compact`, `.regular`, `.large` |

---

## Components/

**What it demonstrates:** Reusable, composable SwiftUI components that follow the design system tokens.

### `PrimaryButton.swift`

A standardized button component used throughout the app. Features:
- Consistent padding, corner radius, and typography from the design system
- Loading state with built-in spinner
- Disabled state with reduced opacity
- Haptic feedback on tap
- Gradient or solid fill variants matching the mystical theme

### `MysticalLoader.swift`

An animated loading indicator with the app's signature mystical aesthetic. Features:
- Multi-layer animation (rotating rings, pulsing glow, floating particles)
- Configurable size via `FaloraIconSize` tokens
- Smooth state transitions using SwiftUI's animation system
- Used during fortune reading processing to maintain the immersive experience

---

## Running the Samples

These files are provided for reading and reference purposes. They are extracted from the full Xcode project and cannot be compiled standalone — they depend on the app's internal types, services, and SwiftUI environment.

To understand how they fit together, see [`ARCHITECTURE.md`](../ARCHITECTURE.md) for layer diagrams and data flow sequences.

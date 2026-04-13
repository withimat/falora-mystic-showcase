<p align="center">
  <img src="screenshots/app-icon.png" width="120" alt="Falora App Icon" />
</p>

<h1 align="center">Falora — AI-Powered Mystical Fortune Telling</h1>

<p align="center">
  <img src="https://img.shields.io/badge/Platform-iOS_17+-000000?style=flat&logo=apple&logoColor=white" />
  <img src="https://img.shields.io/badge/SwiftUI-5.9-F05138?style=flat&logo=swift&logoColor=white" />
  <img src="https://img.shields.io/badge/Architecture-MVVM_+_Coordinator-8E44AD?style=flat" />
  <img src="https://img.shields.io/badge/Localization-TR_·_EN_·_AR-1ABC9C?style=flat" />
  <img src="https://img.shields.io/badge/Real--time-SignalR_WebSocket-3498DB?style=flat" />
  <img src="https://img.shields.io/badge/AI-ElevenLabs_TTS-FF6F00?style=flat" />
</p>

<p align="center">
  <a href="https://apps.apple.com/app/falora">
    <img src="https://developer.apple.com/assets/elements/badges/download-on-the-app-store.svg" height="40" alt="Download on the App Store" />
  </a>
</p>

---

**Falora** is a production iOS app that blends artificial intelligence with mystical content — coffee fortune reading, tarot spreads, dream interpretation, palm reading, birth charts, and daily horoscopes — wrapped in a social experience with real-time chat rooms, a community feed, and a tiered subscription economy. Built entirely in SwiftUI with 137 Swift files across three languages.

<p align="center">
  <img src="screenshots/onboarding.png" width="180" />
  <img src="screenshots/fortune-reading.png" width="180" />
  <img src="screenshots/feed.png" width="180" />
  <img src="screenshots/cosmic-match.png" width="180" />
</p>

---

## Key Features

| Category | Features |
|----------|----------|
| **Fortune Reading** | Coffee fortune (classic & deep), Tarot (6 spread types including Celtic Cross), Dream interpretation, Palm reading (basic & complete), Birth chart (natal), Daily horoscope |
| **AI Narration** | ElevenLabs TTS integration with SHA256 disk caching — Platinum-tier users hear their readings narrated aloud |
| **Social** | Community feed with posts, likes, comments & boosts; Real-time chat rooms via custom SignalR client; Cosmic Match (zodiac compatibility); Leaderboard |
| **Monetization** | 4-tier subscription system (Free → No Ads → Gold → Platinum) via StoreKit 2; Credit-based economy for à la carte readings |
| **Localization** | Full Turkish, English, and Arabic support with RTL layout |
| **Infrastructure** | Firebase Analytics & Cloud Messaging; Google AdMob; Google & Apple Sign-In |

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| **UI Framework** | SwiftUI (iOS 17+, Swift 5.9) |
| **Architecture** | MVVM + Coordinator/Router + Clean Architecture layers |
| **Reactive** | Combine, `@Observable`, `@MainActor` concurrency |
| **Networking** | URLSession (REST), URLSessionWebSocketTask (SignalR) |
| **Real-time** | Hand-written SignalR client — WebSocket + JSON Hub Protocol with exponential backoff reconnection and NWPathMonitor |
| **AI / TTS** | ElevenLabs Multilingual v2 REST API + AVAudioPlayer |
| **Payments** | StoreKit 2 (subscriptions & consumables) |
| **Auth** | AuthenticationServices (Apple Sign-In), GoogleSignIn SDK |
| **Analytics** | Firebase Analytics, Firebase Cloud Messaging |
| **Ads** | Google Mobile Ads (AdMob) |
| **Security** | Keychain Services, CryptoKit (SHA256 cache keys) |
| **Media** | PhotosUI (image picker), AVFoundation (audio playback) |
| **DI** | Custom singleton container with factory methods + SwiftUI Environment injection |

---

## Architecture Overview

The codebase follows a **layered Clean Architecture** approach with clear separation between Domain, Data, Core, and Presentation layers.

```
┌─────────────────────────────────────────────────┐
│                  Presentation                    │
│  Views · ViewModels · Components · Coordinators  │
├─────────────────────────────────────────────────┤
│                     Core                         │
│  Router · Services · DI · DesignSystem · State   │
├─────────────────────────────────────────────────┤
│                    Domain                        │
│              Entities · Models                   │
├─────────────────────────────────────────────────┤
│                     Data                         │
│                 Repositories                     │
└─────────────────────────────────────────────────┘
```

**Navigation** is driven by observable `Router` objects (`RootRouter` → `MainTabRouter` → `ReadFlowCoordinator`) that manage `NavigationPath` instances per tab — no UIKit coordinator classes, pure SwiftUI state.

> See [`ARCHITECTURE.md`](ARCHITECTURE.md) for detailed Mermaid diagrams covering layer dependencies, navigation flow, and data pipelines.

---

## Notable Technical Decisions

### 1. Hand-Written SignalR Client
Instead of relying on a third-party SignalR Swift package, Falora implements the full JSON Hub Protocol over `URLSessionWebSocketTask` — negotiate handshake, `0x1E` message framing, ping/pong keep-alive, and reconnection with exponential backoff (2→4→8→16→30s). `NWPathMonitor` triggers automatic reconnection on network interface changes.

### 2. Tiered Entitlement Engine
A declarative `EntitlementService` maps each `SubscriptionPlan` to a set of feature capabilities (method access, daily quotas, narration, room creation, DM, leaderboard). Every feature check returns a typed `EntitlementResult` (`.allowed`, `.requiresUpgrade(minimumPlan:)`, `.quotaReached(resetAt:)`, `.requiresAuth`), keeping paywall logic out of view code.

### 3. ElevenLabs TTS with Disk Cache
Fortune narrations are synthesized server-side via ElevenLabs' multilingual v2 model. Audio responses are cached on disk keyed by SHA256 hashes of the input text, so repeated readings play instantly without re-fetching.

### 4. Coordinator-Style SwiftUI Routing
Three nested observable routers (`RootRouter` → `MainTabRouter` → `ReadFlowCoordinator`) manage all navigation state. Each tab maintains its own `NavigationPath`, and the read flow coordinator gates each step through the entitlement system before advancing.

---

## Sample Code Highlights

The [`sample-code/`](sample-code/) directory contains representative excerpts (with secrets redacted) showcasing the project's engineering quality:

| File | What It Demonstrates |
|------|----------------------|
| [`SignalRService.swift`](sample-code/SignalRService.swift) | Custom WebSocket + SignalR hub protocol, reconnection strategy, NWPathMonitor |
| [`FortuneNarrationService.swift`](sample-code/FortuneNarrationService.swift) | ElevenLabs TTS integration, SHA256 disk cache, AVAudioPlayer lifecycle |
| [`EntitlementService.swift`](sample-code/EntitlementService.swift) | Multi-tier subscription feature gating with typed result system |
| [`DIContainer.swift`](sample-code/DIContainer.swift) | Singleton DI with factory methods + SwiftUI Environment injection |
| [`DesignSystem/`](sample-code/DesignSystem/) | Token-based design system: Colors, Typography, Spacing |
| [`Components/`](sample-code/Components/) | Reusable UI components: PrimaryButton, MysticalLoader |

> See [`sample-code/README.md`](sample-code/README.md) for detailed explanations.

---

## Demo

🎬 A walkthrough video showcasing the full user journey is available in the [`demo/`](demo/) directory.

---

## Project Stats

| Metric | Value |
|--------|-------|
| Swift files | 137 |
| Languages | 3 (Turkish, English, Arabic) |
| Fortune types | 6 categories, 15+ methods |
| Subscription tiers | 4 (Free, No Ads, Gold, Platinum) |
| Architecture | MVVM + Coordinator + Clean Architecture |

---

## Disclaimer

This is a **showcase repository** containing selected code samples and documentation for portfolio purposes. The full source code, API keys, backend endpoints, and proprietary assets are not included. The app is available on the [App Store](https://apps.apple.com/app/falora).

---

<p align="center">
  Built with ❤️ and SwiftUI
</p>

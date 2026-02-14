# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

VisionClaw is an iOS app that streams real-time video from Meta Ray-Ban smart glasses (via the Meta Wearables DAT SDK) to Google's Gemini Live API for AI-powered voice + vision conversations. It optionally delegates tool calls to an OpenClaw gateway running on the user's Mac.

## Build & Run

All source lives under `samples/CameraAccess/`. Open the Xcode project:

```bash
open samples/CameraAccess/CameraAccess.xcodeproj
```

- **Xcode 15+**, **iOS 17+**, Swift 5
- Bundle ID: `com.xiaoanliu.VisionClaw`
- Signing team is set; you may need to update it for your Apple Developer account

### Secrets setup (required before first build)

```bash
cp samples/CameraAccess/CameraAccess/Secrets.swift.example samples/CameraAccess/CameraAccess/Secrets.swift
```

Edit `Secrets.swift` with your Gemini API key (and optionally OpenClaw credentials). This file is gitignored.

### Running tests

```bash
xcodebuild test -project samples/CameraAccess/CameraAccess.xcodeproj -scheme CameraAccess -destination 'platform=iOS Simulator,name=iPhone 16'
```

Tests are in `samples/CameraAccess/CameraAccessTests/CameraAccessTests.swift` (XCTest, integration tests using mock DAT SDK devices).

## Architecture

MVVM with SwiftUI. All source is in `samples/CameraAccess/CameraAccess/`.

### Key modules

| Module | Files | Role |
|--------|-------|------|
| **Gemini/** | `GeminiConfig.swift`, `GeminiLiveService.swift`, `GeminiSessionViewModel.swift`, `AudioManager.swift` | WebSocket client for Gemini Live API, audio I/O, session lifecycle |
| **OpenClaw/** | `OpenClawBridge.swift`, `ToolCallRouter.swift`, `ToolCallModels.swift` | HTTP client for OpenClaw gateway, tool call routing, data models |
| **ViewModels/** | `StreamSessionViewModel.swift`, `WearablesViewModel.swift`, `DebugMenuViewModel.swift` | DAT SDK streaming orchestration, device registration, debug tools |
| **iPhone/** | `IPhoneCameraManager.swift` | AVCaptureSession fallback for testing without glasses |
| **Views/** | SwiftUI views + `Components/` | UI layer including `GeminiOverlayView` (transcript, status, tool call display) |

### Core data flows

**Video:** Glasses camera (or iPhone back camera) → `StreamSessionViewModel` → throttled to 1fps JPEG (50% quality) → `GeminiLiveService.sendVideoFrame()` over WebSocket.

**Audio in:** Mic → `AudioManager` (AVAudioEngine, resampled to 16kHz Int16 mono, 100ms chunks) → `GeminiLiveService.sendAudio()`. Echo prevention: mic muted while model speaks in iPhone mode.

**Audio out:** Gemini audio (24kHz PCM base64) → `AudioManager.playAudio()` → speaker.

**Tool calls:** Gemini emits `execute` tool call → `ToolCallRouter` → `OpenClawBridge.delegateTask()` (HTTP POST to local Mac) → result sent back to Gemini via `sendToolResponse()`.

### Configuration

- `GeminiConfig.swift` — model name, WebSocket URL, audio/video parameters, system instruction, tool declarations. Reads keys from `Secrets`.
- `Secrets.swift` (gitignored) — API keys and OpenClaw host/tokens. Template at `Secrets.swift.example`.
- `Info.plist` — DAT SDK app IDs, permissions (Bluetooth, camera, mic, photo library), background modes, local networking.

### Frameworks

- **MWDATCore / MWDATCamera** — Meta Wearables DAT SDK (device streaming, registration)
- **MWDATMockDevice** — Mock glasses for debug builds only (excluded from release via build config)
- **AVFoundation** — Audio engine + iPhone camera capture
- **SwiftUI / UIKit** — UI

## Conventions

- `@MainActor` on all view models for thread safety
- Async/await throughout; no Combine pipelines
- All WebSocket sends go through a serial `DispatchQueue` to avoid interleaving
- Audio session mode: `.voiceChat` for iPhone (aggressive AEC), `.videoChat` for glasses
- PRs go to `main` branch; Meta imports PRs internally (not merged directly on GitHub)

## Fork Context
- Forked from: git@github.com:sseanliu/VisionClaw.git

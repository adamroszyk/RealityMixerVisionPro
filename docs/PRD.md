# PRD: Spectator Mode for Apple Vision Pro

| | |
|---|---|
| **Status** | Draft v0.1 |
| **Last updated** | 2026-10-09 |
| **Related** | [TDD](TDD.md) · [Roadmap](../ROADMAP.md) · [Assessment](ASSESSMENT.md) · [Research](RESEARCH.md) |

## 1. Summary

A Swift package plus a free iPhone app. Together they let any RealityKit visionOS app be filmed from a **third-person viewpoint**. The iPhone is the camera, the Vision Pro renders the app's 3D content from the iPhone's position, and the iPhone composites the two live, with the wearer correctly in front of or behind virtual objects. The result can be viewed live, recorded, or sent to a bigger screen.

## 2. Problem

Showing a spatial app to people who aren't wearing the headset is one of the hardest parts of shipping for Vision Pro.

- **First-person only.** Every built-in capture tool records the wearer's own view: Mirror My View, Record My View, High Quality capture, and Developer Capture. They can't show the wearer *inside* the experience, which is the most persuasive shot for marketing, demos, and education.
- **No platform support.** visionOS has no API for rendering a scene from an extra camera or for a third-person composite. iPhone and Vision Pro can't share a coordinate space. SwiftUI windows and attachments are rendered out of process and can't be captured.
- **High cost to build in-house.** Doing it yourself means solving calibration, cross-device sync, real-time video, compositing, and a thermal budget at the same time. Most teams don't, and fall back on post-produced trailers.
- **The prior attempt is a prototype.** [fabio914/RealityMixerVisionPro](https://github.com/fabio914/RealityMixerVisionPro) shows the approach works. It has correctness bugs, main-thread stalls, manual IP entry, and it reparents the app's live entities every frame (see the [Assessment](ASSESSMENT.md)).

**Lessons from earlier platforms.** HoloLens Spectator View and Meta Quest Mixed Reality Capture both lost developer support once integration and maintenance became work every app had to keep doing. **Low integration cost is a core requirement, not a nice-to-have.**

## 3. Target users

| Persona | Need | Key requirement |
|---|---|---|
| **Indie / studio visionOS developer** (primary) | Trailer and App Store footage, social clips, demos for investors and press | Integration in minutes; no effect on the app's performance |
| **Marketing / content creator** | Good-looking mixed-reality footage without VFX work | Good image quality, in-app recording, simple calibration |
| **Event / booth operator** | Show a live third-person view on a TV while someone uses the headset | Reliable connection, reconnects automatically, live output to a display |
| **Educator / enterprise trainer** | Let a room see what a trainee is doing in context | Works on a typical Wi-Fi network, runs for a long session |

## 4. Goals and non-goals

### Goals

1. **G1: Adoption.** A developer can add spectator mode to an existing RealityKit immersive app with about 10 lines of code. They see the first composite within 10 minutes of starting, using the published iPhone app.
2. **G2: Quality.** The footage is good enough to post: 1080p (30 fps, 60 optional), no visible lag or sliding between virtual and real content in recordings, clean edges around people.
3. **G3: No harm to the host app.** Spectator mode never causes dropped frames or main-thread stalls in the headset app at normal settings, and it backs off under thermal pressure.
4. **G4: Reliability.** Automatic discovery, pairing, and reconnection. It survives app backgrounding, Wi-Fi blips, and iPhone app restarts.
5. **G5: Open and honest.** MIT licensed, documented limits, a public compatibility matrix, and fixes offered back upstream.

### Non-goals (for now)

- Capturing system UI pixels (windows, volumes, attachments). Not possible on visionOS today. M4 *re-draws* UI panels instead (see [TDD §8](TDD.md#8-ui-panels-swiftui-windows--attachments)).
- Shared Space apps without an immersive space.
- Multiple simultaneous spectator cameras (considered for later).
- Professional camera rigs (DSLR plus genlock). The iPhone is the only camera for now.
- Android, Meta Quest, or web clients.

## 5. Use cases and user stories

**Developer**
- **US-1:** As a developer, I add the Swift package, apply `.mixedRealityCapture()` to my `RealityView`, and tag the entities I want on camera. Then my app is spectator-ready.
- **US-2:** I can choose which entities appear (for example, hide debug gizmos or the wearer's private HUD).
- **US-3:** I can provide an image-based light so lit materials look right on camera.
- **US-4:** I can turn spectator mode on only in debug or demo builds, so a production build never listens on the network unless I want it to.
- **US-5:** I can check a compatibility matrix to know which RealityKit features render correctly.

**Operator (iPhone)**
- **US-6:** I open the iPhone app and it finds nearby Vision Pros running spectator-enabled apps. I pick one and confirm a short pairing code. No IP address needed.
- **US-7:** A guided step calibrates the camera in under 30 seconds and tells me when calibration is good.
- **US-8:** I can record the composited video to Photos, and optionally the raw camera and virtual layers separately for editing.
- **US-9:** If the connection drops, the app reconnects on its own and keeps the calibration.
- **US-10:** I can send the composite to a TV (AirPlay or HDMI) for a live audience.

**Wearer**
- **US-11:** I can see that a spectator is connected (an unobtrusive indicator) and stop it at any time.

## 6. Requirements

Priority: **P0** = required for 1.0, **P1** = should have, **P2** = later.

### 6.1 Functional

| ID | Requirement | Priority | Milestone |
|---|---|---|---|
| FR-1 | Vision Pro advertises via Bonjour; iPhone discovers and lists available hosts | P0 | M1 |
| FR-2 | Pairing with a short code shown on the Vision Pro; transport encrypted (QUIC/TLS) | P0 | M1 |
| FR-3 | Automatic reconnection without recalibration (within a session) | P0 | M1 |
| FR-4 | Render only the content the developer tagged, **without reparenting or modifying live entities** | P0 | M1 |
| FR-5 | Frame-matched compositing: each rendered frame matches the exact camera frame it was rendered for | P0 | M2 |
| FR-6 | Color plus alpha video (HEVC with alpha), with H.264 fallback | P0 | M2 |
| FR-7 | Guided calibration with quality feedback; supports the on-screen marker and a printed marker | P0 | M2 |
| FR-8 | In-app recording of the composite to Photos | P0 | M2 |
| FR-9 | Developer-provided lighting (IBL) for the spectator render | P1 | M2 |
| FR-10 | One-line SwiftUI API (`.mixedRealityCapture()`) plus a tag component | P0 | M3 |
| FR-11 | iPhone app distributed via TestFlight / App Store | P0 | M3 |
| FR-12 | Spectator indicator and stop control for the wearer | P1 | M3 |
| FR-13 | Layered recording (camera, virtual, alpha) for post-production | P1 | M3 |
| FR-14 | UI panels: SwiftUI views described as data and re-drawn on the iPhone | P1 | M4 |
| FR-15 | Metal / CompositorServices integration (app renders into provided textures) | P1 | M4 |
| FR-16 | Per-pixel depth compositing (rendered depth compared with ARKit person/LiDAR depth) | P1 | M4 |
| FR-17 | Live output to an external display / AirPlay | P2 | M4 |
| FR-18 | Object/marker-based calibration with iOS 27 / visionOS 27 reference objects | P2 | M4 |

### 6.2 Non-functional

| ID | Requirement | Target |
|---|---|---|
| NFR-1 | Main-thread cost on the Vision Pro | ≤ 1 ms per frame on average; no blocking GPU waits on the main thread |
| NFR-2 | Host app frame rate | No additional dropped frames at default settings in the reference demos (measured with RealityKit Trace) |
| NFR-3 | Thermal behaviour | Spectator resolution/frame rate drops automatically at `.serious`; pauses at `.critical` |
| NFR-4 | Output quality | 1080p30 default, 1080p60 / 4K30 camera recording optional |
| NFR-5 | Registration accuracy | Virtual content within about 2 cm of its true position at 2 m after calibration (target, to be validated) |
| NFR-6 | Preview latency | ≤ 250 ms glass-to-glass on a good 5 GHz network; recordings have zero relative lag |
| NFR-7 | Reliability | 30-minute session with no crash or leak; reconnect within 5 s after a network drop |
| NFR-8 | Privacy & security | Off unless enabled; pairing required; no unauthenticated listener; respects `contentCaptureProtected` |
| NFR-9 | Compatibility | visionOS 26+ / iOS 26+ (proposed floor); visionOS 27 / iOS 27 features enabled when available |
| NFR-10 | Code quality | Swift 6 strict concurrency; CI builds on every PR; unit tests for protocol, math, and frame matching |

## 7. UX flows

### Developer integration (target)

```swift
import MixedRealityCapture

RealityView { content in
    content.add(scene)
    scene.components.set(SpectatorVisibleComponent())   // tag what appears on camera
}
.mixedRealityCapture(
    enabled: AppConfig.isDemoBuild,
    lighting: .environment(named: "Studio")             // optional IBL
)
```

### Operator flow (iPhone)

1. Open the app. The **Nearby** list shows "MiniBrush on Studio Vision Pro".
2. Tap the host. Enter the 4-digit code shown in the headset.
3. Calibration: "Point your iPhone's screen at the Vision Pro" (or "Place the printed marker in view of both devices"). A progress ring fills as samples are collected, then shows "Calibrated ✓ (accuracy: good)".
4. The live composite appears. Controls: Record, Layers, Quality, Delay (advanced), Recalibrate, Disconnect.
5. Recordings save to Photos. Layered recordings save to Files.

## 8. Success metrics

| Metric | Target (12 months after 1.0) |
|---|---|
| Time to first composite (new developer, published iPhone app) | ≤ 10 min (measured in usability tests) |
| Lines of code to integrate | ≤ 10 |
| GitHub stars / forks | 500 / 50 |
| Apps known to ship with spectator mode | 10 |
| iPhone app installs | 2,000 |
| Calibration success on first try | ≥ 80 % |
| Crash-free sessions (iPhone app) | ≥ 99.5 % |

## 9. Constraints and assumptions

- Apple's restrictions: no capture of system UI; ARKit tracking needs a Full Space; `RealityRenderer` has no depth output and only takes a symmetric projection; no shared coordinate space between iPhone and Vision Pro. See [Research](RESEARCH.md).
- Development needs a recent Xcode, and therefore a recent macOS.
- Wi-Fi quality varies widely. The design must degrade gracefully (lower bitrate and resolution, larger delay buffer).
- The Vision Pro's thermal and GPU budget is tight. Apple's own High Quality capture lowers render quality while it runs.

## 10. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Extra render hurts host app performance | High: blocks adoption | Mirror scene + async pipeline, adaptive quality, thermal backoff, measured budgets |
| Apple ships a first-party spectator feature | Medium | Stay focused on what Apple doesn't do (third-person, open, customizable); be ready to adopt any new API |
| Calibration drift between the two tracking systems | Medium | Periodic re-calibration prompts, printed marker / object anchors, quality indicator |
| Upstream author inactive or doesn't want the changes | Low | Maintain as a credited fork; keep PRs small and offer them upstream |
| App Store review of an app that receives streams from another device | Low/Medium | Clear purpose string, local-network-only, pairing |

## 11. Open questions

1. **Minimum OS:** visionOS/iOS 26 floor (recommended: new networking and coordinate APIs), or support visionOS 2?
2. **Name and branding:** keep "Reality Mixer Pro" (with the author's blessing) or rename the fork?
3. **iPhone app publisher:** which developer account publishes the App Store app?
4. **Monetization:** fully free, or a free core with paid "pro" recording features?
5. **Full-scene replication on the iPhone** (HoloLens-style state sync): pursue later for apps with simple scenes, or never?

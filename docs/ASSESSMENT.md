# Assessment of the upstream prototype

| | |
|---|---|
| **Reviewed** | [fabio914/RealityMixerVisionPro](https://github.com/fabio914/RealityMixerVisionPro) at commit `7fac251` (last upstream push 2024-10-02) |
| **Date** | 2026-10-09 |
| **Method** | Full source read of the Swift package, the iPhone app, and the examples. **Not built or run** for this review, because the review machine's Xcode was too old. Findings are from code reading. |

## 1. Summary

The prototype proves the approach end to end. The iPhone sends its camera pose, the Vision Pro renders RealityKit content from that pose, and the iPhone composites the result with person segmentation. Calibration uses an image shown on the iPhone screen. The author calls it "a very early prototype", and that's accurate:

- **Good:** a small, readable codebase (about 600 lines in the library); a clever calibration trick; a working FG/BG layering idea; three convincing demos; MIT license.
- **Not production-ready:** main-thread GPU stalls, live-entity reparenting, video bitstream bugs, a busy-loop network thread, no frame synchronization, manual IP entry, no reconnection, no tests or CI, no releases.

Upstream activity: 52 stars, 3 forks, 6 open issues, all filed by the author on 2024-10-02 (#4 reconnection, #5 iOS UI, #6 full-body mode, #7 lights, #8 Swift 6, #9 Bonjour).

## 2. How it works today

| Step | Implementation |
|---|---|
| Discovery | Manual IP entry on the iPhone; TCP port 13370 ([`MixedRealityServer.swift`](../Sources/Receiver/MixedRealityServer.swift)) |
| Calibration | iPhone shows `reference` image; Vision Pro `ImageTrackingProvider` finds it; `cameraToWorld = imageAnchor · flip(−90° X) · cameraPose⁻¹` ([`MixedRealityCaptureManager.swift`](../Sources/Manager/MixedRealityCaptureManager.swift#L144-L155)) |
| Pose stream | iPhone sends position, quaternion, image size ×0.5, and vertical FOV for every `ARFrame` |
| Rendering | `RealityRenderer` + `PerspectiveCamera`. Two passes split at the headset's distance (FG: near→headset, BG: headset→far), then a compute kernel extracts alpha ([`MixedRealityRenderer.swift`](../Sources/Renderer/MixedRealityRenderer.swift)) |
| Video | One 2W×2H BGRA frame `[FG | BG ; FG-alpha | BG-alpha]`, H.264 Main, 5 Mbps default, 30 fps default ([`VideoEncoder.swift`](../Sources/Video/VideoEncoder.swift)) |
| Compositing | SceneKit: FG plane at 0.01 m and BG plane at 100 m in front of the AR camera, with person segmentation in between; alpha threshold 0.25 |

## 3. Architecture issues

1. **Live entities are reparented every frame.** The renderer moves the children of `referenceEntity` into the offscreen `RealityRenderer` and puts them back in a `defer` block ([L141–L144](../Sources/Renderer/MixedRealityRenderer.swift#L141-L144), [L135–L138](../Sources/Renderer/MixedRealityRenderer.swift#L135-L138)). This can disturb physics, collisions, input targeting, and anchoring, and it forces render-server re-commits. Apple's support engineers recommend cloning instead ([forum 773957](https://developer.apple.com/forums/thread/773957)).
2. **The children are rendered as if the parent sat at the world origin.** The world transform of `referenceEntity` is ignored, so content under an anchor or an offset parent renders in the wrong place.
3. **Everything runs on the main thread with blocking GPU waits.** Four command buffers per frame each call `waitUntilCompleted()` ([L218](../Sources/Renderer/MixedRealityRenderer.swift#L218), [L279](../Sources/Renderer/MixedRealityRenderer.swift#L279), [L315](../Sources/Renderer/MixedRealityRenderer.swift#L315), [L351](../Sources/Renderer/MixedRealityRenderer.swift#L351)). After that comes a GPU→CPU `getBytes` of about 11 MB per frame ([VideoEncoder L213](../Sources/Video/VideoEncoder.swift#L213)), all from a `CADisplayLink` on the main run loop.
4. **No frame synchronization.** The iPhone composites the *latest* decoded frame over the *current* camera image. Virtual content therefore lags and slides when the phone moves.
5. **Simplified camera model.** Only the vertical FOV is sent. The principal point is ignored, and the screen-to-lens offset of the on-screen calibration image is ignored (see the TODO at [L148–L151](../Sources/Renderer/MixedRealityRenderer.swift#L148-L151)). Expect centimetre-level misregistration.
6. **Lighting is hard-coded.** `RealityRenderer` uses its default IBL with `intensityExponent = 13` ([L121](../Sources/Renderer/MixedRealityRenderer.swift#L121)). The demos use `UnlitMaterial`; lit materials won't match the headset (issue #7).
7. **Single client, no reconnection.** The server accepts once and never again (issue #4).
8. **Unauthenticated, unencrypted listener** on all interfaces.

## 4. Bugs

| # | Severity | Location | Bug | Effect |
|---|---|---|---|---|
| B1 | High | [`VideoEncoder.swift` L116–L119](../Sources/Video/VideoEncoder.swift#L116-L119) | Keyframe detection is inverted. The presence of `kCMSampleAttachmentKey_NotSync` means the frame is **not** a keyframe. | SPS/PPS are written before non-keyframes and omitted before keyframes. The decoder can only start after it has seen a parameter set *and* a later IDR, which delays startup and makes recovery after loss fragile. |
| B2 | High | [`MixedRealityServer.swift` L49–L50](../Sources/Receiver/MixedRealityServer.swift#L49-L50) | Busy loop: `while !cancelled { while read(32, timeout: 0) … }` with no blocking or sleep. | One CPU core at 100 % for as long as a client is connected, which costs battery and heat on the headset. |
| B3 | Medium | [`MixedRealityCaptureManager.swift` L174–L180](../Sources/Manager/MixedRealityCaptureManager.swift#L174-L180) | `cameraExtrinsic` is assigned, then checked for `nil`, so the check is never true. | If the marker is seen before the first camera update, calibration doesn't apply until the marker is seen again. |
| B4 | Medium | [`VideoEncoder.swift` L211–L213](../Sources/Video/VideoEncoder.swift#L211-L213) | One `CVPixelBuffer` is reused while the previous asynchronous encode may still read it (FIXME in the code). | Torn or corrupted frames under load. |
| B5 | Low | [`MixedRealityCaptureManager.swift` L116](../Sources/Manager/MixedRealityCaptureManager.swift#L116) | Presentation timestamp is always `0`. | The rate controller gets no timing information; output can't be recorded with correct timing. |
| B6 | Medium | `MixedRealityServer` `client` property | Written on the server thread, read on `serverSerialQueue` and main, with no synchronization. | Data race (it will surface under Swift 6 strict concurrency; issue #8). |
| B7 | Low | [`ReceivedPayload.swift` L35](../Sources/Receiver/ReceivedPayload.swift#L35), [`ReceivedFrame.swift` L28](../Sources/Model/ReceivedFrame.swift#L28) | `load(as:)` on raw `Data` assumes alignment and struct layout. | Undefined behaviour if alignment differs; fragile protocol with no versioning. |
| B8 | Low | `MixedRealityCaptureManager` | The encoder and renderer are created once, with the first image size. | A resolution change on the iPhone would desynchronise the stream layout. |
| B9 | Low | [`InitialViewController.swift` L85](../iPhoneProject/RealityMixerVisionPro/RealityMixerVisionPro/Initial/InitialViewController.swift#L85) | `connect(timeout: 10)` on the main thread (FIXME in the code). | The UI freezes for up to 10 s on a bad address. |
| B10 | Low | `Package.swift` | Declares `.visionOS(.v1)`. The library itself works on visionOS 1 (`RealityRenderer` is 1.0+), but the README requires visionOS 2 and the MiniBrush example uses visionOS 2 APIs (`LowLevelMesh`). | Confusing requirements. |

## 5. Performance

Estimated per spectator frame on the Vision Pro, at 30 fps:
- 2× `RealityRenderer` passes;
- 2× alpha-extraction compute passes;
- 4× blits;
- 4× blocking waits on the main thread;
- 1× 1920×1440 BGRA GPU→CPU copy;
- H.264 encode;
- plus a busy-looping network thread.

The result is likely main-thread hitches in the host app and avoidable heat. Target design: async, zero-copy, FG/BG side by side with HEVC alpha. See the [TDD §7–§10](TDD.md#7-rendering-on-the-vision-pro).

## 6. Compatibility

| Area | Status |
|---|---|
| RealityKit content in an `ImmersiveSpace` (Full Space) | ✅ |
| Shared Space / windows / volumes | ❌ (ARKit tracking requires a Full Space) |
| SwiftUI windows, attachments, ornaments | ❌ (rendered out of process; not capturable) |
| Lit materials | ⚠️ default IBL only |
| Anchored content (hands, planes) | ❌/⚠️ (parent world transform ignored) |
| Metal / CompositorServices apps | ❌ |
| Swift 6 strict concurrency | ❌ (issue #8) |
| Toolchain | Xcode 16+ (README); not verified on Xcode 26/27 |

## 7. iPhone app

- UIKit + XIBs + `ARSCNView`. SceneKit is in maintenance mode, so the app should move to SwiftUI with Metal compositing.
- Manual IP and port entry; the address is saved in preferences.
- Calibration UX: a full-screen marker that you hide manually; no feedback on calibration quality.
- No recording (relies on iOS Screen Recording); no landscape handling beyond the defaults; disconnects whenever the app resigns active.
- Uses a 0.25 alpha threshold to hide compression artifacts, so semi-transparent content is lost. Its own FIXME notes semi-transparent content blends wrongly with the segmented person.

## 8. Recommendation

Treat the upstream code as a **working reference design**, not a base to keep extending:

1. Send B1–B5 upstream as small, separate fixes. They are low-risk and help existing users.
2. Rebuild the library along the [TDD](TDD.md): mirror scene, async zero-copy pipeline, QUIC transport, frame matching, and a protocol-based render source.
3. Keep the parts that work: the calibration concept, FG/BG layering (until depth is available), and the demos.

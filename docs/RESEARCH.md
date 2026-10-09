# Research: Spectator Mode on Apple Vision Pro

| | |
|---|---|
| **Compiled** | 2026-10-09 (visionOS 27 / iOS 27 released September 2026) |
| **Sources** | Apple documentation, WWDC sessions (2019–2026), Apple Developer Forums, technotes, prior-art repositories, and vendor docs |
| **Confidence** | ✅ confirmed in a primary source · 🟡 reported by a forum post or secondary source · 💭 our own inference |

Some Apple forum pages returned HTTP 403 during research, so a few forum findings come from search summaries and are marked 🟡. Verify the details before relying on them.

---

## Key findings

1. **No visionOS API exposes system-rendered UI pixels** (windows, volumes, attachments) from any viewpoint other than the wearer's. ✅ However, **the world pose and size of windows and attachments *can* be read** while an immersive space is open. ✅ So UI can be *re-drawn* on the spectator side.
2. **iPhone and Vision Pro cannot share a coordinate space.** An Apple engineer confirmed this in June 2026. The suggested alternatives are image markers on both devices, or iOS 27 object tracking with the same reference objects as visionOS. ✅
3. **`RealityRenderer` (visionOS 1.0+) works for offscreen rendering, but has no depth output, supports only symmetric projections, and can't share entities with the app's scene.** Apple support recommends cloning. ✅
4. **visionOS 27's `LowLevelRenderer`** renders into your command buffer with a depth attachment and off-axis projections, but it draws mesh instances, not entities. ✅
5. **Apple's own High Quality capture lowers on-device render quality.** Any extra spectator render must be budgeted carefully. ✅
6. **Network.framework's new Swift API (iOS/visionOS 26) and Apple's iPad↔visionOS local-network sample** are a ready template for discovery, QUIC, and TLS. Multipeer Connectivity is deprecated. ✅
7. **HEVC with alpha** is supported on visionOS. Alpha is an 8-bit auxiliary layer, and low-latency rate control now works with HEVC. ✅ Whether low latency and alpha work together is unknown. 💭
8. **Earlier spectator systems** (HoloLens Spectator View, Meta Quest MRC, LIV) all ended up with latency compensation and calibration rigs. They lost developer adoption whenever integration became ongoing work for every app. ✅

---

## 1. System UI: can it be captured?

### Pixels: no

- **ReplayKit** (`RPScreenRecorder`) is documented for visionOS. In-app capture reportedly returns only the app's own view on a black background. Apple engineers point to `RPSystemBroadcastPickerView` plus a Broadcast Upload Extension, which delivers the *wearer's* composited view. ✅ [forum 757738](https://developer.apple.com/forums/thread/757738)
- **`ImageRenderer`** only draws views that SwiftUI rasterizes itself (text, images, shapes). Views composited by Core Animation (complex controls, web views, media, most UIKit) come out as placeholders. ✅ [docs](https://developer.apple.com/documentation/swiftui/imagerenderer) Glass and materials reportedly render flat. 🟡
- **`RealityRenderer` output** excludes passthrough and content in other windows (Apple DTS). ✅ [forum 762238](https://developer.apple.com/forums/thread/762238)
- **No API** reads back attachment textures. `ViewAttachmentComponent` exposes only `bounds`, `id`, and `init(rootView:)` (the initializer is new in visionOS 26). ✅ [docs](https://developer.apple.com/documentation/realitykit/viewattachmentcomponent)
- **iOS RealityKit has no attachments.** `ViewAttachmentComponent`, `PresentationComponent`, and `RealityViewAttachments` are visionOS-only. An Apple engineer suggested `ImageRenderer` textures as a workaround. ✅ [forum 756579](https://developer.apple.com/forums/thread/756579)

### Poses: yes

- **SwiftUI.**
  - `GeometryProxy3D.transform(in:)` (visionOS 1.0) and the `.immersiveSpace` coordinate space (visionOS 1.1) give a view's full transform in immersive-space coordinates. Apple's own documentation example uses `proxy.transform(in: .immersiveSpace)`.
  - `onGeometryChange3D` (visionOS 2.0) notifies you when it changes. ✅ [docs](https://developer.apple.com/documentation/swiftui/coordinatespaceprotocol/immersivespace)
- **visionOS 26 unified coordinate conversion.**
  - `GeometryProxy3D.coordinateSpace3D()` together with the Spatial framework's `CoordinateSpace3D.convert(value:to: .worldReference)`.
  - RealityKit `Entity` and `Scene` conform to `CoordinateSpace3D`. ✅ [WWDC25-274](https://developer.apple.com/videos/play/wwdc2025/274/)
  - A forum example shows it updating when a window moves and on recenter. ✅ [forum 799630](https://developer.apple.com/forums/thread/799630)
- **Attachments are entities.** `transformMatrix(relativeTo: nil)` or `relativeTo: .immersiveSpace` (visionOS 2.0) returns their pose. ✅
- **Window scale.** Windows scale dynamically with distance by default (`defaultWorldScaling`, `WorldScalingBehavior`, visionOS 2), so the spectator replica must apply the same scale. ✅
- **Popovers.** `PresentationComponent` (visionOS 26) popovers face the *wearer*, so replicas should billboard toward the headset, not the iPhone. 💭
- **Ornaments and toolbars** sit outside the window's geometry; their offsets have to be modelled. 💭

## 2. Rendering the spectator view on the Vision Pro

### RealityRenderer

- **Availability.** visionOS 1.0+ (also iOS 18, macOS 15, tvOS 26). ✅ [docs](https://developer.apple.com/documentation/realitykit/realityrenderer)
- **Output.** `CameraOutput.Descriptor.singleProjection(colorTexture:)` gives **colour only: no depth and no stereo**. ✅ A developer combining it with MetalSplatter reports having no depth to work with. 🟡 [forum 761846](https://developer.apple.com/forums/thread/761846)
- **Settings.** `colorBackground`, `antialiasing` (`.multisample4X`, visionOS 2.0+), `isToneMappingEnabled` (needs the 27 SDK), and `lighting: ImageBasedLight`. Lit content needs an IBL `resource`. ✅ [forum 770404](https://developer.apple.com/forums/thread/770404)
- **Projection.** `PerspectiveCamera` takes only FOV, near, and far. `ProjectiveTransformCameraComponent` (visionOS 2.0) requires *symmetric* perspective or orthographic matrices. ✅ It's unconfirmed whether `RealityRenderer` honours that component. 💭
- **Entities can't be shared** between the app scene and a `RealityRenderer`, since an entity has one parent. Apple DTS recommends cloning or loading a second copy, and there is no built-in sync. ✅ [forum 773957](https://developer.apple.com/forums/thread/773957)
- **Clones** copy component data but share meshes and materials by reference. Each clone still costs a draw call and ECS work. ✅/🟡
- **Known issues.**
  - visionOS 1.1: custom `System`s aren't ticked every frame in a `RealityRenderer` scene; the workaround is `SceneEvents.Update`. ✅ [release notes](https://developer.apple.com/documentation/visionos-release-notes/visionos-1_1-release-notes)
  - visionOS 27 fixed `OpacityComponent` making opaque materials transparent. ✅ [release notes](https://developer.apple.com/documentation/visionos-release-notes/visionos-27-release-notes)
- **Where it runs.** RealityView content is rendered out of process. `RealityRenderer` encodes into the app's own command buffers, so its CPU cost counts against the app. 💭 [render pipeline](https://developer.apple.com/documentation/visionos/understanding-the-visionos-render-pipeline)

### New APIs

- **visionOS 26:** `MeshInstancesComponent`, `ViewAttachmentComponent(rootView:)`, `PresentationComponent`, observable entities. `RealityRenderer` gained nothing beyond tvOS support. `customPostProcessing` is not on visionOS. ✅ [WWDC25-287](https://developer.apple.com/videos/play/wwdc2025/287/)
- **visionOS 27: `LowLevelRenderer`.**
  - Encodes draws into your command buffer.
  - `Output(color:depth:)` gives a **depth attachment**.
  - Projections include **off-axis** `.perspective(leftTangent:rightTangent:topTangent:bottomTangent:…)` and `.custom(matrix:)`.
  - Supports IBL and ShaderGraph materials through a standalone render context.
  - Draws `LowLevelMeshInstance` arrays, *not entities*. ✅ [docs](https://developer.apple.com/documentation/realitykit/low-level-rendering)
- **visionOS 27:** `LowLevelTexture(deviceResource:)` with IOSurface or shared handles enables zero-copy textures. ✅

### CompositorServices (Metal apps)

- **Extra pass.** An extra offscreen pass is plain Metal: arbitrary view/projection, colour, alpha, and depth. 💭
- **Capture drawable.** visionOS 26 `LayerRenderer.Drawable.Target` includes **`.capture`**, a drawable the system uses for video and AirPlay capture. This is Apple's own precedent for an extra render, though from the wearer's pose. ✅ [docs](https://developer.apple.com/documentation/compositorservices/layerrenderer/drawable/target-swift.enum)
- **Frame pacing.** Displays usually run at 90 Hz. Apple advises lowering `renderQuality` before raising the frame-repeat count. ✅ [WWDC25-294](https://developer.apple.com/videos/play/wwdc2025/294/)
- **macOS 26 `RemoteImmersiveSpace`** (a Mac renders for the headset) supports full or progressive immersion only. It has no passthrough, no RealityKit, and no app data channel, so it isn't viable for mixed-reality spectating. ✅ [docs](https://developer.apple.com/documentation/swiftui/remoteimmersivespace)

### Engines

- **Unity PolySpatial** renders secondary cameras with Unity's renderer and blits them into a RealityKit `DrawableQueue` texture. ✅ [docs](https://docs.unity3d.com/Packages/com.unity.polyspatial.visionos@3.0/manual/RenderTextures.html) Unity's Metal mode has a `HighQualityRecordingMode`. 🟡
- **No shipping Vision Pro title** with third-person mixed-reality capture was found. This project's upstream is the only public implementation. ✅

### Performance

- **Apple's budgets:** ≤ 250 draw calls and triangles in the Shared Space, ≤ 500 in a Full Space. Profile with RealityKit Trace and Metal System Trace. ✅ [docs](https://developer.apple.com/documentation/visionos/reducing-the-rendering-cost-of-realitykit-content-on-visionos)
- **High Quality Record My View** (unfoveated 4K30 from one eye) **reduces render quality while active** and is limited to 3 minutes. ✅ [Apple support](https://support.apple.com/guide/apple-vision-pro/take-a-capture-or-recording-of-your-view-tan1527c9e00/visionos)

## 3. Video and depth

- **`kCMVideoCodecType_HEVCWithAlpha`** is available on visionOS 1.0+. ✅ [Apple interop profile (PDF)](https://developer.apple.com/av-foundation/HEVC-Video-with-Alpha-Interoperability-Profile.pdf)
  - Alpha is an HEVC Annex F auxiliary layer, **8-bit**, with both layers in Main 4:2:0 at the same size.
  - Premultiplied alpha is recommended.
  - `kVTCompressionPropertyKey_TargetQualityForAlpha` (0–1) controls alpha quality independently of bitrate.
- **Low-latency rate control** (`kVTVideoEncoderSpecification_EnableLowLatencyRateControl`, iOS 14.5 / visionOS 1.0) uses an infinite GOP with no reordering. ✅ [WWDC21-10158](https://developer.apple.com/videos/play/wwdc2021/10158/)
  - Apple's 2025 low-latency conferencing sample pairs it with **HEVC**. ✅ [sample](https://developer.apple.com/documentation/videotoolbox/encoding-video-for-low-latency-conferencing)
  - iOS/visionOS 26 add `kVTCompressionPreset_VideoConferencing`. ✅
  - DTS: VBR isn't compatible with low-latency mode, and CBR isn't supported in every mode. ✅ [forum 799459](https://developer.apple.com/forums/thread/799459)
- **Zero-copy.** `VTCompressionSessionGetPixelBufferPool` returns encoder-matched buffers that are cheap to request every frame. Render into them through `CVMetalTextureCache`. ✅/💭
- **Depth in video:**
  - Hue colorization (Intel RealSense) gives about 10.5 bits but produces "flying pixels" at depth edges. ✅ [Intel](https://dev.realsenseai.com/?p=41316)
  - Chroma subsampling hurts depth packing, and inverse depth is preferable. ✅ [arXiv 2206.15183](https://arxiv.org/html/2206.15183v1)
  - Our choice: 10-bit inverse depth in HEVC Main10 luma, or lossless LZ4 at about 256×192, which matches the iPhone's own depth resolution. 💭

## 4. Networking

- **`NetworkConnection`, `NetworkListener`, `NetworkBrowser`** (iOS/visionOS 26): Swift async APIs with TLV and Codable framers, QUIC streams, and datagrams. ✅ [docs](https://developer.apple.com/documentation/network/networkconnection)
- **Apple sample:** [Connecting iPadOS and visionOS apps over the local network](https://developer.apple.com/documentation/visionos/connecting-ipados-and-visionos-apps-over-the-local-network) uses QUIC, a self-signed TLS identity with trust on first use, and a Bonjour TXT device ID. ✅
- **TN3213** ([link](https://developer.apple.com/documentation/technotes/tn3213-moving-from-multipeer-connectivity-to-network-framework)): Multipeer Connectivity is deprecated; prefer QUIC; `peerToPeerIncluded` can reduce performance. ✅
- **Local network privacy** (TN3179): `NSLocalNetworkUsageDescription` and `NSBonjourServices` are needed on visionOS as on iOS. ✅ [technote](https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy)
- **AWDL stalls:** a developer measured periodic 30–50 ms stalls from peer-to-peer Wi-Fi when the router wasn't on channel 149. There's no API to force real-time mode. ✅ [forum 819926](https://developer.apple.com/forums/thread/819926)
- **Wi-Fi Aware** is iOS 26+ only; Vision Pro doesn't support it. ✅
- **`com.apple.developer.low-latency-streaming`** (visionOS 26) is meant for apps that *consume* game streams. It's unclear whether it helps a sender. ✅/💭
- **Bandwidth:** NVIDIA CloudXR recommends 100–200 Mbps on 5/6 GHz Wi-Fi. Plan on roughly 50–150 Mbps and a few milliseconds one way, with spikes. 🟡/💭

## 5. iPhone (ARKit) side

- **Person segmentation:** `personSegmentationWithDepth` needs an A12 or later. The ML model runs at lower resolution than the camera. `ARMatteGenerator` gives a full- or half-resolution matte and a dilated depth: the intended path for custom Metal compositing. ✅ [WWDC19-607](https://developer.apple.com/videos/play/wwdc2019/607/), [docs](https://developer.apple.com/documentation/arkit/armattegenerator)
- **LiDAR:** `sceneDepth` runs at 60 Hz at low resolution, with confidence values. ✅ [WWDC20-10611](https://developer.apple.com/videos/play/wwdc2020/10611/)
- **4K:** `recommendedVideoFormatFor4KResolution` gives 3840×2160 at 30 fps. **Don't retain `ARFrame`s**, because ARKit stalls; copy what you need into your own buffers. ✅ [WWDC22-10126](https://developer.apple.com/videos/play/wwdc2022/10126/)
- **Intrinsics** can change from frame to frame, so send them with every pose. ARKit exposes no lens-distortion parameters. 🟡/💭
- **iOS 27 object tracking:** `ARWorldTrackingConfiguration` detects and tracks reference objects using the same `.referenceobject` files as visionOS. ✅ [WWDC26-283](https://developer.apple.com/videos/play/wwdc2026/283/)

## 6. Calibration and synchronization

- **Image tracking on the Vision Pro** updates far below the camera frame rate, about 1 Hz while the image moves. That's fine for a still calibration step, not for live tracking. ✅ [forum 747118](https://developer.apple.com/forums/thread/747118)
- **visionOS 27** `coordinateSpace(correction: .none)` returns metric anchor poses. The default *rendered* correction nudges them to line up with passthrough, so use `.none` for calibration. ✅ [docs](https://developer.apple.com/documentation/arkit/arkitcoordinatespaceproviding)
- **Object tracking** gets a high-frame-rate option per object in visionOS 27. Barcode detection is enterprise-only. ✅
- **`SharedCoordinateSpaceProvider`** (visionOS 26) works between nearby Vision Pros only. ✅ [docs](https://developer.apple.com/documentation/arkit/sharedcoordinatespaceprovider)
- **No API gives the iPhone's screen-to-lens offset.** Apple's dimensional drawings, hand-eye calibration, or a shared printed marker avoid the problem. ✅/💭
- **Clock sync:** echoing frame IDs is enough for compositing. NTP-style sync is only needed for latency stats. 💭

## 7. Prior art

### HoloLens Spectator View (Microsoft), archived 2024

- **Approach:** state synchronization. The headset broadcasts component changes and the phone **re-renders the scene locally**, with no video stream. ✅ [architecture](https://microsoft.github.io/MixedReality-SpectatorView/doc/SpectatorView.Architecture.html)
- **Costs:**
  - Every asset needed a unique ID in "asset caches", regenerated whenever assets changed.
  - Scanning the scene for changes was expensive, and the HoloLens frame rate dropped when a spectator joined.
  - The fixes were opt-in tagging, polling materials once, and flattening hierarchies. ✅ [state sync README](https://github.com/microsoft/MixedReality-SpectatorView/blob/master/src/SpectatorView.Unity/Assets/SpectatorView/Scripts/StateSynchronization/README.md)
- **Calibration:** the phone shows a marker for the headset to detect (QR on HoloLens 2, ArUco on HoloLens 1). The pro rig used a DSLR, a chessboard for lens parameters, and a manual frame-time adjustment. ✅ [setup](https://github.com/microsoft/MixedReality-SpectatorView/blob/master/doc/SpectatorView.Setup.VideoCamera.md)

### Meta Quest Mixed Reality Capture, LIV, Reality Mixer

- **Quest MRC:**
  - The app clones its main camera to a calibrated pose and streams frames to an OBS plugin.
  - FG/BG split at the headset's distance.
  - A `CastingLatency` setting.
  - Advice: hide objects from the capture camera; design HUDs and effects for other aspect ratios. ✅ [Meta blog](https://developers.meta.com/horizon/blog/mixed-reality-capture-on-quest-workflow-integrations-insights/)
- **Decline:** Meta handed MRC to LIV in 2024, after most Quest apps dropped support. ✅ [Road to VR](https://www.roadtovr.com/meta-ditches-mixed-reality-tools-for-liv/)
- **LIV:**
  - Clip geometry at the player plus per-layer depth; timestamp-based latency compensation.
  - Additive-blend materials skip depth tests and break layering. Post-effects corrupt alpha.
  - Typical latency offset is about 160 ms. ✅/🟡 [LIV docs](https://docs.liv.tv/mixed-reality-capture/unreal/tips-and-config)
- **Reality Mixer (Quest):** BG/FG/alpha layout, a 0.25 alpha threshold for H.264 artifacts, `camDelayMs` in calibration, and tips (5 GHz, keep the phone still and powered). ✅ [repo](https://github.com/fabio914/RealityMixer)

### Rendering on the iPhone instead (state sync) on Apple platforms

- **RealityKit sync is out.** `MultipeerConnectivityService` isn't available on visionOS. `SynchronizationComponent` reportedly fails to compile on visionOS. ✅/🟡 [forum 735245](https://developer.apple.com/forums/thread/735245)
- **SharePlay** works between iOS and visionOS, but iOS joins as a FaceTime participant with no spatial alignment. ✅ [WWDC25-318](https://developer.apple.com/videos/play/wwdc2025/318/)
- **Precedent:** Unity PolySpatial "Play to Device" replicates a scene graph into a RealityKit host. ✅
- **Commercial example:** STAGEit Viewer (iPhone/iPad) shows scenes alongside a Vision Pro user; its method isn't disclosed. ✅

## 8. Apple's built-in capture (all first-person)

| Feature | Notes |
|---|---|
| Mirror My View (AirPlay) | Up to 1080p to iPhone, iPad, Mac, or TV; protected content shows black ✅ [support](https://support.apple.com/en-us/119944) |
| Record My View (High Quality) | Unfoveated 4K30 from one eye, 3-minute limit, lowers render quality ✅ |
| visionOS 27 on-device High Quality capture | No Mac needed; 4K mirroring while recording; no app API ✅ [WWDC26-287](https://developer.apple.com/videos/play/wwdc2026/287/) |
| Developer Capture (Reality Composer Pro) | About 4K30 10-bit HEVC, time-limited 🟡 |
| Enterprise: passthrough in screen capture | `com.apple.developer.screen-capture.include-passthrough`; system broadcast to an upload extension; user starts each session; reported unreliable on visionOS 26 ✅ [WWDC24-10139](https://developer.apple.com/videos/play/wwdc2024/10139/), [forum 803274](https://developer.apple.com/forums/thread/803274) |
| `contentCaptureProtected()` | visionOS 26 (entitlement): blanks views in captures; **we must honour it** ✅ [WWDC25-223](https://developer.apple.com/videos/play/wwdc2025/223/) |

## 9. Design decisions this research supports

| Decision | Based on |
|---|---|
| Keep video streaming from the Vision Pro for 3D content | Exact visual match with no per-app replication (§2, §7) |
| Mirror scene via clone + sync, never reparent | Apple DTS recommendation (§2) |
| FG/BG split now, depth later (`LowLevelRenderer` / Metal) | `RealityRenderer` has no depth (§2) |
| Render a wider symmetric frustum + crop | `RealityRenderer` supports symmetric projections only (§2) |
| UI panels as data, re-drawn on the iPhone | No UI pixels, but poses are readable (§1) |
| QUIC via Network.framework, Bonjour, peer-to-peer off | TN3213, Apple sample, AWDL stalls (§4) |
| HEVC with alpha, zero-copy encoding | VideoToolbox support (§3) |
| Frame-ID echo plus ring buffer of copied frames | ARKit frame-retention guidance; prior art's latency sliders (§5, §7) |
| Printed marker / reference objects for calibration | No shared space; 1 Hz image tracking; `.none` correction (§6) |
| Off by default, pairing, honour `contentCaptureProtected` | Privacy and enterprise guidance (§8) |
| Integration must be about one line | Adoption lessons from Spectator View and Quest MRC (§7) |

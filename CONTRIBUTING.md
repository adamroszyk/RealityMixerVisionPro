# Contributing

Thanks for helping make spectator mode on Vision Pro easy for everyone.

## Where to start

- **Pick a task** from the [Roadmap](ROADMAP.md). Tasks are pull-request-sized and have IDs like `T1.2.3`. Mention the ID in your PR title.
- **Device testing is very valuable.** If you have a Vision Pro and an iPhone, try the examples and file a device-test report: device models, OS versions, network setup, what you saw.
- **Open an issue first** for anything larger than a task, or anything that changes the [TDD](docs/TDD.md).

## Ground rules

- **Small PRs.** One task per PR. Bug fixes that also apply upstream should be sent to [fabio914/RealityMixerVisionPro](https://github.com/fabio914/RealityMixerVisionPro) too, as separate, minimal PRs.
- **Tests first for pure logic.** Protocol coding, math, calibration, and frame matching (`MRCCore`) need unit tests (Swift Testing). Compositor changes need golden-image tests.
- **Swift 6 strict concurrency.** No new `@preconcurrency`, unchecked `Sendable`, or semaphores around shared state.
- **Never block the main thread** on the Vision Pro: no `waitUntilCompleted`, no synchronous networking. Spectator mode must not cost the host app frames.
- **Never modify the app's live entities.** Render from the mirror scene.
- **Privacy.** Spectator mode stays off unless the app enables it. Never render content the developer didn't tag, and never report `contentCaptureProtected` views.
- **Performance claims need numbers.** If a PR affects performance, include RealityKit Trace or Metal System Trace results.

## Development setup

- The current Xcode, with the visionOS and iOS SDKs.
- Clone anywhere. *(Until T0.1.1 lands, the examples expect the folder to be named `RealityMixerVisionPro`.)*
- Run the package tests: `swift test` (macOS, `MRCCore` once it exists), or build the schemes in Xcode.

## Code of conduct

Be kind, assume good intent, and keep discussions technical. Harassment of any kind isn't tolerated.

## License

By contributing, you agree that your contributions are licensed under the [MIT License](LICENSE).

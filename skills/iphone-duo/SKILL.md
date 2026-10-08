---
name: iphone-duo
description: Implementation guide for adapting apps to Apple's iPhone Duo, an iPhone with multiple displays and a fold. Covers size class, safe area asymmetry, reserved regions, vertical bars, arrangement, hinge, scene accessories, and dual front cameras. Always use this skill when iPhone Duo, a foldable iPhone, inner/outer displays, vertical toolbars, hinge angle, reserved region, ArrangementView, CameraCaptureAccessory, or AVCaptureDeviceDirectionCoordinator is mentioned. Also consult it for requests such as "adapt my app to iPhone Duo", "my layout breaks when folded", "the bars become vertical", or "switch cameras when opening or closing", and for any iOS discussion of multiple displays, folds, or pose changes even when iPhone Duo is not named.
---

# Adapting apps to iPhone Duo

An implementation guide based on six Apple Tech Talks, the Human Interface Guidelines, Apple Developer Documentation, and two iPhone Duo Group Labs (2026-09-16 and 2026-09-17) as primary sources.

App Store Connect accepts optimized apps starting on 2026-10-05. iPhone Duo screenshots are required for submissions from 2027-04 onward.

## Start here

iPhone Duo launches on 2026-10-23 with iOS 27.1. The SDK with iPhone Duo support and Device Hub are included in Xcode 27.1 (beta released on 2026-09-18, Release Candidate on 2026-10-05).

iPhone Duo has two displays, inner and outer, and opens and closes around a central hinge. **Your app remains an iPhone app**; opening and closing cause a resize. There is no dedicated user interface idiom: it returns phone. The only new aspect is that the inner display has regular size classes on both the horizontal and vertical axes, a first for iPhone.

An app running full screen on the inner display moves to the outer display when the device closes and keeps running. It does not enter the background. Split View is the exception: with two apps side by side, the most recently used app moves to the outer display and the other enters the background. If you reopen immediately, both return; after a delay, the app that was on the outer display fills the inner display.

The foundation is a layout that handles resizing. Existing support for resizing on iPad or through iPhone Mirroring carries over directly.

### Behavior by SDK

| Build SDK | Behavior |
|---|---|
| Before iOS 27 (Xcode 26) | Runs letterboxed. On the outer display, the aspect ratio is close to iPhone mini, with a black strip on the vertical bars side. On the inner display, it is centered at the same aspect ratio and does not respond to pose changes. |
| iOS 27 | Resizing support is enabled (no opt-out). Uses almost the entire inner display, but a black strip remains along the side below the status bar. |
| iOS 27.1 | Extends to the display edges, with standard bars placed vertically below the status bar. |

You do not need to build a tailored experience for every pose from the start. Apple recommends shipping resizing support and following best practices for launch, then improving the experience afterward.

Test with the iPhone Duo simulator in Xcode 27.1's Device Hub. You can change the hinge angle and select closed, open, book, laptop, and tent poses. Check transitions between poses as well as each pose itself. The simulator also returns reserved regions: partially fold it and draw the regions to check custom layouts that avoid the fold or cameras (see "Testing in the simulator" in `references/layout.md`). The simulator cannot reproduce both the inner and outer displays being on at once, because this requires a camera. The iOS 27.1 simulator can now launch apps that use cameras. It shows no video and reports that no camera is available, but you can check the rest of the UI.

You can also use Device Hub's iOS resizable simulator to check resizing support. Run on iPad and resize the window, or, for iPhone-only apps, use iPhone Mirroring on macOS 27 and resize to extreme sizes in both directions. The iPhone Duo simulator is the best way to test the full native experience. iPhone Mirroring uses an aspect ratio close to the inner display and retains the iPhone interface idiom. It reproduces many issues. Problems caused by safe area and margin asymmetry from vertical bars are harder to find without the iPhone Duo simulator.

## Which references to read

Read only the references needed for the task.

| Reference | Coverage |
|---|---|
| `references/layout.md` | size class, safe area asymmetry, display corners, reserved regions, arrangement |
| `references/bars.md` | vertical bars, item ordering, axis control, badges, overflow, sheets, disabling vertical bars |
| `references/scenes.md` | hinge, multitasking, multiple windows, scene accessories, camera accessories, Core Motion, Web |
| `references/camera.md` | dual front cameras, camera direction, previews, rotation |
| `references/checklist.md` | Migration and validation for existing apps, and an App Store checklist |

## General principles

These principles apply across all areas. Keep them in mind to make decisions more quickly.

**Use size class to make decisions.** Do not infer screen size or device capabilities from user interface idiom, or branch layouts on `UIDevice.current.orientation`, `statusBarOrientation`, or `interfaceOrientation` (including `windowScene.effectiveGeometry.interfaceOrientation`). Compare the width and height of bounds to determine whether the layout is tall or wide, and remove device-size comparisons such as `bounds.height == 844`. The inner display does not rotate according to the app's declared supported interface orientations; it scales instead. Checking orientation will not produce the intended result. Even portrait-only apps have regular/regular size classes on the inner display. You can read the current orientation from the environment or traits, but base decisions on available width. Moving away from settings that lock orientation or size, such as `UIRequiresFullScreen`, is recommended.

**Do not rebuild the view hierarchy through branching.** SwiftUI code that uses an `if` on size class with a different container in each branch discards one view hierarchy when the branch changes. State is lost unless it has been lifted to a parent. This already happened during iPad resizing, but on iPhone Duo it surfaces every time the device goes from the inner display to closed. First consider removing the branch itself, for example by using `ArrangementView`.

**Do not reference the screen directly.** `UIScreen.main` is ambiguous on a two-display device and is already deprecated in the documentation. Use the environment, trait collection, and scene bounds. If you need a screen, obtain it from `window?.windowScene?.screen`. Replace `UIWindow(frame: UIScreen.main.bounds)` with `UIWindow(windowScene:)`, and retrieve `UIScreen` dynamically instead of retaining it. Do not read size just once at launch or in `viewIsAppearing`; read it again in `layoutSubviews` / `viewDidLayoutSubviews`, and put size-change handling in `viewWillTransition(to:with:)`. `displayScale` / `traitCollection.displayScale` is sufficient for display scale.

**Check full-bleed media for cropping.** Check whether `.scaleAspectFill` / `.aspectRatio(contentMode: .fill)` crops content in wide layouts. Switch between fill and fit based on size class or aspect ratio, or specify a focal point. Before shipping an app built with the iOS 27.1 SDK, verify that content adapts when bars become vertical.

**The safe area becomes asymmetric.** Code that assumes opposite edges have equal insets will break. Stop using expressions such as `view.bounds.width - insets.left * 2` and handle each edge separately. Layout margins and content insets are also asymmetric. Use the actual system-provided values instead of reusing one edge's value for the opposite edge.

**Standard components handle much of the adaptation automatically.** Standard navigation, bars, sheets, alerts, and menus adapt to poses and the fold. The system also handles fold avoidance. Every custom replacement increases the work you must handle yourself.

**Start with higher-level APIs.** Fold-related APIs form layers: hinge angle → reserved regions → arrangement view → system components. Before calculating a layout from the hinge angle yourself, consider whether a higher layer is sufficient.

**This is a phone.** An existing iPad design is a good starting point, but do not bring it over unchanged. Remove iPad-only elements or separate them by idiom. Likewise, do not build an app that works only on iPhone Duo or assumes a single pose. Users treat it as one model in the product line. The first step in bringing an iPad app over is to make it universal and add iPhone as a supported platform (there has been no clear answer on whether a compatibility mode like Apple Vision Pro's exists).

**Treat it as one device.** Design opening as a resize that widens the app window. Users hold and place the device in many ways, so do not assume a particular pose. A tabletop pose should offer the same experience with either the camera side or the outer display facing down. If you change the UI by pose, consider whether you can animate transitions smoothly.

## About the iOS 27.1 APIs

As of 2026-09-18, documentation for arrangement, reserved regions, hinge (UIKit), vertical bar controls, the camera direction coordinator, and other APIs has been confirmed to be published as **iOS 27.1+ Beta**. These APIs may change before the final release.

As of 2026-09-18, declarations for every API covered by this guide have been confirmed in the documentation.

If you continue supporting older versions such as iOS 16, wrap iPhone Duo-specific code in availability checks. The scope depends on the app: you can build a separate new UI and freeze the old one, or use finer-grained conditions to keep offering the same features on all devices.

**When implementing, verify with Xcode completion and on a device or simulator.**

## How to respond

Write explanations, proposals, and validation results in the user's language. Preserve the original spelling of API names, type names, identifiers, and source titles.

Because this guide deals with unpublished APIs, **do not invent signatures you cannot verify.** If you cannot confirm them with Xcode completion or on a device or simulator, state that explicitly when proposing an implementation. A plausible but incorrect spelling is more harmful than saying you do not know.

## Sources

- [Three steps to make your app shine on iPhone Duo](https://developer.apple.com/iphone-duo/prepare/)
- [Prepare and submit your apps for iPhone Duo (News)](https://developer.apple.com/news/?id=kkphp5qo)
- [A Summary of the iPhone Duo Group Lab (Apple Developer Forums)](https://developer.apple.com/forums/thread/847644)

- Tech Talks: Prepare your app for iPhone Duo / Raise the bar with iPhone Duo / Strike a pose with adaptive layouts on iPhone Duo / Leverage multiple displays and scenes on iPhone Duo / Build a great camera experience for iPhone Duo / Design for iPhone Duo
- Human Interface Guidelines: Designing for iPhone Duo
- Meet with Apple: iPhone Duo Group Lab (2026-09-16 / 2026-09-17)
- Apple Developer Documentation: Preparing your app for iPhone Duo / Registering a camera capture accessory on iPhone Duo / Choosing a camera by the direction it faces / Supporting device rotation in your camera app / iOS 27.1 Beta API reference

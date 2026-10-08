# Scenes and the hinge

## Hinge

Apps can read hinge status and angle. There are three statuses: **closed, partially folded, and fully open**, with continuous angle updates alongside them.

```swift
// SwiftUI: onHingeChange(isEnabled:_:) (iOS 27.1+ Beta)
// Declaration: (DeviceHingeContext, DeviceHingeContext) -> Void. isEnabled has a default value and can be omitted
someView
    .onHingeChange { _, context in
        // A nil hinge in context means the device has no hinge
        if let hinge = context.hinge, hinge.status == .partiallyOpen {
            value = compute(from: hinge.angle)   // angle is an Angle
        } else {
            value = 0
        }
    }
```

UIKit provides `UIHingeInteraction`.

The closure receives the `DeviceHingeContext` before and after the change. `DeviceHingeContext` contains only `hinge` (`DeviceHinge?`). `DeviceHinge` has `angle` and `status`, and `DeviceHinge.Status` has three values: `closed` / `partiallyOpen` / `fullyOpen`. Always check for `nil`; omitting the check causes unintended behavior on devices without a hinge.

**Keep the use cases separate.** Live hinge data is for interactions and effects. Use arrangement and reserved regions APIs for layout decisions. Building a layout directly from the hinge angle duplicates the adaptation provided by the system.

## Multitasking

Every app participates in multitasking on iPhone Duo. Two apps can appear side by side, or a video and an app can be stacked vertically, but **both are the same from the app's perspective**. Just adapt to the size you are given.

Split View and pinning Picture in Picture to the top are system-provided interactions. There are no APIs for implementing them. Make decisions using size class and scene geometry.

If your app already supports resizing on iPad or in iPhone Mirroring, you have a head start.

The Group Lab assessment was that external displays would probably behave as on a regular iPhone (connecting and running has been confirmed, but this is not a comprehensive validation of the behavior). Multitasking like Stage Manager on iPad is not expected to be supported.

There is no API to specify a window scene's size directly. Users also cannot freely drag to resize as on iPad; size is determined by pose and Split View.

Picture in Picture can be pinned to the top of the display when fully open in landscape, vertically resizing the app into the remaining space. When partially folded, the video expands to half the display. It moves to the outer display only when the device closes, and the system controls that movement. There is no mechanism to pin audio-only content like PiP.

## Multiple windows

iPhone Duo is the first iPhone that can show multiple instances of an app's UI. Apps that support this on iPad also support it on iPhone Duo. The mechanism is the same as on iPad: instances of the same app can appear beside each other or beside other apps. State is shared between instances when they reference storage such as `@AppStorage`.

**New windows can be created only on the inner display**, not the outer display. The dynamic change in this capability is specific to iPhone Duo.

The Group Lab assessment was that audio would not be separated by scene (this was not a definitive answer). Audio control is a separate layer from the UI, and Control Center will not show two volume sliders. To separate audio, you would need to mix it yourself. You can open two scenes of the same app on iPad too, so check the behavior there.

```swift
// Automatically hides when creation is unavailable
UIWindowScene.ActivationAction   // UIKit

// SwiftUI: Use openWindow and supportsMultipleWindows to determine availability
@Environment(\.supportsMultipleWindows) private var supportsMultipleWindows
```

Handle errors in case a request fails. In UIKit, request activation with `UIApplication.activateSceneSession(for:errorHandler:)` (iOS 17.0+) and use `UISceneError.Code` to identify the reason. `.requestDenied` indicates a situation where creation is unavailable, such as on the outer display. `.multipleScenesNotSupported` indicates that the app does not support multiple scenes.

## scene accessories

Scene accessories display content accompanying the main UI on another display at the same time. They are an iPhone and iPad feature, not exclusive to iPhone Duo. One use case is showing a game on an external display while using the iPhone as a controller.

The system manages availability dynamically. It is enabled by default but can change at any time, so **design to respond to changes**.

```swift
// SwiftUI (iOS 27.0+)
CameraView(model: model)
    .sceneAccessory {
        CameraCaptureAccessory(isEnabled: $model.isEnabled) {
            TeleprompterView(model: model)
        }
        .onAvailabilityChange { newValue in
            model.isAvailable = newValue
        }
    }
```

Content passed to `sceneAccessory(content:)` must conform to `SceneAccessoryContent`.

- `CameraCaptureAccessory` (iOS 27.1+ Beta) — Displays additional UI on the outer display while using the camera. See the next section
- `ExternalNonInteractiveAccessory` — Displays noninteractive content on an external display

UIKit provides `UISceneAccessory`; check availability with `UISceneAccessoryRegistration.isAvailable`. `sceneAccessory(content:)`, `UISceneAccessory`, `UISceneAccessoryRegistration`, `registerSceneAccessory(_:)`, and `onAvailabilityChange(perform:)` are all iOS 27.0.

There are no restrictions on what content you can place there. Unlike a widget, it receives a full UIScene, so build it like any other screen in your app. The Group Lab respondent also said they knew of no restrictions on animation or update frequency.

**Only camera apps can activate the inner and outer displays simultaneously.** Clock-like presentations that illuminate the inner display in tent pose are provided by AlarmKit. The Group Lab answer was that the same behavior probably could not be used outside alarms. Group Lab day 1 stated that this requires a system entitlement, but neither its name nor how to request it has been published, and Apple's documentation does not mention an entitlement. **Do not invent one.**

## Registering a camera accessory

This mechanism shows content to the person being photographed or filmed. The rear camera and outer display face the same direction, so you can show a script, countdown, or the camera's field of view to the person standing in front of the camera.

**Design assumptions:**

- The system chooses the destination. The app provides only the content type; it does not specify a display, capture session, or camera
- The outer display accepts touch, but keep interaction to a single action such as tapping the preview to focus. Do not make it a second control screen
- **Keep essential controls on the capture screen.** The system can withdraw the content at any time. Design the capture screen to work on its own, both on devices without an outer display and when the system displays nothing

**Register on the view that displays the capture screen.** Content appears only while that view is onscreen and stops when you navigate to another screen.

```swift
// UIKit
let configuration = UISceneConfiguration()
configuration.delegateClass = ScriptSceneDelegate.self

let accessory = UISceneAccessory.cameraCapture(sceneConfiguration: configuration,
                                               userInfo: model)
registration = registerSceneAccessory(accessory)   // Retain the return value strongly
```

- `cameraCapture(sceneConfiguration:userInfo:)` and `UISceneSession.Role.windowCameraCaptureAccessory` are iOS 27.1+ Beta
- Retain the returned `UISceneAccessoryRegistration` strongly. Call `unregisterSceneAccessory(_:)` only when you stop offering the accessory entirely
- The system assigns the session role. Adding an entry to the scene manifest has no effect. If one scene delegate handles multiple scene types, compare against `windowCameraCaptureAccessory` to distinguish them

**Have both sides read the same object rather than sending state back and forth.** In SwiftUI, the content closure captures surrounding state, so you can pass an observable model directly. In UIKit, pass it through `userInfo` and retrieve it from `UIScene.ConnectionOptions.sceneAccessoryUserInfo` when the scene connects. Keep a strong reference to the object in the app (the accessory is not its storage location).

**Availability and enabled state are separate.**

| | Controlled by | Read/write |
|---|---|---|
| Availability | System only | SwiftUI `onAvailabilityChange(perform:)` / UIKit `isAvailable` |
| Enabled state | App | SwiftUI `CameraCaptureAccessory(isEnabled:)` / UIKit `isEnabled` |

`isAvailable` supports observation, so reading it inside `updateProperties()` tracks changes without notifications.

Conditions that change availability:

- Capture stops, the app leaves the foreground, or the device closes
- Unavailable in Split View while open
- Only the frontmost registration of the same type is displayed. Navigating to a screen that registers its own content makes the previous registration unavailable; returning restores it
- Accessories of different types do not conflict (you can show slides on an external display while showing capture content on the outer display)

When there is no destination, the registration remains inactive and `isAvailable` continues to return false. This lets you use one code path, including on devices without an outer display. **To stop displaying content, provide an on/off control on the capture screen rather than unregistering.** It is enabled by default.

Build the content as a regular view to validate its layout in previews or the simulator. Because the simulator has no camera, always validate camera-dependent behavior on a device.

## Core Motion

If you need attitude aligned with the display orientation, make the view displaying the UI conform to `CMBodyIdentifiable` (iOS 27.0+) and assign it to `CMMotionManager.deviceMotionBody`. `CMDeviceMotion.attitude` then receives values corrected for the display in use. Correction applies only while full screen on that display; otherwise, values use the device's default coordinate system. Make UI layout decisions using size class and traits rather than which face points upward.

## Web

Safari's Viewport Segments API and Device Posture API are experimental features in iOS 27.1. Enable them under Settings > Apps > Safari > Advanced > Feature Flags to try them. There is no mechanism for a web page to identify iPhone Duo, so use responsive design.

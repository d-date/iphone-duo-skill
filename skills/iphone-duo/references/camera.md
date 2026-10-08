# Camera

## Two front cameras

iPhone Duo has two front cameras with square sensors and ultra-wide fields of view: one on the outer display side and one embedded under the inner display.

There are two options.

| | Resolution / frame rate | Depth | Switching when opening or closing |
|---|---|---|---|
| Virtual Front Camera | Up to 1080p / 60fps (only features common to both cameras) | Not supported | Automatic |
| Individual cameras | Inner: 1080p / up to 60fps; outer: up to 4K / 120fps | Supported | App-managed |

Discover the Virtual Front Camera by searching for the front position and Wide or Ultra Wide device types.

```swift
let session = AVCaptureDevice.DiscoverySession(
    deviceTypes: [.builtInWideAngleCamera, .builtInUltraWideCamera],
    mediaType: .video,
    position: .front
)
```

Specify individual cameras with `.builtInOuterUltraWideCamera` and `.builtInInnerUltraWideCamera`. Choose these when you need all features or depth.

**First decide whether Virtual Front Camera is sufficient.** If it is, you do not need to implement switching yourself. Apple's guidance is to start existing apps with Virtual Front Camera and use individual cameras in apps primarily intended for capture.

Virtual Front Camera is a virtual device with `isVirtualDevice` set to `true`. `activePrimaryConstituent` identifies the physical camera currently streaming (it is `nil` until the session runs).

## Camera direction

If you implement simulated autorotation by locking to portrait and rotating individual UI elements, replace it with a rotation coordinator and a direction coordinator. The former tells you how much to rotate the capture screen; the latter tells you which direction the camera in use faces. With two front cameras, you cannot assume that a front camera faces the user. Do not calculate the spatial relationships yourself.

A fixed `position` (front, back, or unspecified) is not enough to determine whether a camera faces the user. That relationship changes when the device opens or turns over. For example, opening it and turning it over makes both the rear camera and outer display camera face the user (Apple's article "Choosing a camera by the direction it faces" illustrates the closed, open, and open-and-turned-over states).

`AVCaptureDeviceDirectionCoordinator` (AVKit) reports camera direction **relative to the app's view**.

```swift
// init(view:deviceTypes:changeHandler:) (iOS 27.1+ Beta). Create on the main actor and retain strongly
directionCoordinator = AVCaptureDeviceDirectionCoordinator(
    view: previewView,
    deviceTypes: [.builtInOuterUltraWideCamera, .builtInInnerUltraWideCamera, .builtInDualWideCamera]
) { [weak self] map in
    self?.cameraDirectionsDidChange(map)
}
```

Pass a view (the reference for direction, not a rendering destination), device types to monitor, and a change handler.

Notes on `deviceTypes`:

- **Include rear cameras too.** A rear camera on iPhone Duo can face the user
- **Virtual Front Camera is not reported.** Specify `.builtInOuterUltraWideCamera` and `.builtInInnerUltraWideCamera` instead
- External, Continuity Camera, and Desk View cameras are ignored even if specified

For example, when the device opens and the view moves to the inner display, both the outer front camera and rear camera are reported as backward-facing, while the inner front camera is forward-facing.

When using both displays simultaneously, **create a coordinator for each view**. The same rear camera is forward-facing relative to the outer display and backward-facing relative to the inner display, because each report is relative to its own view. For the mechanism that enables simultaneous use of both displays, see "Registering a camera accessory" in `references/scenes.md`.

### Writing the change handler

The handler is called once immediately after creation (for the initial state), then on every change, on the main actor. `deviceDirections` is empty until the first call.

The supplied `AVCaptureDeviceDirectionMap` (iOS 27.1+ Beta) has `forwardFacingDeviceDescriptors` and `backwardFacingDeviceDescriptors`. **Forward-facing means facing the same direction as the view, not `position == .front`.** Do not infer direction from `position` or device type. The same code works on single-display iPhones (front is forward, back is backward, unspecified is in neither group, and the handler is called only once).

If the current camera remains forward-facing, do nothing. Select a replacement only when it no longer does. Pass the chosen descriptor to the camera actor and record it as the current camera **only after switching succeeds**. Recording it first creates a mismatch between the actual camera and the record if device creation fails, causing the next notification to skip switching.

The coordinator is tied to a view and therefore isolated to the main actor. The handler receives `AVCaptureDeviceDescriptor`, not `AVCaptureDevice`. This is a sendable representation that is safe to use on the main actor and contains the information needed to create an `AVCaptureDevice`.

**Do not call AVFoundation APIs directly in the handler.** Pass the descriptor to the camera actor and perform operations there.

There are three tasks when direction changes, but **they run in different places**.

| Task | Where |
|---|---|
| Receive the descriptor and pass it to the camera actor | Change handler |
| Reconfigure `AVCaptureSession` to maintain streaming from a camera facing the user | Camera actor |
| Update preview mirroring and UI | main actor |

### Reconfiguring on the camera actor

- `AVCaptureDeviceDescriptor` (iOS 27.1+ Beta) is a `Sendable` value with `deviceType` / `mediaTypes` / `position` / `uniqueID` / `localizedName`. It does not acquire the device
- Create the device on the actor with `AVCaptureDevice(uniqueID:)`. **Handle `nil` because the camera configuration can change during dispatch (do not force-unwrap)**
- Replace a single video input rather than connecting two cameras in a multicamera session
- Swap inputs inside `beginConfiguration()` / `commitConfiguration()`, and restore the original input if `canAddInput` fails

### Preview and mirroring

- The old camera's video appears during switching. Hide the preview when the handler is called and show it again when video from the new camera arrives
- Determine mirroring from the map, not `position`. Connections automatically mirror `position == .front`, so **mirror manually when a rear camera is forward-facing and remove mirroring when a front camera is backward-facing**
- Set `automaticallyAdjustsVideoMirroring = false` before `isVideoMirrored` (setting it while automatic adjustment is enabled throws an exception). Setting it on a connection whose `isVideoMirroringSupported` is `false` also throws an exception
- Replacing an input recreates the preview connection and loses its settings. Reapply them both when receiving the map and after connecting the new device
- After manual configuration, opening or closing can make direction and `position` agree again. Either restore `automaticallyAdjustsVideoMirroring` to `true` then, or set the map-derived value every time so stale settings do not remain

An implementation example is in Apple's article "Choosing a camera by the direction it faces" (https://developer.apple.com/documentation/avkit/choosing-a-camera-by-the-direction-it-faces).

## Preview

Showing the rear camera's full field of view on the inner display leaves unused space. You can position the preview to one side and group controls in the remaining space, or fill the entire display.

```swift
previewLayer.videoGravity = .resizeAspectFill
```

The ultra-wide front cameras let you use their square sensors to choose a wide aspect ratio.

```swift
let current = device.dynamicAspectRatio   // Read-only

// Change it with the dedicated method. Requires lockForConfiguration,
// and only accepts values in the activeFormat's supportedDynamicAspectRatios
try device.lockForConfiguration()
device.setDynamicAspectRatio(ratio) { syncTime, error in }
device.unlockForConfiguration()
```

## Rotation

Adopt `AVCaptureDevice.RotationCoordinator` (iOS 17.0+) to keep previews and photos upright. On iPhone Duo, it also updates when moving between displays.

```swift
let coordinator = AVCaptureDevice.RotationCoordinator(device: device, previewLayer: previewLayer)
previewLayer.connection?.videoRotationAngle = coordinator.videoRotationAngleForHorizonLevelPreview
```

The angle properties support key-value observing. Observe changes and apply them.

Apple's AVFoundation sample "Supporting device rotation in your camera app" (iOS 27.0+, Xcode 27.0+, https://developer.apple.com/documentation/avfoundation/supporting-device-rotation-in-your-camera-app) uses AVCam to show how to apply angles to both the preview and captured output. For the overall app structure, see "AVCam: Building a camera app" (https://developer.apple.com/documentation/avfoundation/avcam-building-a-camera-app).

Implementation notes:

- **A coordinator is bound to a device. Recreate it every time you switch cameras**
- A coordinator created with `nil` for `previewLayer` will not report preview angles even if a layer becomes available later. Recreate it once the layer is ready
- The layer cannot measure its position until it is in a window. Pass it again after `didMoveToWindow()` too
- KVO reports only subsequent changes. **Read and apply the current value before starting observation** (otherwise, the preview remains sideways until the first rotation)
- For captured output, set each output connection's `videoRotationAngle` to `videoRotationAngleForHorizonLevelCapture`. This can differ from the preview value
- Adding an output creates a connection with the default angle, so reapply the current angle every time you add one

```swift
photoOutput.connection(with: .video)?.videoRotationAngle = coordinator.videoRotationAngleForHorizonLevelCapture
```

After adopting the coordinator, disable sensor orientation compensation for performance (`isCameraSensorOrientationCompensationEnabled`, iOS 26.0+; check support with `isCameraSensorOrientationCompensationSupported`). This compensation is enabled on all iPhone Duo front cameras.

```swift
photoOutput.isCameraSensorOrientationCompensationEnabled = false
```

## Video calling apps

The inner front camera is in an unusual position toward the right side of the device. **Shift the UI's visual center toward the camera side** to direct the user's gaze there (Apple's FaceTime does this).

Keep the small self-view clear of the inner camera region. The inner camera occlusion appears only while the camera is active and disappears when it turns off, so any view you move must track its appearance and disappearance. Get the region with reserved regions' `.occlusion` (see `references/layout.md`).

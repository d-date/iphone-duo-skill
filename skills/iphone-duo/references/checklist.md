# Migration checklist

Follow these steps to adapt an existing app to iPhone Duo. Working from top to bottom reduces rework.

## 1. Build and validation environment

- [ ] Build with the iOS 27.1 SDK
- [ ] Select the iPhone Duo simulator in Xcode 27.1's Device Hub
- [ ] Check hinge angles and each pose (closed, open, book, laptop, tent)
- [ ] Check the appearance during transitions between poses too
- [ ] For custom layouts that avoid the fold or cameras, partially fold the device in the simulator, draw reserved regions, and check for overlap (see "Testing in the simulator" in `references/layout.md`)
- [ ] Ask Xcode's coding assistant to "get my app ready for iPhone Duo" to try the App Resizability skill. Export it for other agents with `xcrun agent skills export` (it cannot detect every issue)
- [ ] For apps that use a camera, check the preview on a device
- [ ] Also check the appearance in Device Hub's iOS resizable simulator, iPhone Mirroring on macOS 27, and resized iPad windows. In Mirroring, resize to extreme sizes in both directions. Check the complete native experience in the iPhone Duo simulator
- [ ] Before shipping an app built with the iOS 27.1 SDK, verify that content adapts when bars become vertical

## 2. Review layout decisions

This is the most likely area to break.

- [ ] Replace large/small layout decisions with size class
- [ ] Remove code that infers display size or device capabilities from the user interface idiom
- [ ] Remove layout dependencies on `UIDevice.current.orientation`, `statusBarOrientation`, and `interfaceOrientation` (including `windowScene.effectiveGeometry.interfaceOrientation`); use bounds width and height to determine whether the view is tall or wide
- [ ] Remove references to deprecated `UIScreen.main`, and use the environment, trait collection, and scene bounds for layout decisions. If you need the screen, retrieve it dynamically from the window scene rather than retaining `UIScreen`
- [ ] Replace `UIWindow(frame: UIScreen.main.bounds)` with `UIWindow(windowScene:)`
- [ ] Do not read size just once at launch or in `viewIsAppearing`; read it again in `layoutSubviews` / `viewDidLayoutSubviews`. Put resize handling in `viewWillTransition(to:with:)`
- [ ] Remove fixed widths, breakpoints, and dimensions or comparisons tied to specific displays, such as `bounds.height == 844`
- [ ] Move away from settings that lock orientation or size, such as `UIRequiresFullScreen`
- [ ] Identify size class branches that use separate containers and lift their state to a common ancestor

## 3. safe area

- [ ] Remove assumptions that opposite edges have equal insets and handle each edge separately (the same applies to content insets)
- [ ] Place interactive elements and foreground content inside the safe area
- [ ] Extend backgrounds beyond the safe area and behind bars
- [ ] Validate with the app on both sides of Split View (vertical bars can appear on either side)
- [ ] Remove portrait locking and check the leading and trailing safe area in landscape
- [ ] Look for code that subtracts twice one edge's inset from the width

## 4. Navigation and bars

- [ ] Use bars provided by standard navigation containers (custom bar configurations are not covered)
- [ ] Replace `UITabBar` added as a subview with `UITabBarController` / `TabView`
- [ ] Review toolbar order: primary navigation, then primary actions
- [ ] Provide titles even for items displayed as images
- [ ] Reduce title-only items and custom views that display both text and images (keep items whose text is meaningful in itself, such as prices, in horizontal bars; see `references/bars.md`)
- [ ] Replace count displays with badges
- [ ] Integrate custom overflow into a single system-managed menu
- [ ] Set visibility priorities at the group level
- [ ] Keep keyboard accessory bars attached to the keyboard
- [ ] Check the area around vertical bars with Reduce Transparency enabled
- [ ] Assign icons to tab items
- [ ] Change vertical bar height through rotation, Picture in Picture, and Split View, and check compression and overflow presentation
- [ ] Extend views with background images behind vertical bars using `backgroundExtensionEffect()` / `UIBackgroundExtensionView`
- [ ] Specify placement with `presentationPlacement(_:)` / `preferredPlacement` for sheets that should reveal more background content, such as a map
- [ ] Consider disabling vertical bars on screens where they do not fit (single pages with content concentrated at the bottom, or sheets with one button)

## 5. Layout adaptation

- [ ] Check whether full-bleed media using `.scaleAspectFill` / `.aspectRatio(contentMode: .fill)` is cropped in wide layouts; switch between fill and fit based on size class or aspect ratio, or specify a focal point
- [ ] Inspect centered layouts and consider two columns or displacement
- [ ] Check that continuous scrolling content is not moved between regions
- [ ] Consider reserved regions for complex layouts or custom UI placed outside the safe area
- [ ] Use supporting system components where fold avoidance is needed
- [ ] Move custom buttons that overlap the fold (purchase, add to cart, etc.) using reserved regions
- [ ] Provide a fallback that treats the layout as having no fold when division becomes inactive in the flat state and there are zero active regions
- [ ] If the UI changes by pose, verify that transitions between poses can animate smoothly
- [ ] Check the inner display at large Dynamic Type sizes
- [ ] Use an even number of grid columns (so they divide cleanly at the fold; see `references/layout.md`)

## 6. Multiple displays

- [ ] Support Split View multitasking
- [ ] Handle errors when requesting new scenes and use `UIWindowScene.ActivationAction`
- [ ] Consider scene accessories for accompanying content to display on another display
- [ ] Respond to changes in scene accessory availability
- [ ] For camera apps, register `CameraCaptureAccessory` on the capture screen's view and disable controls based on availability
- [ ] Verify that updates to state shared across multiple instances (such as `@AppStorage`) appear in each instance
- [ ] Consider hinge-based interactions and effects (do not use the hinge for layout)

## 7. Camera

- [ ] Replace simulated autorotation (portrait locking + rotation of individual UI elements) with a rotation coordinator and a direction coordinator
- [ ] Choose a front camera strategy (Virtual Front Camera or individual cameras)
- [ ] If using individual cameras, implement switching when the device opens or closes
- [ ] Adopt a direction coordinator when using individual cameras. Create one per view when using both displays simultaneously
- [ ] Use a camera actor rather than calling AVFoundation directly from the change handler
- [ ] Consider preview mirroring when direction changes
- [ ] Adopt a rotation coordinator, then disable sensor orientation compensation
- [ ] For video calling apps, keep the self-view clear of the inner camera occlusion

## 8. App Store

- [ ] Submit an app optimized for iPhone Duo through App Store Connect (accepted starting on 2026-10-05)
- [ ] Prepare the required iPhone Duo screenshots for submissions from 2027-04 onward
- [ ] Prepare screenshots and app previews that show the appearance in each orientation
- [ ] Check how the app looks on iPhone Duo using App Store Connect's preview tool
- [ ] Submit a featuring nomination and describe iPhone Duo optimization and support for all poses in Helpful Details

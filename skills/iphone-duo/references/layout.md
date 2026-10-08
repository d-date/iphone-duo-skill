# Layout

## size class

| State | Horizontal | Vertical |
|---|---|---|
| Outer display, portrait | compact | regular |
| Outer display, landscape | compact | compact |
| Inner display | regular | regular |

Designing for two cases, outer = compact width and inner = regular width, provides a foundation for every pose. You do not need a separate layout for each pose.

```swift
// SwiftUI
@Environment(\.horizontalSizeClass) private var horizontalSizeClass

// UIKit
traitCollection.horizontalSizeClass
// Track changes with registerForTraitChanges(_:action:) (iOS 17.0+)
```

Avoid fixed widths, breakpoints, and dimensions tied to a specific screen. If you need to distinguish portrait from landscape on the outer display, the size classes differ across the two axes, so you can use them. First question whether that distinction is necessary at all.

### Do not rely on the screen, orientation, or device size

Create windows from a scene, read size again whenever it changes, and use bounds to determine whether the layout is tall or wide.

```swift
// UIKit: Create the window from a scene
// Before: UIWindow(frame: UIScreen.main.bounds)
let window = UIWindow(windowScene: windowScene)

final class GalleryViewController: UIViewController {
    // Read size again whenever it changes. Reading it once at launch or in viewIsAppearing is not enough
    override func viewDidLayoutSubviews() {
        super.viewDidLayoutSubviews()
        // Before: Decisions based on UIScreen.main.bounds, UIDevice.current.orientation, bounds.height == 844, etc.
        let isWide = view.bounds.width > view.bounds.height
        updateColumns(isWide: isWide)
        // For full-bleed media, switch to fit if a wide layout crops important content
        heroImageView.contentMode = isWide ? .scaleAspectFit : .scaleAspectFill
    }
}
```

```swift
// SwiftUI: Receive changes to the available size with onGeometryChange(for:of:action:)
struct GalleryView: View {
    @State private var isWide = false

    var body: some View {
        Gallery(isWide: isWide)
            .onGeometryChange(for: Bool.self) { proxy in
                proxy.size.width > proxy.size.height
            } action: { newValue in
                isWide = newValue
            }
    }
}
```

`Gallery`, `updateColumns(isWide:)`, and `heroImageView` are custom names used for this example.

### Preserve state across branches

SwiftUI code that uses an `if` on size class with a different container in each branch discards one view hierarchy when the branch changes. State is lost unless it has been lifted to a parent.

```swift
// Avoid: A different container in each branch
if horizontalSizeClass == .regular {
    NavigationSplitView { ... } detail: { ... }
} else {
    NavigationStack { ... }
}
```

The app is not recreated when it moves between the inner and outer displays. This is resizing, not destruction and recreation, so standard navigation containers preserve state. First consider removing the branch itself, for example by using only `NavigationSplitView` or handling the difference with `ArrangementView`.

If you want different portrait and landscape layouts on the inner display, you can obtain the current orientation from the environment and traits, but base decisions on available width. On iPad, a narrow window can exist even in landscape. Let containers such as split views decide how to present columns, and size grids to the actual width (for example, two columns in landscape and one in portrait).

### Three-column layouts

An iPad app with three columns effectively displays two columns at once on the inner display. In portrait, it displays only the detail, with a button at the upper left to show the sidebar as an overlay. Use `NavigationSplitView` in SwiftUI and `UISplitViewController` in UIKit. Partially folding the device like a book automatically adjusts the split to 50/50. For information-dense apps, displaying tabs as a sidebar is another option.

### Sheets and tabs

For a sheet over a map with a tab bar inside it (such as Find My), if you are unsure whether to separate the tab bar and sheet on a larger display: keep them together if the tabs switch the sheet's contents. If each tab has a separate sheet, separating them is an option.

## safe area

Assume asymmetry. Vertical bars sit on one side, so left and right insets differ. In Split View, your app can be on either side.

```swift
// Avoid: Assuming equal insets on opposite edges
let width = view.bounds.width - view.safeAreaInsets.left * 2

// Handle each edge separately
let width = view.bounds.inset(by: view.safeAreaInsets).width
```

Content insets are also asymmetric. Use the actual system-provided values instead of using one edge's value to equalize both sides. Portrait-only apps often account only for the top and bottom safe area, making them a common place for left/right assumptions to remain.

The placement rule is: foreground inside the safe area, background extending beyond it. SwiftUI places content inside the safe area by default, so you only need to account for this when extending backgrounds.

```swift
// SwiftUI: Extend only the background
Color.accentColor.ignoresSafeArea()

// UIKit
foreground.frame = view.bounds.inset(by: view.safeAreaInsets)
backgroundView.frame = view.bounds
```

Remove the portrait lock and check the leading and trailing safe area in landscape. Expressions such as `safeAreaInsets.left * 2`, which double one edge's inset, are typical failure points. iPad apps with sidebars have an advantage because they already handle the leading edge.

Read insets through `GeometryProxy.safeAreaInsets` in SwiftUI and `UIView.safeAreaInsets` in UIKit. UIKit also provides `UIView.LayoutRegion` (iOS 26.0+), which returns guides for individual regions. Use `layoutGuide(for:)` / `edgeInsets(for:)` to obtain safe area, margins, and readable content with corner adaptation.

For apps without bars that need to avoid only the camera and status bar region rather than the entire safe area, use corner adaptation on `UIView.LayoutRegion` to respect that region while drawing to the remaining edges (for example, full-screen games).

## Display corners

The iOS 26 Concentricity API has been updated to support iPhone Duo's display shape.

The four corners of the outer display do not have equal radii: the side farther from the hinge is rounder. Corners also appear in positions where you previously did not need to handle them. Even if you already support corners on iPad, check whether there are additional cases to handle.

```swift
// SwiftUI
ConcentricRectangle()

// UIKit
view.cornerConfiguration = .uniformCorners(radius: .containerConcentric(minimum: 0))
```

## reserved regions

These are regions occupied by the hinge and the inner and outer cameras. There are three kinds.

| Region | When it exists |
|---|---|
| Outer front camera | Always present. Expands into the Dynamic Island for Live Activities. |
| Inner front camera | Only while the camera is active. |
| Fold | When partially folded. Divides the inner display into multiple regions. |

Treat them like regions your layout has already adapted to, such as iPadOS window controls. Standard components avoid them automatically. Use the APIs when custom components need to avoid them.

```swift
// SwiftUI
GeometryReader { proxy in
    let regions = proxy.reservedRegions(kind: .division)   // Fold
    let frames = regions.map(\.frame)
}

// UIKit
let regions = view.reservedRegions(kind: .division)

// Include inactive regions too
proxy.reservedRegions(kind: .division, options: .includeInactive)

// Cameras use .occlusion
proxy.reservedRegions(kind: .occlusion)
```

The SwiftUI declaration is `reservedRegions(kind:options:layoutDirectionBehavior:)`. The default for `options` is empty, and the default for `layoutDirectionBehavior` is `.mirrors`. In addition to `frame`, `ReservedRegion` has `isActive`, `kind`, and `margins` (iOS 27.1+ Beta).

An even column count lets a grid divide evenly at the fold. Use `.includeInactive` if you want to keep the count even regardless of the fold's state.

**Do not hard-code Duo-specific dimensions.** System components such as alerts automatically move to positions that are easy to read and tap, whether the device is folded like a book or standing on a table. Custom components in the content area do not move automatically. If a button tied directly to revenue, such as purchase, add to cart, or start a free trial, overlaps the fold, use reserved regions to check whether the fold is active and choose a new position. This API is for moving an individual element.

Custom UI placed over the outer camera region can have touches suppressed or delivered unexpectedly. Check the region with the reserved regions API.

### Testing in the simulator

The simulator also returns reserved regions. Partially fold the device and overlay a drawing of the returned division region to verify that custom layouts intended to avoid the fold or cameras do not overlap the regions.

```swift
struct DivisionRegionOverlay: View {
    var body: some View {
        GeometryReader { proxy in
            ForEach(proxy.reservedRegions(kind: .division)) { region in
                Rectangle()
                    .fill(.orange.opacity(0.3))
                    .frame(width: region.frame.width, height: region.frame.height)
                    .position(x: region.frame.midX, y: region.frame.midY)
            }
        }
    }
}

// content.overlay { DivisionRegionOverlay() }
```

Values checked on 2026-09-28 in Xcode 27.1 beta's iOS 27.1 simulator: when the inner display is partially folded in landscape, `.division` is a vertical strip 40pt wide at the center, with `margins` of 20pt on each side. The closed outer display returns two `.occlusion` regions. Use the returned regions; do not hard-code these dimensions into your layout.

When flat, division becomes inactive. Provide a fallback that behaves as though there is no fold when there are zero active regions.

## displacement (moving elements)

A **design pattern** that adjusts existing elements' frames to the available space. There is no API named `displacement`. To move elements yourself, query reserved regions and apply the results to your layout.

Guidelines:

- Move independently adaptable elements on their own; move related elements together while preserving their relationships.
- Avoid excessive movement that weakens the visual relationship to the original position.
- Do not move continuously scrolling content such as articles, feeds, documents, or lists. Scrolling already provides adaptation, and movement disrupts continuity.
- Choose destinations that suit the element's purpose and how the device is being used.

## arrangement (laying out two views)

A layout container with two views, primary and secondary. It chooses their arrangement based on display size, orientation, and reserved regions. It becomes available in iOS 27.1.

```swift
// SwiftUI
NavigationStack {
    ArrangementView {
        PlayerView()
    } secondary: {
        UpNextView()
    }
    .arrangementViewStyle(.split)
}

// UIKit
let vc = UIArrangementViewController()
vc.setViewController(playerVC, for: .primary)
vc.setViewController(upNextVC, for: .secondary)
```

There are two styles.

- **split**: Splits horizontally in a wide layout and vertically in a tall layout. Restrict the axis with `.split.axes(.horizontal)`; if that axis does not match the primary axis, it becomes a single view.
- **overlay**: Normally overlays the views, preferring a side-by-side layout when folded. Read the stacking order through `@Environment(\.overlayArrangementZIndex)` in SwiftUI or the `zIndex` from `state(for:)` in UIKit.

As a migration guide, use split for divisions built with `HStack` / `VStack`, and overlay for layering built with `ZStack`. If neither existing pattern fits, consider overlay when the foreground/background relationship is clear, or split when you have main content and detail and want neither hidden.

**Note that the nesting restrictions run in opposite directions.**

- Do not place navigation containers such as `NavigationSplitView` **inside** `ArrangementView`.
- Do not place `ArrangementView` **inside** `List` or `ScrollView`.

Place navigation outside `ArrangementView`.

An arrangement view animates the two views sliding apart when the pose changes, as the TV app separates video and controls when partially folded. This produces a more natural transition than switching custom UI for each pose.

**It is not exclusive to iPhone Duo.** It works the same way on nonfolding devices when the specified conditions are met. With enough width for two columns, it shows two; otherwise, it shows a single view. This is the same behavior as moving between the inner and outer displays. Use it to move away from switching between `HStack` / `VStack` / `ZStack` with `if` branches.

## Tailoring the experience to poses

**Before detecting poses and branching, consider whether arrangement view and reserved regions are sufficient.** Declare relationships between views and let the system arrange them, rather than directly checking for laptop or book. Apple's Music app uses an arrangement view and consults a reserved region to find the split position.

Even within Apple, there is no conclusion on whether partially folded use will be a primary mode or merely a brief state during pose changes. Do not invest too heavily in it immediately at launch.

Following best practices gives you a layout that works in every pose. Apple itself cites building a dedicated experience for every pose as a failed approach. If a pose suits your app's purpose, such as a video or podcast player moving controls to the lower half when placed on a table, tailor that experience specifically.

## Accessibility

- The inner display is particularly beneficial at large Dynamic Type sizes. Use a readable content guide or similar tools to handle both text-size changes and resizing.
- VoiceOver works on both displays simultaneously.
- Check the area around vertical bars with Reduce Transparency enabled (see `references/bars.md`).

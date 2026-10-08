# Vertical bars

## What changes

On the outer display and in open landscape, bars that normally sit at the top and bottom **move to the side**. This frees vertical space for content and puts controls within thumb reach. Open portrait is the exception: it retains standard horizontal bars because there is enough vertical space.

App elements are not the only things that move to the side. The Dynamic Island, status bar, toolbar (including navigation buttons), and tab bar all move. In Split View, each app places them on its own outer edge. Bars retain their position relative to the hardware, so they stay on the same side in right-to-left languages.

iPhone Duo is the only device whose status bar can be either horizontal or vertical. In an open, tall layout, both bars and the status bar are horizontal, with the status bar at the upper right. In an open, wide layout, the status bar runs vertically on the vertical bars side.

These are not new components: **the same components change their layout**. Think of rotating a horizontal bar 90 degrees into a vertical stack.

## Prerequisites

1. Rebuild with the latest SDK.
2. Use the bars provided by navigation containers.

When used directly, `UIToolbar` / `UINavigationBar` / `UITabBar` **remain horizontal and never become vertical**. Vertical bar placement is a container feature. In SwiftUI, combine `NavigationStack` or `NavigationSplitView` with `toolbar`; in UIKit, let `UINavigationController` and `UITabBarController` handle it.

### Custom tab bars

A `UITabBar` created and added as a subview does not move into a vertical bar. Replacing it with `UITabBarController` or SwiftUI's `TabView` provides support without additional work. Propose that replacement first.

**There is no API that gives a custom tab bar the same behavior as the system tab bar.** Label display during scrubbing and compression decisions are not exposed individually. If you need them, you must use system components.

Keeping a fully custom tab bar requires low-level APIs to match vertical bar dimensions and involves substantial work. You must also handle all of the following yourself.

- Vertical bar position is not fixed. When rotating while closed, it stays on the camera side, so it can appear on the left. In open Split View, the left app's bars sit on the left edge.
- The system tab bar shows each tab's label when the user presses down and starts dragging.

For placement guidance, an API was announced for iOS 27.1 that queries the region where the system would draw a bar, specifying a position such as the trailing bar. However, **the API was not named in this Group Lab and has not been identified in the documentation. Do not guess its name.** The documented `ReservedRegion.Kind` cases are `division` and `occlusion`; no kind corresponding to bars has been found.

## Item ordering

From top to bottom, place primary navigation (back, close) first, followed by primary actions (such as done). Preserve the original groups for the remaining items.

```swift
// Primary actions
// SwiftUI
ToolbarItem(placement: .topBarPinnedTrailing) { ... }
// UIKit
navigationItem.pinnedTrailingGroup = ...

// Custom back and close
// SwiftUI
ToolbarItem(placement: .cancellationAction) { ... }
// UIKit
navigationItem.leftItemsSupplementBackButton = false
```

A navigation controller adds the back button automatically. Bars may not be vertical in every pose, so **keep relative placement consistent across poses**. Users should not have to relearn control positions.

## Items suited to vertical and horizontal bars

Horizontal bars have fixed item heights and variable widths; vertical bars have fixed widths and variable heights. This makes vertical bars suitable for symbol-only items.

The choice of display format is the same as for horizontal bars. Top and bottom bars prefer icons, falling back to text when no icon is provided. Overflow shows both title and icon. The only new decision for vertical bars is whether the content is better suited to a vertical or horizontal layout. Items with icons move to vertical bars; text-only items remain horizontal.

**Provide both a title and an icon, and let the system choose.** Even items displayed as images need titles for overflow and expanded presentations.

The same applies to tab items. They move to vertical bars even without icons, but the system tab bar displays icons by default and reveals labels during interaction, so missing icons make it harder to understand. Assign appropriate SF Symbols.

```swift
// SwiftUI
Label("Share", systemImage: "square.and.arrow.up")
// UIKit: Set both title and image on UIBarButtonItem
```

Use `axisBehavior` to change the default axis.

```swift
// Keep a custom item that switches between a symbol and text horizontal
.axisBehavior(.horizontalOnly)      // SwiftUI
item.axisBehavior = .horizontalOnly // UIKit

// Prefer vertical placement for a custom view that supports it
.axisBehavior(.verticalPreferred)
```

The system edit button automatically stays horizontal. UIKit custom views and complex views are also horizontal by default.

### Making more content suitable for vertical bars

Decide whether text is needed by asking **whether it merely reinforces the symbol or conveys independent information**. If it is only supplementary, the symbol alone is enough. Information such as counts can move to badges.

```swift
.badge(unreadCount)          // SwiftUI
item.badge = .count(7)       // UIKit (iOS 26.0+)
```

When the text itself carries meaning, such as a cart button displaying an amount, keep it in a horizontal bar.

Read whether vertical bars are present through `@Environment(\.toolbarVerticalEdge)` in SwiftUI or `traitCollection.verticalBarEdge` in UIKit. Use this to change a custom view's presentation. Ensure custom views fit the bar's fixed width or provide a vertical layout.

## Overflow

Items move into overflow in outer-display landscape or when the keyboard or similar UI appears. Vertical bar height changes in many situations, including outer-display rotation, video Picture in Picture, and Split View. Previously, iPhone screens rarely stacked a bottom bar and tab bar vertically; with vertical bars, multiple bars share the same edge. Compression can therefore occur even with ample height. **If your app has never handled compression or overflow, check this first.** You can start without waiting for the simulator.

**First decide whether the toolbar or tab bar should remain visible longer.** On navigation-focused screens, use the default behavior of compressing the toolbar first. On task-focused screens, compress the tab bar first to keep actions available.

```swift
// SwiftUI
.toolbarVerticalCompressionBehavior(.prefersToolbarItems)
// UIKit
navigationItem.verticalBarCompressionBehavior = .prefersBarItems
// UIVerticalBarCompressionBehavior has .automatic / .prefersBarItems / .prefersTabBar
```

By default, items overflow from bottom to top. Adjust the order with visibility priorities. Set priorities for groups first, then individual items as needed. Keep frequently used controls and items showing important state visible longer.

```swift
.visibilityPriority(.high)        // SwiftUI (ToolbarItemVisibilityPriority)
item.visibilityPriority = .high   // UIKit (UIBarButtonItemVisibilityPriority)
// Custom values use init(higherThan:) / init(lowerThan:)
```

Integrate custom overflow into a system-managed menu. Use `ToolbarOverflowMenu` in SwiftUI and `UINavigationItem.additionalOverflowItems` in UIKit. Do not bring in symbols from other platforms; reserve the ellipsis for overflow.

## Disabling vertical bars

Vertical bars do not suit every app. Disabling them can make better use of space in a single-page app with content concentrated at the bottom, such as a calculator, or in a sheet with only a close button.

```swift
// SwiftUI
NavigationStack {
    ContentView()
        .toolbarVerticalBehavior(.disabled)
}

// UIKit
override var preferredVerticalBarBehavior: UIVerticalBarBehavior { .disabled }
```

Disabling them in an outer-display sheet uses the space up to the front camera and also repositions the status bar.

Treat `toolbarVerticalBehavior(_:)` as a stable choice. Do not change it on every navigation transition or toggle it based on a single view's state. Changing the value reflows content between vertical and horizontal bars and changes the status bar axis and safe area insets. If you only want to hide bars on a specific screen, use `toolbarVisibility(_:for:)`. Each container resolves the value from a particular view: the frontmost view in `NavigationStack`, the selected view in `TabView`, and the furthest trailing column in `NavigationSplitView`.

For an immersive full-screen app such as AR, hide the status bar. You do not need vertical controls if they do not suit the content.

## Sheets

Whether a sheet's bars become vertical **depends on its content**. The rough guidance from the Group Lab was that this choice generally persists when the device rotates; it was not stated definitively. However, apps such as Find My that choose the sheet's own width change the placement of the tab bar and sheet with rotation. For control-focused sheets, such as writing tools, horizontal bars may work better because vertical bars reduce the space available for interaction.

Outer-display sheets place controls at the side by default. On the inner display, default centered sheets use horizontal bars in both orientations. With custom placement, leading sheets have no vertical bars and trailing sheets have vertical bars.

Specify placement with `presentationPlacement(_:)` in SwiftUI and `UISheetPresentationController.preferredPlacement` in UIKit (both iOS 27.0). SwiftUI's `PresentationPlacement` argument accepts leading and trailing in addition to `.automatic` (the default). **Only sheets use this placement**; it does not affect other presentations such as popovers.

```swift
// To show more of the content behind the sheet, such as a map
.sheet(isPresented: $isShowingDetail) {
    DetailView()
        .presentationPlacement(.leading)
}
```

### inspector

`.inspector(isPresented:content:)` becomes a trailing column with a horizontal regular size class, and a sheet with a horizontal compact size class or on the outer display. It can appear alongside a `NavigationSplitView` sidebar. For bars when expanded, see "Other details".

## Extending backgrounds behind vertical bars

Views with hero or background images can extend the image beneath vertical bars. The effect mirrors the view, places the copy outside the safe area edge, and blurs it.

```swift
// SwiftUI (iOS 26.0+)
BannerView()
    .backgroundExtensionEffect()

// UIKit: Use UIBackgroundExtensionView (iOS 26.0+)
```

For readability and performance, normally apply it to only one background.

## Other details

- In a split view, only the detail column uses vertical bars. The sidebar and content columns remain horizontal. Do not give an expanded inspector its own vertical bar, because the detail column already has one; bars inside the inspector are horizontal.
- Keep keyboard accessory bars attached to the keyboard rather than moving them to the vertical axis.
- Do not add extra spacing in the app, regardless of bar orientation.
- Like horizontal bars, vertical bars have no scroll edge effect by default. With the Reduce Transparency accessibility setting, an opaque rectangle is drawn behind vertical bars. This rectangle is slightly narrower than the vertical bar safe area, so check the appearance with the setting enabled.
- Flexible spacers default to zero size on the vertical axis; fixed spacers retain their minimum size.

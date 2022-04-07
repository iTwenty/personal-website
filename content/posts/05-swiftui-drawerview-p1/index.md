---
title: "Create Android style navigation drawer in SwiftUI"
date: 2022-04-01T14:28:35+05:30
draft: true
tags: [ios, android, swiftui]
GHIssueID:
---

Android's Jetpack Compose has a nifty navigation drawer that is useful for adding navigation to different sections of an app. iOS doesn't have any equivalent UI element, either in UIKit or in SwiftUI. However, the view is pretty easy to create in SwiftUI - and that is exactly what we will be doing over the course of this post. This is going to be our end goal:

{{< youtube idGlGvZMSKg >}}

A breakdown of the features our DrawerView will support -

- We should be able to open/close the drawer programatically
- Main view should gradually fade as drawer view slides in
- We should be able to use drag gestures to open/close the drawer
- Drawer should "interact" with the drag gesture.

We will discuss more on the last point when we get to it's implementation.

The public API of DrawerView looks like this -

```swift
@State var isOpen = false

DrawerView(isOpen: $isOpen) {
    // Main content view
} drawer: {
    // Drawer content view
}
```

The `isOpen` boolean binding controls whether the drawer is open or closed.

Let's go ahead and create the `DrawerView.swift` file which satisfies this API -

```swift
import SwiftUI

// 1
struct DrawerView<MainContent: View, DrawerContent: View>: View {
    // 2
    @Binding var isOpen: Bool
    private let main: () -> MainContent
    private let drawer: () -> DrawerContent

    init(isOpen: Binding<Bool>,
         @ViewBuilder main: @escaping () -> MainContent,
         @ViewBuilder drawer: @escaping () -> DrawerContent) {
        self._isOpen = isOpen
        self.main = main
        self.drawer = drawer
    }

    var body: some View {
        main()
    }
}
```

1. Main content and drawer content can be different types of views. So DrawerView needs two different generic types to represent them. 
2. The boolean binding is not used for now, but we will use it later to close the drawer when any part of main content is tapped.

### Lay things out
We need to make sure that main view occupies the full size of the screen and drawer view occupies full height and some fraction of main view's width. We want this fraction to be easily configurable. In SwiftUI, the way to read the size of any view is by using [`GeometryReader`](https://developer.apple.com/documentation/swiftui/geometryreader). GeometryReader constructor takes a single closure and passes an instance of `GeometryProxy` to this closure. Using this proxy, we can get (among other things) the size of the view containing the GeometryReader[^geometryreader_swiftui_lab].

```swift
struct DrawerView<MainContent: View, DrawerContent: View>: View {
    // 1
    private let overlap: CGFloat = 0.7
    ....
    ....

    var body: some View {
        GeometryReader { proxy in
            // 2
            let drawerWidth = proxy.size.width * overlap
            // 3
            ZStack(alignment: .topLeading) {
                // 4
                main().frame(maxWidth: .infinity, maxHeight: .infinity)
                // 5
                drawer().frame(minWidth: drawerWidth, idealWidth: drawerWidth, 
                               maxWidth: drawerWidth, maxHeight: .infinity)
            }
        }
    }
}
```

{{< preview src="drawer_preview_1.png" >}}
struct ContentView: View {
    @State var isOpen = false

    var body: some View {
        DrawerView(isOpen: $isOpen) {
            Color.red
            Button("Show drawer") {
                withAnimation {
                    isOpen.toggle()
                }
            }
        } drawer: {
            Color.green
            Button("Hide drawer") {
                withAnimation {
                    isOpen.toggle()
                }
            }
        }
    }
}
{{< /preview >}}

1. The `overlap` CGFloat property is a fraction between between 0 to 1. It governs how wide the drawer should be compared to the main content.
2. Within our GeometryReader closure, we calculate the width of drawer using the width of the proxy and overlap fraction.
3. We embed both main and drawer views inside a ZStack. Drawer comes after main since we want drawer to show on top of main view.
4. We embed main view in an infinitely sized frame. Practically, this means the frame will extend to occupy same size as it's first *non layout neutral* parent. Since both ZStack and GeometryReader are layout neutral, the frame will end up occupying the size of the device screen.
5. We embed drawer in a frame with infinite height, but width is constrained to what we calculated in step 2.

### Show/hide drawer programtically

Note that in ContentView, we are toggling the isOpen boolean on button taps. Tapping the buttons won't do anything yet. To fix this, we need to modify the position of drawer along X axis in response to the boolean value. This can be accomplished by adding an `offset` modifier to drawer view.


```swift
var body: some View {
    GeometryReader { proxy in
        let drawerWidth = proxy.size.width * overlap
        ZStack(alignment: .topLeading) {
            main().frame(maxWidth: .infinity, maxHeight: .infinity)
            drawer()
                .frame(minWidth: drawerWidth, idealWidth: drawerWidth,
                       maxWidth: drawerWidth, maxHeight: .infinity)
                .offset(x: isOpen ? 0 : -drawerWidth, y: 0)
        }
    }
}
```

{{< preview src="drawer_preview_2.gif" >}}
struct ContentView: View {
    @State var isOpen = false

    var body: some View {
        DrawerView(isOpen: $isOpen) {
            Color.red
            Button("Show drawer") {
                withAnimation {
                    isOpen.toggle()
                }
            }
        } drawer: {
            Color.green
            Button("Hide drawer") {
                withAnimation {
                    isOpen.toggle()
                }
            }
        }
    }
}
{{< /preview >}}

When isOpen is true, we set drawer X-axis offset to 0. This will show the drawer in it's original position i.e overlapping the main view. When isOpen is false, we set the offset equal to negative of drawer width. This will effectively "hide" the drawer by moving it off screen. We don't need to change Y-axis offset.

Tapping the buttons in ContentView should now toggle the drawer visibility with a nice animation. We don't need to write any animation code besides wrapping the changes to isOpen in a `withAnimation` block. SwiftUI is smart enough to figure out what properties need to change based on this boolean and smoothly animate between their start and end values (in this case - the X-axis offsets). Pretty cool, huh?

### Fade main view gradually

For putting more focus on the drawer when it is open, we can fade the main view progressively as drawer opens. We also need to make sure that -
- Main view's content is disabled from user interaction while drawer is open
- Tapping the main view content anywhere closes the drawer

This might sound like a lot of work, but it's actually very trivial to achieve thanks to the power of modifiers. SwiftUI is smart enough to infer that if a modifier is not going to affect the view tree, it can effectively be "removed" from the view. To understand what this means, try running this piece of code on a simulator -


```swift
Color.red.opacity(1)
    .onTapGesture {
        print("Tapped")
    }
```

Whole screen will be red and tapping anywhere will print a message in console. Now change opacity to 0. This time, no message will be printed since setting opacity to 0 effectively hides the view and so SwiftUI also removes the tap gesture associated with it.[^inert_modifiers]

Using this technique, we can add an overlay for the main view -

```swift
struct DrawerView<MainContent: View, DrawerContent: View>: View {
    ...
    // 1
    private let overlayColor = Color.gray
    private let overlayOpacity = 0.7

    ...
    var body: some View {
        GeometryReader { proxy in
            let drawerWidth = proxy.size.width * overlap
            ZStack(alignment: .topLeading) {
                main()
                    .frame(maxWidth: .infinity, maxHeight: .infinity)
                    // 3
                    .overlay(mainOverlay)
                drawer()
                    .frame(minWidth: drawerWidth, idealWidth: drawerWidth,
                           maxWidth: drawerWidth, maxHeight: .infinity)
                    .offset(x: isOpen ? 0 : -drawerWidth, y: 0)
            }
        }
    }

    // 2
    private var mainOverlay: some View {
        overlayColor.opacity(isOpen ? overlayOpacity : 0.0)
            .onTapGesture {
                withAnimation {
                    isOpen.toggle()
                }
            }
    }
}
```

{{< preview src="drawer_preview_3.gif" >}}
struct ContentView: View {
    @State var isOpen = false

    var body: some View {
        DrawerView(isOpen: $isOpen) {
            Color.red
            Button("Show drawer") {
                withAnimation {
                    isOpen.toggle()
                }
            }
        } drawer: {
            Color.green
            Button("Hide drawer") {
                withAnimation {
                    isOpen.toggle()
                }
            }
        }
    }
}
{{< /preview >}}

1. We define the color of the overlay and it's opacity when drawer is fully open. The opacity will be 0 when drawer is closed.
2. We define a view for overlay whose opacity depends on `isOpen` boolean and add a tap gesture to this view which toggles the boolean.
3. We add this view as an overlay to our main view.

Thanks to the overlay, the underlying main view content becomes non-interactive while the drawer is open and interactive again when drawer is closed.

### Adding support for drag gestures

Our DrawerView still lacks one critical functionality that Android's view has - Drag support. To avoid making this post too long, we will focus on adding this support in another post which can be found here.

---
[^geometryreader_swiftui_lab]: To know more about GeometryReader, I highly recommend checking this article - https://swiftui-lab.com/geometryreader-to-the-rescue/

[^inert_modifiers]: I couldn't find any official documentation for this behaviour. The closest thing is "Inert Modifiers" as talked about in WWDC21's "[Demystify SwiftUI](https://developer.apple.com/videos/play/wwdc2021/10022/)" talk (37:30 onwards).
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

struct DrawerView<MainContent: View, DrawerContent: View>: View {
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

Main content and drawer content can be different types of views. So DrawerView needs two different generic types to represent them.
 
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
            main().frame(width: proxy.size.width, height: proxy.size.height)
            // 4
            drawer().frame(width: drawerWidth, height: proxy.size.height)
        }
    }
}
```

{{< preview src="drawer_preview_1.png" >}}
struct DrawerView_Previews: PreviewProvider {
    static var previews: some View {
        DrawerView(isOpen: .constant(false)) {
            Color.red
            Text("Main content")
        } drawer: {
            Color.green
            Text("Drawer Content")
        }
    }
}
{{< /preview >}}


---
[^geometryreader_swiftui_lab]: To know more about GeometryReader, I highly recommend checking this article - https://swiftui-lab.com/geometryreader-to-the-rescue/
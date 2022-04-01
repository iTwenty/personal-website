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

The public API of our DrawerView looks like this -

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
        Text("Hello world")
    }
}

struct DrawerView_Previews: PreviewProvider {
    static var previews: some View {
        DrawerView(isOpen: .constant(false)) {
            Text("Main content")
        } drawer: {
            Text("Drawer Content")
        }
    }
}
```

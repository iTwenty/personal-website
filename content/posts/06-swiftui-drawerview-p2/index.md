---
title: "Drag Gestures in SwiftUI"
date: 2022-04-09T15:11:44+05:30
draft: true
tags: [ios, android, swiftui]
GHIssueID:
---

*NOTE : This post builds upon the DrawerView implementation from the [previous post]({{< ref "/posts/05-swiftui-drawerview-p1/index.md" >}}).*

Continuing our DrawerView implementation, let's now add a drag gesture to open/close the drawer. We want to accomplish the following things -

- When drawer is closed, dragging from left edge of screen towards right should start opening the drawer.
- When drawer is open, dragging from right to left anywhere should start closing the drawer.
- Drawer should interact with our finger's movement:
    - If we move finger right, drawer should slide in by corresponding amount. 
    - If we move finger left, drawer should slide out by corresponding amount.
- If we release our finger, drawer should either open or close completely depending on either:
    - How fast our drag was.
    - How far did we drag our finger.

Check the video in the introduction of [this post]({{< ref "/posts/05-swiftui-drawerview-p1/index.md" >}}) to get an idea of the drag interaction we are aiming for.

The actual SwiftUI boilerplate code required to achieve this functionality is fairly small. Our main focus is going to be the math behind the drag interaction.

### Represent fractional open/closed states

The `isOpen` boolean we are using to represent drawer state can only represent fully open or fully closed states. For drag support, we need a way to represent the transient partially open/closed states as well. Let's add the required state as a `CGFloat` property.

```swift
struct DrawerView<MainContent: View, DrawerContent: View>: View {
    ...
    @Binding var isOpen: Bool
    // 1
    @State private var openFraction: CGFloat
    ...

    init(isOpen: Binding<Bool>,
         @ViewBuilder main: @escaping () -> MainContent,
         @ViewBuilder drawer: @escaping () -> DrawerContent) {
        self._isOpen = isOpen
        // 2
        self.openFraction = isOpen.wrappedValue ? 1 : 0
        self.main = main
        self.drawer = drawer
    }

    var body: some View {
        let _ = print(openFraction)
        GeometryReader { proxy in
            let drawerWidth = proxy.size.width * overlap
            ZStack(alignment: .topLeading) {
                ...
            }
            // 3
            .onChange(of: isOpen) { newValue in
                withAnimation {
                    self.openFraction = newValue ? 1 : 0
                }
            }
        }
    }
}
```

1. We add a private CGFloat state which represents the "openness" of the drawer. It's value will be between 0 and 1 with 0 representing fully closed state and 1 representing fully open state. For example - if the value of `openFraction` is 0.3, it means drawer is 30% open. It is a private property since we want users of DrawerView to still rely on `isOpen` binding.
2. We set the initial value of `openFraction` based on the value of `isOpen`.
3. We also need to ensure that `openFraction` is updated to correct value whenever `isOpen` changes.

You might be tempted to use the `didSet` observer on `isOpen` for #3, but it will not work since `isOpen` is of type `Binding<Bool>` and not `Bool`. `didSet` will be called when the binding changes, not when the enclosed boolean changes. So we need to use the `onChange` modifier for this purpose. Check [this article](https://www.hackingwithswift.com/books/ios-swiftui/responding-to-state-changes-using-onchange) by Paul Hudson to know more.

### Replace `isOpen` with `openFraction`

Now that we have added fractional state, we need to actually use it render our view's body, rather than `isOpen`. Right now, we are using the boolean for two things -

1. Changing the opacity of our main view.
2. Setting the offset of our drawer view.

Opacity is easy - we just use the value of openFraction is for opacity directly. Offset is a bit more complex. openFraction of 0 corresponds to offset of `-drawerWidth` and openFraction of 1 corresponds to offset of 0. So we need a way to map a value in range [0, 1] to corresponding value in range [-drawerWidth, 0].

The general formula for remapping a value `t` from range `[a, b]` to range `[c, d]`  is -

$$
f(t)=\frac{(t-a) \times (d - c)}{b - a} + c
$$

To understand how this formula is derived, I recommend checking [this article](https://victorkarp.com/godot-engine-how-to-remap-a-range-of-numbers/).

Let's go ahead and use openFraction for rendering our DrawerView's body.

```swift
struct DrawerView<MainContent: View, DrawerContent: View>: View {
    ....
    var body: some View {
        GeometryReader { proxy in
            let drawerWidth = proxy.size.width * overlap
            ZStack(alignment: .topLeading) {
                main()
                    .frame(maxWidth: .infinity, maxHeight: .infinity)
                    .overlay(mainOverlay)
                drawer()
                    .frame(minWidth: drawerWidth, idealWidth: drawerWidth,
                           maxWidth: drawerWidth, maxHeight: .infinity)
                    // 4
                    .offset(x: xOffset(drawerWidth), y: 0)
            }
            .onChange(of: isOpen) { newValue in
                withAnimation {
                    self.openFraction = newValue ? 1 : 0
                }
            }
        }
    }

    private var mainOverlay: some View {
        // 3
        overlayColor.opacity(openFraction)
            .onTapGesture {
                withAnimation {
                    isOpen.toggle()
                }
            }
    }

    // 1
    private func remap(value: CGFloat, from: ClosedRange<CGFloat>, to: ClosedRange<CGFloat>) -> CGFloat {
        let toDiff = to.upperBound - to.lowerBound
        let fromDiff = from.upperBound - from.lowerBound
        return (value - from.lowerBound) * toDiff / fromDiff + to.lowerBound
    }

    // 2
    private func xOffset(_ drawerWidth: CGFloat) -> CGFloat {
        remap(value: openFraction, from: 0...1, to: -drawerWidth...0)
    }
}
```

1. We add a `remap` function which does the range remapping described above.
2. We add an `xOffset` helper function, passing to it the final drawer width to be used as start value for the `to` range of remap. Rest of the values are either constants or view's properties.
3. We update our main view overlay's opacity based on `openFraction` rather than `isOpen`.
4. We use the helper `xOffset` function for setting the drawer's offset, rather than `isOpen`.

If we run the application now, nothing will have changed visually, but we have laid down all the groundwork needed for adding an interactive drag gesture. So let's go ahead and do that.

### Gestures in SwiftUI
Gestures in SwiftUI are applied to specific views in the view hierarchy. To recognize a gesture event on a particular view, we first create and configure the gesture and then use the `gesture(_:including:)` view modifier.

To receive updates from the gesture itself, we have to use gesture modifiers. There are three gesture modifiers -

1. `updating(_:body:)` 
2. `onChanged(_:)`
3. `onEnded(_:)`

For our drag gesture, we are primarily interested in the last two. `onChanged` is called when the gesture begins and - for continuous gestures like drag gesture - each time the gesture's value changes.
`onEnded` is invoked - unsurprisingly - when the gesture ends. Note that SwiftUI only invokes `onEnded` if the gesture succeeds. The definition of success depends on the gesture under consideration. For example, `DragGesture` takes in a `minimumDistance` parameter. If the user begins dragging, but lifts his finger before moving at least `minimumDistance` pixels from the gesture start point, the gesture won't be considered a success and `onEnded` won't be invoked.

---

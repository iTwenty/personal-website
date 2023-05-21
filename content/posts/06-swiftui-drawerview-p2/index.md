---
title: "Drag Gestures in SwiftUI"
summary: "Explore drag gestures in SwiftUI while building a navigation drawer."
date: 2022-04-09T15:11:44+05:30
draft: false
tags: [ios, android, swiftui]
---

*NOTE : This post builds upon the DrawerView implementation from the [previous post]({{< ref "/posts/05-swiftui-drawerview-p1/index.md" >}}).*

Continuing our DrawerView implementation, let's now add a drag gesture to open/close the drawer. We want to accomplish the following things -

- When drawer is closed, dragging from left edge of screen towards right should start opening the drawer.
- When drawer is open, dragging from right to left anywhere should start closing the drawer.
- Drawer should interact with our finger's movement:
    - If we move finger right, drawer should slide in by corresponding amount. 
    - If we move finger left, drawer should slide out by corresponding amount.
- If we release our finger, drawer should snap to open or closed state depending on:
    - How fast our drag was OR
    - How far did we drag our finger.

Check the video in the introduction of [this post]({{< ref "/posts/05-swiftui-drawerview-p1/index.md" >}}) to get an idea of the drag interaction we are aiming for.

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

1. We add a private CGFloat property which represents the "openness" of the drawer. It's value will be between 0 and 1 with 0 representing fully closed state and 1 representing fully open state. For example - if the value of `openFraction` is 0.3, it means drawer is 30% open. It is a private property since we want users of DrawerView to still rely on `isOpen` binding.
2. We set the initial value of `openFraction` based on the value of `isOpen`.
3. We also need to ensure that `openFraction` is updated to correct value whenever `isOpen` changes.

You might be tempted to use the `didSet` observer on `isOpen` for #3, but it will not work since `isOpen` is of type `Binding<Bool>` and not `Bool`. `didSet` will be called when the binding changes, not when the enclosed boolean changes. So we need to use the `onChange` modifier for this purpose. Check [this article](https://www.hackingwithswift.com/books/ios-swiftui/responding-to-state-changes-using-onchange) by Paul Hudson to know more.

### Replace `isOpen` with `openFraction`

Now that we have added fractional state, we need to actually use it render our view's body, rather than `isOpen`. Right now, we are using the boolean for two things -

1. Changing the opacity of our main view.
2. Setting the offset of our drawer view.

Opacity is easy - we just use the value of openFraction for opacity directly. Offset is a bit more complex. openFraction of 0 corresponds to offset of `-drawerWidth` and openFraction of 1 corresponds to offset of 0. So we need a way to map a value in range [0, 1] to corresponding value in range [-drawerWidth, 0].

The general formula for remapping a value `t` from source range `[a, b]` to target range `[c, d]`  is -

$$
f(t)=\frac{(t-a) \times (d - c)}{b - a} + c
$$

To understand how this formula is derived, I recommend checking [this article](https://victorkarp.com/godot-engine-how-to-remap-a-range-of-numbers/).

Let's go ahead and use `openFraction` for rendering our DrawerView's body.

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
    private func remap(_ value: CGFloat, from source: ClosedRange<CGFloat>, to target: ClosedRange<CGFloat>) -> CGFloat {
        let targetDiff = target.upperBound - target.lowerBound
        let sourceDiff = source.upperBound - source.lowerBound
        return (value - source.lowerBound) * targetDiff / sourceDiff + target.lowerBound
    }

    // 2
    private func xOffset(_ drawerWidth: CGFloat) -> CGFloat {
        remap(openFraction, from: 0...1, to: -drawerWidth...0)
    }
}
```

1. We add a `remap` function which does the range remapping described above.
2. We add an `xOffset` helper function, passing to it the final drawer width to be used as start value for the `target` range of remap. Rest of the values are either constants or view's properties.
3. We update our main view overlay's opacity based on `openFraction` rather than `isOpen`.
4. We use the helper `xOffset` function for setting the drawer's offset, rather than `isOpen`.

If we run the application now, nothing will have changed visually, but we have made the drawer offset be driven by `openFraction` value between 0 to 1. Next, we need a way to remap the movement of user's finger in drag gesture to a value a between 0 and 1, which we will assign to `openFraction`. But first, let's take a quick look at how gestures work in SwiftUI.

### Gestures in SwiftUI

Gestures in SwiftUI are applied to specific views in the view hierarchy. To recognize a gesture event on a particular view, we first create and configure the gesture and then use the `gesture(_:including:)` view modifier.

To receive updates from the gesture itself, we have to use gesture modifiers, of which there are three -

1. `updating(_:body:)` 
2. `onChanged(_:)`
3. `onEnded(_:)`

For our drag gesture, we are primarily interested in the last two. `onChanged` is called when the gesture begins and - for continuous gestures like drag gesture - each time the gesture's value changes.
`onEnded` is invoked - unsurprisingly - when the gesture ends. Note that SwiftUI only invokes `onEnded` if the gesture succeeds. The definition of success depends on the gesture under consideration. For example, `DragGesture` takes in a `minimumDistance` parameter. If the user begins dragging, but lifts his finger before moving at least `minimumDistance` pixels, the gesture won't be considered a success and `onEnded` won't be invoked.

Both these modifiers provide a parameter of type `DragGesture.Value` which we can use to read properties like gestures's start location, it's current translation etc. We are primarily interested in `translation` CGSize property. This property is updated everytime the finger moves and tells us the total translation from the start of the drag gesture to the current event of the drag gesture.

Another helpful tip to remember - In iOS, right/down means positive and left/up means negative. Origin is generally at top left corner.

### Add drag gesture to DrawerView

Before diving into implementation of gesture, it is helpful to understand how the drag values change in response to drag gesture. So let's go ahead and attach a DragGesture to our ZStack and log some interesting values as we drag across the view.

```swift
struct DrawerView<MainContent: View, DrawerContent: View>: View {
    ...
    var body: some View {
        GeometryReader { proxy in
            let drawerWidth = proxy.size.width * overlap
            ZStack(alignment: .topLeading) {
                ...
            }
            // 1
            .gesture(dragGesture(proxy.size.width))
            ...
        }
    }
    ...
    // 2
    private func dragGesture(_ mainWidth: CGFloat) -> some Gesture {
        return DragGesture().onChanged { value in
            print("onChanged startX : \(value.startLocation.x), moveX : \(value.translation.width)")
        }.onEnded { value in
            print("onEnded startX : \(value.startLocation.x), moveX : \(value.translation.width)")
        }
    }
}
```

{{< preview src="drawer_preview_4.gif" >}}
CONSOLE OUTPUT :
onChanged startX : 98.666656, moveX : 10.333344
onChanged startX : 98.666656, moveX : 12.666672
onChanged startX : 98.666656, moveX : 14.666672
onChanged startX : 98.666656, moveX : 16.333344
.....
onChanged startX : 98.666656, moveX : 44.666672
onChanged startX : 98.666656, moveX : 43.666672
onChanged startX : 98.666656, moveX : 43
.....
onChanged startX : 98.666656, moveX : 2.666672
onChanged startX : 98.666656, moveX : 1.333344
onChanged startX : 98.666656, moveX : -0.333328
onChanged startX : 98.666656, moveX : -1.666656
onChanged startX : 98.666656, moveX : -3
.....
onChanged startX : 98.666656, moveX : -73
onChanged startX : 98.666656, moveX : -73.333328
onChanged startX : 98.666656, moveX : -73.666656
onEnded startX : 98.666656, moveX : -73.666656
{{< /preview >}}

1. We attach a drag gesture to our ZStack using `gesture` modifier.
2. To the drag gesture, we attach `onChanged` and `onEnded` modifiers using which we log the X values of drag start location and drag translation. We are not interested in Y values.

As we can see in the logs, startX remains same across the whole gesture. moveX increases as we move our finger from left to right, starts decreasing as we move from right to left. As we keep moving past the drag start location, moveX becomes negative and keeps decreasing. The full range of values possible for moveX goes from 0 to `proxy.size.width` when we start dragging from extreme left edge and move all the way to right edge. When done the other way round i.e start drag from right edge all the way to left edge, moveX will have a value between 0 to `-proxy.size.width`.

Next, let's focus on what we want our drag gesture to accomplish. I will call `proxy.size.width` as mainWidth from now on, since it denotes the width of our main view.

1. When the gesture begins and drawer is closed -
    - Ignore -ve moveX values. Drawer is already as closed as it can be.
    - Remap +ve moveX values that will fall in range [0, mainWidth] to corresponding value in range [0, 1].
2. When the gesture begins and drawer is open
    - Ignore +ve moveX values. Drawer is already as open as it can be. 
    - Remap -ve moveX values that will fall in range [-mainWidth, 0] to corresponding value in range [0, 1].

The value we get from either of these steps becomes the new value of `openFraction`. Let's write this down in code and see what it looks like -

```swift
// 2
private func dragGesture(_ mainWidth: CGFloat) -> some Gesture {
    return DragGesture().onChanged { value in
        if isOpen, value.translation.width < 0 {
            openFraction = openFraction(value.translation.width, from: -mainWidth...0)
        } else if !isOpen, value.translation.width > 0 {
            openFraction = openFraction(value.translation.width, from: 0...mainWidth)
        }
    }.onEnded { value in
        print("onEnded startX : \(value.startLocation.x), moveX : \(value.translation.width)")
    }
}

// 1
private func openFraction(_ moveX: CGFloat, from source: ClosedRange<CGFloat>) -> CGFloat {
    remap(moveX, from: source, to: 0...1)
}
```

1. We introduce a new helper function that maps `moveX` values in the passed `source` range to values in target range `0...1`. We call this function `openFraction` since it's return value is meant to assigned to `openFraction` property
2. In the `onChanged` modifier of our Drag gesture, we code-ify the drag gesture conditions mentioned in previous paragraph.

If we run the app at this point, we will notice that drag gestures work perfectly as long as our finger is pressed down and moved. Once we lift the finger though, drawer doesn't snap to either open or closed state. This is something we need to do in `onEnded` modifier.

### Snapping to closest state on gesture end

In the introduction of this post, we mentioned that the snapping behaviour will depend on variables like how far along the user dragged his finger, how fast the drag was just before user lifted his finger etc. This might sound like a lot of work involving some complex mathematics, but it's actually fairly trivial to achieve since SwiftUI already does the complex parts for you. The `value` parameter passed to `onEnded` modifier has property called [`predictedEndTranslation`](https://developer.apple.com/documentation/swiftui/draggesture/value/predictedendtranslation) whose documentation says "A prediction, based on the current drag velocity, of what the final translation will be if dragging stopped now."

By passing the width of this predicted translation to the `openFraction` function we defined earlier, we can get `predictedOpenFraction`. If this prediction is less than 0.5, we set `openFraction` to 0, else we set `openFraction` to 1.

```swift
private func dragGesture(_ mainWidth: CGFloat) -> some Gesture {
    return DragGesture().onChanged { value in
    ...
    }.onEnded { value in
        // 1
        let fromRange = isOpen ? -mainWidth...0 : 0...mainWidth
        let predictedMoveX = value.predictedEndTranslation.width
        let predictedOpenFraction = openFraction(predictedMoveX, from: fromRange)
        if predictedOpenFraction > 0.5 {
            withAnimation {
                // 2
                openFraction = 1
                isOpen = true
            }
        } else {
            withAnimation {
                // 2
                openFraction = 0
                isOpen = false
            }
        }
    }
}
```

1. Just like we did in `onChanged` modifier, we need to ensure that we use the correct source range for our drag gesture.
2. We need to ensure that we keep `isOpen` and `openFraction` in sync, or bad things will happen.

### Open drawer only if drag begins close to left edge

Our DrawerView implementation is almost done. It responds beautifully to drag gestures. Try performing various types of drags like small slow flicks, small fast flicks, long slow drags etc and see how the view responds naturally to all these gestures. One thing you might notice is that if drawer is in closed state, dragging right from anywhere triggers the opening animation. Ideally, we want to restrict it to only trigger when the drag originates near the left edge of the screen. This can be easily done by adding another property called `dragOpenThreshold` which is a fraction of main view's width. If the X co-ordinate of drag's start location's is less than `mainWidth * dragOpenThreshold`, only then do we start sliding the drawer in.

```swift
struct DrawerView<MainContent: View, DrawerContent: View>: View {
    ...
    // 1
    private let dragOpenThreshold = 0.1

    ...
    private func dragGesture(_ mainWidth: CGFloat) -> some Gesture {
        return DragGesture().onChanged { value in
            if isOpen, value.translation.width < 0 {
                openFraction = openFraction(value.translation.width, from: -mainWidth...0)
            // 2
            } else if !isOpen, value.startLocation.x < mainWidth * dragOpenThreshold, value.translation.width > 0 {
                openFraction = openFraction(value.translation.width, from: 0...mainWidth)
            }
        }.onEnded { value in
            // 3
            if openFraction == 1 || openFraction == 0 {
                return
            }
            ...
    }
```

1. We introduce a new property `dragOpenThreshold`. This is a fractional value between 0 to 1, much like `openFraction`.
2. While opening, we consider the drag gesture to be valid only if the X co-ordinate of gesture's start location is less than ` mainWidth * dragOpenThreshold`.
3. We add a safety check in `onEnded` to not update the drawer state if `onChange` modifier doesn't manage to change the value of `openFraction`.

### Conclusion

And that is pretty much it! Source code for this post can be found [here](https://github.com/iTwenty/DrawerView).

---

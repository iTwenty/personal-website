---
title: "SwiftUI List selection bug with NavigationStack"
date: 2026-05-05T19:13:28+05:30
draft: true
tags: [swiftui,ios]
authors: [itwenty]
---

Recently, I ran into a fairly annoying SwiftUI bug while working with a multi-select List.
The setup is pretty straightforward -

- A `NavigationStack`
- A `List` with [multi selection](https://developer.apple.com/documentation/swiftui/list/init(selection:content:)-4sffx) enabled
- A button that toggles `EditMode`
- A separate button that pushes a new View on the `NavigationStack`

Here's the minimal code to demonstrate the setup and the bug -

```swift
import SwiftUI

enum Route: Hashable {
    case profile
    case detail(Int)
}

struct ContentView: View {
    @State private var selected = Set<Int>()
    @State private var navigation = NavigationPath()

    var body: some View {
        NavigationStack(path: $navigation) {
            List(0..<100, selection: $selected) { index in
                Text("Row \(index)")
            }
            .safeAreaInset(edge: .bottom) {
                Text("Selected: \(selected.count)")
                    .padding()
                    .frame(maxWidth: .infinity)
                    .background(.ultraThickMaterial)
            }
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    EditButton()
                }

                ToolbarItem(placement: .topBarTrailing) {
                    Button("Profile") {
                        navigation.append(Route.profile)
                    }
                }
            }
            .navigationDestination(for: Route.self) { route in
                switch route {
                case .profile: Text("Profile Screen")
                case .detail(let index): Text("Details for \(index)")
                }
            }
        }
    }
}
```

And here's the GIF showing the bug in action -

// TODO Insert GIF or video

The gist of it is -

- Select a few rows in the List in edit mode
- Push a new View on the NavigationStack while still in edit mode with rows selected
- Go back to the List
- List no longer highlights the rows you have selected, even though they are still present in the underlying State.

Once you select any other row, the highlights come back. I guess this is because of the List refreshing it's content
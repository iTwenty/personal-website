---
title: "Read workouts using Healthkit - and keep them updated!"
summary: "Learn how to read workouts data using Healthkit and keep the data synced with further updates."
date: 2024-10-13T12:03:05+05:30
draft: false
tags: []
authors: [itwenty]
---

Recently, I got a chance to dive deep into Apple's Healthkit and Workout related APIs while developing [MergeFit](https://apps.apple.com/us/app/mergefit/id6714483581).

The app's home screen displays a list of user's recent workouts. Since the app displays workouts that can be created, updated or deleted externally by other apps at any time, it is important to keep the workouts list synced with these external changes. In this post, I outline the approach I took to achieve this goal since I found the official Apple docs a bit unclear on this topic.

Before we dive into the details, let’s start with a quick introduction to how HealthKit represents workouts.

#### Intro to Healthkit & HKWorkout

Healthkit splits user's health data into two fundamental types - Data that can't change over time and data that can.

- **Can't change** - Blood type, Date of birth, Biological sex etc. These are called Characteristics.
- **Can change** - Height, Weight, Heart Rate etc. These are called Samples.

Samples are represented by the abstract class `HKSample`. Each sample has an associated start and end time. For some samples, like height or weight, these times are the same because the data is captured at a single moment.

Samples are further subdivided into four concrete types -

1. `HKCategorySample`: This is used for data that can only have a fixed set of values. For example, sleep type (awake, light sleep, deep sleep, REM etc) or the result of a pregnancy test (positive, negative, or indeterminate).
2. `HKQuantitySample`: This type handles data that involves a numeric value and unit. Examples include height (measured in units of length) or heart rate (measured in counts/min).
3. `HKCorrelation`: This sample type is specifically for grouping related data, like correlating blood pressure readings or food intake. I haven’t worked much with this type yet.
4. `HKWorkout`: This type is specifically for workouts. The start and end times of the sample represent the beginning and end of the workout session.

Workout samples are a bit special inside Healthkit in that they can have other sample types associated with them. This makes sense since we usually do need the associated heart rate, energy burned, distance covered, route taken etc. to get a full picture of any workout.

***
**NOTE**: I highly recommend watching the 2014 WWDC talk "[Introduction to Healthkit](https://www.youtube.com/watch?v=F0zSHz-dxrg)" to get a full idea about what I glossed over very quickly in this section. There's a lot more to Healthkit and this talk does a great job covering it all from the ground up.
***

#### Reading workout data using HKSampleQuery

To read HKWorkout samples within a specific time period - let's say last 365 days - we can use `HKSampleQuery`.

```swift
import Foundation
import HealthKit
import Observation

@Observable
@MainActor
final class HKWrapper {
    @ObservationIgnored private let store = HKHealthStore()
    var workouts: [HKWorkout]?

    // 1
    private func samplesPredicate() -> NSPredicate {
        let oneYearAgo = Calendar.current.date(byAdding: .day, value: -365, to: .now)
        return HKQuery.predicateForSamples(withStart: oneYearAgo, end: .now)
    }

    // 2
    private func workoutSamplesPredicate() -> HKSamplePredicate<HKWorkout> {
        HKSamplePredicate.workout(samplesPredicate())
    }

    // 3
    func readWorkouts() async {
        let query = HKSampleQueryDescriptor(predicates: [workoutSamplesPredicate()],
                                            sortDescriptors: [SortDescriptor(\.startDate, order: .reverse)])
        do {
            workouts = try await query.result(for: store)
        } catch {
            print(error.localizedDescription)
        }
    }
}
```

1. `samplesPredicate()` constructs a general predicate that can be used to query any data for last 365 days.
2. `workoutSamplesPredicate()` wraps the samples predicate to make it specific to workout samples.
3. `readWorkouts()` uses the predicate and a specifies a sort order to query healthkit data store for workouts.

The `Descriptor` suffix to `HKSampleQuery` is a Healthkit convention for structured concurrency variants of closure-based queries. Sample queries return a snapshot of Healthkit data matching the predicate at the moment the query is executed. Since we want the initial list of workouts, as well as any subsequent updates, we need to look at other types of queries.

#### Reading workout data using HKAnchoredObjectQuery

Healthkit has two different types of queries for handling data updates - `HKObserverQuery` and `HKAnchoredObjectQuery`. Both are long running queries, meaning they continue running in the background after we execute them. Observer queries are older and limited by the fact that they can only notify the app when updates happen, but don't tell what data changed. We need to run another query from within the observer query's callback to get updated data. Anchored object queries are more versatile in this regard and can be used for both initial data fetch, as well as subsequent updates to the fetched data.

Let's set up an anchored object query for both initial workouts fetch, as well as any updates that follow. Initial data fetch part is a simple change to our `readWorkouts()` method -

```swift
private var workoutsAnchor: HKQueryAnchor? = nil

func readWorkouts() async {
    let query = HKAnchoredObjectQueryDescriptor(predicates: [workoutSamplesPredicate()],
                                                anchor: workoutsAnchor)
    do {
        let result = try await query.result(for: store)
        workoutsAnchor = result.newAnchor
        workouts = result.addedSamples.sorted { lhs, rhs in
            lhs.startDate > rhs.startDate
        }
    } catch {
        print(error.localizedDescription)
    }
}
```

The returned resut is not an array like for sample queries, but a `Result` type which contains three properties:

- `addedSamples` contains all samples added since last time query was run, or all samples matching the predicate on first run.
- `deletedObjects` contains all samples deleted since last time query was run. It is empty on first run.
- `newAnchor` is an opaque HKQueryAnchor type used for batching samples reads.

Unlike HKSampleQuery, anchored object query doesn't have a way to specify sort order in the query itself. So we need to sort the results explicitly after receiving them.. We are not reading workouts in batches, but we still need to save the anchor since we will use it for listening to workouts updates, which we will set up next.

#### Keeping workout data updated while app is running

Once we have fetched the initial list of workouts and saved the anchor, we can run *another* anchored query whose result is a long lived AsyncSequence. Let's write a function to fetch this sequence and handle it's results -

```swift
// 1
@ObservationIgnored private var workoutUpdatesTask: Task<Void, Never>?

func startWorkoutUpdatesTask() {
    guard let anchor = workoutsAnchor else {
        return
    }
    // 2
    workoutUpdatesTask?.cancel()
    let query = HKAnchoredObjectQueryDescriptor(predicates:[workoutSamplesPredicate()],
                                                anchor: anchor)
    let results = query.results(for: store)
    workoutUpdatesTask = Task {
        do {
            for try await result in results {
                try Task.checkCancellation()
                // 3
                let addedCount = result.addedSamples.count
                let deletedCount = result.deletedObjects.count
                if addedCount == 0, deletedCount == 0 {
                    continue
                }
                var updatedWorkouts = workouts
                updatedWorkouts?.append(contentsOf: result.addedSamples)
                updatedWorkouts?.removeAll { workout in
                    result.deletedObjects.contains { $0.uuid == workout.uuid }
                }
                updatedWorkouts?.sort { lhs, rhs in
                    return lhs.startDate > rhs.startDate
                }
                workouts = updatedWorkouts
            }
        } catch {
            print(error.localizedDescription)
        }
    }
}
```

1. We add a Task to keep a handler for the async sequence we will be starting.
2. We use this task handler to cancel the previous task before starting a new long running task.
3. Everytime a workout is updated outside of our app, we will receive the update as a `Result` type we saw in previous section. We need to handle both added and deleted samples inside the loop.
4. We don't need to update the anchor here since it will be the same as the one we passed in.

The way we handle added and deleted samples is open to any number of implementations. I have opted for a brute force approach of just appending new items to the existing sorted list and then re-sorting it. Using something like binary search to find proper insertion index will be faster, but this approach is fast enough as it is.

Next, we need a trigger point from where to start this long running task. A good place is right after our initial fetch query succeeds -

```swift
func readWorkouts() async {
    let query = HKAnchoredObjectQueryDescriptor(predicates: [workoutSamplesPredicate()],
                                                anchor: workoutsAnchor)
    do {
        let result = try await query.result(for: store)
        workoutsAnchor = result.newAnchor
        workouts = result.addedSamples.sorted { lhs, rhs in
            lhs.startDate > rhs.startDate
        }
        // Start the updates task here
        startWorkoutUpdatesTask()
    } catch {
        print(error.localizedDescription)
    }
}
```

At this point, our list will stay updated with any workout changes, as long as the app is running. Once our app moves to background - but is not terminated - our update task will be cancelled. This is one of the limitations of anchored object queries. They only run as long as the app itself is running. To enable delivery of updates while app is in background, we need to use a third type of query - `HKObserverQuery`.

#### Background delivery of workout updates using observer queries

Observer queries - limiting as they are - have a nice trick up their sleeve. They can receive updates even when the app is in background or terminated. When an update occurs, iOS will deliver it to our observer query, silently launching the app for a few seconds if it isn't running. Before setting them up, we need to enable background delivery by adding an [entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com_apple_developer_healthkit_background-delivery?language=objc) to the app. Once added, we can set up query using the same predicate we used for our anchored object query -

```swift
func enableBackgroundWorkoutUpdates() async {
    let workoutType = HKObjectType.workoutType()
    do {
        // 1, 2
        try await store.enableBackgroundDelivery(for: workoutType, frequency: .immediate)
    } catch {
        print(error.localizedDescription)
    }

    let backgroundQuery = HKObserverQuery(sampleType: workoutType,
                                          predicate: samplesPredicate()) { query, completion, optError in
        if let error = optError {
            print(error.localizedDescription)
            return
        }
        
        // 3
        UserDefaults.standard.setValue(true, forKey: "update_workouts")

        // 4
        completion()
    }

    store.execute(backgroundQuery)
}
```

There doesn't seem to be an async/await variant of observer query, so we use the closure-based variant. There are a few things to keep in mind when setting up observer queries -

1. We first need to enable background delivery in code (in addition to the entitlement). 
2. The `frequency` parameter lets us specify how often the system should deliver the updates to our app. Since we want workout updates to be delivered as soon they happen, we specify `.immediate`.
3. As mentioned earlier, observer queries only tell us when the data changes, but not what changed. We can run an anchored object query within the completion handler of the observer query, or we can just set a flag like I did and refresh the workout list on app launch based on this flag.
4. It is important to call the completion block when we are done to let system know we have successfully processed this background update. If we don't call this block, Healthkit will continue delivering the update to our app using an exponential backoff. After trying 3 times, Healthkit will assume our app can't process the updates and stop delivery them altogether.
5. We should call this method as early as possible in our app's lifecycle. `application:didFinishLaunchingWithOptions:` is a good place for it.

#### Outro

By using a combination of three different queries - anchored object query for initial data fetch, another anchored object query for listening to foreground updates and an observer query for background updates - we have ensured our workouts list stays updated at all times. An argument can be made for forgoing this complexity and simply using a sample query with an option for manually refreshing the list in the app's UI. In fact, this was the approach I took in the initial stages of developing MergeFit. But having the app stay up to date without any input from user's end makes for a nicer end user experience. A basic functional app for the code mentioned in this post, with a simple UI for displaying workouts, can be found [here](https://github.com/iTwenty/WorkoutsFetch).
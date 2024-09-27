---
title: "Read workouts using Healthkit - and keep them updated!"
date: 2024-09-27T22:25:05+05:30
draft: true
tags: []
authors: [itwenty]
---

Recently, I got a chance to dive deep into Apple's Healthkit and Workout related APIs while developing [MergeFit](https://apps.apple.com/us/app/mergefit/id6714483581).

The app's home screen displays a list of user's recent workouts. Since the app displays workouts that can be created, updated or deleted externally by other apps at any time, it is important to keep the workouts list synced with these external changes. In this post, I outline the best approach to achieve this goal since I found the official Apple docs lacking in clarity.

Before we get to the juicy bits, here's a (very) quick intro to how Healthkit represents workouts.

### Intro to Healthkit & HKWorkout

Healthkit splits user's health info into two fundamental types - Data that can't change over time and data that can.

Can't change - Blood type, Date of birth, Biological sex etc. These are called Characteristics.
Can change - Height, Weight, Heart Rate etc. These are called Samples.

Samples are represented by the class `HKSample`. Each sample has an associated start and end time. The times can be the same if the sample is measured in an instant - like height, weight etc.

Samples are further subdivided into four types -

### Reading workout data

To read HKWorkout samples within a specific time period - let's say last three months - we can use `HKSampleQuery`.

```swift
// Predicate that can be used to query any samples over last 3 months.
private func samplesPredicate() -> NSPredicate {
    let threeMonthsAgo = Calendar.current.date(byAdding: .month, value: -3, to: .now)
    return HKQuery.predicateForSamples(withStart: threeMonthsAgo, end: .now)
}

// Predicate for workouts in last 3 months.
private func workoutSamplesPredicate() -> HKSamplePredicate<HKWorkout> {
    HKSamplePredicate.workout(samplesPredicate())
}

func readWorkouts() async  -> [HKWorkout]? {
    let query = HKSampleQueryDescriptor(predicates: [workoutSamplesPredicate()],
                                        sortDescriptors: [SortDescriptor(\.startDate, order: .reverse)])
    do {
        let result = try await query.result(for: store)
        return result
    } catch {
        return nil
    }
}
```

***
**NOTE:** The `Descriptor` suffix to queries is Healthkit convention for structured concurrency variants or closure-based queries. We will be using structured concurrency variants of queries throughout this post.
***

This gives us the workouts, but doesn't update the list when workouts are updated outside the app. 

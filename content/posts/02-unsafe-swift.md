---
title: "Exploring memory layout in Swift"
date: 2020-08-16T19:46:16+05:30
draft: true
tags: [ios, swift]
---

### Memory Layout of structs
Swift has an enum called `MemoryLayout` that provides information about a type's size in memory, it's alignment and it's stride, all in bytes. To understand what each of these means, let's define a struct with two properties and print out it's MemoryLayout -

```swift
struct TestStruct {
    let integer: Int32
    let boolean: Bool
}


print(MemoryLayout<TestStruct>.size)
print(MemoryLayout<TestStruct>.alignment)
print(MemoryLayout<TestStruct>.stride)
```

This prints out 

```
5
4
8
```
The size of struct is the sum of size of it's individual properties. `TestStruct` has a Int32 property which takes 4 bytes and a Bool property which takes 1 byte. So the size of struct itself is 5 bytes.

When multiples structs are placed sequentially in memory (like in an array), structs really like to placed at a standard distance from each other, padding the remaining bytes as necessary. This distance is denoted by alignment.

The actual memory footprint of a struct is a multiple of alignment, rather than size. This memory footprint is called stride.

Visually, multiple instances of TestStruct in an array `a` would look something like this in memory -



### Memory related APIs.
Swift provides access to pointers in a type safe way. 
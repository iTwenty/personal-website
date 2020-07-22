---
title: "Understanding dyld @executable_path, @loader_path and @rpath"
date: 2020-07-10T10:39:18+05:30
draft: false
tags:
  - ios, compiler
---
If you have added any third party dynamic framework to an iOS app, you might have run into a cryptic error that reads something like:

```
dyld: Library not loaded: @rpath/TestKit.framework/TestKit
 Referenced from: <long_path_name>/TestApp.app/TestApp
 Reason: image not found
```

What is this `@rpath` thing in the error message? `@rpath` stands for **R**unpath search **path**. To understand what it means and why we need it, we need to take a step back and look at how dynamic libraries (called dylibs in macOS and iOS world) link with other dylibs and executables. We also need to understand the meaning of `@executable_path` and `@loader_path` before looking into `@rpath` This is best demonstrated by examples.

***
**NOTE :** I am using C for example code, but the concept stays the same for both Objective C and Swift. Reason for choosing C is easier compilation from command line.
***

### What is @executable_path?

Starting in a fresh empty directory - `~/tests/blog/` in my case - let's create two test files -

Cat.c
```c
#include <stdio.h>

void catSound() {
  printf("MEOW!\n");
}
```

main.c
```c
void catSound();

int main(int argc, char** argv) {
  catSound();
  return 0;
}
```

Compile `Cat.c` into a dylib and `main.c` into an executable.

```
❯❯❯❯ clang -dynamiclib Cat.c -o libCat.dylib
❯❯❯❯ clang -L. -lCat main.c -o main
```

Here, `-L` stands for library search path. `-l` specifies the name of dylib to link against, without the `lib` prefix and `.dylib` suffix. `-o` specifies the name of final output file.

Let's go ahead and run the `main` executable.

```
❯❯❯❯ ./main
MEOW!
```

We get the cat sound, as expected. Let's inspect how the dylib has been linked to our executable. We can use `otool` for this task. The `-L` option prints paths to all dynamic libraries used by an executable. Let's call these paths install paths.

```
❯❯❯❯ otool -L main
main:
	libCat.dylib (compatibility version 0.0.0, current version 0.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1281.100.1)
```

The first install path `libCat.dylib` is a relative path. This means our `main` executable expects to find `libCat.dylib` in same directory as it is executed from. If we try to run `main` from any other directory, we get an error.

```
❯❯❯❯ cd ~/tests/
❯❯❯❯ ./blog/main
dyld: Library not loaded: libCat.dylib
 Referenced from: /Users/jaydeep/tests/./blog/main
 Reason: image not found
[1]  29123 abort   ./blog/main
```

We can change the relative install path `libCat.dylib` in `main` using a utility called `install_name_tool`[^install_name_tool]. But what should we change it to? We could change it to absolute path, and that would work on our system, but if you distribute `main` and `libCat.dylib` to someone else, it will likely fail on their system.

Solution here is to use a special variable called `@executable_path`. When dyld encounters this variable at link time, it gets resolved to path of directory containing the executable. In my case since `main` is in `~/tests/blog/`, `@executable_path` resolves to `~/test/blog/`. Let's fix the install path using `install_name_tool` -

```
❯❯❯❯ install_name_tool -change libCat.dylib @executable_path/libCat.dylib main
```

Let's run `main` from `~/tests/` directory again. 

```
❯❯❯❯ cd ~/tests/
❯❯❯❯ ./blog/main
MEOW!
```
Great!

### What is @loader_path?

To understand `@loader_path`, we need to add some complexity to our test case. Let's create another dylib in a different directory, and make this dylib depend on our Cat dylib.

Animal/Animal.c
```c
void makeCatSound();

void makeAnimalSound() {
  makeCatSound();
}
```

Compile `Animal/Animal.c` into a dylib. We will need to link it to `libCat.dylib`.

```
❯❯❯❯ clang -dynamiclib -L. -lCat Animal/Animal.c -o Animal/libAnimal.dylib
```

Modify `main.c` to make generic animal sounds rather than cat sounds, and recompile it. We will need to link against `libAnimal` rather than `libCat`.

```c
void animalSound();

int main(int argc, char** argv) {
  animalSound();
  return 0;
}
```

```
❯❯❯❯ clang -LAnimal -lAnimal main.c -o main
```

Just like before, we know that we can execute `main` from `~/tests/blog/`, but not from `~/tests` or any other directory. We also know the fix for this. So let's go ahead and do that.

```
❯❯❯❯ install_name_tool -change Animal/libAnimal.dylib @executable_path/Animal/libAnimal.dylib main
```

Running `main` from `~/tests/` fails this time though.

```
❯❯❯❯ cd ~/tests/
❯❯❯❯ ./blog/main
dyld: Library not loaded: libCat.dylib
 Referenced from: /Users/jaydeep/tests/blog/Animal/libAnimal.dylib
 Reason: image not found
[1]  29869 abort   ./blog/main
```

The error tells us that `libAnimal` could not find `libCat`. Let's check the install paths for `libAnimal.dylib`

```
❯❯❯❯ otool -L Animal/libAnimal.dylib
Animal/libAnimal.dylib:
	Animal/libAnimal.dylib (compatibility version 0.0.0, current version 0.0.0)
	libCat.dylib (compatibility version 0.0.0, current version 0.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1281.100.1)
```

We need to fix the relative install path to `libCat.dylib` here. But what should we change it to? Using `@executable_path` would work - but for our `main` executable only. Dylibs are meant to shared across muliple clients[^clients], all of which can be in different paths. This means `@executable_path` will resolve to different values depending on the executable being run.

Let's take a step back and look at the dependency tree we have. `main` depends on `libAnimal` and `libAnimal` depends on `libCat`. `libCat` doesn't depend on anything.[^libSystem]

```
libCat.dylib <--- Animal/libAnimal.dylib <--- main
```

`@executable_path` will always resolve to path of `main`. No matter if it's `main` doing the loading of `libAnimal`, or `libAnimal` doing the loading of `libCat`. dyld provides another variable - `@loader_path` - that resolves to path of client doing the loading. In the above tree there are two loads happening. Let's write down the values of the two variables for both loads.

| load        | @executable_path | @loader_path     |
|---------------------|------------------|----------------------|
| main -> libAnimal  | ~/tests/blog/  | ~/test/blog/     |
| libAnimal -> libCat | ~/tests/blog/  | ~/test/blog/Animal/ |

Knowing this, we can now change the install path in Animal dylib from `libCat.dylib` to `@loader_path/../libCat.dylib`

```
❯❯❯❯ install_name_tool -change libCat.dylib @loader_path/../libCat.dylib Animal/libAnimal.dylib
```

Running `main` will now succeed from any directory. In fact, if you were to add a new executable `foo/main` that depends on `libAnimal`, you would only need to set the install path for `libAnimal.dylib` in `foo/main` itself[^third_party_dylibs]. No change is needed to either `libCat` or `libAnimal` as long as relative path between them remains the same, but that is unavoidable. They are now "shared libraries" in true sense of the word.

> **NOTE** : For executables, `@loader_path` and `@executable_path` mean the same thing.

### Short note on Install ID

Before moving on to `@rpath`, let's understand a small concept called install IDs. Check the `otool -L` output for any dylib

```
❯❯❯❯ otool -L Animal/libAnimal.dylib
	Animal/libAnimal.dylib (compatibility version 0.0.0, current version 0.0.0)
	@loader_path/../libCat.dylib (compatibility version 0.0.0, current version 0.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1281.100.1)
```

For dylibs, the first entry is not an install path, but rather an install ID. When another client links to this dylib, it is the dylib's install ID that gets copied as install path in the client.

### What is @rpath?

In a large project with muliple clients in different locations depending on each other, having to keep track of `@loader_path` can get messy quickly. In such cases, we can use `@rpath`. Unlike the two variables we have seen till now, `@rpath` doesn't have any special meaning to dyld. It is upto us to define a value (or values) for `@rpath` for each client. `@rpath` adds a level of indirection that can simplify things. 

Let's modify our test case to make use of `@rpath`. Add another executable `foo/main.c` with same source as `main.c`. We won't compile it yet. Our directory structure looks like this :

```
❯❯❯❯ tree blog
blog
├── Animal
│   ├── Animal.c
│   └── libAnimal.dylib
├── Cat.c
├── foo
│   └── main.c
├── libCat.dylib
├── main
└── main.c
```

First step is to pick a path as an anchor path in our directory structure. Let's pick `~/tests/blog/` as our anchor.

Next, let's change the dylib install IDs to `@rpath/zzz` where zzz is relative path from anchor to the dylib.

For `libCat.dylib` this works out to `@rpath/libCat.dylib`
```
❯❯❯❯ install_name_tool -id @rpath/libCat.dylib libCat.dylib
```
For `Animal/libAnimal.dylib` this works out to `@rpath/Animal/libAnimal.dylib`
```
❯❯❯❯ install_name_tool -id @rpath/Animal/libAnimal.dylib Animal/libAnimal.dylib
```

Next, let's add an `@rpath` to our executables with value equal to `@loader_path/zzz` where zzz is relative path from executable to our anchor.

For `foo/main.c` this works out to `@loader_path/../`

```
❯❯❯❯ clang -LAnimal -lAnimal -rpath "@loader_path/../" foo/main.c -o foo/main
```

For `main.c` this works out to `@loader_path`. We can use `install_name_tool` to add `@rpath` to the compiled executable, but since the install IDs for `libAnimal` and `libCat` have changed after `main.c` was compiled, it's best to re-compile and re-link it so that it picks up the updated IDs.

```
clang -LAnimal -lAnimal -rpath "@loader_path" main.c -o main
```

After these changes, we can move our `~/test/blog/` directory anywhere, even a different system, and the two executables will continue to run - as long as the directory structure within `~/tests/blog/` itself doesn't change.

Note that you can define more than one `@rpath` value for an executable, either at link time or later using `install_name_tool`. dyld will try all values in order to check for existence of dylibs.

### Back to the error message

Knowing all this, we are now in a far better position to understand what the initial error message means, and how to go about fixing it. Let's analyze the error -

```
dyld: Library not loaded: @rpath/TestKit.framework/TestKit
 Referenced from: <long_path_name>/TestApp.app/TestApp
 Reason: image not found
```

The executable is `<long_path_name>/TestApp.app/TestApp`. Dylib is `TestKit`. The executable could not find the dylib at `@rpath/TestKit.framework/TestKit`.

For iOS apps, all third party frameworks reside inside a `Frameworks` directory inside app directory. So the actual path to dylib is `<long_path_name>/TestApp.app/Frameworks/TestKit.framework/TestKit`. The anchor directory is `<long_path_name>/TestApp.app/Framewoks/`. So to figure out the reason for this error, we can check two things -

- `<long_path_name>/TestApp.app/TestApp` has `@rpath` value of `@loader_path/Frameworks/`. `@executable_path/Frameworks/` works too since both mean the same thing for executables. If you have the source code, you can check this in target's build settings (`LD_RUNPATH_SEARCH_PATHS`). If not, `otool -l` is your friend.
- `TestKit` dylib has install ID of `@rpath/TestKitFramework.framework/TestKit`.  If you have the source, check this in build settings (`LD_DYLIB_INSTALL_NAME`). If not, `otool -l` is your friend again.
- `TestKit.framework` is actually present inside `Frameworks` directory. You need to embed the framework inside the app for this.

When you add a new framework to an app, Xcode takes care of all this settings for you. But it's good to know what's happening behind the scenes :)

---
`man dyld` offers a lot more insight into the things glossed over in this post.

[^install_name_tool]: Check out the manpage for `install_name_tool` to see what all it can do. It's a short read.

[^libSystem]: `libCat` depends on `libSystem` which is located at `/usr/lib/libSystem.dylib`. We can ignore this for our purpose.

[^third_party_dylibs]: For any client, having correct install paths to all dylibs it depends on is unavoidable, unless the dylib is present at a standard location like `/usr/lib/`.

[^clients]: Client can mean either an executable or a dylib.
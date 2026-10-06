# zenoh in Dart and Flutter

[zenoh](https://zenoh.io) is a protocol for publishing, subscribing and querying data, and it runs on everything from
servers to microcontrollers [1]. A program asks for data by its key and receives it without knowing where the
publisher runs [1]. Programs connect peer to peer, through routers, or in a mix of the two, decided when they run [1].
In this guide, a phone publishes what its accelerometer reads, and a program on your laptop watches it.

You build the two programs by hand, chapter by chapter, and run them against each other:

- **`sensorctl`**, a Dart command-line application on your laptop. It is the collector, the program the operator uses.
  It watches a stream of sensor readings, asks for their history, sets the rate they arrive at, and reports which nodes
  are alive. Its terminal is its user interface, built the way an application's screen is.
- **`sensor_node`**, a Flutter application on Android. It is the sensor node. It reads the device's accelerometer and
  gyroscope, publishes what it reads, answers the collector's questions and takes its commands.

By chapter 3 the two are talking. The phone publishes what its accelerometer reads, and the program on your laptop
displays it. From chapter 4, `sensorctl` can also run a sensor node of its own, a stand-in that takes the phone's
place when no device is at hand. By chapter 5 the phone publishes over your Wi-Fi network.

Both programs follow the same architecture, MVVM, and share one pure-Dart package, which holds their zenoh code. They
are built with test-driven development, TDD. From chapter 1 on, every behavior starts as a test that fails, and a test
that pins what already works is written green, with the chapter saying so. You type every line of your own code. The
tools create each project's starting files, and some of the package's example programs are copied in to run.

## The chapters

| # | chapter | what you build |
|---|---|---|
| 0 | **[Getting started](chapters/00-getting-started.md)** | the toolchain, a git repository holding a Dart project that depends on `zenoh_dart`, and two of the package's example programs talking to each other in two terminals |
| 1 | **[A session of your own](chapters/01-a-session-of-your-own.md)** | a session opened with a configuration you wrote, behind `ZenohService`, in a core package that a pub workspace shares with the phone app to come; your first tests, red then green; the program wired by a provider container |
| 2 | **[The node on your phone](chapters/02-the-node-on-your-phone.md)** | the Flutter app `sensor_node`, in the same workspace, publishing the phone's accelerometer on `sensor/phone/accel` through the core; the chapter's claim tested against real zenoh, and the app built from its screen in; the node run on the emulator and then on a phone over its USB cable, with the package's `z_sub` receiving on the laptop |
| 3 | **[The collector on your laptop](chapters/03-the-collector-on-your-laptop.md)** | `sensorctl`'s first command, `watch`: a subscription behind `ZenohService` and a repository that turns each payload back into a reading, then a view that redraws one line in place, a view model and the providers; the chapter's claim tested against real zenoh, and the program built from its terminal in; `watch` run against the node on the emulator |

The rest are being written. Chapter 4 gives `sensorctl` a sensor node that needs no device. Chapter 5 takes the phone
onto your Wi-Fi network, with a router for a network where the laptop cannot reach the phone. The chapters after it
add one thing each: serialization, queryables, queries, commands, liveliness, quality of service, stopping properly, a
second view, and scouting and security.

The chapters take zenoh's ideas in the order the two programs need them. For what each idea is, the reference is
zenoh.io's documentation [1], and each chapter's reading list gives its page first, then the books' [2], [3].

## Before you start

- **A Linux machine on x86_64, with glibc 2.34 or newer.** `zenoh_dart` ships zenoh's native library for Linux on
  x86_64 and for Android, so the laptop side of this guide needs Linux on x86_64.
- **Some Dart, and enough Flutter to have finished Flutter's first codelab.** The guide does not teach the languages.
- **No zenoh needed.** If you do know zenoh already, from C, C++, Python, Rust or ROS 2, the chapters carry short asides
  that say what is the same here and what is not.
- **An Android emulator and an Android phone, from chapter 2 on**, with `adb`. The emulator's system image must be
  `x86_64`, and a phone needs API 24 or newer. The phone runs on a USB cable first, and on your Wi-Fi network from
  chapter 5.
- **Everything else you install yourself.** Chapter 0 names each tool with the version it was checked with: git, fvm,
  the Flutter SDK, which fvm fetches, and VS Code, which is optional. Chapter 0 starts from an empty folder.

Chapter 0 ends with the exact versions it was checked with, and each later chapter with the versions that changed.

## What the guide builds on

Each chapter's reading list tags a source with its number here.

- [1] Eclipse Foundation, "Zenoh documentation," *zenoh.io*, 2026, for zenoh 1.x. Accessed: Oct. 1, 2026. [Online].
  Available: <https://zenoh.io/docs/>

  Zenoh's own documentation, the reference for what each idea is. Where it and a book disagree, the guide follows it.
- [2] A. Corsaro, *The Zenoh Book*, 2026. Accessed: Oct. 1, 2026. [Online]. Available:
  <https://corsaro.me/zenoh/book/>

  The ideas, what a thing is and why it exists. The chapters take its vocabulary, with the author's permission, and
  nothing is copied from it.
- [3] A. Corsaro, *Zenoh Programming in Rust*, draft, written for Zenoh 1.4.0, 2025. Accessed: Oct. 1, 2026.
  [Online]. Available: <https://kydos.github.io/zenoh-book/>

  The shape of the API and its options, chapter by chapter. Its examples are in Rust, and nothing is copied from it.
- [4] bluecorn, *zenoh_dart*, version 1.0.0-rc.1, the Dart binding for zenoh, built on zenoh-c 1.8.0, 2026. [Online].
  Available: <https://pub.dev/packages/zenoh_dart>

  Most chapters start from one of the programs in its `example/` folder. You run it first and see the behavior, or read
  its code when the chapter before has just run it, and then build that idea into the application. Every statement
  about the Dart API is checked against the package's source at its version.
- [5] Google, "Guide to app architecture," *Flutter documentation*, May 5, 2026. Accessed: Oct. 1, 2026. [Online].
  Available: <https://docs.flutter.dev/app-architecture/guide>

  The MVVM layering both programs use.

## Copyright and license

Every fenced code block and every file listing in this guide is licensed under the **Apache License 2.0**
([LICENSE](LICENSE)). Use them in any project, including a commercial one, with no further permission. The
surrounding text is **© 2026 Hugo Alberto Garcia, all rights reserved**. Read it, follow it and link to it freely. To
mirror, translate or reuse it elsewhere, open an issue and ask. The full statement is in [COPYRIGHT](COPYRIGHT).

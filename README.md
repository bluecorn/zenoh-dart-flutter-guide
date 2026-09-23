# zenoh in Dart and Flutter

A guide that teaches [zenoh](https://zenoh.io) in Dart and Flutter. You build two programs by hand, chapter by chapter,
and run them against each other:

- **`sensorctl`**, a Dart command-line application on your laptop — the collector, the program the operator uses. It
  watches a stream of sensor readings, asks for their history, sets the rate they arrive at, and reports which nodes are
  alive. Its terminal is its user interface, built the way an application's screen is.
- **`sensor_node`**, a Flutter application on Android — the sensor node. It reads the device's accelerometer and
  gyroscope, publishes what it reads, answers the collector's questions and takes its commands.

By chapter 3 the two are talking: the phone publishes what its accelerometer reads, and the command-line program on
your laptop displays it. From chapter 4 a stand-in sensor node inside `sensorctl` takes the phone's place whenever no
device is at hand, and the program watching it cannot tell the difference — which is most of what zenoh is for.

Both programs follow the same architecture, MVVM, and share one pure-Dart package that holds everything touching zenoh.
You write every line yourself; nothing here is generated or cloned.

## The chapters

| # | chapter | what you build |
|---|---|---|
| 0 | **[Getting started](chapters/00-getting-started.md)** | the toolchain, a git repository holding a Dart project that depends on `zenoh_dart`, and two of the package's example programs talking to each other in two terminals |
| 1 | **[A session of your own](chapters/01-a-session-of-your-own.md)** | a session opened with a configuration you wrote, behind `ZenohService`, the one class that imports the package, in a core package that a pub workspace shares with the phone app to come; your first tests, red then green; the program wired by a provider container |

The rest are being written. Chapter 2 puts the sensor node on the Android emulator and then on a phone, and chapter 3
gives the laptop the program that watches it; from there each chapter adds one zenoh idea to both programs at once —
the phone on your network, serialization, queryables, queries, commands, liveliness, quality of service, lifecycle —
and the last one goes beyond your network.

## Before you start

- **A Linux machine on x86_64.** `zenoh_dart` ships zenoh's native library for Linux on x86_64 and for Android, so the
  laptop side of this guide needs one of those, with glibc 2.34 or newer.
- **Some Dart, and enough Flutter to have finished Flutter's first codelab.** The guide does not teach the languages.
- **No zenoh needed.** If you do know zenoh already, from C, C++, Python, Rust or ROS 2, the chapters carry short asides
  that say what is the same here and what is not.
- **An Android emulator and an Android phone, from chapter 2 on**, with `adb`. The emulator's system image must be
  `x86_64`, which is one of the three Android architectures `zenoh_dart` ships a library for; the phone works as it
  is, on a USB cable first and on your Wi-Fi network from chapter 5.
- Everything else — the SDK, the editor — is named in the chapter that first needs it, with the version it was checked
  with; installing it is yours to do. Chapter 0 starts from an empty folder.

Each chapter ends with the exact versions it was checked with, and asks for those or newer.

## What the guide builds on

- **[`zenoh_dart`](https://github.com/bluecorn/zenoh_dart)**, the Dart binding for zenoh, published on
  [pub.dev](https://pub.dev/packages/zenoh_dart). Each chapter starts from one of the programs in the package's
  `example/` folder: you run it first, see the behaviour, and then build that idea into the application.
- **[The Zenoh Book](https://corsaro.me/zenoh/book/)** by Angelo Corsaro, for the ideas: what a thing is, why it exists
  and when to use it. The chapters follow its order, name the pages to read instead of explaining a concept a second
  time, and take their vocabulary from it, with the author's permission. Nothing is copied from it.
- **[Zenoh Programming in Rust](https://kydos.github.io/zenoh-book/)**, by the same author, for the shape of the API and
  its options, chapter by chapter. It is a draft written against Zenoh 1.4.0 and its examples are in Rust; nothing is
  copied from it. Where this guide states something about the Dart API, it has been read in `zenoh_dart` itself.
- **[Flutter's architecture guide](https://docs.flutter.dev/app-architecture/guide)**, for the MVVM layering both
  programs use.

## Copyright and licence

Every fenced code block and every file listing in this guide is licensed under the **Apache License 2.0**
([LICENSE](LICENSE)): use them in any project, including a commercial one, with no further permission. The surrounding
text is **© 2026 Hugo Alberto Garcia, all rights reserved** — read it, follow it and link to it freely; to mirror,
translate or reuse it elsewhere, open an issue and ask. The full statement is in [COPYRIGHT](COPYRIGHT).

# zenoh in Dart and Flutter

A guide that teaches [zenoh](https://zenoh.io) in Dart and Flutter. You build two programs by hand, chapter by chapter,
and run them against each other:

- **`sensorctl`**, a Dart command-line application on your laptop — the operator's tool. It watches a stream of sensor
  readings, asks for their history, sets the rate they arrive at, and reports which nodes are alive. Its terminal is its
  user interface, built the way an application's screen is.
- **`sensor_node`**, a Flutter application on Android — the sensor. It reads the device's accelerometer and gyroscope,
  publishes what it reads, answers the tool's questions and takes its commands.

Part 1 needs no phone: the command-line tool talks to a stand-in sensor you write, and to the example programs that come
with the `zenoh_dart` package. Part 2 replaces the stand-in with a real device, and the tool does not change — which is
most of what zenoh is for.

Both programs follow the same architecture, MVVM, and share one pure-Dart package that holds everything touching zenoh.
You write every line yourself; nothing here is generated or cloned.

## The chapters

| # | chapter | what you build |
|---|---|---|
| 0 | **[Getting started](chapters/00-getting-started.md)** | the toolchain, a git repository holding a Dart project that depends on `zenoh_dart`, and two of the package's example programs talking to each other in two terminals |

The rest are being written, in the order the zenoh book takes: sessions and configuration, publishers, subscribers,
serialization, queryables, queries, liveliness, quality of service and lifecycle — then the Flutter application on
Android, and the network beyond one machine.

## Before you start

- **A Linux machine on x86_64.** `zenoh_dart` ships zenoh's native library for Linux on x86_64 and for Android, so the
  laptop side of this guide needs one of those, with glibc 2.34 or newer.
- **Some Dart, and enough Flutter to have finished Flutter's first codelab.** The guide does not teach the languages.
- **No zenoh needed.** If you do know zenoh already, from C, C++, Python, Rust or ROS 2, the chapters carry short asides
  that say what is the same here and what is not.
- Everything else — the SDK, the editor, the Android emulator — is installed in the chapter that first needs it.
  Chapter 0 starts from an empty folder.

Each chapter ends with the exact versions it was checked with, and asks for those or newer.

## What the guide builds on

- **[`zenoh_dart`](https://github.com/bluecorn/zenoh_dart)**, the Dart binding for zenoh, published on
  [pub.dev](https://pub.dev/packages/zenoh_dart). Each chapter starts from one of the programs in the package's
  `example/` folder: you run it first, see the behaviour, and then build that idea into the application.
- **[Zenoh Programming in Rust](https://kydos.github.io/zenoh-book/)** by Angelo Corsaro. The guide follows the book's
  order and its vocabulary, and links to it for each concept instead of explaining it a second time. Its examples are in
  Rust; nothing is copied from it.
- **[Flutter's architecture guide](https://docs.flutter.dev/app-architecture/guide)**, for the MVVM layering both
  programs use.

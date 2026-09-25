# 2 — The node on your phone

This chapter follows chapters 3 and 5 of *Zenoh Programming in Rust*, *The Zenoh Data Model* and *Publishers and
Put*, and *The Zenoh Book*'s pages on key expressions and pub/sub. It starts from `z_pub`, a program shipped with the
`zenoh_dart` package.

## 1 — What you build, and what you will see

By the end of this chapter the sensor node exists. It is a Flutter app, `sensor_node`, in the same workspace as
`sensorctl`, and it runs on the Android emulator and then on a phone. It opens a session with chapter 1's
`ZenohService`, from the settings that listen. It reads the device's accelerometer and publishes each reading as
text on the key expression `sensor/phone/accel`. On the laptop, the package's `z_sub` connects to the node through
`adb` and prints what arrives:

```
>> [Subscriber] Received PUT ('sensor/phone/accel': '0.000,9.776,0.812')
>> [Subscriber] Received PUT ('sensor/phone/accel': '0.000,9.776,0.812')
```

On the emulator, about 15 of these arrive each second, all the same until you move the virtual device from the
command line. On the phone they are your own numbers, and they change as you tilt it. The app's screen shows the
latest reading and how many it has published.

For the first time, your own code publishes and the laptop receives it. Chapter 3 replaces `z_sub` with a collector
of your own.

You build it in two passes, each led by a test. First the data side: stand-ins prove the core until the chapter's
claim holds, and then the phone's own sensor is connected to it. Then the app, from the screen in, each layer added
when the one above needs it. By the end of the chapter all four layers of the pattern are there, each with the least
that makes the claim true:

- the services: one for the sensor, and the shared `ZenohService` with one new thing in it
- the repository, which owns the key expression
- the view model
- the view, one widget

The test that states the claim lives in `sensor_core`. It runs on the laptop against real zenoh, with the sensor
faked. The app's own tests use fakes and never touch zenoh. You check the claim about a phone and a laptop by running
it, in sections 10 and 11.

> **If you already know zenoh.** This is `z_pub` with the key `sensor/phone/accel` and a `text/plain` encoding on
> every put. The session listens on the device's loopback, and the laptop reaches it with `adb forward`, with no
> router. Section 3 explains why the node declares a publisher and does not call `put` on the session.

> **If you already know Flutter.** The app is `flutter create --empty`, `flutter_riverpod`, one `ConsumerWidget`, and
> one plugin, `sensors_plus`. It is a member of a pub workspace, so you resolve and test it from the folder above it.
> `flutter run` is the only command you run inside the app's folder.

## 2 — What to read

**[The Zenoh Book](https://corsaro.me/zenoh/book/core-concepts/key-expressions/), *Core Concepts → Key
Expressions* and *[Publishers & Subscribers](https://corsaro.me/zenoh/book/core-concepts/pub-sub/)*, for the
idea.** A key expression is zenoh's only naming system, in place of topics and URLs. Its two wildcards, `*` and `**`,
let a subscriber ask for everything under `sensor/`. Take its naming advice too. Keys form a hierarchy of domain,
entity and attribute, so this guide's readings live on `sensor/<node>/<kind>`.

From the pub/sub page, take what a publisher is and what a *sample* holds. A publisher is a declared intent to
produce on a key, so routes are worked out once, not on every write. A sample is what a subscriber receives. The
[Peer Mode](https://corsaro.me/zenoh/book/routing/peer-mode/) page describes zenoh's default, where peers find each
other by multicast. This guide switches that off and gives each peer an address.

**[Zenoh Programming in Rust](https://kydos.github.io/zenoh-book/chapter_03.html), chapter 3, *The Zenoh Data
Model*, and [chapter 5, *Publishers and Put*](https://kydos.github.io/zenoh-book/chapter_05.html), for the shape of
the API.** From chapter 3, take how a key is formed and the three wildcard forms. Take also that a payload is bytes,
with an *encoding* beside it that says what the bytes are, and that a sample's timestamp comes from zenoh's clock, not
yours.

From chapter 5, take the difference between a one-shot put and a declared publisher, and the rule of thumb for
choosing: more than a few writes a second calls for a publisher. Skip *Publisher Options*, which is chapter 11 of this
guide, and *Matching Listener*.

Five things look different in Dart:

| in the books | in `zenoh_dart` |
|---|---|
| `session.declare_publisher(key).await` | `session.declarePublisher(keyExpr)`, with nothing to await |
| `publisher.put(payload).encoding(…).await` | `publisher.put(text, encoding: Encoding.textPlain)`, which takes a `String` and returns nothing |
| `session.declare_subscriber(key).await`, then `recv_async()` in a loop | `session.declareSubscriber(keyExpr)`, whose `stream` is a Dart `Stream<Sample>` |
| `sample.key_expr()`, `sample.payload()`, `sample.encoding()` | `sample.keyExpr` and `sample.payload`, plain `String`s, and `sample.encoding`, a `String?`, here `'text/plain'` |
| `ZBytes` in and out | a `String` in and out for this chapter, and bytes from chapter 6 |

> **A note on versions.** *Zenoh Programming in Rust* is written against Zenoh 1.4.0, and `zenoh_dart` 1.0.0-rc.1 is
> built on 1.8.0, so a detail there may have changed since. Every statement this guide makes about the Dart API was
> read in the package itself.

**[`sensors_plus`](https://pub.dev/packages/sensors_plus) and [`flutter_riverpod`](https://riverpod.dev), for more
than a sentence on each piece.** You add `sensors_plus` for the sensor in section 8, and `flutter_riverpod`, Riverpod
for Flutter, for the app's providers and view model in section 9. Each of their pieces gets a sentence where it first appears, on what it does in the app. For
the rest, read their documentation.

## 3 — A declared publisher

Run `z_pub`, the package's example of a *declared publisher*, before you write one. Chapter 0's `z_put` put one value
and ended. The node you are about to build puts many values a second and never ends, and zenoh has a different
object for that.

**1. Copy one more example.** Chapter 1 ended in the top folder. Read the package's folder from the workspace's
package config, the one at the top, and copy the example beside the others:

```sh
# in zenoh_sensors
pkg=$(sed -n 's|.*"rootUri": "file://\(.*/zenoh_dart-[^/"]*\)".*|\1|p' .dart_tool/package_config.json)
cp "$pkg/example/z_pub.dart" apps/sensorctl/example/
```

**2. Start the subscriber.** In a terminal at the top folder, start `z_sub` as in chapter 1, listening on the
loopback, and leave it running:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_sub.dart -l tcp/127.0.0.1:7447 --no-multicast-scouting
```

```
⋮
…Opening session...
Declaring Subscriber on 'demo/example/**'...
Press CTRL-C to quit...
```

**3. Start the publisher.** In a second terminal, also at the top folder, start `z_pub` with the three options
`z_put` took: connect to the subscriber, no scouting, and no listener of its own. Give it no key and no value, so
that its defaults show:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_pub.dart \
  -e tcp/127.0.0.1:7447 --no-multicast-scouting --cfg 'listen/endpoints:[]'
```

```
⋮
…Opening session...
Declaring Publisher on 'demo/example/zenoh-dart-pub'...
Press CTRL-C to quit...
Putting Data ('demo/example/zenoh-dart-pub': '[   0] Pub from Dart!')...
Putting Data ('demo/example/zenoh-dart-pub': '[   1] Pub from Dart!')...
⋮
```

**4. Watch them arrive.** In the first terminal, `z_sub` prints one line a second:

```
>> [Subscriber] Received PUT ('demo/example/zenoh-dart-pub': '[   0] Pub from Dart!')
>> [Subscriber] Received PUT ('demo/example/zenoh-dart-pub': '[   1] Pub from Dart!')
⋮
```

**The key falls under the subscriber's expression.** `z_pub`'s default key is `demo/example/zenoh-dart-pub`, and
`z_sub` asked for `demo/example/**`, so every put matches. The two programs agree on nothing else. A subscriber names
a *pattern*, a publisher names a *key*, and zenoh delivers where the two intersect. Section 5 makes the same
arrangement with `sensor/**` on one side and `sensor/phone/accel` on the other.

**`z_pub` declares first, then puts.** `z_put` called `put` on the session once. `z_pub` first declared a publisher
on its key, as `Declaring Publisher on …` shows, and then put through that publisher once a second.

Both books give the reason. A put on the session resolves its route every time, and a declared publisher settles the
route once and reuses it. So a program that writes more than a few times a second declares a publisher. The node
writes on one key for as long as it runs, at least several times a second, so the service you build in section 6
declares one.

**Each put names its encoding.** `[   0]`, `[   1]` are `z_pub`'s own counter, as in the zenoh-c example it mirrors.
Each put also marks its payload `text/plain`, an *encoding* that travels with the bytes and tells the receiver what
they are. The node does the same, in one line of section 6. Chapter 6 covers what encodings are for, and the one
this guide moves to.

> **If you already know zenoh.** This is zenoh-c's `z_pub`, flag for flag, including the `text/plain` encoding on
> each put and the `--add-matching-listener` option, which this guide does not use.

> **In VS Code.** Both terminals can be VS Code's. Open the second with the **+** at the top right of the terminal
> panel. Every new terminal starts in `zenoh_sensors`, where both commands run.

Both programs are still running. Section 4 stops them and creates the app.

## 4 — The app, in the same workspace

Create the sensor node, a Flutter app, in the workspace chapter 1 made, beside `sensorctl`. Both programs then depend
on the same `sensor_core` by name, and one `pubspec.lock` at the top holds every version. When this section is done,
`apps` holds a second folder:

```
zenoh_sensors/
├── apps/
│   ├── sensor_node/                the app, created by fvm flutter create
│   │   ├── .gitignore
│   │   ├── .idea/                  for editors of the IntelliJ family; git leaves it out
│   │   ├── .metadata               what flutter create made, for its own upgrade tool
│   │   ├── analysis_options.yaml
│   │   ├── android/                the Android project Gradle builds
│   │   ├── lib/
│   │   │   └── main.dart
│   │   ├── pubspec.yaml
│   │   ├── README.md
│   │   └── sensor_node.iml         the same, and left out the same way
│   └── sensorctl/
├── packages/
│   └── sensor_core/
├── pubspec.lock
└── pubspec.yaml                    now lists three members
```

**1. Stop the two examples, then create the app.** `z_sub` and `z_pub` are still running from the last section.
Press Ctrl-C in the second terminal and then in the first terminal. `z_pub` prints zenoh 1.8.0's `ERROR` line as it
closes, the one chapter 0 explained. `z_sub` closes last, when it is connected to no one, so it prints nothing. Then,
from the top folder:

```sh
# in zenoh_sensors
fvm flutter create --empty --platforms=android apps/sensor_node
```

`--empty` asks the app template for its smallest start: a `main.dart` of 16 lines that shows `Hello World!`, with no
counter demo and no test folder to throw away. `--platforms=android` writes the Android project and nothing for iOS,
the desktops or the web.

The tool also runs `pub get` inside the new app, which leaves a lock file and a package config there. Step 4 removes
both. The tool also picks an *application id*, `com.example.sensor_node`, the name Android knows the app by. The
default is fine for an app that never leaves your devices. An app you publish gets its own id, in
`android/app/build.gradle.kts`.

> **In VS Code.** Type the command in VS Code's terminal, as in chapter 0 for `sensorctl`. Do not use the Command
> Palette's **Flutter: New Project**. It runs the same template, but then it reopens VS Code on the new app's folder.

**2. Make it a member of the workspace.** Replace `zenoh_sensors/apps/sensor_node/pubspec.yaml`:

```yaml
name: sensor_node
description: The sensor node, an Android app that publishes the phone's sensors over zenoh.
publish_to: none
version: 1.0.0+1

environment:
  sdk: ^3.13.2

resolution: workspace

dependencies:
  flutter:
    sdk: flutter

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^6.0.0

flutter:
  uses-material-design: true
```

Three lines differ from the template, the same three that chapter 1 changed for `sensorctl`. The description changes
from `A new Flutter project.` to the app's own. The version was `0.1.0+1`. The `+1` is Android's build number, which
counts uploads to a store and stays at 1 here. `resolution: workspace` is new.

Keep `flutter_lints`, the lint set Flutter's template ships, until section 9. There the template's last file goes,
and the app switches to the guide's own rules, as `sensorctl` did in chapter 1 once its template was gone.

**3. Add the app to the workspace.** The top folder's `pubspec.yaml` gains one member. Replace
`zenoh_sensors/pubspec.yaml`:

```yaml
name: zenoh_sensors
publish_to: none

environment:
  sdk: ^3.13.2

dependencies:
  sensor_core: ^1.0.0

workspace:
  - apps/sensorctl
  - apps/sensor_node
  - packages/sensor_core
```

**4. Resolve once, from the top.**

```sh
# in zenoh_sensors
fvm dart pub get
```

`pub get` prints three things you can ignore:

- It deletes the app's own lock file and package config, with a link about *stray files*, because a workspace keeps
  one of each, at the top.
- It lists the dependencies it changed, and some went *down*, such as `test` to 1.31.1. `flutter_test` pins the test
  packages it is built with, and a workspace shares one set of versions, so the core's tests now use the Flutter
  SDK's. The tests do not change.
- The note chapter 0's `pub add` printed, about packages with newer versions it cannot use, now counts more of them.
  These are packages the Flutter SDK pins, and you have nothing to do.

**5. Let the app use the network.** An Android app may open a socket only if its manifest asks for the `INTERNET`
permission. `flutter create` asks for it only in the *debug* manifest, because Flutter's own tools need it to talk to
a running app. Without it in the main manifest, the app's session would be refused the day you build it for release.
Add the permission as the main manifest's second line. Replace
`zenoh_sensors/apps/sensor_node/android/app/src/main/AndroidManifest.xml`:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET"/>
    <application
        android:label="sensor_node"
        android:name="${applicationName}"
        android:icon="@mipmap/ic_launcher">
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:launchMode="singleTop"
            android:taskAffinity=""
            android:theme="@style/LaunchTheme"
            android:configChanges="orientation|keyboardHidden|keyboard|screenSize|smallestScreenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
            android:hardwareAccelerated="true"
            android:windowSoftInputMode="adjustResize">
            <!-- Specifies an Android theme to apply to this Activity as soon as
                 the Android process has started. This theme is visible to the user
                 while the Flutter UI initializes. After that, this theme continues
                 to determine the Window background behind the Flutter UI. -->
            <meta-data
              android:name="io.flutter.embedding.android.NormalTheme"
              android:resource="@style/NormalTheme"
              />
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <!-- Don't delete the meta-data below.
             This is used by the Flutter tool to generate GeneratedPluginRegistrant.java -->
        <meta-data
            android:name="flutterEmbedding"
            android:value="2" />
    </application>
    <!-- Required to query activities that can process text, see:
         https://developer.android.com/training/package-visibility and
         https://developer.android.com/reference/android/content/Intent#ACTION_PROCESS_TEXT.

         In particular, this is used by the Flutter engine in io.flutter.plugin.text.ProcessTextPlugin. -->
    <queries>
        <intent>
            <action android:name="android.intent.action.PROCESS_TEXT"/>
            <data android:mimeType="text/plain"/>
        </intent>
    </queries>
</manifest>
```

You edit nothing else in the Android project. The app runs on Android API 24 and up, Flutter's default, and the
package needs no more than that.

**6. Run the template app on the emulator.** Build the template app before any code of yours goes in, because the
first build is the slow one and the place a missing piece of the Android SDK shows up. List your virtual devices, then
start one by its id:

```sh
# in zenoh_sensors
fvm flutter emulators
```

```sh
# in zenoh_sensors
fvm flutter emulators --launch <the id from the list>
```

The emulator's window opens and the device boots. `fvm flutter devices` lists it once it has. Then go into the app's
folder and run the app:

```sh
# in zenoh_sensors
cd apps/sensor_node
```

```sh
# in zenoh_sensors/apps/sensor_node
fvm flutter run
```

`flutter run` is the only command in this guide that runs inside a package's folder. It compiles the app and has
Gradle pack it into an APK, with zenoh's native libraries, which the package's build hook copies in. Then it installs
the APK on the device and starts it.

Chapter 1's rule is for programs that load the libraries from the laptop's disk. This app loads them from its own APK,
so the rule does not apply. `flutter run` still resolves the workspace from the top, and its first lines say so.

The first build takes minutes, because Gradle downloads what it needs and compiles the Android side of Flutter once.
Later builds take seconds. On every build, Gradle may print a few red `WARNING:` lines about a *restricted method* in
`java.lang.System`. They come from Gradle running on a recent Java, such as the one Android Studio carries, and they
say nothing about your app.

When the build finishes, the emulator shows `Hello World!` in the middle of a white screen. The terminal prints
`Flutter run key commands` and waits. `r` reloads the app after you change a file, and `q` stops it. Press `q` now,
and go back to the top:

```sh
# in zenoh_sensors/apps/sensor_node
cd ../..
```

> **In VS Code.** The status bar, bottom right, names the device VS Code runs on. Click it to pick the emulator, or
> **No Device** to start one. Then open `apps/sensor_node/lib/main.dart` and choose **Run** above `main`. It is the
> same build, with the output in the Debug Console, and the red square stops it. Section 10 gives the app an entry in
> Run and Debug.

> **If you already know Flutter.** Hot reload works as usual. Nothing in this guide needs it, because each step
> replaces files whole and runs tests from a terminal, but you can use it.

The app is in the workspace and builds on the emulator. Section 5 writes the test that states the chapter's claim.

## 5 — The test of the chapter's claim

Write the test that states the chapter's claim, and watch it fail. The test comes first, as in chapter 1. It is the
chapter's promise, written as code.

**1. State the claim in one sentence.** The test's name repeats it.

> A reading from the sensor is published on `sensor/phone/accel` as text, and a subscriber receives it.

**2. Make test files take turns.** The core is about to have a second test file, and both open a sensor node that
listens on `tcp/127.0.0.1:7447`. `dart test` runs test files in parallel by default, so the second file to reach
`open()` finds the port taken and fails. Its error names a network failure, not the port, because zenoh reports every
failure to open with one code. Create `zenoh_sensors/dart_test.yaml`:

```yaml
# Test files that open zenoh sessions share one loopback port, so they run
# one at a time.
concurrency: 1
```

The runner reads this file from the folder it starts in. That is the top folder, where the tests run and where the
editor's template starts them.

**3. Make room.** Create two folders in the library and two under `test/`:

```sh
# in zenoh_sensors
mkdir -p packages/sensor_core/lib/src/domain packages/sensor_core/lib/src/repositories
mkdir -p packages/sensor_core/test/repositories packages/sensor_core/test/support
```

`domain/` holds the model, `repositories/` the layer that owns key expressions, and `support/` what the tests share:
stand-ins and one helper.

**4. Write the laptop's end of the test.** The test has two ends. The node's end is the code under test: your
`ZenohService`, which gets a publication in section 6, and the repository that uses it. The laptop's end is only a
witness. It subscribes and collects what arrives.

`ZenohService` cannot subscribe yet. Subscribing is chapter 3's subject, when `sensorctl watch` needs it. So the test
does what `z_sub` does on the laptop, and opens a plain zenoh session straight from the package. It uses the settings
chapter 1 wrote for the collector, `SessionSettings.collectorNode()`, so the network arrangement is chapter 1's and
only the object is different. Create `zenoh_sensors/packages/sensor_core/test/support/collector.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:zenoh_dart/zenoh.dart';

/// A collector opened with the package directly, as `z_sub` is on the laptop.
/// There is no collector-side service until chapter 3.
Future<Session> openCollector() {
  final config = Config();
  SessionSettings.collectorNode().asJson5.forEach(config.insertJson5);
  return Session.open(config: config);
}

/// Long enough for a sample, or a declaration, to cross the loopback, which
/// takes milliseconds.
const delivery = Duration(milliseconds: 500);
```

`delivery` is the only wait in this chapter's tests. A put returns before the sample has traveled anywhere, so the
test gives it time to arrive. Half a second is generous for a trip of tens of milliseconds on the loopback. Tests may
import `zenoh_dart`, but under `lib/` only the service may.

**5. Write the test.** Create `zenoh_sensors/packages/sensor_core/test/repositories/sensor_node_repository_test.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:test/test.dart';
import 'package:zenoh_dart/zenoh.dart';

import '../support/collector.dart';
import '../support/fakes.dart';

void main() {
  test('a sensor reading reaches a subscriber on sensor/phone/accel', () async {
    // The node's end: its session, from the settings that listen.
    final zenoh = ZenohService(SessionSettings.sensorNode());
    addTearDown(zenoh.dispose);
    await zenoh.open();

    // The laptop's end: a witness that subscribes and keeps what arrives, as
    // z_sub does. A plain session, because ZenohService cannot subscribe yet.
    final collector = await openCollector();
    addTearDown(collector.close);
    final subscriber = collector.declareSubscriber('sensor/**');
    addTearDown(subscriber.close);
    final received = <Sample>[];
    subscriber.stream.listen(received.add);

    // The code to implement: a repository that publishes one reading
    // from a stand-in sensor, through the node's session.
    const reading = Reading(x: 0, y: 9.776, z: 0.812);
    final sensor = FakeSensorService(Stream.value(reading));
    final repository = SensorNodeRepository(zenoh, sensor, nodeName: 'phone');
    final published = await repository.publish().toList();
    // A put returns before the sample arrives, so give it time to cross.
    await Future<void>.delayed(delivery);

    // The claim: one sample, on the exact key, as text, marked text/plain,
    // and the reading handed on for the screen.
    final keys = received.map((sample) => sample.keyExpr).toList();
    expect(keys, ['sensor/phone/accel']);
    expect(received.single.payload, '0.000,9.776,0.812');
    expect(received.single.encoding, 'text/plain');
    expect(published, [reading]);
  });
}
```

The test is section 3 again, with your code in the publisher's place. The node's session opens first and listens, as
in chapter 1. The collector connects, subscribes to `sensor/**` as `z_sub` does on the laptop, and collects every
sample into a list.

Then the code under test runs. A repository gets the service, a sensor that delivers exactly one reading, and the
node's name. `publish()` returns the stream of what it published, and `toList()` runs that stream to the end.

The four expectations state the claim: one sample on the *key* `sensor/phone/accel`, its payload the reading as
`x,y,z` to three decimals, marked `text/plain`, and the reading handed on for a screen to show. The numbers are the
emulator's resting pose, which you see on the laptop in section 10.

**6. Write just enough for it to compile.** Five files. Three of them are not skeletons. A model and a contract have
no behavior to fake, and the stand-in for the sensor is whole from the start, so you write all three once, here.

The model holds one reading. Create `zenoh_sensors/packages/sensor_core/lib/src/domain/reading.dart`:

```dart
/// One reading of the accelerometer: meters per second squared on each axis,
/// gravity included.
class Reading {
  /// A reading of [x], [y] and [z].
  const new({required this.x, required this.y, required this.z});

  /// Along the device's x axis, to the right.
  final double x;

  /// Along the device's y axis, towards the top of the screen.
  final double y;

  /// Along the device's z axis, out of the screen.
  final double z;
}
```

The model holds the three values the claim sends, and nothing else. The time the device took the reading joins it in
chapter 6, where time goes on the wire. It has no equality and no `copyWith`, because no test in this chapter compares
two readings.

The sensor's contract is an interface. Create `zenoh_sensors/packages/sensor_core/lib/src/services/sensor_service.dart`:

```dart
import 'package:sensor_core/src/domain/reading.dart';

/// The device's motion sensors, as the node reads them.
///
/// The accelerometer reports meters per second squared on three axes, gravity
/// included, so a device lying flat on its back reads about 9.81 on z. The
/// axes are the device's own: x to the right, y towards the top of the screen,
/// z out of the screen.
abstract interface class SensorService {
  /// The accelerometer's readings, as the device delivers them.
  Stream<Reading> accelerometer();
}
```

The interface lives in the core, but its implementation cannot, because `sensors_plus` is a Flutter plugin and needs
Flutter to load. The core is plain Dart, shared with `sensorctl` and tested with `dart test`. So the core keeps the
*contract*, and the doc comment states it.

Gravity is in the numbers, so a device at rest reads about 9.81 on one axis.

The repository starts empty. Create
`zenoh_sensors/packages/sensor_core/lib/src/repositories/sensor_node_repository.dart`:

```dart
import 'package:sensor_core/src/domain/reading.dart';
import 'package:sensor_core/src/services/sensor_service.dart';
import 'package:sensor_core/src/services/zenoh_service.dart';

/// The node's side of the readings: it owns the key expression and publishes
/// what the sensor delivers.
class SensorNodeRepository {
  /// A repository publishing [sensor]'s readings through [zenoh], for the node
  /// called [nodeName].
  new(this.zenoh, this.sensor, {required this.nodeName});

  /// The service the readings are published through.
  final ZenohService zenoh;

  /// The sensor the readings come from.
  final SensorService sensor;

  /// The node's name, the middle segment of its key expressions.
  final String nodeName;

  /// Publishes every reading as text, and hands each on.
  Stream<Reading> publish() => const Stream.empty();
}
```

The core's exports gain the three new files. Replace `zenoh_sensors/packages/sensor_core/lib/sensor_core.dart`:

```dart
/// The zenoh data layer that `sensorctl` and the phone app share.
library;

export 'src/domain/reading.dart';
export 'src/repositories/sensor_node_repository.dart';
export 'src/services/sensor_service.dart';
export 'src/services/session_settings.dart';
export 'src/services/zenoh_service.dart';
```

The test proves the repository with a stand-in for the sensor, and section 8 connects the real one. The stand-in
implements the contract above, which it imports through the exports you just replaced, and plays whatever readings
the test gives it. Create
`zenoh_sensors/packages/sensor_core/test/support/fakes.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';

class FakeSensorService implements SensorService {
  new(this.readings);

  final Stream<Reading> readings;

  @override
  Stream<Reading> accelerometer() => readings;
}
```

A fake needs no doc comments. The rule is for what a package makes public, and a test's stand-ins are nobody's API.

**7. Run it, and read the failure.**

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'reaches a subscriber'
```

```
⋮
  Expected: ['sensor/phone/accel']
    Actual: []
     Which: at location [0] is [] which shorter than expected
⋮
```

No sample arrived on `sensor/phone/accel`. The sessions are real and connected, as chapter 1's tests showed, but the
repository's `publish()` is a skeleton that publishes nothing. **This test stays red until section 7.** Each of its
four expectations is made true by a smaller test of one part, in the next two sections, and each of those is red
before its code:

```
a sensor reading reaches a subscriber on sensor/phone/accel
 ├─ on the key sensor/phone/accel    section 7, cycles 1 and 2: the repository composes the key
 ├─ the payload 0.000,9.776,0.812    section 6: text put through a publication arrives
 │                                   section 7, cycle 3: a reading becomes that text
 ├─ marked text/plain                section 6: the publication marks what it puts
 └─ the reading handed on            section 7, cycle 3
```

Section 7's fourth cycle makes cancelling the stream close the publication. The app needs that in section 9, and this
test does not check it.

> **In VS Code.** The Testing view shows the new file beside the old one, and ▶ on the test fails it the same way.
> Run the whole core from there, and the two files run one after the other, as `dart_test.yaml` says.

The outer test is red. Section 6 gives the service its publication.

## 6 — The publisher, behind the service

Give the service a publication, a class of its own around the package's publisher. The service is the only file that
imports the package. A publisher is a package object, so it cannot travel upward as it is.

Instead, the service hands out a *publication* of its own. It is a small class beside the service, in the same file,
with the package's publisher inside and two methods, `put` a string and `close`. Chapter 1's rule stays true, and the
repository never sees a type from `zenoh_dart`.

**Cycle 1 — a put through a publication reaches a subscriber as text.** This is the service's own test, so it lives
in the service's file and opens real sessions, like every test there. The block shows only what changes, as in
chapter 1, and this time the imports change too. The lines above `⋮` are the file's imports as they now read. Add two
imports and a ninth test to `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:test/test.dart';
import 'package:zenoh_dart/zenoh.dart';

import '../support/collector.dart';

⋮

  test('a put through a publication reaches a subscriber as text', () async {
    // The node's end: its session, from the settings that listen.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);
    await sensorNode.open();

    // The laptop's end: a plain session that subscribes, as z_sub does.
    final collector = await openCollector();
    addTearDown(collector.close);
    final subscriber = collector.declareSubscriber('sensor/phone/accel');
    addTearDown(subscriber.close);
    final received = <Sample>[];
    subscriber.stream.listen(received.add);

    // The new contract to implement: declare a publication on a key,
    // then put text through it.
    sensorNode
        .declarePublication('sensor/phone/accel')
        .put('0.000,9.776,0.812');
    // A put returns before the sample arrives, so give it time to cross.
    await Future<void>.delayed(delivery);

    // The claim: the text arrives, marked text/plain.
    final payloads = received.map((sample) => sample.payload).toList();
    expect(payloads, ['0.000,9.776,0.812']);
    expect(received.single.encoding, 'text/plain');
  });
}
```

The call in the test sets the shape of the service: `declarePublication` with a key, then `put` with a string. The
text must arrive marked `text/plain` without the caller saying so.

Neither method exists yet, so first write the emptiest publication that compiles. It is chapter 1's class with one
method added, and above it a second class, `Publication`, which that method returns. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'package:sensor_core/src/services/session_settings.dart';
import 'package:zenoh_dart/zenoh.dart';

/// Starts zenoh's own log, printed to standard output: for a program with a
/// terminal. [level] applies unless `RUST_LOG` is set. Once per process,
/// before any session opens.
void initZenohLogging(String level) => Zenoh.initLog(level);

/// A declared publisher on one key expression, as the service hands it out:
/// text in, marked `text/plain` on every put.
class Publication {
  new _();

  /// Puts [text] on the publication's key expression, marked `text/plain`.
  void put(String text) {}

  /// Undeclares the publisher. Safe to call twice.
  void close() {}
}

/// The owner of the zenoh session. It hands upward only plain Dart values and
/// its own [Publication]s.
class ZenohService {
  /// A service for one role's [settings]. Nothing opens until [open].
  new(this.settings);

  /// Which side of the topology this session is on.
  final SessionSettings settings;

  Session? _session;

  /// Opens the session with the settings. When this returns, each connection
  /// they ask for is made, or its first attempt has failed and zenoh keeps
  /// retrying it.
  Future<void> open() async {
    _session = await Session.open(config: _config());
  }

  /// The session's identity: up to thirty-two hexadecimal characters.
  String get zid => _opened.zid.toHexString();

  /// The identities of the peers this session is connected to.
  List<String> get peerIds =>
      _opened.peersZid().map((id) => id.toHexString()).toList();

  /// Declares a publisher on [keyExpr]. The service closes it on [dispose].
  Publication declarePublication(String keyExpr) => Publication._();

  /// Closes the publications and the session. Safe before [open], and more
  /// than once.
  void dispose() {
    _session?.close();
    _session = null;
  }

  Config _config() {
    final config = Config();
    settings.asJson5.forEach(config.insertJson5);
    return config;
  }

  Session get _opened {
    final session = _session;
    if (session == null) {
      throw StateError('open() has not been called on this ZenohService');
    }
    return session;
  }
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'through a publication'
```

```
⋮
  Expected: ['0.000,9.776,0.812']
    Actual: []
     Which: at location [0] is [] which shorter than expected
⋮
```

Nothing arrived, because the stub's `put` does nothing.

**Write the obvious implementation.** No constant makes a sample arrive at another session, so write the real thing.
Replace `zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'package:sensor_core/src/services/session_settings.dart';
import 'package:zenoh_dart/zenoh.dart';

/// Starts zenoh's own log, printed to standard output: for a program with a
/// terminal. [level] applies unless `RUST_LOG` is set. Once per process,
/// before any session opens.
void initZenohLogging(String level) => Zenoh.initLog(level);

/// A declared publisher on one key expression, as the service hands it out:
/// text in, marked `text/plain` on every put.
class Publication {
  new _(this._publisher);

  final Publisher _publisher;

  /// Puts [text] on the publication's key expression, marked `text/plain`.
  void put(String text) => _publisher.put(text, encoding: Encoding.textPlain);

  /// Undeclares the publisher. Safe to call twice.
  void close() => _publisher.close();
}

/// The owner of the zenoh session. It hands upward only plain Dart values and
/// its own [Publication]s.
class ZenohService {
  /// A service for one role's [settings]. Nothing opens until [open].
  new(this.settings);

  /// Which side of the topology this session is on.
  final SessionSettings settings;

  Session? _session;
  final _publications = <Publication>[];

  /// Opens the session with the settings. When this returns, each connection
  /// they ask for is made, or its first attempt has failed and zenoh keeps
  /// retrying it.
  Future<void> open() async {
    _session = await Session.open(config: _config());
  }

  /// The session's identity: up to thirty-two hexadecimal characters.
  String get zid => _opened.zid.toHexString();

  /// The identities of the peers this session is connected to.
  List<String> get peerIds =>
      _opened.peersZid().map((id) => id.toHexString()).toList();

  /// Declares a publisher on [keyExpr]. The service closes it on [dispose].
  Publication declarePublication(String keyExpr) {
    final publication = Publication._(_opened.declarePublisher(keyExpr));
    _publications.add(publication);
    return publication;
  }

  /// Closes the publications and the session. Safe before [open], and more
  /// than once.
  void dispose() {
    for (final publication in _publications) {
      publication.close();
    }
    _publications.clear();
    _session?.close();
    _session = null;
  }

  Config _config() {
    final config = Config();
    settings.asJson5.forEach(config.insertJson5);
    return config;
  }

  Session get _opened {
    final session = _session;
    if (session == null) {
      throw StateError('open() has not been called on this ZenohService');
    }
    return session;
  }
}
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'through a publication'
```

It passes.

**`declarePublisher` is the declared publisher of section 3.** It takes the key and returns the package's
`Publisher`, and the service hands up a wrapper around it. Declaring needs an open session, so it goes through
`_opened` and throws the same `StateError` as `zid` when there is none.

**`Encoding.textPlain` goes on every put**, inside `Publication.put`, as in `z_pub`. The repository writes the text
and never deals with the encoding. This is the only place the guide names this encoding, until chapter 6 replaces it.

**The service closes what it declared.** Every publication goes into a list, and `dispose()` closes them before it
closes the session. A publication can also be closed on its own, earlier. Closing one twice is safe. The package's
documentation says so of its publisher, and section 12 pins it for yours.

**The constructor is private.** Only this file can call `Publication._`, so a publication comes only from a service,
and the language enforces it.

**What the test guarantees:** a string put through a publication on a key reaches a subscriber to that key as a
sample with the text and the `text/plain` encoding.

The service can publish. Section 7 builds the repository that uses it, and the outer test goes green.

## 7 — The repository, and the outer test goes green

Build the repository in four cycles, and the outer test goes green. The repository owns the key expressions. In this
chapter it owns one, `sensor/<node>/accel`, built from the node's name. It reads the sensor, puts each reading
through a publication as text, and hands each reading on for the screen.

Each cycle runs against a *fake* service. These are *layer tests*, the second kind of test in this guide. Each tests
one layer and fakes the layer below. Section 5's outer test runs the repository against real zenoh, and it goes green
at the end of this section. In between, each cycle checks what the repository did, through a stand-in that records
it.

**1. Write a stand-in for the service.** The fakes file keeps the sensor's stand-in and gains two more, for the
service and its publications. Replace `zenoh_sensors/packages/sensor_core/test/support/fakes.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';

class FakeSensorService implements SensorService {
  new(this.readings);

  final Stream<Reading> readings;

  @override
  Stream<Reading> accelerometer() => readings;
}

class FakePublication implements Publication {
  new(this.keyExpr);

  final String keyExpr;
  final puts = <String>[];
  bool isClosed = false;

  @override
  void put(String text) => puts.add(text);

  @override
  void close() => isClosed = true;
}

class FakeZenohService implements ZenohService {
  final publications = <FakePublication>[];

  @override
  Publication declarePublication(String keyExpr) {
    final publication = FakePublication(keyExpr);
    publications.add(publication);
    return publication;
  }

  @override
  SessionSettings get settings => SessionSettings.sensorNode();

  @override
  Future<void> open() async {}

  @override
  String get zid => 'a-fake';

  @override
  List<String> get peerIds => const [];

  @override
  void dispose() {}
}
```

`ZenohService` is an ordinary class, not an interface, and the fake still says `implements ZenohService`. In Dart
every class is also an interface, so a stand-in can promise the same members without inheriting a line of the real
thing.

The fake's publications record the key they were declared on, what was put through them, and whether they were closed.
A repository test needs to see nothing else. The five members the fake never uses answer with the least they can, as a
fake should.

*Fake* is this guide's word for every stand-in it writes by hand. In stricter words, `FakeSensorService` is a
**stub**. It hands the code under test what the test gave it, and records nothing. `FakeZenohService` and its
publications are **spies**. They record what was asked of them, and the test reads the record afterwards.

None of them is a **mock**. A mock is told in advance which calls to expect, checks them itself, and usually comes
from a framework. This guide's tests read what a stand-in recorded after the fact, so the guide has no mocks.

**Cycle 1 — the phone publishes on `sensor/phone/accel`.** Add a second test to
`zenoh_sensors/packages/sensor_core/test/repositories/sensor_node_repository_test.dart`:

```dart
⋮

  test('the phone publishes on sensor/phone/accel', () async {
    // Stand-ins: a service that records what is declared on it, and a
    // sensor with nothing to deliver.
    final zenoh = FakeZenohService();
    final sensor = FakeSensorService(const Stream.empty());

    // The code to implement: the repository declares its publication.
    final repository = SensorNodeRepository(zenoh, sensor, nodeName: 'phone');
    await repository.publish().toList();

    // The claim: one publication, on the phone's key.
    final keys = zenoh.publications.map((p) => p.keyExpr).toList();
    expect(keys, ['sensor/phone/accel']);
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'the phone publishes'
```

```
⋮
  Expected: ['sensor/phone/accel']
    Actual: []
     Which: at location [0] is [] which shorter than expected
⋮
```

No publication was declared, because the skeleton's `publish()` does nothing yet.

**Fake it.** Return the key as a constant, as chapter 1 did for the identity. Replace
`zenoh_sensors/packages/sensor_core/lib/src/repositories/sensor_node_repository.dart`:

```dart
import 'package:sensor_core/src/domain/reading.dart';
import 'package:sensor_core/src/services/sensor_service.dart';
import 'package:sensor_core/src/services/zenoh_service.dart';

/// The node's side of the readings: it owns the key expression and publishes
/// what the sensor delivers.
class SensorNodeRepository {
  /// A repository publishing [sensor]'s readings through [zenoh], for the node
  /// called [nodeName].
  new(this.zenoh, this.sensor, {required this.nodeName});

  /// The service the readings are published through.
  final ZenohService zenoh;

  /// The sensor the readings come from.
  final SensorService sensor;

  /// The node's name, the middle segment of its key expressions.
  final String nodeName;

  /// The key expression the readings are published on.
  String get keyExpr => 'sensor/phone/accel';

  /// Publishes every reading as text, and hands each on.
  Stream<Reading> publish() {
    zenoh.declarePublication(keyExpr);
    return const Stream.empty();
  }
}
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'the phone publishes'
```

It passes. The key is a constant, and the node's name is ignored, so the next cycle names a second node.

**Cycle 2 — a node named `sim` publishes on `sensor/sim/accel`.** Add a third test to
`zenoh_sensors/packages/sensor_core/test/repositories/sensor_node_repository_test.dart`:

```dart
⋮

  test('a node named sim publishes on sensor/sim/accel', () async {
    // Stand-ins: a service that records what is declared on it, and a
    // sensor with nothing to deliver.
    final zenoh = FakeZenohService();
    final sensor = FakeSensorService(const Stream.empty());

    // The code to implement: the key built from the node's name.
    final repository = SensorNodeRepository(zenoh, sensor, nodeName: 'sim');
    await repository.publish().toList();

    // The claim: one publication, on the sim node's key.
    final keys = zenoh.publications.map((p) => p.keyExpr).toList();
    expect(keys, ['sensor/sim/accel']);
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'named sim'
```

```
⋮
  Expected: ['sensor/sim/accel']
    Actual: ['sensor/phone/accel']
     Which: at location [0] is 'sensor/phone/accel' instead of 'sensor/sim/accel'
⋮
```

The constant gives every node the phone's key.

**Triangulate.** Build the key from the node's name, because no constant satisfies both tests. Replace
`zenoh_sensors/packages/sensor_core/lib/src/repositories/sensor_node_repository.dart`:

```dart
import 'package:sensor_core/src/domain/reading.dart';
import 'package:sensor_core/src/services/sensor_service.dart';
import 'package:sensor_core/src/services/zenoh_service.dart';

/// The node's side of the readings: it owns the key expression and publishes
/// what the sensor delivers.
class SensorNodeRepository {
  /// A repository publishing [sensor]'s readings through [zenoh], on the key
  /// expression of the node called [nodeName].
  new(this.zenoh, this.sensor, {required String nodeName})
    : keyExpr = 'sensor/$nodeName/accel';

  /// The service the readings are published through.
  final ZenohService zenoh;

  /// The sensor the readings come from.
  final SensorService sensor;

  /// The key expression the readings are published on.
  final String keyExpr;

  /// Publishes every reading as text, and hands each on.
  Stream<Reading> publish() {
    zenoh.declarePublication(keyExpr);
    return const Stream.empty();
  }
}
```

Run both:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'publishes on'
```

Both pass. The app's providers pass `'phone'` in section 9, and chapter 4's node without a device passes `'sim'`.

**What the tests guarantee:** a repository declares its publication on `sensor/<node>/accel`, built from the node's
name, which is given once, when the repository is made.

**Cycle 3 — a reading is put as `x,y,z` to three decimals, and handed on.** Add a fourth test to
`zenoh_sensors/packages/sensor_core/test/repositories/sensor_node_repository_test.dart`:

```dart
⋮

  test('a reading is put as x,y,z to three decimals and handed on', () async {
    // Stand-ins: a service that records what is put through it, and a
    // sensor that delivers one reading.
    final zenoh = FakeZenohService();
    const reading = Reading(x: 0, y: 9.776, z: 0.812);
    final sensor = FakeSensorService(Stream.value(reading));

    // The code to implement: each reading put as text, then handed on.
    final repository = SensorNodeRepository(zenoh, sensor, nodeName: 'phone');
    final published = await repository.publish().toList();

    // The claim: the text put through the publication, and the same reading
    // handed on.
    expect(zenoh.publications.single.puts, ['0.000,9.776,0.812']);
    expect(published, [reading]);
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'three decimals'
```

```
⋮
  Expected: ['0.000,9.776,0.812']
    Actual: []
     Which: at location [0] is [] which shorter than expected
⋮
```

Nothing was put, because `publish()` declares the publication and then returns an empty stream.

**Write the obvious implementation.** This is the repository's first real code. Replace
`zenoh_sensors/packages/sensor_core/lib/src/repositories/sensor_node_repository.dart`:

```dart
import 'dart:async';

import 'package:sensor_core/src/domain/reading.dart';
import 'package:sensor_core/src/services/sensor_service.dart';
import 'package:sensor_core/src/services/zenoh_service.dart';

/// The node's side of the readings: it owns the key expression and publishes
/// what the sensor delivers.
class SensorNodeRepository {
  /// A repository publishing [sensor]'s readings through [zenoh], on the key
  /// expression of the node called [nodeName].
  new(this.zenoh, this.sensor, {required String nodeName})
    : keyExpr = 'sensor/$nodeName/accel';

  /// The service the readings are published through.
  final ZenohService zenoh;

  /// The sensor the readings come from.
  final SensorService sensor;

  /// The key expression the readings are published on.
  final String keyExpr;

  /// Publishes every reading as `x,y,z` to three decimals, and hands each on.
  /// Listening declares the publication.
  Stream<Reading> publish() {
    final controller = StreamController<Reading>();
    controller.onListen = () {
      final publication = zenoh.declarePublication(keyExpr);
      sensor.accelerometer().listen((reading) {
        publication.put(_asText(reading));
        controller.add(reading);
      }, onDone: controller.close);
    };
    return controller.stream;
  }

  static String _asText(Reading reading) {
    final x = reading.x.toStringAsFixed(3);
    final y = reading.y.toStringAsFixed(3);
    final z = reading.z.toStringAsFixed(3);
    return '$x,$y,$z';
  }
}
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'three decimals'
```

It passes, and so do the first two cycles' tests, `-n 'publishes on'`. `publish()` returns a stream that does nothing
until someone listens. When someone does, it declares the publication and listens to the sensor. It puts each
reading as text, then adds the reading to the stream, so the screen sees what went on the wire, in the same order.
When the sensor's stream ends, this one ends too.

The three lines that make the text are this chapter's wire format. They stay in the repository until chapter 6 moves
them into a codec, the refactor that chapter is built around.

**What the test guarantees:** each reading goes on the wire as `x,y,z` to three decimals, and is handed on in the
same order.

**Cycle 4 — cancelling the stream closes the publication.** A node that stops publishing should tell zenoh, and here
a node stops when the stream's listener cancels. Add a fifth test to
`zenoh_sensors/packages/sensor_core/test/repositories/sensor_node_repository_test.dart`, and the import it needs
at the top:

```dart
import 'dart:async';

import 'package:sensor_core/sensor_core.dart';
import 'package:test/test.dart';
import 'package:zenoh_dart/zenoh.dart';

import '../support/collector.dart';
import '../support/fakes.dart';

⋮

  test('cancelling the stream closes the publication', () async {
    // Stand-ins: a service that records what is closed, and a sensor that
    // stays quiet, as a real one does between readings.
    final zenoh = FakeZenohService();
    final readings = StreamController<Reading>();
    final sensor = FakeSensorService(readings.stream);
    final repository = SensorNodeRepository(zenoh, sensor, nodeName: 'phone');

    // The code to implement: a cancel that closes the publication, even
    // while the sensor is quiet.
    final subscription = repository.publish().listen((_) {});
    await subscription.cancel();

    // The claim: the publication the repository declared is closed.
    expect(zenoh.publications.single.isClosed, isTrue);
  });
}
```

The sensor is a controller that never delivers anything. So the listener cancels while the node waits for a reading,
which is when a real node is cancelled too. Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'cancelling'
```

```
⋮
  Expected: true
    Actual: <false>
⋮
```

The publication was declared and never closed.

**Write the obvious implementation.** The controller gets a second callback, for the cancel. It must reach the
publication and the sensor subscription that `onListen` made, so both become `late` variables. Replace
`zenoh_sensors/packages/sensor_core/lib/src/repositories/sensor_node_repository.dart`:

```dart
import 'dart:async';

import 'package:sensor_core/src/domain/reading.dart';
import 'package:sensor_core/src/services/sensor_service.dart';
import 'package:sensor_core/src/services/zenoh_service.dart';

/// The node's side of the readings: it owns the key expression and publishes
/// what the sensor delivers.
class SensorNodeRepository {
  /// A repository publishing [sensor]'s readings through [zenoh], on the key
  /// expression of the node called [nodeName].
  new(this.zenoh, this.sensor, {required String nodeName})
    : keyExpr = 'sensor/$nodeName/accel';

  /// The service the readings are published through.
  final ZenohService zenoh;

  /// The sensor the readings come from.
  final SensorService sensor;

  /// The key expression the readings are published on.
  final String keyExpr;

  /// Publishes every reading as `x,y,z` to three decimals, and hands each on.
  /// Listening declares the publication; cancelling closes it.
  Stream<Reading> publish() {
    late final Publication publication;
    late final StreamSubscription<Reading> readings;
    final controller = StreamController<Reading>();
    controller.onListen = () {
      publication = zenoh.declarePublication(keyExpr);
      readings = sensor.accelerometer().listen((reading) {
        publication.put(_asText(reading));
        controller.add(reading);
      }, onDone: controller.close);
    };
    controller.onCancel = () async {
      await readings.cancel();
      publication.close();
    };
    return controller.stream;
  }

  static String _asText(Reading reading) {
    final x = reading.x.toStringAsFixed(3);
    final y = reading.y.toStringAsFixed(3);
    final z = reading.z.toStringAsFixed(3);
    return '$x,$y,$z';
  }
}
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'cancelling'
```

It passes. The repository uses a `StreamController` because of this cycle. An `async*` function notices that its
listener has gone only at its next `yield`. A node cancelled while its sensor is quiet would keep its publication open
until a reading happened to arrive.

The controller's `onCancel` runs at the cancel, whatever the sensor is doing. It also runs when the stream ends by
itself, which closes the publication then too, as when the outer test's sensor delivers its one reading and stops.

**What the test guarantees:** cancelling the node's stream closes its publication, even while the sensor is quiet.

**2. Run the outer test again.** Nothing else stands in its way:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'reaches a subscriber'
```

It passes. Then run the whole core, the way it runs from here on:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

14 tests pass, the two files one after the other.

**What the outer test guarantees:** a reading from the sensor is published on `sensor/phone/accel` as text, and a
subscriber to `sensor/**` receives it. The sample carries the exact key, the text and the encoding, and the
repository hands the reading on.

The claim holds on the laptop. Sections 8 to 11 make it hold on a device.

## 8 — The sensor, on the device

Connect the real sensor. Section 5's stand-in proved what the core does with readings, and now the phone's own
accelerometer delivers them. Implement the sensor's contract over `sensors_plus`. This is its one implementation in
this chapter, and it lives in the app, because the plugin does.

**1. Add two dependencies.** The app needs the core, for `Reading` and the interface, and the plugin. Replace
`zenoh_sensors/apps/sensor_node/pubspec.yaml`:

```yaml
name: sensor_node
description: The sensor node, an Android app that publishes the phone's sensors over zenoh.
publish_to: none
version: 1.0.0+1

environment:
  sdk: ^3.13.2

resolution: workspace

dependencies:
  flutter:
    sdk: flutter
  sensor_core: ^1.0.0
  sensors_plus: ^7.1.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^6.0.0

flutter:
  uses-material-design: true
```

Then resolve from the top:

```sh
# in zenoh_sensors
fvm dart pub get
```

> **In VS Code.** Open `apps` › `sensor_node` › `pubspec.yaml`, add the two lines, and save. The extension runs
> `pub get`.

**2. Make room.** Mirror `lib/` under `test/`, as the core does:

```sh
# in zenoh_sensors
mkdir -p apps/sensor_node/lib/data/services apps/sensor_node/test/data/services
```

**3. Write the test.** The plugin's stream exists only on a device. So the service takes the function that produces
the stream as a constructor argument. By default it is the plugin's own function, and a test passes in one that plays
events the test made. Create `zenoh_sensors/apps/sensor_node/test/data/services/device_sensor_service_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sensor_node/data/services/device_sensor_service.dart';
import 'package:sensors_plus/sensors_plus.dart';

void main() {
  test('an accelerometer event becomes a reading, field for field', () async {
    // A stand-in for the plugin's function: one event, made by the test.
    final event = AccelerometerEvent(0.1, 9.8, 0.2, DateTime(2026, 9, 23, 12));
    final service = DeviceSensorService(
      events: ({samplingPeriod = SensorInterval.normalInterval}) =>
          Stream.value(event),
    );

    // The code to implement: each event mapped to a reading.
    final readings = await service.accelerometer().toList();

    // The claim: one reading, with the event's values on the three axes.
    final fields = readings.map((r) => (r.x, r.y, r.z)).toList();
    expect(fields, [(0.1, 9.8, 0.2)]);
  });
}
```

The test checks that the service turns the plugin's event into your `Reading`. `flutter_test` is the app's test
package. For a test with no widget, its `test` and `expect` are the core's.

**4. Write just enough to compile.** For now the service answers each event with a reading of zeros. Create
`zenoh_sensors/apps/sensor_node/lib/data/services/device_sensor_service.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:sensors_plus/sensors_plus.dart';

/// The shape of the plugin's accelerometer function, so that a test can hand
/// in another.
typedef AccelerometerEvents = Stream<AccelerometerEvent> Function({
  Duration samplingPeriod,
});

/// The device's sensors, read through `sensors_plus`.
class DeviceSensorService implements SensorService {
  /// A service over the plugin's accelerometer, or over what a test hands in.
  new({this._events = accelerometerEventStream});

  final AccelerometerEvents _events;

  @override
  Stream<Reading> accelerometer() =>
      _events().map((_) => const Reading(x: 0, y: 0, z: 0));
}
```

It follows the rules the app gets in section 9, as every app file does from here: doc comments, and the `new`
constructor chapter 1 introduced.

**5. Run it, from the top.** Run the app's tests with `flutter test`, which is `dart test` plus the Flutter
framework. Give it the path to the test from the top folder, as for every other test:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node/test/data/services/device_sensor_service_test.dart
```

```
⋮
  Expected: [(double, double, double):(0.1, 9.8, 0.2)]
    Actual: [(double, double, double):(0.0, 0.0, 0.0)]
     Which: at location [0] is (double, double, double):<(0.0, 0.0, 0.0)> instead of (double, double, double):<(0.1, 9.8, 0.2)>
⋮
```

It fails on its assertion. The reading has zeros on all three axes. The first Flutter test build
takes a little longer than a Dart one. It loads nothing from zenoh, because the app's tests never open a session and
the package loads its native library only when something asks for it.

**6. Write the obvious implementation.** Map each event to a reading. Replace
`zenoh_sensors/apps/sensor_node/lib/data/services/device_sensor_service.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:sensors_plus/sensors_plus.dart';

/// The shape of the plugin's accelerometer function, so that a test can hand
/// in another.
typedef AccelerometerEvents = Stream<AccelerometerEvent> Function({
  Duration samplingPeriod,
});

/// The device's sensors, read through `sensors_plus`.
class DeviceSensorService implements SensorService {
  /// A service over the plugin's accelerometer, or over what a test hands in.
  new({this._events = accelerometerEventStream});

  final AccelerometerEvents _events;

  @override
  Stream<Reading> accelerometer() =>
      _events().map((event) => Reading(x: event.x, y: event.y, z: event.z));
}
```

Run it again:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node/test/data/services/device_sensor_service_test.dart
```

It passes. `accelerometerEventStream` is the plugin's function, and `AccelerometerEvents` writes down its shape, so a
test can pass in another function of the same shape. `{this._events = …}` is a named parameter that fills a private
field, which Dart allows from 3.10. Outside the class its name is `events`. The class's only method has no doc
comment of its own, because it inherits the interface's.

The mapping copies the three axes. The model fits the real sensor as it is, so nothing in the core changes. The event
also carries the time it was taken, and `Reading` leaves it out until chapter 6, where time goes on the wire.

The real sensor also has a rate, and the rate is a request. The plugin asks for its default, one reading every 200
milliseconds, 5 a second. Android runs one sensor per device and delivers to every program at the fastest rate any of
them asked for.

So the emulator, whose own system asks for more, gives the app about 15 readings a second. The phone this chapter was
checked with gave about 120 while its screen was on, and about 7 with the screen off. Chapter 9 makes a command of the
rate.

**What the test guarantees:** an accelerometer event becomes a reading with the same values on the three axes.

The device's sensor is behind the contract. Section 9 builds the app around it.

## 9 — Providers, a view model, a screen, and `main`

Build the app from the screen in. The screen shows the readings, a view model holds what it shows, the providers
reach the core, and a `main` starts it all. First add one dependency. Replace
`zenoh_sensors/apps/sensor_node/pubspec.yaml`:

```yaml
name: sensor_node
description: The sensor node, an Android app that publishes the phone's sensors over zenoh.
publish_to: none
version: 1.0.0+1

environment:
  sdk: ^3.13.2

resolution: workspace

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^3.4.3
  sensor_core: ^1.0.0
  sensors_plus: ^7.1.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^6.0.0

flutter:
  uses-material-design: true
```

Then resolve from the top:

```sh
# in zenoh_sensors
fvm dart pub get
```

`flutter_riverpod` is chapter 1's `riverpod`, plus the widgets that put a container into a widget tree. The app
keeps `flutter_lints` until step 5, because the template's `main.dart` stays until then and would not pass the
guide's rules. Then create the folders:

```sh
# in zenoh_sensors
mkdir -p apps/sensor_node/lib/config apps/sensor_node/lib/ui/node
mkdir -p apps/sensor_node/test/ui/node apps/sensor_node/test/support
```

**1. Write the screen's test.** Start from what the user sees. The least that shows the node at work is the latest
reading, to three decimals, and how many readings there have been. The test's name is its claim: the screen shows the
latest reading and the count. Create `zenoh_sensors/apps/sensor_node/test/ui/node/node_screen_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:sensor_node/ui/node/node_screen.dart';
import 'package:sensor_node/ui/node/node_view_model.dart';

import '../../support/fakes.dart';

void main() {
  testWidgets('the screen shows the latest reading and the count', (
    tester,
  ) async {
    // Stand-in: a view model that holds a fixed state.
    final container = ProviderContainer.test(
      overrides: [
        nodeViewModelProvider.overrideWith(
          () => FakeNodeViewModel(const NodeState(latest: aReading, count: 42)),
        ),
      ],
    );

    // The code to implement: the screen, built once from that state.
    await tester.pumpWidget(
      UncontrolledProviderScope(
        container: container,
        child: const MaterialApp(home: NodeScreen()),
      ),
    );

    // The claim: the reading to three decimals, and the count.
    expect(find.text('x 0.100'), findsOneWidget);
    expect(find.text('y 9.776'), findsOneWidget);
    expect(find.text('z 0.812'), findsOneWidget);
    expect(find.text('42 readings'), findsOneWidget);
  });
}
```

The test needs three things that do not exist yet: the view model's state and its provider, a stand-in view model that
holds a fixed state, and the screen. Write the emptiest of each.

The screen reads its state from a view model. A view model holds the data its screen needs, shaped for that screen.
The emptiest one holds a state and never changes it. Create
`zenoh_sensors/apps/sensor_node/lib/ui/node/node_view_model.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:sensor_core/sensor_core.dart';

/// What the node's screen shows: the latest reading, and how many there were.
class NodeState {
  /// A state with [latest] as the newest reading and [count] readings so far.
  const new({this.latest, this.count = 0});

  /// The newest reading, or null before the first.
  final Reading? latest;

  /// How many readings the node has published.
  final int count;
}

/// Keeps the latest reading and counts them, for the node's screen.
class NodeViewModel extends Notifier<NodeState> {
  @override
  NodeState build() => const NodeState();
}

/// The node screen's view model.
final nodeViewModelProvider = NotifierProvider<NodeViewModel, NodeState>(
  NodeViewModel.new,
);
```

`build` returns the state the view model starts with: no reading yet, and a count of 0.

`NodeViewModel` is a Riverpod `Notifier`, and `NodeState` is the state it holds. When a `Notifier` replaces its state,
Riverpod rebuilds every widget that watches it, so the screen will show each new reading.

The stand-in is the real view model with `build` replaced. It never listens to anything, and it holds whatever state
the test gives it. Create `zenoh_sensors/apps/sensor_node/test/support/fakes.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:sensor_node/ui/node/node_view_model.dart';

class FakeNodeViewModel extends NodeViewModel {
  new(this.fixed);

  final NodeState fixed;

  @override
  NodeState build() => fixed;
}

const aReading = Reading(x: 0.1, y: 9.776, z: 0.812);
```

Then write the emptiest screen. Create `zenoh_sensors/apps/sensor_node/lib/ui/node/node_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

/// The node's one screen: the latest reading to three decimals, and the count.
class NodeScreen extends ConsumerWidget {
  /// The screen; it watches the node's view model.
  const new({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) => const Scaffold();
}
```

`UncontrolledProviderScope` puts a container you made into the widget tree, which is how a widget test overrides a
provider. `pumpWidget` builds the screen once, and `find.text` looks for the exact strings.

Run it:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node/test/ui/node/node_screen_test.dart
```

```
⋮
Expected: exactly one matching candidate
  Actual: _TextWidgetFinder:<Found 0 widgets with text "x 0.100": []>
   Which: means none were found but one was expected
⋮
```

The screen shows no text yet.

**Write the obvious implementation.** The screen watches the view model and shows its state. Replace
`zenoh_sensors/apps/sensor_node/lib/ui/node/node_screen.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:sensor_node/ui/node/node_view_model.dart';

/// The node's one screen: the latest reading to three decimals, and the count.
class NodeScreen extends ConsumerWidget {
  /// The screen; it watches [nodeViewModelProvider].
  const new({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final node = ref.watch(nodeViewModelProvider);
    final latest = node.latest;
    return Scaffold(
      body: Center(
        child: DefaultTextStyle.merge(
          style: Theme.of(context).textTheme.headlineSmall,
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              Text('x ${_fixed(latest?.x)}'),
              Text('y ${_fixed(latest?.y)}'),
              Text('z ${_fixed(latest?.z)}'),
              Text('${node.count} readings'),
            ],
          ),
        ),
      ),
    );
  }

  static String _fixed(double? value) => (value ?? 0).toStringAsFixed(3);
}
```

Run it again:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node/test/ui/node/node_screen_test.dart
```

It passes. A `ConsumerWidget` gets a `ref` with its context, and `ref.watch` rebuilds the widget whenever the view
model's state changes, which on the device is every reading. The screen is four lines of text in a column, in the
theme's headline size so you can read them from across a desk. Chapter 13 adds a second screen.

**What the test guarantees:** the screen shows the latest reading's three values to three decimals, and the count.

The screen reads a stand-in's state so far. Step 2 gives the real view model something to hold.

**2. Write the view model's test.** The view model's data is the readings. It listens to `readingsProvider`, the
app's one way in to them, and keeps the latest reading and the count. Riverpod rebuilds whatever watches that state,
the screen from step 1, so the screen never reads a stream itself.

The test's name is its claim: the view model keeps the latest reading and counts them. Create
`zenoh_sensors/apps/sensor_node/test/ui/node/node_view_model_test.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:sensor_core/sensor_core.dart';
import 'package:sensor_node/config/providers.dart';
import 'package:sensor_node/ui/node/node_view_model.dart';

void main() {
  test('the view model keeps the latest reading and counts them', () async {
    // Stand-in: the readings provider, overridden with two readings, so
    // nothing below the view model is built.
    const first = Reading(x: 0, y: 9.776, z: 0.812);
    const second = Reading(x: 0, y: 0, z: 9.81);
    final container = ProviderContainer.test(
      overrides: [
        readingsProvider.overrideWith(
          (ref) => Stream.fromIterable([first, second]),
        ),
      ],
    )..listen(nodeViewModelProvider, (_, _) {});

    // Let both readings flow through before reading the state.
    await pumpEventQueue();

    // The code to implement: the view model's state, built from the
    // readings it heard.
    final state = container.read(nodeViewModelProvider);

    // The claim: two readings counted, and the second one kept.
    expect(state.count, 2);
    expect(state.latest, second);
  });
}
```

The test replaces `readingsProvider`, which does not exist yet. Declare it with no readings, so the test has something
to replace. It is a `StreamProvider`, a provider whose value comes from a stream. Step 3 connects it to the core.
Create `zenoh_sensors/apps/sensor_node/lib/config/providers.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:sensor_core/sensor_core.dart';

/// The readings the node publishes. This first version yields none.
final readingsProvider = StreamProvider<Reading>((ref) => const Stream.empty());
```

The provider graph gives the test its seam. The test fakes nothing below the view model. It *overrides the readings
provider* with a stream of two readings, so nothing below is ever built.

`ProviderContainer.test` makes a container that disposes itself when the test ends. The cascade, `..listen`, keeps the
view model alive, as a widget watching it would, and it is how the lint rules want a second call on a value just made.
`pumpEventQueue` lets both readings flow through before the test reads the state.

Run it:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node/test/ui/node/node_view_model_test.dart
```

```
⋮
  Expected: <2>
    Actual: <0>
⋮
```

The count is 0, because the skeleton never listens to the readings.

**Write the obvious implementation.** The view model listens to the readings and replaces its state on each one.
Replace `zenoh_sensors/apps/sensor_node/lib/ui/node/node_view_model.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:sensor_core/sensor_core.dart';
import 'package:sensor_node/config/providers.dart';

/// What the node's screen shows: the latest reading, and how many there were.
class NodeState {
  /// A state with [latest] as the newest reading and [count] readings so far.
  const new({this.latest, this.count = 0});

  /// The newest reading, or null before the first.
  final Reading? latest;

  /// How many readings the node has published.
  final int count;
}

/// Keeps the latest reading and counts them, for the node's screen.
class NodeViewModel extends Notifier<NodeState> {
  @override
  NodeState build() {
    ref.listen(readingsProvider, (_, next) {
      if (next case AsyncData(:final value)) {
        state = NodeState(latest: value, count: state.count + 1);
      }
    });
    return const NodeState();
  }
}

/// The node screen's view model.
final nodeViewModelProvider = NotifierProvider<NodeViewModel, NodeState>(
  NodeViewModel.new,
);
```

Run it again:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node/test/ui/node/node_view_model_test.dart
```

It passes. On each reading the view model replaces its state as a whole. In `build`, `ref.listen` subscribes the view
model to the readings provider for as long as the view model lives. A
stream provider delivers its values wrapped in `AsyncValue`, as loading, data or error. The pattern match takes the
data case and ignores the others.

Chapter 12 handles loading and errors: what the screen shows while the session opens, and when it fails. Before
then, the screen shows zeros until the first reading arrives, and a failure shows only in the log, which `main` sets
up below.

**What the test guarantees:** the view model counts every reading the node publishes, and keeps the latest.

**3. Wire the readings to the core.** `readingsProvider` yields nothing yet. Follow the data back to where it starts.
The readings come from the core, in three links:

- `DeviceSensorService` reads the sensor.
- `SensorNodeRepository` publishes each reading on `sensor/phone/accel` and hands it on.
- `readingsProvider` brings that stream into the app once the session is open.

Each link gets a provider, and so does what the links need in turn: the service the repository publishes through, its
settings, and the session it opens. Replace `zenoh_sensors/apps/sensor_node/lib/config/providers.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:sensor_core/sensor_core.dart';
import 'package:sensor_node/data/services/device_sensor_service.dart';

/// Which side of the topology this app is on: the sensor node.
final sessionSettingsProvider = Provider<SessionSettings>(
  (ref) => SessionSettings.sensorNode(),
);

/// The app's one zenoh session, disposed with the container.
final zenohServiceProvider = Provider<ZenohService>((ref) {
  final service = ZenohService(ref.watch(sessionSettingsProvider));
  ref.onDispose(service.dispose);
  return service;
});

/// The device's sensors, behind the interface the core defines.
final sensorServiceProvider = Provider<SensorService>(
  (ref) => DeviceSensorService(),
);

/// The node's repository: this phone's readings, on `sensor/phone/accel`.
final sensorNodeRepositoryProvider = Provider<SensorNodeRepository>(
  (ref) => SensorNodeRepository(
    ref.watch(zenohServiceProvider),
    ref.watch(sensorServiceProvider),
    nodeName: 'phone',
  ),
);

/// The session, opened once; what depends on it waits for this.
final sessionProvider = FutureProvider<void>(
  (ref) => ref.watch(zenohServiceProvider).open(),
);

/// The readings the node publishes, once the session is open.
final readingsProvider = StreamProvider<Reading>((ref) async* {
  await ref.watch(sessionProvider.future);
  yield* ref.watch(sensorNodeRepositoryProvider).publish();
});
```

As in chapter 1, each provider holds one piece of the app and names the providers it needs. Each piece is made once
and shared, the container disposes them together, and a test can swap any one for a stand-in with an override.

The six form one chain, from the settings to the readings. The view model from step 2 sits at its end, and the screen
from step 1 watches the view model.

The first two are chapter 1's, with one change. This program is a **sensor node**, so it takes the settings that
listen. The service is disposed through its provider, as before. `sensorServiceProvider` is declared against the
interface, so a test, or chapter 4, can put another implementation there. The repository gets the two services and
the node's name.

`sessionProvider` is a new kind, a `FutureProvider`. It opens the session once, and anything that depends on it can
wait for that. `readingsProvider` is still a `StreamProvider`. Now it waits, with `ref.watch(sessionProvider.future)`,
and then yields the repository's stream.

A provider that watches another is rebuilt when the other changes. So when chapter 12 reconnects the session, the
readings stream restarts with it, and nothing above needs to know. When the provider is disposed, it cancels the
stream's listener, and section 7's cycle 4 closes the publication.

A provider makes nothing until something reads it. Listening to the view model starts the whole chain: the session
opens, the publication is declared on `sensor/phone/accel`, the sensor is asked for readings, and the first reading is
put. On the device the screen does that listening, so the phone starts publishing when the screen is first built.

**4. Add logging for an app.** Chapter 1 started zenoh's log with `initZenohLogging`, which prints to the process's
standard output, the terminal for `sensorctl`. An Android app has no terminal, and Android throws its standard output
away, so on a device that call shows nothing.

The package has a second way, a *sink*. It delivers the log's records as a stream, for the program to print as it
likes. The core offers it as a function beside the first. The classes stay the same, and one function is added.
Replace `zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'package:sensor_core/src/services/session_settings.dart';
import 'package:zenoh_dart/zenoh.dart';

/// Starts zenoh's own log, printed to standard output: for a program with a
/// terminal. [level] applies unless `RUST_LOG` is set. Once per process,
/// before any session opens.
void initZenohLogging(String level) => Zenoh.initLog(level);

/// Zenoh's own log at [level] as lines, for a program with no terminal to
/// print to, such as an app. Once per process, before any session opens, and
/// never after [initZenohLogging].
Stream<String> zenohLog(String level) =>
    Zenoh.initLogWithSink(minSeverity: LogSeverity.values.byName(level))
        .map((record) => 'zenoh ${record.severity.name}: ${record.message}');

/// A declared publisher on one key expression, as the service hands it out:
/// text in, marked `text/plain` on every put.
class Publication {
  new _(this._publisher);

  final Publisher _publisher;

  /// Puts [text] on the publication's key expression, marked `text/plain`.
  void put(String text) => _publisher.put(text, encoding: Encoding.textPlain);

  /// Undeclares the publisher. Safe to call twice.
  void close() => _publisher.close();
}

/// The owner of the zenoh session. It hands upward only plain Dart values and
/// its own [Publication]s.
class ZenohService {
  /// A service for one role's [settings]. Nothing opens until [open].
  new(this.settings);

  /// Which side of the topology this session is on.
  final SessionSettings settings;

  Session? _session;
  final _publications = <Publication>[];

  /// Opens the session with the settings. When this returns, each connection
  /// they ask for is made, or its first attempt has failed and zenoh keeps
  /// retrying it.
  Future<void> open() async {
    _session = await Session.open(config: _config());
  }

  /// The session's identity: up to thirty-two hexadecimal characters.
  String get zid => _opened.zid.toHexString();

  /// The identities of the peers this session is connected to.
  List<String> get peerIds =>
      _opened.peersZid().map((id) => id.toHexString()).toList();

  /// Declares a publisher on [keyExpr]. The service closes it on [dispose].
  Publication declarePublication(String keyExpr) {
    final publication = Publication._(_opened.declarePublisher(keyExpr));
    _publications.add(publication);
    return publication;
  }

  /// Closes the publications and the session. Safe before [open], and more
  /// than once.
  void dispose() {
    for (final publication in _publications) {
      publication.close();
    }
    _publications.clear();
    _session?.close();
    _session = null;
  }

  Config _config() {
    final config = Config();
    settings.asJson5.forEach(config.insertJson5);
    return config;
  }

  Session get _opened {
    final session = _session;
    if (session == null) {
      throw StateError('open() has not been called on this ZenohService');
    }
    return session;
  }
}
```

A program calls one of the two, once, before anything opens, because the logging slot is process-wide and the first
call claims it. Call `zenohLog` after `initZenohLogging` and it throws a `StateError`. Call them the other way round,
and the second call is silently ignored. The level is one of `trace`, `debug`, `info`, `warn` and `error`, as before.

**5. Write `main`, and switch on the app's rules.** The template's last file goes now, and the app's lint set comes
with it. `very_good_analysis` replaces the template's `flutter_lints`. Replace
`zenoh_sensors/apps/sensor_node/pubspec.yaml`:

```yaml
name: sensor_node
description: The sensor node, an Android app that publishes the phone's sensors over zenoh.
publish_to: none
version: 1.0.0+1

environment:
  sdk: ^3.13.2

resolution: workspace

dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^3.4.3
  sensor_core: ^1.0.0
  sensors_plus: ^7.1.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  very_good_analysis: ^11.0.0

flutter:
  uses-material-design: true
```

The include is the switch, and the two exclusions are the template's own, for folders that hold no Dart of yours.
Replace `zenoh_sensors/apps/sensor_node/analysis_options.yaml`:

```yaml
include: package:very_good_analysis/analysis_options.yaml

analyzer:
  exclude:
    - build/**
    - android/**
```

Resolve from the top:

```sh
# in zenoh_sensors
fvm dart pub get
```

Then replace `zenoh_sensors/apps/sensor_node/lib/main.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:sensor_core/sensor_core.dart';
import 'package:sensor_node/ui/node/node_screen.dart';

void main() {
  zenohLog('error').listen(debugPrint);

  runApp(
    ProviderScope(
      retry: (retryCount, error) => null,
      child: const MaterialApp(home: NodeScreen()),
    ),
  );
}
```

**`zenohLog('error').listen(debugPrint)`** prints every record zenoh logs at `error` into Flutter's log, which
`flutter run` shows in its terminal. When all is well there are none. Change `'error'` to `'info'`, and the first two
lines after launch are `Using ZID: …` and `Zenoh can be reached at: tcp/127.0.0.1:7447`, the node announcing that it
listens on the device's loopback.

**`ProviderScope`** is the container as a widget. Its `retry` turns off Riverpod's default of retrying a failed
provider, with growing delays, up to 10 times. `sessionProvider` can fail, and a session that cannot open should say
so once, without trying again in the background for a minute. Chapter 12 makes reconnecting an explicit action.

**6. Run every test of the app**, from the top:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node
```

3 tests pass. Nothing in them opens a session or reads a sensor. The app's tests cover the app's own logic, for the
whole guide. `fvm dart analyze` from the top folder reports no issues for the three packages together, as it does at
the end of every section from here.

> **In VS Code.** The Testing view lists the app's tests under `sensor_node`, beside the core's. The same template
> entry in `launch.json` runs both kinds, from the top folder.

The app is complete, and its tests pass. Section 10 runs it on the emulator.

## 10 — On the emulator

Run the node on the virtual device, and watch its readings arrive on the laptop. Then move the device from the
command line, and watch them change. You need three terminals: the app in the first, the subscriber in the second,
and the emulator's console in the third.

**1. Start the emulator and the node.** Check that `adb` is on your `PATH`, because every `adb` command in this
guide calls it by name. It is in the `platform-tools` folder of the Android SDK, as
[Android's page on `adb`](https://developer.android.com/tools/adb) describes.

```sh
# in zenoh_sensors
adb version
```

It prints its version. If the shell reports `command not found`, add that `platform-tools` folder to your `PATH`, and
open a new terminal.

`adb` must see only one device, so unplug your phone if it is connected. List your virtual devices:

```sh
# in zenoh_sensors
fvm flutter emulators
```

Start one by the id in the list's first column:

```sh
# in zenoh_sensors
fvm flutter emulators --launch <the id from the list>
```

The emulator's window opens and the device boots. `fvm flutter devices` lists it once it has. Go into the app's
folder:

```sh
# in zenoh_sensors
cd apps/sensor_node
```

Run the app. The build takes a few seconds, because Gradle reuses what it built in section 4.

```sh
# in zenoh_sensors/apps/sensor_node
fvm flutter run
```

When `Flutter run key commands` appears, the emulator shows four lines: `x`, `y` and `z`, and a count that climbs
about 15 times a second. Zenoh prints nothing, because nothing went wrong.

**2. Reach it from the laptop.** Your app on the emulator listens on `127.0.0.1:7447`. `127.0.0.1` is the loopback
address. It only reaches programs on the same machine. The emulator is a separate machine with its own network, so its
loopback and your laptop's loopback are two different places. A program on the laptop cannot reach the app directly.

`adb forward tcp:7447 tcp:7447` builds a bridge between the two:

```
 laptop                                      emulator (a separate machine)
┌────────────────────────────┐              ┌────────────────────────────┐
│ z_sub, the collector       │              │ the app, the sensor node   │
│ a peer that connects to    │              │ a peer that listens on     │
│   tcp/127.0.0.1:7447       │              │   tcp/127.0.0.1:7447       │
│        │                   │              │        ▲                   │
│        ▼                   │   adb's own  │        │                   │
│ adb listens on             │   link       │ adb's helper connects to   │
│   127.0.0.1:7447  ═════════╪══════════════╪══► 127.0.0.1:7447          │
└────────────────────────────┘              └────────────────────────────┘
```

Build the bridge:

```sh
# in zenoh_sensors
adb forward tcp:7447 tcp:7447
```

- **The first `tcp:7447` is the laptop side.** `adb` now listens on port 7447 of your laptop's loopback.
- **The second `tcp:7447` is the device side.** Each connection that arrives on the laptop's port is carried over
  `adb`'s link, the USB cable for a phone, and connected to port 7447 on the device's loopback, where the app listens.
- **Nothing flows yet.** The bridge waits for a program to connect to it. Next, `z_sub` connects to
  `127.0.0.1:7447`, which from the laptop looks like a local program. The app sees a connection arriving on its own
  loopback. Neither side knows `adb` is in between.
- **It is plain TCP.** `adb` does not know about zenoh, and several laptop programs can connect through the same
  forward at once.
- **It lasts** until the emulator stops, or until you run `adb forward --remove-all`.

`flutter run` uses the same mechanism for itself. `adb forward --list` shows two rules on the emulator: yours,
`tcp:7447 tcp:7447`, and one Flutter made between two random ports so its terminal can talk to the running app.

**Zenoh sees two peers joined by one TCP link.** Neither peer scouts by multicast. The sensor node listens on the
endpoint `tcp/127.0.0.1:7447`, `nodeEndpoint` in its `listen/endpoints`. The collector, `z_sub` here, connects to the
same endpoint and listens on none. The link is a direct peer-to-peer connection with no router, as the book's
[Peer Mode](https://corsaro.me/zenoh/book/routing/peer-mode/) page describes.

In a second terminal at the top folder, start `z_sub` with its own listener off and the key expression that covers
every node:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_sub.dart \
  -e tcp/127.0.0.1:7447 --no-multicast-scouting --cfg 'listen/endpoints:[]' -k 'sensor/**'
```

```
⋮
…Opening session...
Declaring Subscriber on 'sensor/**'...
Press CTRL-C to quit...
>> [Subscriber] Received PUT ('sensor/phone/accel': '0.000,9.776,0.812')
>> [Subscriber] Received PUT ('sensor/phone/accel': '0.000,9.776,0.812')
⋮
```

About 15 lines arrive each second, all the same. The virtual device lies still, nearly upright, at the pose the
emulator gives it when it starts, so `y` carries most of gravity. Your virtual device shows the same three numbers,
because they are the emulator's defaults.

The plugin asked for its default, 5 readings a second. The device delivers about 15, because the system's own
programs asked the same sensor for more, as section 8 described.

**3. Move the device.** The emulator's console takes sensor values, and `adb` forwards a command to it. In a third
terminal, lay the device flat on its back:

```sh
# in zenoh_sensors
adb emu sensor set acceleration 0:0:9.81
```

In the second terminal, within a second, gravity moves to `z`:

```
>> [Subscriber] Received PUT ('sensor/phone/accel': '0.000,0.000,9.810')
⋮
```

Then stand it upright, facing you:

```sh
# in zenoh_sensors
adb emu sensor set acceleration 0:9.81:0
```

```
>> [Subscriber] Received PUT ('sensor/phone/accel': '0.000,9.810,0.000')
⋮
```

**A value you set holds until you set another.** The emulator's motion model does not move the device back by
itself. A stream that seems frozen after this means the device is lying where you put it, and the app is fine. Put it
back where it started:

```sh
# in zenoh_sensors
adb emu sensor set acceleration 0:9.77631:0.812349
```

The numbers on the emulator's screen followed every change too, and its count kept climbing.

**4. Update the editor.** Change two entries in `launch.json`: `z_sub` now connects, because the node listens from
here on, and the app gets its own entry. Replace `zenoh_sensors/.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "z_sub",
      "type": "dart",
      "request": "launch",
      "program": "${workspaceFolder}/apps/sensorctl/example/z_sub.dart",
      "cwd": "${workspaceFolder}",
      "args": [
        "-e", "tcp/127.0.0.1:7447", "--no-multicast-scouting", "--cfg", "listen/endpoints:[]",
        "-k", "sensor/**"
      ]
    },
    {
      "name": "z_put",
      "type": "dart",
      "request": "launch",
      "program": "${workspaceFolder}/apps/sensorctl/example/z_put.dart",
      "cwd": "${workspaceFolder}",
      "args": [
        "-e", "tcp/127.0.0.1:7447", "--no-multicast-scouting", "--cfg", "listen/endpoints:[]",
        "-k", "demo/example/test", "-p", "Hello from the guide"
      ]
    },
    {
      "name": "sensorctl",
      "type": "dart",
      "request": "launch",
      "program": "${workspaceFolder}/apps/sensorctl/bin/sensorctl.dart",
      "cwd": "${workspaceFolder}"
    },
    {
      "name": "sensor_node",
      "type": "dart",
      "request": "launch",
      "program": "${workspaceFolder}/apps/sensor_node/lib/main.dart",
      "cwd": "${workspaceFolder}/apps/sensor_node"
    },
    {
      "name": "tests",
      "type": "dart",
      "request": "launch",
      "templateFor": "",
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

The app's entry starts in the app's folder, for the reason section 4 gave. Its `type` is `dart` for a Flutter app
too, and the extension tells the two apart by the project. The `tests` template is unchanged, and it runs the app's
tests as well as the core's, from the top.

> **In VS Code.** Pick the emulator in the status bar, choose **sensor_node** in Run and Debug, and press ▶. It is
> the same as `flutter run`, with the Debug Console for its output and hot reload on the toolbar. Then run **z_sub**
> from the same list, after `adb forward` in a terminal. `sensorctl` still works too. It connects to the node now,
> says who it is, and reports one peer, the node.

**5. Stop.** Press Ctrl-C in the second terminal to stop `z_sub`, the collector. It prints zenoh 1.8.0's `ERROR` line,
because it was connected to the node when it closed. Press `q` in the first terminal to stop the app, which stays
installed on the device.

In the third terminal, remove the forward:

```sh
# in zenoh_sensors
adb forward --remove-all
```

Then close the emulator's window. Go back to the top folder in the first terminal:

```sh
# in zenoh_sensors/apps/sensor_node
cd ../..
```

The node publishes from the emulator, and the laptop receives it. Section 11 runs the same commands on your phone.

## 11 — On your phone

Run the same commands on a phone over its USB cable. **`adb` must see only one device**, or every command needs `-s`
and the device's serial. If the emulator is running, close its window.

**1. Plug in, and run.** Turn on USB debugging and unlock the phone. `fvm flutter devices` lists it, perhaps beside
the laptop itself and a browser, which this app cannot run on because it is made for Android only. Go into the app's
folder:

```sh
# in zenoh_sensors
cd apps/sensor_node
```

Run the app:

```sh
# in zenoh_sensors/apps/sensor_node
fvm flutter run
```

The first build for a phone compiles for a different processor than the emulator's, so it takes a while again. The
app appears on the phone with your own numbers in it, about 9.8 on the axis pointing at the sky, and a count. The
terminal is busier than it was for the emulator. A phone sends Android's own log lines from the app's process,
starting `I/` and `D/`, and none of them is zenoh's.

**2. Forward the port, and subscribe.** In the second terminal, at the top folder:

```sh
# in zenoh_sensors
adb forward tcp:7447 tcp:7447
```

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_sub.dart \
  -e tcp/127.0.0.1:7447 --no-multicast-scouting --cfg 'listen/endpoints:[]' -k 'sensor/**'
```

```
⋮
…Opening session...
Declaring Subscriber on 'sensor/**'...
Press CTRL-C to quit...
>> [Subscriber] Received PUT ('sensor/phone/accel': '…,…,…')
>> [Subscriber] Received PUT ('sensor/phone/accel': '…,…,…')
⋮
```

Tilt the phone and watch the laptop follow. The number of lines a second depends on the phone and on what else on it
reads the sensor.

The app asks for 5 a second, and Android delivers at the fastest rate any program asked for. The phone this chapter
was checked with delivered about 120 a second while its screen was on, because another program was reading the sensor
100 times a second. With the screen off, it delivered about 7.

**3. Lock it.** Press the power button, wait a few seconds, and watch the second terminal. The lines keep coming. On
the phone this chapter was checked with, they slowed from 120 a second to 7, and were back at 120 within a second of
unlocking.

The laptop reaches the node through the cable, on the phone's own loopback, and locking the phone did not interrupt
that. The lock took away the other program's fast request. Chapter 5 takes the phone off the cable, where locking it
cuts the network.

If the cable comes out, `flutter run` says `Lost connection to device.` and ends. The forward goes with the
connection, and the app goes on running on the phone. Plug the cable back in and start again from step 1.

**4. Stop.** Press Ctrl-C in the second terminal to stop `z_sub`, the collector. It prints zenoh 1.8.0's `ERROR` line,
because it was connected to the node when it closed. In the same terminal, remove the forward:

```sh
# in zenoh_sensors
adb forward --remove-all
```

Go back to the first terminal, and press `q` to exit the app running on the phone. The app stays installed on the
phone. Then go back to the top folder:

```sh
# in zenoh_sensors/apps/sensor_node
cd ../..
```

> **In VS Code.** The status bar lists the phone once `adb` sees it. Pick it, and **sensor_node** in Run and Debug
> runs on the phone. Nothing else changes.

The claim holds on your phone. Section 12 adds three more tests and looks at what changed in the architecture.

## 12 — What the tests pin, and what changed in the architecture

Pin three facts with tests, and look at what the chapter built.

**1. Add three more tests.** They pass at once, because they pin what the code already does, as chapter 1's last three
did. One is about zenoh, and two are rules of your own. Add them to
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test(
    'a subscriber that arrives after a put sees only what follows',
    () async {
      // The node's end: a session and its publication.
      final sensorNode = ZenohService(SessionSettings.sensorNode());
      addTearDown(sensorNode.dispose);
      await sensorNode.open();
      final publication = sensorNode.declarePublication('sensor/phone/accel');

      // The laptop's end: a plain session, with no subscriber yet.
      final collector = await openCollector();
      addTearDown(collector.close);

      // A put before anyone subscribes.
      publication.put('before');

      // Subscribe late. Wait once for the declaration to reach the node's
      // side, and once for the next put to arrive.
      final subscriber = collector.declareSubscriber('sensor/phone/accel');
      addTearDown(subscriber.close);
      final received = <Sample>[];
      subscriber.stream.listen(received.add);
      await Future<void>.delayed(delivery);
      publication.put('after');
      await Future<void>.delayed(delivery);

      // The claim: pub/sub keeps nothing, so only the later put arrives.
      final payloads = received.map((sample) => sample.payload).toList();
      expect(payloads, ['after']);
    },
  );

  test('disposing the service closes its publications', () async {
    // A rule of the pattern: dispose closes what the service declared.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    await sensorNode.open();
    final publication = sensorNode.declarePublication('sensor/phone/accel');

    sensorNode.dispose();

    // The claim: a put after dispose is an error, because the publisher
    // underneath is closed.
    expect(() => publication.put('late'), throwsStateError);
  });

  test('closing a publication twice is safe', () async {
    // A rule of the pattern, which the repository's cancel and the
    // service's dispose both rely on.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);
    await sensorNode.open();
    final publication = sensorNode.declarePublication('sensor/phone/accel')
      ..close();

    // The claim: the second close returns normally.
    expect(publication.close, returnsNormally);
  });
}
```

Run the whole core:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

17 tests pass.

**Pub/sub keeps nothing.** A subscriber that arrives after a put never sees it. It sees only what is put after its
declaration has reached the publisher's side, so the test waits once after subscribing, for the declaration to travel,
and once after the put.

A node that publishes to nobody sends its readings nowhere, and a collector that connects a minute later starts from
the next one. Zenoh calls this data in motion. Chapter 7 gives the node a memory, and chapter 8 a way to ask for
it.

**Two rules of the pattern.** Disposing the service closes every publication it declared, so a put through one
afterwards is an error. It is the package's own `StateError`, because the publisher underneath is closed. Closing a
publication twice is safe, and the repository's cancel and the service's dispose both rely on that.

**What the tests guarantee:** a late subscriber sees only the puts after its declaration, a put after `dispose()` is
an error, and a second `close()` is safe.

**2. Look at what changed in the architecture.** Chapter 1 filled the layer on the right. This chapter drew the whole
stack of the app and filled each layer with the least that makes the claim true:

```
NodeScreen  →  NodeViewModel  →  SensorNodeRepository  →  ZenohService + Publication  →  zenoh_dart
   view          view model           repository       ↘   service                        package
                                                         SensorService  →  sensors_plus
                                                         (DeviceSensorService, in the app)
```

Four rules began here, and they hold for the rest of the guide.

**The repository owns the key expression.** `sensor/<node>/accel` is built in one place, from a name given once, and
nothing above or below the repository knows the shape of a key. The codec that will own the *payload* comes in
chapter 6. Until then the three lines that make `x,y,z` sit in the repository.

**The service hands out its own types.** `Publication` is declared beside the service and wraps the package's
publisher. The repository asks the service for one by key and puts strings through it. Chapter 3 adds a subscription
the same way.

**The app's tests never touch zenoh or a device.** The plugin's function is replaced by one that plays an event the
test made. The readings provider is overridden with a stream. The view model is replaced by one that holds a fixed
state. Every claim about zenoh is
tested in the core, on the laptop, against real sessions. The claim about the device is checked by running it.

**Live data is a stream provider that depends on the connection.** `readingsProvider` waits for `sessionProvider` and
then yields the repository's stream, and a view model listens to it and keeps state. When the connection is rebuilt,
the stream is rebuilt with it, and cancelling the stream closes what the node declared.

**What comes next.** Chapter 3 builds `sensorctl watch`, the collector's stack: a subscription in the service, a
repository that receives readings, a view model, and the terminal as the view. Each layer is tested against a fake of
the one below, as this chapter's app was, and `watch` replaces `z_sub`.

Chapter 4 adds `simulate`, a second sensor node inside `sensorctl`, written to the contract `SensorService` set here,
for when you would rather not start a device. Chapter 5 takes the phone off the cable and onto your Wi-Fi.

The chapter's code is done. Section 13 commits it.

## 13 — Files and versions at the end of this chapter

**1. Commit.** Commit everything this chapter made, in one commit:

```sh
# in zenoh_sensors
git add .
git commit -m "The node on your phone: sensor_node, and readings on sensor/phone/accel"
```

The commit holds 47 files, 41 of them new:

- 33 are the app's: 19 under `android/`, the 5 at its top, and the 9 Dart files you wrote.
- 9 are the core's: 6 new and 3 changed.
- The other 5 are `z_pub.dart`, `dart_test.yaml`, the changed `launch.json`, and the 2 files at the top.

Git leaves out these, which the app's `.gitignore` and `android/.gitignore` list:

- `.dart_tool/` and `build/` at every level (`flutter test` made one at the top, and `flutter run` one in the app)
- the app's `.idea/` and `*.iml` files
- under `android/`, the Gradle wrapper, which Flutter writes again when it needs it, and `local.properties`, which
  holds the SDK's path on this machine

Keep both `.gitignore` files as the tool wrote them.

> **In VS Code.** **View › Source Control** lists the same changes. Choose the **+** on the **Changes** line to stage
> them all, type the message in the box above them, and choose **Commit**.

`zenoh_sensors` now holds this, in four commits:

```
zenoh_sensors/
├── .dart_tool/
├── .fvm/
├── .fvmrc
├── .git/
├── .gitignore
├── .vscode/
│   ├── launch.json
│   └── settings.json
├── apps/
│   ├── sensor_node/
│   │   ├── .dart_tool/
│   │   ├── .gitignore
│   │   ├── .idea/
│   │   ├── .metadata
│   │   ├── analysis_options.yaml
│   │   ├── android/
│   │   ├── build/
│   │   ├── lib/
│   │   │   ├── config/
│   │   │   │   └── providers.dart
│   │   │   ├── data/
│   │   │   │   └── services/
│   │   │   │       └── device_sensor_service.dart
│   │   │   ├── main.dart
│   │   │   └── ui/
│   │   │       └── node/
│   │   │           ├── node_screen.dart
│   │   │           └── node_view_model.dart
│   │   ├── pubspec.yaml
│   │   ├── README.md
│   │   ├── sensor_node.iml
│   │   └── test/
│   │       ├── data/
│   │       │   └── services/
│   │       │       └── device_sensor_service_test.dart
│   │       ├── support/
│   │       │   └── fakes.dart
│   │       └── ui/
│   │           └── node/
│   │               ├── node_screen_test.dart
│   │               └── node_view_model_test.dart
│   └── sensorctl/
│       ├── .dart_tool/
│       ├── .gitignore
│       ├── analysis_options.yaml
│       ├── bin/
│       │   └── sensorctl.dart
│       ├── CHANGELOG.md
│       ├── example/
│       │   ├── common_args.dart
│       │   ├── z_info.dart
│       │   ├── z_pub.dart
│       │   ├── z_put.dart
│       │   └── z_sub.dart
│       ├── lib/
│       │   └── config/
│       │       └── providers.dart
│       ├── pubspec.yaml
│       ├── README.md
│       └── test/
├── build/
├── dart_test.yaml
├── packages/
│   └── sensor_core/
│       ├── .dart_tool/
│       ├── .gitignore
│       ├── analysis_options.yaml
│       ├── CHANGELOG.md
│       ├── lib/
│       │   ├── sensor_core.dart
│       │   └── src/
│       │       ├── domain/
│       │       │   └── reading.dart
│       │       ├── repositories/
│       │       │   └── sensor_node_repository.dart
│       │       └── services/
│       │           ├── sensor_service.dart
│       │           ├── session_settings.dart
│       │           └── zenoh_service.dart
│       ├── pubspec.yaml
│       ├── README.md
│       └── test/
│           ├── repositories/
│           │   └── sensor_node_repository_test.dart
│           ├── services/
│           │   └── zenoh_service_test.dart
│           └── support/
│               ├── collector.dart
│               └── fakes.dart
├── pubspec.lock
└── pubspec.yaml
```

This chapter was checked with these versions. Newer ones should work. If something does not, go back to these.

| what | version |
|---|---|
| Linux | Ubuntu 26.04.1 on x86_64, glibc 2.43 |
| git | 2.53.0 |
| fvm | 4.3.1 |
| Flutter, and the Dart it carries | 3.47.2, Dart 3.13.2; sections 10–13 also by hand on 3.47.5, Dart 3.13.4 |
| VS Code | 1.138.0 |
| Dart and Flutter extensions | 3.142.0 |
| Android platform tools, `adb` | 1.0.41 (37.0.1) |
| Android emulator, and the virtual device | 37.1.11; API 37, x86_64 |
| Java, which Gradle runs on | 25.0.3, the one Android Studio carries |
| the phone | Pixel 9a, Android 17 |
| `zenoh_dart`, built on zenoh 1.8.0 | 1.0.0-rc.1 |
| `riverpod`, `flutter_riverpod` | 3.4.3 |
| `sensors_plus` | 7.1.0 |
| `args` | 2.7.0 |
| `very_good_analysis` | 11.0.0 |
| `test` | 1.31.1 |

---

*The code listings in this chapter are licensed under the Apache License 2.0. The text is © 2026 Hugo Alberto Garcia,
all rights reserved — see [COPYRIGHT](../COPYRIGHT).*

# 1 — A session of your own

This chapter follows chapter 4 of *Zenoh Programming in Rust*, *Sessions and Configuration*, and *The Zenoh Book*'s
page on sessions. It starts from `z_info`, a program shipped with the `zenoh_dart` package.

## 1 — What you build, and what you will see

By the end of this chapter, `sensorctl` is a program you wrote. It opens a zenoh session with a configuration you
wrote, says who it is and who it found, and closes. On the way, the top folder becomes a **workspace** with a second
package, `sensor_core`. The code that touches zenoh moves there, behind one class, `ZenohService`. For the whole
guide, that class stays the only file that imports `zenoh_dart`. A test drives the move. It is the first test you
write, and the first you watch fail.

With the package's `z_sub` running in a second terminal, as in chapter 0, `sensorctl` prints:

```
sensorctl is 8d2ee6a92269c47d3b9fd2897f946465
connected to 1 peer:
  8278f066730daedae9902fc949189943
```

Both ids differ on your machine, and change every time you run it, because a session picks a new id each time it
opens. The first is `sensorctl`'s own. The second is `z_sub`'s, and **nothing discovered it.** You tell `sensorctl`
where to look, as in chapter 0 you told `z_sub` where to listen. Zenoh finds no one on your behalf, because every
program in this guide says where it is and who it talks to, until the last chapter.

> **If you already know zenoh.** This is `z_info` with two changes. The configuration is in code, because the
> topology is fixed from here on: every program a peer, multicast scouting and gossip off, the sensor node listening
> on the loopback and the collectors connecting to it. And the zenoh calls go behind one class from the start,
> because the Flutter app in chapter 2 shares that class.

> **If you have not used a pub workspace, or Riverpod outside Flutter.** Both arrive in this chapter with the
> smallest example that needs them, and each is explained where it appears. A workspace is one `pubspec.yaml` at the
> top that resolves the dependencies of every package below it. A `ProviderContainer` is Riverpod without a widget
> tree, with the same providers the Flutter app uses in chapter 2.

## 2 — What to read

This guide builds on two books, and each does a different job. Read a little of each before you start.

**[The Zenoh Book](https://corsaro.me/zenoh/book/core-concepts/sessions/), *Core Concepts → Sessions*, for the
idea.** Read what a session is, your program's one connection to everything zenoh does. Read also what the three
modes mean and cost, what closing a session involves, and what it means to open more than one in a single process.
The last point matters here, because the test you write in this chapter opens two.

**[Zenoh Programming in Rust](https://kydos.github.io/zenoh-book/chapter_04.html), chapter 4, *Sessions and
Configuration*, for the shape of the API.** Read how to open a session, what the default configuration assumes, and
how configuration from a file compares with configuration built in code, which is what you do here. Read also session
info and graceful shutdown. Skip *Runtime Configuration via Admin Space*, because it changes a running router's
settings, and this guide has no router until chapter 5.

Both books use Rust. Read them for the ideas and the shapes of the API. Five things look different in Dart:

| in the books | in `zenoh_dart` |
|---|---|
| `zenoh::open(config).await.unwrap()` | `await Session.open(config: config)`, which throws on failure |
| `config.insert_json5("mode", r#""peer""#)` | `config.insertJson5('mode', '"peer"')`, with the same paths, the same `/` between nested keys, and the same JSON5 values |
| `session.info().zid().await` | `session.zid`, and `.toHexString()` to print it |
| `routers_zid()`, `peers_zid()` return async streams you `.collect()` | `routersZid()` and `peersZid()` return plain lists |
| `session.close().await.unwrap()` | `session.close()`, which returns nothing, needs no `await`, and is safe to call twice |

**Opening a session completes the handshake before it returns.** Chapter 4 makes the point briefly. Scouting, binding
and any connection the configuration asked for are all finished by the time you have the session, and this chapter
relies on that from its first test to its last line.

> **A note on versions.** *Zenoh Programming in Rust* is written against Zenoh 1.4.0, and `zenoh_dart` 1.0.0-rc.1 is
> built on 1.8.0, so a detail there may have changed since. Every statement this guide makes about the Dart API was
> read in the package itself.

## 3 — The program that opens a session

Write your first program of your own. First run a third example, `z_info`, which does what `sensorctl` is about to
do: it opens a session, says who it is and who it found, and closes. A program is easier to write once you have
watched it work.

**1. Go to the program's folder.** Chapter 0 ended in the top folder.

```sh
# in zenoh_sensors
cd apps/sensorctl
```

**2. Copy one more example.** Read the package's folder from `.dart_tool/package_config.json`, then copy the example,
as in chapter 0:

```sh
# in zenoh_sensors/apps/sensorctl
pkg=$(sed -n 's|.*"rootUri": "file://\(.*/zenoh_dart-[^/"]*\)".*|\1|p' .dart_tool/package_config.json)
cp "$pkg/example/z_info.dart" example/
```

**3. Start the subscriber.** In a terminal in that folder, start `z_sub` again and leave it running. It stands in for
the sensor node until chapter 2 builds one.

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart run example/z_sub.dart -l tcp/127.0.0.1:7447 --no-multicast-scouting
```

```
⋮
…Opening session...
Declaring Subscriber on 'demo/example/**'...
Press CTRL-C to quit...
```

Two marks in that block appear in every expected output. `⋮` stands for lines the guide does not show, here what pub
prints before the program speaks: `Running build hooks...`, and sometimes `Building package executable...` and
`Built …`, on one line or several, depending on what pub had to do. `…` stands for the part of a line that differs on
your machine, such as an identity or a timestamp. Everything else is printed exactly as shown.

**4. Ask who is there.** Open a second terminal, go to the same folder, and run `z_info`:

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart run example/z_info.dart \
  -e tcp/127.0.0.1:7447 --no-multicast-scouting --cfg 'listen/endpoints:[]'
```

```
⋮
…Opening session...
own id: …
routers ids:
peers ids:
…
… ERROR ThreadId(…) zenoh::api::admin: Unable to publish transport event: session closed
```

**Each session has its own id.** `own id` is 16 bytes, printed as 32 hexadecimal characters. A new one is made every
time a session opens, so yours differs from the one above. Zenoh uses the id in timestamps and in the names in its
admin space. In this guide you use it to tell one running program from another.

**`routers ids:` is empty.** A router is a separate program, `zenohd`, that sessions connect through instead of
connecting to each other. This guide does not use one until chapter 5, so the line needs no fix.

**`peers ids:` is `z_sub`.** It is the only other zenoh program on your machine, and it is there because you told both
programs where to be. The three options are the ones chapter 0 gave `z_put`. `-e` connects to the subscriber.
`--no-multicast-scouting` stops this program announcing itself on your network. `--cfg 'listen/endpoints:[]'` stops
it listening. Without that last one, a peer also listens on every interface, on a port picked at random. Nothing
searched, and nothing was discovered.

**Ignore the red `ERROR` line.** Zenoh 1.8.0 prints it when a connected session closes, as chapter 0 explained.
Nothing failed.

**5. Write your own program.** It replaces the template's `Hello world`. Replace
`zenoh_sensors/apps/sensorctl/bin/sensorctl.dart`:

```dart
import 'package:zenoh_dart/zenoh.dart';

Future<void> main() async {
  Zenoh.initLog('error');

  final config = Config()
    ..insertJson5('mode', '"peer"')
    ..insertJson5('scouting/multicast/enabled', 'false')
    ..insertJson5('scouting/gossip/enabled', 'false')
    ..insertJson5('listen/endpoints', '[]')
    ..insertJson5('connect/endpoints', '["tcp/127.0.0.1:7447"]');

  final session = await Session.open(config: config);
  try {
    print('sensorctl is ${session.zid.toHexString()}');
    final peers = session.peersZid();
    final noun = peers.length == 1 ? 'peer' : 'peers';
    print('connected to ${peers.length} $noun:');
    for (final peer in peers) {
      print('  ${peer.toHexString()}');
    }
  } finally {
    session.close();
  }
}
```

> **In VS Code.** Open the file from the Explorer on the left, `apps` › `sensorctl` › `bin` › `sensorctl.dart`, and
> replace what is in it.

`Zenoh.initLog('error')` turns on zenoh's own logging, at the level that prints errors and nothing else. It comes first
in `main`, because a session that fails to open throws an exception with one code for nearly every cause. The sentence
that says what went wrong is in that log. Without it, a wrong endpoint and an unreachable machine look the same.

The five settings are this guide's topology, written down once:

- **`mode: peer`**, because there is no router to be a client of.
- **Multicast scouting and gossip off**, because both find sessions you were not told about, and nothing here should
  be found by accident.
- **`listen/endpoints: []`**, because `sensorctl` is a collector. It starts conversations and never accepts one, so it
  needs no address of its own. Without this setting, a peer listens on every interface.
- **`connect/endpoints`**, the one address it needs: the node's, on the loopback, where `z_sub` is waiting.

The rest is what `z_info` did. `Session.open` is a `Future` because the connection happens while you wait. When it
returns, the session is open, and the connection it was told to make has been made, or its first attempt has failed.
Zenoh keeps retrying a failed connection in the background, so a sensor node that starts later is still found.
`session.zid` is the id, and `peersZid()` is the list you saw. `close()` sits in a `finally` so that it runs even if
printing throws, a habit chapter 12 builds on when it handles signals.

**6. Run it.** In the second terminal, with `z_sub` still running in the first:

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart run bin/sensorctl.dart
```

```
⋮
…sensorctl is …
connected to 1 peer:
  …
… ERROR ThreadId(…) zenoh::api::admin: Unable to publish transport event: session closed
```

Your program found the subscriber and printed the same id `z_info` printed.

**7. Give it an entry in Run and Debug.** Chapter 0 wrote entries for the two examples, and your program gets one too.
The file keeps the two entries you have and gains a third. Replace `zenoh_sensors/.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "z_sub",
      "type": "dart",
      "request": "launch",
      "program": "${workspaceFolder}/apps/sensorctl/example/z_sub.dart",
      "cwd": "${workspaceFolder}/apps/sensorctl",
      "args": ["-l", "tcp/127.0.0.1:7447", "--no-multicast-scouting"]
    },
    {
      "name": "z_put",
      "type": "dart",
      "request": "launch",
      "program": "${workspaceFolder}/apps/sensorctl/example/z_put.dart",
      "cwd": "${workspaceFolder}/apps/sensorctl",
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
      "cwd": "${workspaceFolder}/apps/sensorctl"
    }
  ]
}
```

`sensorctl` needs no `args`, because everything the examples took from the command line is in its own code now.

> **In VS Code.** With `z_sub` still running, choose **sensorctl** at the top of the Run and Debug view and press ▶.
> The Debug Console prints what the terminal printed. Only the entry itself is new, and its `cwd` line. That line
> starts the program in the folder that holds its own `pubspec.yaml`, because the package unpacked zenoh's native
> library there in chapter 0. **The next section moves that folder, and all three entries change with it.** The
> change comes from the workspace, and you see it happen.

**8. Stop the subscriber, and ask once more.** Press Ctrl-C in the first terminal, then run `sensorctl` again:

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart run bin/sensorctl.dart
```

```
⋮
…sensorctl is …
connected to 0 peers:
```

**The session still opened.** Nothing was listening at `tcp/127.0.0.1:7447`. `sensorctl` spent about half a second
trying, then carried on with no peers and no complaint. It printed no `ERROR` line either, because a session that was
never connected has nothing to complain about as it closes.

**An open session is not necessarily a connected one.** `open()` succeeding means your configuration was valid and
zenoh is running, and it says nothing about the other end. A test later in this chapter pins this down, so it stays
true as the code moves.

**9. Delete the template's leftovers.** When `dart create` made this project in chapter 0, it wrote a small library,
`lib/sensorctl.dart`, with a `calculate()` function and a test for it. Its program imported that library and printed
`Hello world: 42!`, which proved the toolchain worked. Your program imports none of it, so both files are dead. Delete
them:

```sh
# in zenoh_sensors/apps/sensorctl
rm lib/sensorctl.dart test/sensorctl_test.dart
```

`lib/` and `test/` come back with code of your own. `lib/` returns at the end of this chapter, holding the providers
that wire the program together. Its view models and its terminal view arrive in chapter 3, when `bin/sensorctl.dart`
shrinks to the few lines that start them. `test/` gets this program's first test in chapter 3, when there is something
of its own to test. The deletion follows a rule of this guide: **delete code the moment nothing uses it.**

**10. Switch on stricter lints, now that the code is yours.** `dart create` gave the program the Dart team's
recommended lint set, `lints`, which the template's own code was written to. Everything in this folder is yours now,
and this guide holds its own code to a stricter set,
[`very_good_analysis`](https://pub.dev/packages/very_good_analysis). It has about 200 rules, and they apply whole, with
none switched off in the code you write. Add it as a development dependency:

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart pub add --dev very_good_analysis
```

Then replace the template's analysis options and their long comment. Replace
`zenoh_sensors/apps/sensorctl/analysis_options.yaml`:

```yaml
include: package:very_good_analysis/analysis_options.yaml

analyzer:
  exclude:
    - example/**
```

The first line switches the rules on. The exclusion is for the folder of copied examples. They are the package's
programs, written to the package's rules, so your lint set does not apply to them. Now run the analyzer on the program
you wrote:

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart analyze
```

```
Analyzing sensorctl...

   info - bin/sensorctl.dart:15:5 - Don't invoke 'print' in production code. Try using a logging framework. - avoid_print
   info - bin/sensorctl.dart:18:5 - Don't invoke 'print' in production code. Try using a logging framework. - avoid_print
   info - bin/sensorctl.dart:20:7 - Don't invoke 'print' in production code. Try using a logging framework. - avoid_print

3 issues found.
```

Three findings, one per `print`. `print` is for a developer reading a console while debugging, and a program's real
output goes through `stdout`, a stream a shell can redirect and a test can capture. Write the same program through
`stdout`. Replace `zenoh_sensors/apps/sensorctl/bin/sensorctl.dart`:

```dart
import 'dart:io';

import 'package:zenoh_dart/zenoh.dart';

Future<void> main() async {
  Zenoh.initLog('error');

  final config = Config()
    ..insertJson5('mode', '"peer"')
    ..insertJson5('scouting/multicast/enabled', 'false')
    ..insertJson5('scouting/gossip/enabled', 'false')
    ..insertJson5('listen/endpoints', '[]')
    ..insertJson5('connect/endpoints', '["tcp/127.0.0.1:7447"]');

  final session = await Session.open(config: config);
  try {
    stdout.writeln('sensorctl is ${session.zid.toHexString()}');
    final peers = session.peersZid();
    final noun = peers.length == 1 ? 'peer' : 'peers';
    stdout.writeln('connected to ${peers.length} $noun:');
    for (final peer in peers) {
      stdout.writeln('  ${peer.toHexString()}');
    }
  } finally {
    session.close();
  }
}
```

`stdout` comes from `dart:io`, and `writeln` writes one line to it. The output does not change. Run the analyzer
again:

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart analyze
```

It prints `No issues found!`, and from here it prints that after every section of this guide, for every package. The
rules ask for things as you go: a doc comment on every public class and member, `package:` imports inside `lib/`, and
Dart 3.13's shorter way to write a constructor, which you meet in section 5. Each is explained where it first appears.

> **In VS Code.** The Problems panel, **View › Problems**, lists the same three findings as soon as you save the new
> `analysis_options.yaml`, with a squiggle under each `print`. They go when you save the new program.

`sensorctl` is yours, and it runs clean. Section 4 makes the top folder a workspace and adds the core package.

## 4 — A workspace, and a package to share

Make the core package, and the workspace that lets two programs share it. The zenoh code you are about to write is not
only `sensorctl`'s. In chapter 2 a Flutter app on a phone opens a session of its own, with the same class, the same
settings and the same tests. So the code belongs in a package both programs depend on. A **pub workspace** is one
folder at the top that resolves the dependencies of everything below it. It also changes where you run things from,
for the rest of the guide.

**1. Go back to the top folder.**

```sh
# in zenoh_sensors/apps/sensorctl
cd ../..
```

**2. Make the core package.** `dart create` does not make the folder above the one it creates, so make `packages`
first:

```sh
# in zenoh_sensors
mkdir packages
fvm dart create --template=package packages/sensor_core
```

You now have a second Dart package with its own `pubspec.yaml`, its own `lib/`, and its own `test/` holding one
passing test. Nothing connects it to `sensorctl` yet.

`dart create` also wrote a demonstration library, a class called `Awesome` in `lib/src/sensor_core_base.dart`, with a
test and an `example/` folder. None of it is yours, and the lint set this package is about to get would flag it, so
delete it, as you deleted the program's leftovers:

```sh
# in zenoh_sensors
rm -r packages/sensor_core/example
rm packages/sensor_core/lib/src/sensor_core_base.dart packages/sensor_core/test/sensor_core_test.dart
```

The library's own file still exports the class you just deleted. Give it a library comment and no exports, until
section 5 gives it something. Replace `zenoh_sensors/packages/sensor_core/lib/sensor_core.dart`:

```dart
/// The zenoh data layer that `sensorctl` and the phone app share.
library;
```

The top folder now looks like this:

```
zenoh_sensors/
├── apps/
│   └── sensorctl/
└── packages/
    └── sensor_core/
        ├── lib/
        │   ├── sensor_core.dart
        │   └── src/
        ├── test/
        └── pubspec.yaml
```

**3. Write the workspace's own `pubspec.yaml`.** Create `zenoh_sensors/pubspec.yaml`:

```yaml
name: zenoh_sensors
publish_to: none

environment:
  sdk: ^3.13.2

dependencies:
  sensor_core: ^1.0.0

workspace:
  - apps/sensorctl
  - packages/sensor_core
```

This package holds no code and is never published, as `publish_to: none` says. It names the members and owns the one
resolution they share. It also depends on `sensor_core`, although it imports nothing. `dart test` builds a package's
native libraries only for the package of the folder it runs in and that package's dependencies. A top folder that
depended on nothing would leave the core's tests, run from here, without zenoh's library. With the dependency,
running them from here builds it.

**4. Tell each package that it belongs to the workspace.** Add one line, `resolution: workspace`, to each member's
`pubspec.yaml`. Replace `zenoh_sensors/apps/sensorctl/pubspec.yaml`:

```yaml
name: sensorctl
description: Watch, query and command the zenoh sensor network from a terminal.
version: 1.0.0

environment:
  sdk: ^3.13.2

resolution: workspace

dependencies:
  args: ^2.7.0
  zenoh_dart: ^1.0.0-rc.1

dev_dependencies:
  test: ^1.25.6
  very_good_analysis: ^11.0.0
```

Four of `dart create`'s lines go at the same time:

- The description was `A sample command-line application.`, which is true of a sample and not of this program.
- The commented-out `repository:` pointed at `my_org/my_repo`.
- `path` was never imported by anything here.
- `lints` stopped being read when section 3 switched the lint set.

**A dependency nothing uses is dead, like an unused file.** It still resolves, still pins a version, and still needs
explaining to whoever reads the file next.

Make the same change to the core, with the same lines removed and the lint set swapped. It has no `dependencies:` yet,
because it depends on nothing until section 6 gives it `zenoh_dart`. Replace
`zenoh_sensors/packages/sensor_core/pubspec.yaml`:

```yaml
name: sensor_core
description: The zenoh data layer that sensorctl and the phone app share.
version: 1.0.0

environment:
  sdk: ^3.13.2

resolution: workspace

dev_dependencies:
  test: ^1.25.6
  very_good_analysis: ^11.0.0
```

The core's analysis options are the program's, without the exclusion, because nothing is copied into this package.
Replace `zenoh_sensors/packages/sensor_core/analysis_options.yaml`:

```yaml
include: package:very_good_analysis/analysis_options.yaml
```

**5. Resolve once, from the top.**

```sh
# in zenoh_sensors
fvm dart pub get
```

It prints two or three lines about deleting an old lock file and an old package config, with a link to a page that
explains them. Nothing is wrong. Until now each package resolved its own dependencies and kept its own
`pubspec.lock`, and a workspace has one of each for everything. From here there is a single `pubspec.lock` and a
single `.dart_tool/` beside the top `pubspec.yaml`, and the per-package ones are gone.

**6. Delete what chapter 0 left behind.**

```sh
# in zenoh_sensors
rm -rf apps/sensorctl/.dart_tool
```

`zenoh_dart`'s build hook stages zenoh's native libraries into the `.dart_tool/` of whatever is being resolved, and it
has just staged them at the top. The copy chapter 0 staged in `apps/sensorctl/.dart_tool/` was still there, because
`pub get` removed only the package config beside it. A program started inside `apps/sensorctl` would find that old
copy first and run on it, a second, ageing copy of the library that no longer gets updated. The command deletes it, so
from here there is only one copy.

**7. Run the program from the top folder:**

```sh
# in zenoh_sensors
fvm dart run sensorctl:sensorctl
```

```
⋮
…sensorctl is …
connected to 0 peers:
```

`sensorctl:sensorctl` is the *package name*, then the *program name*: the file `bin/sensorctl.dart` inside the package
`sensorctl`. Zero peers is right, because nothing is listening now. You stopped `z_sub` at the end of the last
section.

**From here, run programs and tests from `zenoh_sensors`,** never from inside a package: `fvm dart run
sensorctl:sensorctl` for the program, and `fvm dart test packages/sensor_core` for the core's tests. The reason is the
library the build hooks staged. It is at the top now, and a program started inside `apps/sensorctl` cannot find it.
The copy you just deleted was hiding that. The tests find the library because the top folder's `pubspec.yaml` depends
on `sensor_core`, and `dart test` stages native libraries only for the folder's own package and its dependencies. The
next sections rely on this, and so does every chapter after.

**8. Update the editor.** All three entries in `zenoh_sensors/.vscode/launch.json` start programs in
`apps/sensorctl`, which no longer holds the package's libraries. Replace `zenoh_sensors/.vscode/launch.json`:

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
      "args": ["-l", "tcp/127.0.0.1:7447", "--no-multicast-scouting"]
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
    }
  ]
}
```

In each entry, `cwd` is now the folder VS Code has open, the top of the workspace. This is the terminal's rule,
written where the editor can read it.

> **In VS Code.** Press ▶ on **sensorctl** again. It prints the same as before. With the old file it would fail to
> start, with an error about not finding `libzenoh_dart.so`, because the old `cwd` points at a folder that no longer
> holds the library.

The top folder is a workspace with two members, and everything runs from it. Section 5 writes the chapter's first
test.

## 5 — The test of the chapter's claim

Write the test that states the chapter's claim, and watch it fail. From here on, write each test before the code it
tests. This is test-driven development, or TDD, and the rest of the guide is built this way.

The test makes you state what you want precisely enough to run. **If you cannot write the assertion, you do not yet
know what you are building.** The test is also the first caller of the new code, so the way the test calls it sets the
code's shape.

**1. State the claim in one sentence.**

> A sensor node and a collector, configured the way this guide configures them, find each other on the loopback.

The sentence is the test's name, and everything else in the chapter exists to make it true.

**2. Make room.**

```sh
# in zenoh_sensors
mkdir -p packages/sensor_core/lib/src/services packages/sensor_core/test/services
```

`test/` mirrors `lib/`. A file's test sits at the same path under `test/` as the file under `lib/src/`. The
arrangement holds for the whole guide, and it is easier to start it now than to impose it later.

**3. Write the test.** Create `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:test/test.dart';

void main() {
  test('a sensor node and a collector find each other on loopback', () async {
    // The two ends: the sensor node, which listens, and a collector,
    // which connects to it. Each closes when the test ends.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    // The code to implement: two sessions opened from the guide's settings.
    // The sensor node opens first, so the collector has something to reach.
    await sensorNode.open();
    await collectorNode.open();

    // The claim: each one's peers hold the other's identity, and the two
    // identities differ.
    expect(collectorNode.peerIds, contains(sensorNode.zid));
    expect(sensorNode.peerIds, contains(collectorNode.zid));
    expect(collectorNode.zid, isNot(sensorNode.zid));
  });
}
```

The test opens two sessions in one Dart process, and they do to each other what `sensorctl` and `z_sub` did in two
terminals. Zenoh is the subject, so a claim about zenoh is checked against zenoh itself. Two kinds of test in this
guide open real sessions: the test of each chapter's claim, and the tests of `ZenohService`. Every other test runs
against stand-ins.

**The sensor node opens first.** A collector needs something to connect to, and its settings say where. Swap the two
lines, and the collector starts before anything is listening, as in section 4's last run.

**`addTearDown`** hands the closing to the test runner, so the sessions close even when an expectation fails. Without
it, a failing test leaves port 7447 held, and the next run fails for a reason unrelated to your code.

**The three expectations state the claim, in order.** The first two are the claim itself: each one's list of peers
holds the other's identity. The third is the guard. If both sessions reported the same id, `contains` would pass
while nothing had been found.

**4. Write just enough for it to compile.** Three files, none of which does anything yet. The settings come first.
Create `zenoh_sensors/packages/sensor_core/lib/src/services/session_settings.dart`:

```dart
/// A session's settings, by role: the sensor node's, or a collector's.
class SessionSettings {
  const new _();

  /// The sensor node's settings.
  factory sensorNode() => const SessionSettings._();

  /// A collector's settings.
  factory collectorNode() => const SessionSettings._();
}
```

Two of the lint set's rules show here for the first time:

- **Every public class and member has a doc comment**, the `///` lines. Each is one sentence saying what the thing
  is, and the editor shows it wherever the name is used. A comment is a claim, so each one in this guide has been
  checked against the code below it.
- **A constructor is written `new`**, without repeating the class's name. `const new _()` is the private constructor
  `SessionSettings._`, and `factory sensorNode()` is the factory `SessionSettings.sensorNode`, as Dart 3.13 lets you
  write them. Calling them has not changed.

Then the service. Create `zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'package:sensor_core/src/services/session_settings.dart';

/// The one class that talks to zenoh. It owns the session and hands plain
/// Dart values upward.
class ZenohService {
  /// A service for one role's [settings]. Nothing opens until [open].
  new(this.settings);

  /// Which side of the topology this session is on.
  final SessionSettings settings;

  /// Opens the session.
  Future<void> open() async {}

  /// The session's identity.
  String get zid => '';

  /// The identities of the peers this session is connected to.
  List<String> get peerIds => const [];

  /// Closes the session.
  void dispose() {}
}
```

The third rule shows here. Inside `lib/`, a file imports another by its `package:` path, never by a relative one. The
doc comments say what each member will do, and the bodies do nothing yet. The package's library file, the empty one
from section 4, now exports the two files. Replace `zenoh_sensors/packages/sensor_core/lib/sensor_core.dart`:

```dart
/// The zenoh data layer that `sensorctl` and the phone app share.
library;

export 'src/services/session_settings.dart';
export 'src/services/zenoh_service.dart';
```

**These are the emptiest classes that compile.** They answer every question with an empty value. Watching them fail
checks that the test is worth keeping. If something this empty could satisfy the test, the test would check only the
shape of a class, and it would keep passing long after the code stopped working.

**5. Run it, and read the failure.**

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

```
⋮
  Expected: contains ''
    Actual: []
     Which: does not contain ''
⋮
```

The collector's list of peers should hold the sensor node's identity, and it is empty. The test fails on its claim,
because the skeletons answer with empty values.

**This test stays red until the end of the chapter.** Each of its expectations is made true by a smaller test in
sections 6 and 7, each red before its code:

```
a sensor node and a collector find each other on loopback
 ├─ the collector's peers hold the sensor node's id    section 7, cycle 4: the endpoints go in
 ├─ the sensor node's peers hold the collector's id    section 7, cycle 4
 └─ the two ids differ                                 section 6, cycles 1 and 2: each session's own identity
```

Section 7's cycle 3, neither side announces itself on the network, is asked for by the guide's topology, not by this
test. It comes before cycle 4, so that cycle 4's green means the address did the work.

This test is the outer of two loops. It states the chapter's claim and turns green once, at the end. The inner loop
does the work in small cycles. Each cycle has its own test, which goes red and then green.

**6. Run tests from the top folder in VS Code too.** VS Code needs an entry for that. The file keeps the three entries
you have and gains a fourth, with no program in it. Replace `zenoh_sensors/.vscode/launch.json`:

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
      "args": ["-l", "tcp/127.0.0.1:7447", "--no-multicast-scouting"]
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
      "name": "tests",
      "type": "dart",
      "request": "launch",
      "templateFor": "",
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

An entry with `templateFor` and no `program` is a template. The editor takes its `cwd` for every test it runs from the
Testing view, and for the **Run** and **Debug** links it shows above a test. Without it, a test would start inside
`packages/sensor_core`, where there is no copy of zenoh's library, and fail before it began.

> **In VS Code.** Open the Testing view, the flask in the Activity Bar, and press ▶ beside the test, or click **Run**
> above it in the editor. It fails the same way.

The outer test is red. Section 6 starts the inner cycles with each session's identity.

## 6 — Two cycles: an identity

Give each session its identity, in two cycles. The outer test stays red for the rest of the chapter. The work happens
in **cycles**, and a cycle is small: write one test, run it and watch it fail, do the least that makes it pass, and
run it again. Four cycles build the service, two here and two in section 7. Inside a cycle, run only that cycle's
test, by a piece of its name, with `-n`.

**Cycle 1 — a service has an identity once it is open.** From here the guide shows a test file by what changes in
it. `⋮` stands for everything already there, and what follows goes at the end of the file, before `main`'s closing
brace, which the block shows so that you can see where. When the imports change, a block shows them above the `⋮`,
as they now read. Add a second test to `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('a service has an identity once it is open', () async {
    // The sensor node's end, closed when the test ends.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);

    // The code to implement: an identity, once the service is open.
    await sensorNode.open();

    // The claim: the identity is not empty.
    expect(sensorNode.zid, isNotEmpty);
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'has an identity'
```

```
⋮
  Expected: non-empty
    Actual: ''
⋮
```

The identity is empty, because the skeleton's `zid` answers `''`.

**Fake it.** Do the least that makes the test pass, which here is one constant in the same skeleton. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'package:sensor_core/src/services/session_settings.dart';

/// The one class that talks to zenoh. It owns the session and hands plain
/// Dart values upward.
class ZenohService {
  /// A service for one role's [settings]. Nothing opens until [open].
  new(this.settings);

  /// Which side of the topology this session is on.
  final SessionSettings settings;

  /// Opens the session.
  Future<void> open() async {}

  /// The session's identity.
  String get zid => 'the-sensor-node';

  /// The identities of the peers this session is connected to.
  List<String> get peerIds => const [];

  /// Closes the session.
  void dispose() {}
}
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'has an identity'
```

It passes. `dart test` says so in one line, `All tests passed!`, and a passing run prints nothing more. So from here
the guide shows nothing for a pass, and only the lines that matter for a failure.

This technique is called **fake it**. The test asks for an identity that is not empty, and a constant is one. It is
not the real answer, and the next test will replace it. Faking is wrong only as the *last* step. The constant is the
same for every service, so the next cycle opens two.

**Cycle 2 — two services have different identities.** Add a third test to
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('two services have different identities', () async {
    // Two ends, closed when the test ends.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    // The code to implement: each open service gets its own identity.
    await sensorNode.open();
    await collectorNode.open();

    // The claim: the two identities differ.
    expect(collectorNode.zid, isNot(sensorNode.zid));
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'different identities'
```

```
⋮
  Expected: not 'the-sensor-node'
    Actual: 'the-sensor-node'
⋮
```

Both services answer the same constant.

**Triangulate.** Open a real session in each service, because no constant differs from itself. Adding a second case
that the shortcut cannot satisfy is called **triangulation**, and it is how a test forces code into existence.

Two real sessions need the package, so the core now depends on it, as section 4 put off. Replace
`zenoh_sensors/packages/sensor_core/pubspec.yaml`:

```yaml
name: sensor_core
description: The zenoh data layer that sensorctl and the phone app share.
version: 1.0.0

environment:
  sdk: ^3.13.2

resolution: workspace

dependencies:
  zenoh_dart: ^1.0.0-rc.1

dev_dependencies:
  test: ^1.25.6
  very_good_analysis: ^11.0.0
```

Then resolve from the top:

```sh
# in zenoh_sensors
fvm dart pub get
```

> **In VS Code.** Open `packages` › `sensor_core` › `pubspec.yaml`, add the two lines, and save. The Dart extension
> runs `pub get` whenever a `pubspec.yaml` is saved.

Then open a session in the service. Replace `zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'package:sensor_core/src/services/session_settings.dart';
import 'package:zenoh_dart/zenoh.dart';

/// The one class that talks to zenoh. It owns the session and hands plain
/// Dart values upward.
class ZenohService {
  /// A service for one role's [settings]. Nothing opens until [open].
  new(this.settings);

  /// Which side of the topology this session is on.
  final SessionSettings settings;

  Session? _session;

  /// Opens the session. When this returns, it is open.
  Future<void> open() async {
    _session = await Session.open(config: Config());
  }

  /// The session's identity: thirty-two hexadecimal characters.
  String get zid => _opened.zid.toHexString();

  /// The identities of the peers this session is connected to.
  List<String> get peerIds => const [];

  /// Closes the session. Safe before [open], and more than once.
  void dispose() {
    _session?.close();
    _session = null;
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

Run both:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'identit'
```

Both pass. `-n 'identit'` matches both test names, the quickest check that the new code did not break the old test.
The imports are in alphabetical order of their packages, which is another of the lint rules, and the analyzer says so
if they are not.

**The session is what the test asked for.** `Session.open` is awaited, the handle is kept, and `zid` reports it as 32
hexadecimal characters, the same id `z_info` printed in section 3.

**`_opened` turns a mistake into a sentence.** Asking a service for its id before opening it is a programming error.
So it throws a `StateError` that names what was not done. Without it, the error would be a null error from inside the
package.

**The configuration is one nothing has asked for yet.** `Session.open(config: Config())` is zenoh's *default*, the
one section 3 warned about, which listens on every interface and announces itself to the network. It is here because
it is the least that opened a session, and no test has asked for better yet. **Section 7 asks.**

**What the tests guarantee:** a service has a real zenoh identity once it is open, and two services have different
ones.

Each session has its own identity. Section 7 gives the sessions the guide's topology, and the outer test goes green.

## 7 — Two cycles: a topology, and a connection

Give both sides the guide's topology in two cycles, and the outer test goes green. The outer test is still red, and
the service still opens zenoh's default configuration. Cycle 3 takes away the two ways a session finds sessions it
was never told about. Cycle 4 gives each side its address.

Cycle 3 comes first, because a green means only as much as the red before it. With the addresses in and scouting
still on, the outer test could go green because the network found the sessions, and nothing in its output would say
so. With scouting off first, the addresses are the only way left, and the green that follows means what it says.

**Cycle 3 — neither side announces itself on the network.** Section 3 wrote five settings into `bin/sensorctl.dart`.
Three of them are the same on both sides: every session in this guide is a `peer`, and it neither scouts nor gossips.
Scouting is how zenoh finds sessions nobody configured. A new session calls out on a multicast address, and every
session that hears it answers with where it can be reached. The Zenoh Book's
[Multicast Scouting](https://corsaro.me/zenoh/book/routing/scouting/) page shows that exchange in one picture. Gossip
is the second-hand version, where a session passes on to the sessions it knows the addresses of the others it has met.

Both stay off until the last chapter, because nothing before it should be found by accident. This cycle's test opens
no session. These three settings make something *not* happen. A test that opens two sessions can watch them find each
other, but it cannot watch them not find anyone else, because there is no one else in the test to find. To see this
once the chapter is finished, delete the two `scouting` entries from the settings. The outer test still passes.

So a setting whose whole effect is an absence is read back as data and compared with what it should say. The test
reads the settings through a new property, `asJson5`. It is a map from a key path to a JSON5 value. These are the
entries `insertJson5` takes, the same pairs section 3 wrote by hand. Add a fourth test to
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('neither side announces itself on the network', () {
    // The code to implement: both sides' settings, read back as data. An
    // absence cannot be watched, so it is checked as the value behind it.
    final sensorSettings = SessionSettings.sensorNode().asJson5;
    final collectorSettings = SessionSettings.collectorNode().asJson5;

    // The claim: every session is a peer that neither scouts nor gossips.
    for (final settings in [sensorSettings, collectorSettings]) {
      expect(settings, containsPair('mode', '"peer"'));
      expect(settings, containsPair('scouting/multicast/enabled', 'false'));
      expect(settings, containsPair('scouting/gossip/enabled', 'false'));
    }
  });
}
```

Then give `SessionSettings` the emptiest `asJson5` that compiles, so that the test fails on its claim. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/session_settings.dart`:

```dart
/// A session's settings, by role: the sensor node's, or a collector's.
class SessionSettings {
  const new _();

  /// The sensor node's settings.
  factory sensorNode() => const SessionSettings._();

  /// A collector's settings.
  factory collectorNode() => const SessionSettings._();

  /// The settings as the entries `Config.insertJson5` takes: a key path and a
  /// JSON5 value each.
  Map<String, String> get asJson5 => const {};
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'announces'
```

```
⋮
  Expected: contains pair 'mode' => '"peer"'
    Actual: {}
     Which:  doesn't contain key 'mode'
⋮
```

The settings are empty, so `mode` is missing.

**Write the obvious implementation.** This is the third way to green, after fake it and triangulate: when you know what
to write, write it. There is no fake to make here. When the thing under test is data, the least that passes and the
real thing are the same three lines, so the test restates the code. Its job is to make those three lines impossible to
delete without a test going red, and no other test in this chapter can do that. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/session_settings.dart`:

```dart
/// A session's settings, by role: the sensor node's, or a collector's. The
/// three settings that never vary are in [asJson5] too.
class SessionSettings {
  const new _();

  /// The sensor node's settings.
  factory sensorNode() => const SessionSettings._();

  /// A collector's settings.
  factory collectorNode() => const SessionSettings._();

  /// The settings as the entries `Config.insertJson5` takes: a key path and a
  /// JSON5 value each.
  Map<String, String> get asJson5 => const {
    'mode': '"peer"',
    'scouting/multicast/enabled': 'false',
    'scouting/gossip/enabled': 'false',
  };
}
```

Then build the service's configuration from it. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'package:sensor_core/src/services/session_settings.dart';
import 'package:zenoh_dart/zenoh.dart';

/// The one class that talks to zenoh. It owns the session and hands plain
/// Dart values upward.
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

  /// The session's identity: thirty-two hexadecimal characters.
  String get zid => _opened.zid.toHexString();

  /// The identities of the peers this session is connected to.
  List<String> get peerIds => const [];

  /// Closes the session. Safe before [open], and more than once.
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

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'announces'
```

It passes. Run the two identity tests too, with `-n 'identit'`. They still pass. Both sessions open as peers that do
not scout, and each gets an id.

No test asked for `_config()`. `Config()` is zenoh's default configuration, and `forEach(config.insertJson5)` passes
each key and value of the map to `insertJson5`, so the map holds every difference from the default. No test shows
these three settings take effect yet, because `peer` is already zenoh's default mode and the other two switch scouting
off, which is an absence. Cycle 4 adds the endpoints to the same map, and its outer test fails unless this line
applies them.

**What the test guarantees:** every session this guide opens is a peer that neither looks for other sessions nor
passes on word of the ones it has met.

**Cycle 4 — they find each other.** Before you run the outer test again, remove the last neutral answer from section 5.
`peerIds` still returns an empty constant, and a red produced by a stub says nothing about zenoh. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'package:sensor_core/src/services/session_settings.dart';
import 'package:zenoh_dart/zenoh.dart';

/// The one class that talks to zenoh. It owns the session and hands plain
/// Dart values upward.
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

  /// The session's identity: thirty-two hexadecimal characters.
  String get zid => _opened.zid.toHexString();

  /// The identities of the peers this session is connected to.
  List<String> get peerIds =>
      _opened.peersZid().map((id) => id.toHexString()).toList();

  /// Closes the session. Safe before [open], and more than once.
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

`peersZid()` is the list `z_info` printed under `peers ids:`. As with `zid`, the service turns each id into its 32
characters before handing it up, so nothing above the service needs a type from the package.

Now run the outer test:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'find each other'
```

```
⋮
  Expected: contains '…'
    Actual: []
     Which: does not contain '…'
⋮
```

Compare this red with the one at the end of section 5. The id now comes from zenoh, 32 characters. The empty list also
comes from zenoh. The two sessions are peers in one process, both with scouting off, and neither has the other's
address. Cycle 3 set this up, and every program in this guide starts this way. A session finds another only when it
is given an address.

The sensor node waits at an address, and the collector goes to it. That is data again, so the test is of the same
kind as the last one. Add a fifth test to `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('the collector connects to where the sensor node listens', () {
    // The code to implement: the endpoints, read back as data.
    const address = '["tcp/127.0.0.1:7447"]';
    final sensorSettings = SessionSettings.sensorNode().asJson5;
    final collectorSettings = SessionSettings.collectorNode().asJson5;

    // The claim: the sensor node listens at the address and connects to
    // nothing, and the collector listens nowhere and connects to it.
    expect(sensorSettings, containsPair('listen/endpoints', address));
    expect(sensorSettings, containsPair('connect/endpoints', '[]'));
    expect(collectorSettings, containsPair('listen/endpoints', '[]'));
    expect(collectorSettings, containsPair('connect/endpoints', address));
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'connects to where'
```

```
⋮
  Expected: contains pair 'listen/endpoints' => '["tcp/127.0.0.1:7447"]'
    Actual: {
              'mode': '"peer"',
              'scouting/multicast/enabled': 'false',
              'scouting/gossip/enabled': 'false'
            }
     Which:  doesn't contain key 'listen/endpoints'
⋮
```

The settings have no endpoints yet.

**Write the obvious implementation.** The two lists are the only difference between the two sides, so they become
the value's two fields, and the two factories fill them in opposite ways. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/session_settings.dart`:

```dart
/// A session's settings, by role: where it listens and where it connects.
/// The three settings that never vary are in [asJson5] too.
class SessionSettings {
  const new _({required this.listenEndpoints, required this.connectEndpoints});

  /// The sensor node's settings: it listens on the loopback and connects
  /// nowhere.
  factory sensorNode() => const SessionSettings._(
    listenEndpoints: [nodeEndpoint],
    connectEndpoints: [],
  );

  /// A collector's settings: it listens nowhere and connects to the sensor
  /// node.
  factory collectorNode() => const SessionSettings._(
    listenEndpoints: [],
    connectEndpoints: [nodeEndpoint],
  );

  /// Where the sensor node waits: the loopback, port 7447.
  static const nodeEndpoint = 'tcp/127.0.0.1:7447';

  /// The endpoints the session listens on.
  final List<String> listenEndpoints;

  /// The endpoints the session connects to.
  final List<String> connectEndpoints;

  /// The settings as the entries `Config.insertJson5` takes: a key path and a
  /// JSON5 value each.
  Map<String, String> get asJson5 => {
    'mode': '"peer"',
    'scouting/multicast/enabled': 'false',
    'scouting/gossip/enabled': 'false',
    'listen/endpoints': _json5List(listenEndpoints),
    'connect/endpoints': _json5List(connectEndpoints),
  };

  static String _json5List(List<String> items) =>
      '[${items.map((item) => '"$item"').join(', ')}]';
}
```

`nodeEndpoint` is the only address this guide uses until chapter 5: the loopback, port 7447, where `z_sub` waited in
chapter 0 and where the sensor node on your phone waits from chapter 2. `listen/endpoints: []` on the collector is the
setting section 3 explained. Without it, a peer listens on every interface, on a port picked at random. `_json5List`
writes a Dart list as the JSON5 text `insertJson5` wants, `["tcp/127.0.0.1:7447"]` or `[]`.

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'connects to where'
```

It passes. Now run the outer test, for the last time:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'find each other'
```

It passes, and the chapter's claim is true. Run the whole file once, the way it runs from here on:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

5 tests pass.

**What the outer test guarantees:** a sensor node and a collector, configured as this guide configures them, find each
other on the loopback, and have done so when the collector's `open()` returns. The test asserts straight after the
second `open()`, with no wait and no retry. It holds because opening a session completes the handshake before it
returns, the point from chapter 4 of *Zenoh Programming in Rust* that section 2 flagged.

The service is finished. `bin/sensorctl.dart` still has its own five settings and opens its own session. In section 8
you move the program onto the service. That is the refactor, and the five tests stay green before and after.

## 8 — The refactor, and a provider container

Move `bin/sensorctl.dart` onto the service, and then onto a *provider container*. The program still opens zenoh by
itself, with its own copy of the five settings the service now carries and tests. A provider container is how every
program in this guide is put together from here on. This is the chapter's refactor. The behavior does not change,
the tests are green before and after, and the program prints what it printed in section 3.

**1. Check that the tests are green before you start.** A refactor starts from green, so that anything red on the way
is yours:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

5 tests pass.

**2. Let the program depend on the core.** One line is new. Replace `zenoh_sensors/apps/sensorctl/pubspec.yaml`:

```yaml
name: sensorctl
description: Watch, query and command the zenoh sensor network from a terminal.
version: 1.0.0

environment:
  sdk: ^3.13.2

resolution: workspace

dependencies:
  args: ^2.7.0
  sensor_core: ^1.0.0
  zenoh_dart: ^1.0.0-rc.1

dev_dependencies:
  test: ^1.25.6
  very_good_analysis: ^11.0.0
```

Then resolve from the top:

```sh
# in zenoh_sensors
fvm dart pub get
```

A workspace member depends on another by name. There is no path to write, and the version is the one
`packages/sensor_core/pubspec.yaml` declares, so pub resolves it to the folder next door. `zenoh_dart` stays, although
nothing of yours in this package imports it after this section. The three examples in `apps/sensorctl/example/` are
the package's own programs, and `z_sub` stands in for the sensor node until chapter 2 gives you one of your own.

> **In VS Code.** Open `apps` › `sensorctl` › `pubspec.yaml`, add the line, and save. The extension runs `pub get`.

**3. Start logging from the core.** Section 3's program started zenoh's logging with `Zenoh.initLog('error')`. That
call lives in the package, which `bin/sensorctl.dart` is about to stop importing, so the core offers it as a function
of its own. The class is unchanged, and one function is added above it. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'package:sensor_core/src/services/session_settings.dart';
import 'package:zenoh_dart/zenoh.dart';

/// Starts zenoh's own log at [level], printed to standard output: for a
/// program with a terminal. Once per process, before any session opens.
void initZenohLogging(String level) => Zenoh.initLog(level);

/// The one class that talks to zenoh. It owns the session and hands plain
/// Dart values upward.
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

  /// The session's identity: thirty-two hexadecimal characters.
  String get zid => _opened.zid.toHexString();

  /// The identities of the peers this session is connected to.
  List<String> get peerIds =>
      _opened.peersZid().map((id) => id.toHexString()).toList();

  /// Closes the session. Safe before [open], and more than once.
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

Logging belongs to the whole process, so it is a function beside the service. The package starts it once, and
ignores a later call, from a second service say, without a word. So `main` calls it once, before anything opens, as
the package's examples do.

The level you pass is a fallback. The `RUST_LOG` environment variable wins when it is set, which is how you turn on
more detail without editing the program. Misspell the level, `'errod'` say, and you get no log at all and no
complaint, because zenoh reads the word as a filter that matches nothing.

**4. Refactor the program.** Replace `zenoh_sensors/apps/sensorctl/bin/sensorctl.dart`:

```dart
import 'dart:io';

import 'package:sensor_core/sensor_core.dart';

Future<void> main() async {
  initZenohLogging('error');

  final service = ZenohService(SessionSettings.collectorNode());
  try {
    await service.open();
    stdout.writeln('sensorctl is ${service.zid}');
    final peers = service.peerIds;
    final noun = peers.length == 1 ? 'peer' : 'peers';
    stdout.writeln('connected to ${peers.length} $noun:');
    for (final peer in peers) {
      stdout.writeln('  $peer');
    }
  } finally {
    service.dispose();
  }
}
```

Compare it with section 3's. The five settings are gone, because they are `SessionSettings.collectorNode()` now, and
they are tested. `Config` and `Session` went with them. The program imports `sensor_core` and `dart:io` and nothing
else, and no longer knows that `zenoh_dart` exists. What is left is what the program is for: say who it is, and who it
found.

One thing moved. `open()` is inside the `try` now, which it could not be in section 3. A service exists before its
session does, and disposing one that never opened is safe, a rule section 9 pins with a test.

**5. Start the subscriber, and run the program.** In a terminal at the top folder, start `z_sub` and leave it running.
Everything starts from the top folder now, the examples too:

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

In a second terminal:

```sh
# in zenoh_sensors
fvm dart run sensorctl:sensorctl
```

```
⋮
…sensorctl is …
connected to 1 peer:
  …
… ERROR ThreadId(…) zenoh::api::admin: Unable to publish transport event: session closed
```

The output is section 3's three lines and zenoh 1.8.0's `ERROR` line as the session closes. Nothing needs fixing. The
program behaves as before, and only its structure changed. A refactor changes a program's structure and keeps its
behavior.

**6. Build the program from a provider container.** The program made its service by hand in `main`,
`ZenohService(SessionSettings.collectorNode())`. For one object that is fine. From chapter 3 the program has a view
model, which needs a repository, which needs the service, and a test must be able to swap any one of them for a
stand-in.

This guide builds that graph with [Riverpod](https://riverpod.dev). Every object is declared once, as a *provider*,
next to the others in one file. A *provider container* builds them on demand, disposes them together, and lets a test
override any one of them. In chapter 2's Flutter app the same providers live under a `ProviderScope` widget. A
`ProviderContainer` is that without a widget tree, so a pure-Dart program can use it.

`riverpod` is new. Replace `zenoh_sensors/apps/sensorctl/pubspec.yaml` once more:

```yaml
name: sensorctl
description: Watch, query and command the zenoh sensor network from a terminal.
version: 1.0.0

environment:
  sdk: ^3.13.2

resolution: workspace

dependencies:
  args: ^2.7.0
  riverpod: ^3.4.3
  sensor_core: ^1.0.0
  zenoh_dart: ^1.0.0-rc.1

dev_dependencies:
  test: ^1.25.6
  very_good_analysis: ^11.0.0
```

Then resolve from the top:

```sh
# in zenoh_sensors
fvm dart pub get
```

`lib/` comes back, with the folder chapter 3 fills. Create `zenoh_sensors/apps/sensorctl/lib/config/providers.dart`:

```dart
import 'package:riverpod/riverpod.dart';
import 'package:sensor_core/sensor_core.dart';

/// Which side of the topology this program is on: a collector.
final sessionSettingsProvider = Provider<SessionSettings>(
  (ref) => SessionSettings.collectorNode(),
);

/// The program's one zenoh session, disposed with the container.
final zenohServiceProvider = Provider<ZenohService>((ref) {
  final service = ZenohService(ref.watch(sessionSettingsProvider));
  ref.onDispose(service.dispose);
  return service;
});
```

The program's folder now looks like this:

```
zenoh_sensors/
└── apps/
    └── sensorctl/
        ├── bin/
        │   └── sensorctl.dart
        ├── example/
        └── lib/
            └── config/
                └── providers.dart
```

There are two providers. `sessionSettingsProvider` says which side of the topology this program is on: a collector.
It is a provider of its own because it is the entry that gets overridden, by a test and by `simulate`, the sensor node
inside this same program from chapter 4. `zenohServiceProvider` builds the service from it, and `ref.watch` reads
another provider's value. `ref.onDispose(service.dispose)` ties the session's life to the container's, so when the
container is disposed, so is the service, and the session closes.

**Dispose the service through the provider, never by hand.** That rule holds from here to the end of the guide.

Then build the program from the container. Replace `zenoh_sensors/apps/sensorctl/bin/sensorctl.dart`:

```dart
import 'dart:io';

import 'package:riverpod/riverpod.dart';
import 'package:sensor_core/sensor_core.dart';
import 'package:sensorctl/config/providers.dart';

Future<void> main() async {
  initZenohLogging('error');

  final container = ProviderContainer();
  try {
    final service = container.read(zenohServiceProvider);
    await service.open();
    stdout.writeln('sensorctl is ${service.zid}');
    final peers = service.peerIds;
    final noun = peers.length == 1 ? 'peer' : 'peers';
    stdout.writeln('connected to ${peers.length} $noun:');
    for (final peer in peers) {
      stdout.writeln('  $peer');
    }
  } finally {
    container.dispose();
  }
}
```

`container.read(zenohServiceProvider)` asks the container for the service. The container builds it, and the settings
it depends on, the first time it is asked, and hands back the same one after that. `container.dispose()` in the
`finally` is the only closing left in the program.

> **In VS Code.** With `z_sub` still running, press ▶ on **sensorctl**. The entry has not changed, because the
> program is still `bin/sensorctl.dart`, started from the top folder.

Run it again, with `z_sub` still running in the first terminal:

```sh
# in zenoh_sensors
fvm dart run sensorctl:sensorctl
```

```
⋮
…sensorctl is …
connected to 1 peer:
  …
… ERROR ThreadId(…) zenoh::api::admin: Unable to publish transport event: session closed
```

The output is unchanged.

**7. Stop the subscriber, and check that the tests are still green.** Press Ctrl-C in the first terminal, then run the
tests once more:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

5 tests pass. They cover the service, which still behaves as before, and a refactor relies on that. They do not cover
`main`, which has no test yet. You checked `main` by its output, twice. The program gets its own test in chapter 3,
when it has a view to test.

The program runs on the service, built by a container. Section 9 pins three more facts with tests.

## 9 — What the tests pin

Pin three more facts with tests. The four cycles drove the service into existence, and every line in it has a test
that asked for it. The three new tests go in green, because nothing is left to drive. Their job is to keep something
true as the code moves, and to say it to whoever reads the file next. One is about zenoh, and two are rules of your
own.

**An open session is not necessarily a connected one.** Section 3 showed this with the subscriber stopped. `open()`
returning means the configuration was valid and zenoh is running. Whether anyone is at the other end is a separate
question, and the list of peers answers it. This misunderstanding is the one most likely to cost you an afternoon
later, so it gets a test of the same kind as the outer one, with real zenoh at the service. Its name says what it
pins: *a collector opens even when no sensor node is listening*, and finds no one.

**Two rules of the pattern.** The other two tests are of a new kind, and the file should show the difference, because
it matters when one of them fails. A *contract test* pins a promise your own code makes, a rule of this guide's
pattern that would hold whatever zenoh did. Section 8 relied on one without a test: `open()` moved inside the `try`
because disposing a service that never opened is safe. Section 6 made another: asking a service for its identity
before opening it is a programming error, and the error says so.

A promise made in prose is either backed by a test or withdrawn, so both get one. The dispose test also pins the half
section 8 did not need yet, disposing more than once, because from now on the container disposes the service, and
nothing may break if something else already did.

**1. Add the three tests.** They pass at once, because they pin what the code already does. Add them to
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('a collector opens even when no sensor node is listening', () async {
    // A collector alone: nothing listens at its address.
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(collectorNode.dispose);

    await collectorNode.open();

    // The claim: the session opens, and finds no one.
    expect(collectorNode.peerIds, isEmpty);
  });

  test('disposing is safe before open, and more than once after', () async {
    // A rule of the pattern: disposing never throws, whatever the state.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    expect(sensorNode.dispose, returnsNormally);

    await sensorNode.open();
    sensorNode.dispose();

    // The claim: a second dispose after open is safe too.
    expect(sensorNode.dispose, returnsNormally);
  });

  test('asking an unopened service for its identity is an error', () {
    // A rule of the pattern: an unopened service has no identity to give.
    final sensorNode = ZenohService(SessionSettings.sensorNode());

    // The claim: asking throws a StateError.
    expect(() => sensorNode.zid, throwsStateError);
  });
}
```

The dispose test closes its own session, because that is its subject, so it is the one test that opens a session
without `addTearDown`. `returnsNormally` and `throwsStateError` are matchers like `contains`. The first says a call
must not throw, and the second that it must, with that type. `peerIds` before `open()` would throw the same
`StateError` through the same guard, so one test of the guard is enough. Run the whole file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

8 tests pass. The first of the three new ones takes about half a second, the time zenoh 1.8.0 spends trying an
address where nothing answers before it gives up. `sensorctl` spent the same half second in section 3.

**What the tests guarantee:** a collector whose sensor node is absent still opens, with no peers, so `open()`
succeeding never means "connected". Disposing a service is safe before it opens and more than once after, so the
container may dispose what a test or a `finally` already did. And a service asked for its identity before `open()`
throws a `StateError` that names what was not done.

The file is now the chapter's test file, complete: one outer test that states the claim, four inner ones that drove
the code into existence, and three pins. Read top to bottom, it tells the story of the chapter, so the tests stay in
this order and are not grouped by class or by method.

The tests are complete. Section 10 looks at what changed in the architecture.

## 10 — What changed in the architecture

Chapter 0 drew the layers and had nothing in them. This chapter filled in the one on the right:

```
view  →  view model  →  repository  →  codec  →  ZenohService  →  zenoh_dart
                                                 ─────────────
                                                  this chapter
```

`ZenohService` is the *service* of the pattern Flutter's
[architecture guide](https://docs.flutter.dev/app-architecture/guide) describes: one class per data source, which
wraps its API and hands plain values up. `SessionSettings` sits beside it, the topology as a value, tested as data.
Five rules began here, and they hold for the rest of the guide.

**Only the service imports the package.** Everything it hands upward is plain Dart: a `String` for an identity, a
`List<String>` for the peers. Nothing above it needs a type from `zenoh_dart`, and nothing above it needs zenoh to be
tested. The examples you copied in chapter 0 import the package too, but they are the package's programs.

**The service owns the session.** Flutter's guide says a service holds no state. This one holds a `Session`, because
the session is the connection, the one stateful thing in the transport, and it needs one owner with one lifecycle.
That owner is disposed through the provider, so the session lives as long as the container does.

**Real zenoh is used in the service's tests and nowhere else.** Two sessions in one process, on the loopback, is
where the guide checks each zenoh claim it makes, once, at the layer that touches it. Everything to the left of the
service is tested against a hand-written stand-in for the layer to its right, so those tests are fast and run without
a network.

**Every package uses `very_good_analysis`, with every rule on in your code.** So every public name has a doc comment,
imports inside `lib/` use `package:`, and output goes through `stdout`. The analyzer runs clean at the end of every
section, and each rule is explained where you first meet it.

**Providers wire the program.** Flutter's guide builds its objects with constructor injection and a
`ChangeNotifier`. This guide uses Riverpod for both of its applications, because `ChangeNotifier` ships with Flutter
and a pure-Dart program cannot use it, and one mechanism serves both programs. From now on,
`lib/config/providers.dart` is the one place a program's objects are declared, the container builds them, and an
override is how a test replaces one.

**Why the core package exists before the app that shares it.** In chapter 2 the phone app depends on `sensor_core`
as `sensorctl` does now, with the same `ZenohService`, opened from `SessionSettings.sensorNode()`, and the same tests,
run on the laptop. The package boundary also keeps the first rule enforceable. `zenoh_dart` is `sensor_core`'s
dependency, and a program that wants zenoh gets the service.

**What comes next.** Chapter 2 builds the sensor node itself: a Flutter app on the Android emulator and then on a
phone, in this same workspace, depending on this same `sensor_core`. It opens a session from
`SessionSettings.sensorNode()`, puts the device's accelerometer behind a service of its own, and publishes readings on
`sensor/phone/accel`, which the package's `z_sub` receives on your laptop.

Chapter 3 builds `watch` and the collector's layers to the left of the service: a repository that owns the key
expressions, a view model, and the terminal as the view. Each is tested against a stand-in for the one to its right,
and `watch` replaces `z_sub`. Chapter 4 adds `simulate`, a second sensor node inside `sensorctl` itself, for when you
would rather not start a device.

## 11 — Files and versions at the end of this chapter

**1. Commit.** Commit everything this chapter made, in one commit:

```sh
# in zenoh_sensors
git add .
git commit -m "A session of your own: sensor_core, ZenohService and its tests"
```

The commit holds 19 files:

- 12 are new: the core package, the providers, `z_info.dart` and the workspace's own `pubspec.yaml`.
- 4 changed, the program's lint rules among them.
- 2 were deleted, in section 3.
- 1 is the lock file, which git reports as moved from `apps/sensorctl/` to the top, because a workspace keeps one.

`.dart_tool/` stays out at every level, and so does `.fvm/`, as before.

> **In VS Code.** **View › Source Control** lists the same 19 changes. Choose the **+** on the **Changes** line to
> stage them all, type the message in the box above them, and choose **Commit**.

`zenoh_sensors` now holds this, in three commits: the two from chapter 0, and `A session of your own: sensor_core,
ZenohService and its tests`.

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
│       │   ├── z_put.dart
│       │   └── z_sub.dart
│       ├── lib/
│       │   └── config/
│       │       └── providers.dart
│       ├── pubspec.yaml
│       ├── README.md
│       └── test/
├── packages/
│   └── sensor_core/
│       ├── .dart_tool/
│       ├── .gitignore
│       ├── analysis_options.yaml
│       ├── CHANGELOG.md
│       ├── lib/
│       │   ├── sensor_core.dart
│       │   └── src/
│       │       └── services/
│       │           ├── session_settings.dart
│       │           └── zenoh_service.dart
│       ├── pubspec.yaml
│       ├── README.md
│       └── test/
│           └── services/
│               └── zenoh_service_test.dart
├── pubspec.lock
└── pubspec.yaml
```

This chapter was checked with these versions. Newer ones should work. If something does not, go back to these.

| what | version |
|---|---|
| Linux | Ubuntu 26.04.1 on x86_64, glibc 2.43 |
| git | 2.53.0 |
| fvm | 4.3.1 |
| Flutter, and the Dart it carries | 3.47.2, Dart 3.13.2 |
| VS Code | 1.138.0 |
| Dart and Flutter extensions | 3.142.0 |
| `zenoh_dart`, built on zenoh 1.8.0 | 1.0.0-rc.1 |
| `riverpod` | 3.4.3 |
| `args` | 2.7.0 |
| `very_good_analysis` | 11.0.0 |
| `test` | 1.32.0 |

---

*The code listings in this chapter are licensed under the Apache License 2.0. The text is © 2026 Hugo Alberto Garcia,
all rights reserved — see [COPYRIGHT](../COPYRIGHT).*

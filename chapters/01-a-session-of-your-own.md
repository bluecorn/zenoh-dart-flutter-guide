# 1 — A session of your own

## 1 — What you build, and what you will see

By the end of this chapter, `sensorctl` is a program you wrote. It opens a zenoh session with its own configuration,
says which session it is and which peers it is connected to, and closes.

On the way, you make the top folder a **workspace** with a second package, `sensor_core`. You move the code that calls
zenoh into it, behind one class, `ZenohService`. After that, no other file in a `lib/` folder imports `zenoh_dart`. A
test drives the move. It is the first test you write, and the first you watch fail.

With the package's `z_sub` running in another terminal, `sensorctl` prints:

```
sensorctl is 8d2ee6a92269c47d3b9fd2897f946465
connected to 1 peer:
  8278f066730daedae9902fc949189943
```

Both ids differ on your machine. The first is `sensorctl`'s own. The second is `z_sub`'s, and **nothing discovered
it.** You give `sensorctl` the address to connect to, and `z_sub` the address to listen on. Until chapter 14, every
program you write has multicast scouting off, and you tell it where to listen or where to connect.

You run `sensorctl` as `fvm dart run sensorctl:sensorctl`. In the code, `sensorctl` is `main`, at the top of this
stack, and each layer has its section:

```
sensorctl                    what you type
  main                       section 8: prints the id and the peers, and disposes the container
    ProviderContainer        section 8: builds the service from its settings
      ZenohService           sections 5 to 7: opens the session, and gives its id and its peers
      SessionSettings        sections 5 and 7: where each side listens or connects
```

Section 3 first writes the program flat, in one file, and section 4 makes the workspace that holds the core.

> **Zenoh guidance.** `sensorctl` is `z_info` with its configuration in code, in a class that tests check,
> and its zenoh calls behind one class. Until chapter 5, each of your programs is a peer with multicast scouting and
> gossip off. The sensor node listens on the loopback, and the collectors connect to it. The zenoh calls go behind one
> class so that the Flutter app in chapter 2 can share it.

> **If you have not used a pub workspace, or Riverpod outside Flutter.** A workspace is one `pubspec.yaml` at the top
> that resolves the dependencies of the packages it lists. A `ProviderContainer` stores the state of your providers.
> In a Flutter app a `ProviderScope` widget creates one for you, and in a pure-Dart program you create it yourself.
> You make the workspace in section 4 and the container in section 8.

## 2 — What to read

| | page | what to take from it |
|---|---|---|
| [1] zenoh.io | no page on sessions | |
| [1] zenoh.io | [*Deployment*](https://zenoh.io/docs/getting-started/deployment/), *Peer to peer* | how peers scout for each other, by multicast and by gossip. The page calls the link between two peers a session, and this guide calls it a connection |
| [1] zenoh.io | [*Configuration*](https://zenoh.io/docs/manual/configuration/) | a configuration as a JSON5 file, and `--cfg` for one entry |
| [2] *The Zenoh Book* | [*Core Concepts → Sessions*](https://corsaro.me/zenoh/book/core-concepts/sessions/) | what a session is, its modes, closing it, and opening more than one in a process. The page calls a session a connection, and this guide keeps that word for the link between two nodes |
| [2] *The Zenoh Book* | [*Routing → Multicast Scouting*](https://corsaro.me/zenoh/book/routing/scouting/) | the exchange that multicast scouting is, in one picture |
| [3] *Zenoh Programming in Rust* | [chapter 4, *Sessions and Configuration*](https://kydos.github.io/zenoh-book/chapter_04.html) | opening a session, the default configuration, configuration from a file or in code, session info, and closing. Skip *Runtime Configuration via Admin Space*, which changes a running router's settings |
| `very_good_analysis` | [its documentation](https://pub.dev/packages/very_good_analysis) | the lint set you switch on in section 3 |
| `test` | [its documentation](https://pub.dev/packages/test) | how tests, matchers and `addTearDown` work, from section 5 |
| Riverpod | [its documentation](https://riverpod.dev) | providers and the container, from section 8 |

**The Rust API in Dart.** *Zenoh Programming in Rust* [3] shows the API in Rust. Five things look different in Dart:

| in the book | in `zenoh_dart` |
|---|---|
| `zenoh::open(config).await.unwrap()` | `await Session.open(config: config)`, which throws on failure |
| `config.insert_json5("mode", r#""peer""#)` | `config.insertJson5('mode', '"peer"')`, with the same paths, the same `/` between nested keys, and the same JSON5 values |
| `session.info().zid().await` | `session.zid`, and `.toHexString()` to print it |
| `routers_zid()`, `peers_zid()` return async streams you `.collect()` | `routersZid()` and `peersZid()` return plain lists |
| `session.close().await.unwrap()` | `session.close()`, which returns nothing, needs no `await`, and is safe to call twice |

## 3 — The program that opens a session

Write your first program of your own. Before you do, run a third example, `z_info`, which does what `sensorctl` will
do. It opens a session, says which session it is and which peers it is connected to, and closes.

**1. Go to `sensorctl`'s folder.**

```sh
# in zenoh_sensors
cd apps/sensorctl
```

**2. Copy one more example.** Find `zenoh_dart`'s folder in `.dart_tool/package_config.json`, and copy the example
from it:

```sh
# in zenoh_sensors/apps/sensorctl
pkg=$(sed -n 's|.*"rootUri": "file://\(.*/zenoh_dart-[^/"]*\)".*|\1|p' .dart_tool/package_config.json)
cp "$pkg/example/z_info.dart" example/
```

**3. Start the subscriber.** In a terminal in `apps/sensorctl`, start `z_sub` again and leave it running. It takes the
sensor node's place until you build the node in chapter 2.

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

The expected output in this guide uses two marks. `⋮` stands for lines the guide does not show. Here they are what
pub prints before the program's own output: `Running build hooks...`, and sometimes `Building package executable...`
and `Built …`, on one line or several, depending on what pub had to do. `…` stands for the part of a line that
differs on your machine, such as an id or a timestamp. Everything else prints as shown.

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

**Each session has its own id.** The id is 16 bytes. Zenoh treats it as one 128-bit number, so it prints it in
hexadecimal without leading zeros, in up to 32 characters. Zenoh makes a new one every time a session opens, so yours
differs from the one above. Zenoh uses the id in timestamps and in the names in its admin space. In this guide you use
it to tell one running program from another.

**`routers ids:` is empty.** No router is running. A router is a zenoh node in router mode, usually the program
`zenohd`, and this guide uses one first in chapter 5.

**`peers ids:` is `z_sub`.** It is the only other zenoh program you started, and it is in the list because you told
`z_sub` where to listen and `z_info` where to connect. `-e` connects to the subscriber. `--no-multicast-scouting`
keeps this program from announcing itself on your network. `--cfg 'listen/endpoints:[]'` keeps it from listening.
Without that option, a peer also listens on every interface, on a port picked at random.

**Ignore the red `ERROR` line.** Zenoh 1.8.0 prints it when a connected session closes. Nothing failed.

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

`Zenoh.initLog('error')` turns on zenoh's own logging at the `error` level. A session that fails to open throws an
exception with one code for nearly every cause, and the log says what went wrong. The call comes first in `main`, so
that the log is on before the session opens.

The five settings put `sensorctl` into this guide's topology:

- **`mode: peer`**, because there is no router to be a client of.
- **Multicast scouting and gossip off**, because both discover sessions you did not configure, and your sessions
  should connect only to the addresses you give them.
- **`listen/endpoints: []`**, because `sensorctl` is a collector. It opens connections and never accepts one, so it
  needs no address of its own.
- **`connect/endpoints`**, the sensor node's address on the loopback, where `z_sub` is listening.

The rest is what `z_info` did. `Session.open` returns a `Future`. Zenoh's open blocks for a time its configuration
chooses, so the package runs it on a thread of its own, and your program keeps running meanwhile.

When the future completes, the session is open. Each connection in the configuration is made by then, or the wait that
`scouting/delay` sets, half a second by default, has run out. Zenoh keeps retrying a connection that is not made, in
the background, so a program that keeps its session open still connects to a sensor node that starts later.

`session.zid` is the id, and `peersZid()` is the list you saw. `close()` sits in a `finally`, so that it runs even if
printing throws. In chapter 12 you build on that `finally` to handle signals.

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

Your program connected to the subscriber and printed the subscriber's id, the one `z_info` listed under `peers ids:`.
`sensorctl` needs no command-line options, because its settings are in its code.

> **In VS Code.** Give the program an entry in **Run and Debug**. Keep the two entries the file has, and add a
> third. Replace `zenoh_sensors/.vscode/launch.json`:
>
> ```json
> {
>   "version": "0.2.0",
>   "configurations": [
>     {
>       "name": "z_sub",
>       "type": "dart",
>       "request": "launch",
>       "program": "${workspaceFolder}/apps/sensorctl/example/z_sub.dart",
>       "cwd": "${workspaceFolder}/apps/sensorctl",
>       "args": ["-l", "tcp/127.0.0.1:7447", "--no-multicast-scouting"]
>     },
>     {
>       "name": "z_put",
>       "type": "dart",
>       "request": "launch",
>       "program": "${workspaceFolder}/apps/sensorctl/example/z_put.dart",
>       "cwd": "${workspaceFolder}/apps/sensorctl",
>       "args": [
>         "-e", "tcp/127.0.0.1:7447", "--no-multicast-scouting", "--cfg", "listen/endpoints:[]",
>         "-k", "demo/example/test", "-p", "Hello from the guide"
>       ]
>     },
>     {
>       "name": "sensorctl",
>       "type": "dart",
>       "request": "launch",
>       "program": "${workspaceFolder}/apps/sensorctl/bin/sensorctl.dart",
>       "cwd": "${workspaceFolder}/apps/sensorctl"
>     }
>   ]
> }
> ```
>
> With `z_sub` still running, choose **sensorctl** at the top of the Run and Debug view and press ▶. The Debug
> Console prints what the terminal printed. Each entry's `cwd` is `apps/sensorctl`, where `zenoh_dart`'s build hook
> stages zenoh's native libraries. In section 4 the build hook stages them in the top folder, and you change all
> three entries to match.

**7. Stop the subscriber, and ask once more.** Press Ctrl-C in the first terminal, then run `sensorctl` again:

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart run bin/sensorctl.dart
```

```
⋮
…sensorctl is …
connected to 0 peers:
```

**The session still opened.** Nothing was listening at `tcp/127.0.0.1:7447`. `open()` returned after its half
second, and `sensorctl` printed no peers. It printed no `ERROR` line either, because the session was never connected.

**In peer mode, an open session is not necessarily a connected one.** `open()` succeeding means your configuration was
valid and the zenoh runtime started. It says nothing about the other end. In section 9 you pin this with a test, so
that it stays true as the code changes.

**8. Delete the template's leftovers.** When `dart create` made this project, it wrote a small library,
`lib/sensorctl.dart`, with a `calculate()` function and a test for it. The template's program imported the library and
printed `Hello world: 42!`, which proved the toolchain worked. Your program imports none of it, so both files are
dead. Delete them:

```sh
# in zenoh_sensors/apps/sensorctl
rm lib/sensorctl.dart test/sensorctl_test.dart
```

You fill `lib/` and `test/` again with code of your own. At the end of this chapter, `lib/` holds the providers that
wire the program together. In chapter 3 the program gains its view model and its terminal view, with their tests.
**Delete code as soon as nothing uses it.**

**9. Switch on a stricter lint set.** `dart create` gave the program the Dart team's recommended lint set, `lints`,
which the template's code was written to. Hold the code you write to a stricter set,
[`very_good_analysis`](https://pub.dev/packages/very_good_analysis). It has about 200 rules, and you switch none of them
off. Add it as a development dependency:

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart pub add --dev very_good_analysis
```

The template's options file is mostly a long comment. Replace `zenoh_sensors/apps/sensorctl/analysis_options.yaml`:

```yaml
include: package:very_good_analysis/analysis_options.yaml

analyzer:
  exclude:
    - example/**
```

The first line switches the rules on. The exclusion leaves out `example/`, because its programs are the package's,
written to the package's rules. Now run the analyzer on the program you wrote:

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

The analyzer reports 3 findings, one per `print`, from the rule `avoid_print`. Write the same program through
`stdout`, from `dart:io`. Replace `zenoh_sensors/apps/sensorctl/bin/sensorctl.dart`:

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

`writeln` writes one line to `stdout`. The output does not change. Run the analyzer again:

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart analyze
```

It prints `No issues found!`, and so does every run of it at the end of a section from here on. As you go, the rules
require a doc comment on every public class and member, `package:` imports inside `lib/`, and Dart 3.13's shorter way
to write a constructor, which you meet in section 5.

> **In VS Code.** The Problems panel, **View › Problems**, lists the same three findings as soon as you save the new
> `analysis_options.yaml`, with a squiggle under each `print`. They go when you save the new program.

## 4 — A workspace, and a package to share

Make the core package, and the workspace that lets two programs share it. The zenoh code you are about to write is not
only `sensorctl`'s. In chapter 2 a Flutter app on a phone opens a session of its own, with the same class, the same
settings and the same tests. So the code belongs in a package both programs depend on.

A **pub workspace** is one folder at the top that resolves the dependencies of the packages it lists. It also changes
where you run programs and tests from.

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
delete it:

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
- `lints` is no longer read, because `analysis_options.yaml` includes `very_good_analysis` instead.

**Remove a dependency nothing uses.** It still resolves and pins a version, and whoever reads the file next has to
ask what it is for.

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

It prints four lines about deleting an old lock file and an old package config, one of each for each member, and a
link to a page that explains them. Nothing is wrong. Until now each package resolved its own dependencies and kept its
own `pubspec.lock`, and a workspace has one of each for every member. From here there is a single `pubspec.lock` and a
single package config, beside the top `pubspec.yaml`.

**6. Delete the old copy of zenoh's libraries.**

```sh
# in zenoh_sensors
rm -rf apps/sensorctl/.dart_tool
```

`zenoh_dart`'s build hook stages zenoh's native libraries into the `.dart_tool/` of whatever is being resolved, and it
has just staged them at the top. The older copy in `apps/sensorctl/.dart_tool/` was still there, because `pub get`
removed only the package config beside it. A program started inside `apps/sensorctl` would load that old
copy, which no longer gets updated. The command deletes it, so from here there is only one copy.

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

`sensorctl:sensorctl` is the *package name*, then the *program name*, which here is the file `bin/sensorctl.dart`
inside the package `sensorctl`. Zero peers is right, because nothing is listening now.

**From here, run programs and tests from `zenoh_sensors`:** `fvm dart run sensorctl:sensorctl` for the program, and
`fvm dart test packages/sensor_core` for the core's tests. The library the build hooks staged is at the top now, and a
program started inside `apps/sensorctl` cannot find it. The tests find it because the top folder's `pubspec.yaml`
depends on `sensor_core`, and `dart test` stages native libraries only for the folder's own package and its
dependencies.

> **In VS Code.** All three entries in `zenoh_sensors/.vscode/launch.json` start programs in `apps/sensorctl`, which no
> longer holds the package's libraries. Replace `zenoh_sensors/.vscode/launch.json`:
>
> ```json
> {
>   "version": "0.2.0",
>   "configurations": [
>     {
>       "name": "z_sub",
>       "type": "dart",
>       "request": "launch",
>       "program": "${workspaceFolder}/apps/sensorctl/example/z_sub.dart",
>       "cwd": "${workspaceFolder}",
>       "args": ["-l", "tcp/127.0.0.1:7447", "--no-multicast-scouting"]
>     },
>     {
>       "name": "z_put",
>       "type": "dart",
>       "request": "launch",
>       "program": "${workspaceFolder}/apps/sensorctl/example/z_put.dart",
>       "cwd": "${workspaceFolder}",
>       "args": [
>         "-e", "tcp/127.0.0.1:7447", "--no-multicast-scouting", "--cfg", "listen/endpoints:[]",
>         "-k", "demo/example/test", "-p", "Hello from the guide"
>       ]
>     },
>     {
>       "name": "sensorctl",
>       "type": "dart",
>       "request": "launch",
>       "program": "${workspaceFolder}/apps/sensorctl/bin/sensorctl.dart",
>       "cwd": "${workspaceFolder}"
>     }
>   ]
> }
> ```
>
> In each entry, `cwd` is now the folder VS Code has open, the top of the workspace. Press ▶ on **sensorctl** again.
> It prints the same as before. With the old file it would fail to start, with an error about not finding
> `libzenoh_dart.so`, because the old `cwd` points at a folder that no longer holds the library.

The top folder is a workspace with two members, and everything runs from it. Section 5 writes the chapter's first
test.

## 5 — The test of the chapter's claim

Write the test that states the chapter's claim, and watch it fail. From here on, write each test before the code it
tests. This is test-driven development, or TDD.

The test makes you state what you want precisely enough to run. **If you cannot write the assertion, you do not yet
know what you are building.** The test is also the first caller of the new code, so the way the test calls it sets the
code's shape.

**1. State the claim in one sentence.**

> A sensor node and a collector, configured the way this guide configures them, find each other on the loopback.

The test's name says the same in fewer words.

**2. Make room.**

```sh
# in zenoh_sensors
mkdir -p packages/sensor_core/lib/src/services packages/sensor_core/test/services
```

`sensor_core` now looks like this:

```
zenoh_sensors/packages/sensor_core/
├── lib/
│   └── src/
│       └── services/
└── test/
    └── services/
```

`test/` mirrors `lib/src/`. The tests of `lib/src/services/zenoh_service.dart` go in
`test/services/zenoh_service_test.dart`.

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

    // The claim: each one's peers hold the other's id, and the two ids differ.
    expect(collectorNode.peerIds, contains(sensorNode.zid));
    expect(sensorNode.peerIds, contains(collectorNode.zid));
    expect(collectorNode.zid, isNot(sensorNode.zid));
  });
}
```

The test opens two sessions in one Dart process:

```
  one Dart process: the test
 ┌──────────────────────────────────────────────────────────────────────────┐
 │  sensorNode                               collectorNode                  │
 │  a session, a peer                        a session, a peer              │
 │  listening on tcp/127.0.0.1:7447 ◄─────── connecting to it               │
 │  peerIds: [collectorNode's id]            peerIds: [sensorNode's id]     │
 └──────────────────────────────────────────────────────────────────────────┘
```

Zenoh is the subject, so a claim about zenoh is checked against zenoh itself. Two kinds of test in this guide open
real sessions: the test of each chapter's claim, and the tests of `ZenohService`. No other test does.

**The sensor node opens first.** A collector needs something to connect to, and its settings say where. Swap the two
lines, and the collector opens before anything is listening.

**`addTearDown`** hands the closing to the test runner, so the sessions close even when an expectation fails. Without
it, a failing test leaves its sessions open until the run ends. A later test in the same run then cannot listen on
port 7447, because the leftover session holds it. That test fails for a reason that is not in its own code.

**The three expectations state the claim, in order.** The first two are the claim itself, that each one's list of peers
holds the other's id. The third is the guard. If both sessions reported the same id, `contains` would pass while the two
sessions had not connected.

The test asks for three things that do not exist yet:

- `SessionSettings`, with a factory for each role, `sensorNode()` and `collectorNode()`.
- `ZenohService`, built from the settings, with `open()`, `dispose()`, `zid` and `peerIds`.
- The package's exports of both, so that the test can import them.

**4. Write just enough for it to compile.** The settings come first. Create
`zenoh_sensors/packages/sensor_core/lib/src/services/session_settings.dart`:

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

- **Every public class and member has a doc comment**, the `///` lines. Each says what the thing is, and the editor
  shows it wherever the name is used. A comment is a claim, so each one in this guide has been checked against the
  code it describes.
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

  /// The session's id.
  String get zid => '';

  /// The ids of the peers this session is connected to.
  List<String> get peerIds => const [];

  /// Closes the session.
  void dispose() {}
}
```

The third rule shows here. Inside `lib/`, a file imports another by its `package:` path, never by a relative one. The
doc comments say what each member will do, and the bodies do nothing yet. The package's library file now exports the
two files. Replace `zenoh_sensors/packages/sensor_core/lib/sensor_core.dart`:

```dart
/// The zenoh data layer that `sensorctl` and the phone app share.
library;

export 'src/services/session_settings.dart';
export 'src/services/zenoh_service.dart';
```

**The classes compile and do nothing.** They answer every question with an empty value. Watching them fail checks
that the test is worth keeping. If something this empty could satisfy the test, the test would check only the shape
of a class, and it would keep passing long after the code stopped working.

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

The collector's list of peers should hold the sensor node's id, and it is empty. The test fails on its claim, because
the skeletons answer with empty values.

**This test stays red until the end of section 7.** Sections 6 and 7 make its expectations true, in four cycles, each
red before its code:

```
a sensor node and a collector find each other on loopback
 ├─ the collector's peers hold the sensor node's id    section 7, cycle 4: the endpoints go in
 ├─ the sensor node's peers hold the collector's id    section 7, cycle 4
 └─ the two ids differ                                 section 6, cycles 1 and 2: each session's own id
```

Section 7's cycle 3, neither side announces itself on the network, is asked for by the guide's topology, which this
test does not check. It comes before cycle 4, so that cycle 4's green means the address did the work.

This test is the outer of two loops. It states the chapter's claim and turns green once, at the end of section 7. The
inner loop does the work in small cycles. Each cycle has its own test, which goes red and then green.

> **In VS Code.** Run tests from the top folder in VS Code too. VS Code needs an entry for that. Keep the three
> entries you have and add a fourth, with no program in it. Replace `zenoh_sensors/.vscode/launch.json`:
>
> ```json
> {
>   "version": "0.2.0",
>   "configurations": [
>     {
>       "name": "z_sub",
>       "type": "dart",
>       "request": "launch",
>       "program": "${workspaceFolder}/apps/sensorctl/example/z_sub.dart",
>       "cwd": "${workspaceFolder}",
>       "args": ["-l", "tcp/127.0.0.1:7447", "--no-multicast-scouting"]
>     },
>     {
>       "name": "z_put",
>       "type": "dart",
>       "request": "launch",
>       "program": "${workspaceFolder}/apps/sensorctl/example/z_put.dart",
>       "cwd": "${workspaceFolder}",
>       "args": [
>         "-e", "tcp/127.0.0.1:7447", "--no-multicast-scouting", "--cfg", "listen/endpoints:[]",
>         "-k", "demo/example/test", "-p", "Hello from the guide"
>       ]
>     },
>     {
>       "name": "sensorctl",
>       "type": "dart",
>       "request": "launch",
>       "program": "${workspaceFolder}/apps/sensorctl/bin/sensorctl.dart",
>       "cwd": "${workspaceFolder}"
>     },
>     {
>       "name": "tests",
>       "type": "dart",
>       "request": "launch",
>       "templateFor": "",
>       "cwd": "${workspaceFolder}"
>     }
>   ]
> }
> ```
>
> An entry with `templateFor` and no `program` is a template. The editor takes its `cwd` for every test it runs from
> the Testing view, and for the **Run** and **Debug** links it shows above a test. Without it, a test would start
> inside `packages/sensor_core`, where there is no copy of zenoh's library, and a test that opens a session would
> fail.
>
> Open the Testing view, the flask in the Activity Bar, and press ▶ beside the test, or click **Run** above it
> in the editor. It fails the same way.

The outer test is red. Section 6 starts the inner cycles with each session's id.

## 6 — Two cycles: an id

Give each session its id, in two cycles. The work happens in **cycles**. A cycle is small: write one test, run it and
watch it fail, do the least that makes it pass, and run it again. Four cycles build the service, two here and two
in section 7. Inside a cycle, run only that cycle's test, by a piece of its name, with `-n`.

**Cycle 1 — a service has an id once it is open.** From here the guide shows a test file by what changes in it. `⋮`
stands for everything already there, and what follows goes at the end of the file, before `main`'s closing brace,
which the block shows so that you can see where. When the imports change, a block shows them above the `⋮`, as they
now read. Add a second test to `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('a service has an id once it is open', () async {
    // The sensor node's end, closed when the test ends.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);

    // The code to implement: an id, once the service is open.
    await sensorNode.open();

    // The claim: the id is not empty.
    expect(sensorNode.zid, isNotEmpty);
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'has an id'
```

```
⋮
  Expected: non-empty
    Actual: ''
⋮
```

The id is empty, because the skeleton's `zid` answers `''`.

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

  /// The session's id.
  String get zid => 'the-sensor-node';

  /// The ids of the peers this session is connected to.
  List<String> get peerIds => const [];

  /// Closes the session.
  void dispose() {}
}
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'has an id'
```

It passes. `dart test` ends a passing run with `All tests passed!`, so from here the guide shows nothing for a pass,
and only the lines that matter for a failure.

This technique is called **fake it**. The test asks for an id that is not empty, and a constant is one. It is not the
real answer, and the next test replaces it. Faking is wrong only as the *last* step. The constant is the same for
every service, so the next cycle opens two.

**Cycle 2 — two services have different ids.** Add a third test to
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('two services have different ids', () async {
    // Two ends, closed when the test ends.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    // The code to implement: each open service gets its own id.
    await sensorNode.open();
    await collectorNode.open();

    // The claim: the two ids differ.
    expect(collectorNode.zid, isNot(sensorNode.zid));
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'different ids'
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

Two real sessions need the package, so the core now depends on it. Replace
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

  /// The session's id: up to thirty-two hexadecimal characters.
  String get zid => _opened.zid.toHexString();

  /// The ids of the peers this session is connected to.
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
fvm dart test packages/sensor_core -n ' id'
```

Both pass. `-n ' id'`, with its leading space, matches both test names, the quickest check that the new code did not
break the old test. The imports are in alphabetical order of their packages, which is another of the lint rules, and
the analyzer says so if they are not.

**The session is what the test asked for.** `Session.open` is awaited, the handle is kept, and `zid` reports the id in
up to 32 hexadecimal characters.

**`_opened` names the mistake.** Asking a service for its id before opening it is a programming error, so `_opened`
throws a `StateError` that says what was not done. Without it, the error would be a bare null error.

**The configuration is zenoh's default.** `Session.open(config: Config())` listens on every interface and joins
multicast scouting, as the package's README warns. It is the least that opens a session, and no test asks for more
yet. Section 7's tests do.

**What the tests guarantee:** a service has a real zenoh id once it is open, and two services have different ones.

Each session has its own id. Section 7 gives the sessions the guide's topology, and the outer test goes green.

## 7 — Two cycles: a topology, and a connection

Give both sides the guide's topology in two cycles, and the outer test goes green. The outer test is still red, and
the service still opens zenoh's default configuration. Cycle 3 takes away the two ways a session discovers sessions it
was never told about. Cycle 4 gives each side its address.

Cycle 3 comes first. With the addresses in and scouting still on, the outer test could go green because scouting
discovered the sessions, and nothing in its output would say so. With scouting off first, the addresses are the only
way left for the two sessions to connect.

**Cycle 3 — neither side announces itself on the network.** Three of the five settings you wrote into
`bin/sensorctl.dart` are the same on both sides. Each session `SessionSettings` describes is a `peer` that neither
scouts nor gossips. Scouting is how zenoh discovers sessions nobody configured. A new session calls out on a multicast
address, and every session that hears it answers with where it can be reached:

```
 a new session  ──── who is there? ────────────►  UDP multicast, 224.0.0.224:7446
 a new session  ◄─── here I am, at tcp/… ───────  every session that hears it
```

Gossip is the second-hand version, where a session passes on to the sessions it knows the addresses of the others it
has met. Both stay off in your programs until chapter 14, so that your sessions connect only to the addresses you give
them.

This cycle's test opens no session. Two of these settings make something *not* happen, and `mode: peer` is already
zenoh's default. Although a test that opens two sessions can watch them connect, it cannot watch them not discover
anyone else, because there is no one else in the test. To see this once the chapter is finished, delete the two
`scouting` entries from the settings. The outer test still passes.

So a setting whose whole effect is an absence is read back as data and compared with what it should say. The test
reads the settings through a new property, `asJson5`. It is a map from a key path to a JSON5 value, the entries
`insertJson5` takes. Add a fourth test to `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('neither side announces itself on the network', () {
    // The code to implement: both sides' settings, read back as data. An
    // absence cannot be watched, so it is checked as the value behind it.
    final sensorSettings = SessionSettings.sensorNode().asJson5;
    final collectorSettings = SessionSettings.collectorNode().asJson5;

    // The claim: both sides are peers that neither scout nor gossip.
    for (final settings in [sensorSettings, collectorSettings]) {
      expect(settings, containsPair('mode', '"peer"'));
      expect(settings, containsPair('scouting/multicast/enabled', 'false'));
      expect(settings, containsPair('scouting/gossip/enabled', 'false'));
    }
  });
}
```

Then give `SessionSettings` an `asJson5` that compiles and holds nothing, so that the test fails on its claim. Replace
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

**Write the obvious implementation.** This is the third way to green, after fake it and triangulate. When you know
what to write, write it.

There is no fake to make here. When the thing under test is data, the least that passes and
the real thing are the same three lines, so the test restates the code. Its job is to make those three lines
impossible to delete without a test going red, and no other test in this chapter can do that. Replace
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
  /// they ask for is made, or the wait set by `scouting/delay` has run out,
  /// and zenoh keeps retrying the connections not made yet.
  Future<void> open() async {
    _session = await Session.open(config: _config());
  }

  /// The session's id: up to thirty-two hexadecimal characters.
  String get zid => _opened.zid.toHexString();

  /// The ids of the peers this session is connected to.
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

It passes. Run the two id tests too, with `-n ' id'`. They still pass. Both sessions open as peers that do not scout,
and each gets an id.

No test asked for `_config()`. `Config()` is zenoh's default configuration, and `forEach(config.insertJson5)` passes
each key and value of the map to `insertJson5`, so the map holds every difference from the default. No test shows the
two scouting settings take effect yet, because their effect is an absence. Cycle 4 adds the endpoints to the same map,
and its outer test fails unless this line applies them.

**What the test guarantees:** both sides are peers that neither scout nor gossip.

**Cycle 4 — they find each other.** Before you run the outer test again, remove the last neutral answer. `peerIds`
still returns an empty constant, and a red produced by a stub says nothing about zenoh. Replace
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
  /// they ask for is made, or the wait set by `scouting/delay` has run out,
  /// and zenoh keeps retrying the connections not made yet.
  Future<void> open() async {
    _session = await Session.open(config: _config());
  }

  /// The session's id: up to thirty-two hexadecimal characters.
  String get zid => _opened.zid.toHexString();

  /// The ids of the peers this session is connected to.
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

`peersZid()` is the list `z_info` printed under `peers ids:`. As with `zid`, the service turns each id into a
hexadecimal string before handing it up, so nothing above the service needs a type from the package.

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

The id in `Expected:` now comes from zenoh, and so does the empty list. The two sessions are peers in one process,
both with scouting off, and neither has an address to connect to. With scouting off, two sessions connect only when
one is given the other's address.

The sensor node listens at an address, and the collector connects to it. That is data again, so the test is of the same
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

  /// Where the sensor node listens: the loopback, port 7447.
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

`nodeEndpoint` is the only address in this guide until chapter 5. The sensor node on your phone listens there from
chapter 2. `_json5List` writes a Dart list as the JSON5 text `insertJson5` wants, `["tcp/127.0.0.1:7447"]` or `[]`.

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'connects to where'
```

It passes. Now run the outer test:

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
second `open()`, with no wait and no retry. That holds because `open()` waits until the connections in its
configuration are made, within its half second, and the sensor node is already listening.

The service is finished. `bin/sensorctl.dart` still has its own five settings and opens its own session. In section 8
you move the program onto the service. That is the refactor, and the five tests stay green before and after.

## 8 — The refactor, and a provider container

Move `bin/sensorctl.dart` onto the service, and then onto a *provider container*. The program still opens zenoh by
itself, with its own copy of the five settings the service now carries and tests. From here on, a provider container
builds the objects of every program you write. This is the chapter's refactor. The behavior does not change, the tests
are green before and after, and the program's output stays the same.

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
nothing under `lib/` or `bin/` in this package imports it after this section. The three examples in
`apps/sensorctl/example/` are the package's own programs, and `z_sub` takes the sensor node's place until you build
the node in chapter 2.

> **In VS Code.** Open `apps` › `sensorctl` › `pubspec.yaml`, add the line, and save. The extension runs `pub get`.

**3. Start logging from the core.** The program starts zenoh's logging with `Zenoh.initLog('error')`. That call lives
in the package, which `bin/sensorctl.dart` is about to stop importing, so the core offers it as a function of its own.
The class is unchanged, and one function is added above it. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'package:sensor_core/src/services/session_settings.dart';
import 'package:zenoh_dart/zenoh.dart';

/// Starts zenoh's own log, printed to standard output: for a program with a
/// terminal. [level] applies unless `RUST_LOG` is set. Once per process,
/// before any session opens.
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
  /// they ask for is made, or the wait set by `scouting/delay` has run out,
  /// and zenoh keeps retrying the connections not made yet.
  Future<void> open() async {
    _session = await Session.open(config: _config());
  }

  /// The session's id: up to thirty-two hexadecimal characters.
  String get zid => _opened.zid.toHexString();

  /// The ids of the peers this session is connected to.
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

The five settings are gone from `main`, because `SessionSettings.collectorNode()` holds them now, and tests check
them. `Config` and `Session` went with them. The program imports `sensor_core` and `dart:io` and nothing else, and no
longer imports `zenoh_dart`. What is left is the program's job, to say which session it is and which peers it is
connected to.

`open()` is inside the `try` now. A service exists before its session does, and disposing one that never opened is
safe, a rule section 9 pins with a test.

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

It prints three lines, and zenoh 1.8.0's `ERROR` line as the session closes. Nothing needs fixing. The program behaves
as before, and only its structure changed. A refactor changes a program's structure and keeps its behavior.

**6. Build the program from a provider container.** The program made its service by hand in `main`,
`ZenohService(SessionSettings.collectorNode())`. For one object that is fine. From chapter 3 the program has a view
model, which needs a repository, which needs the service. A test must be able to swap any one of them for a fake.

Build that graph with [Riverpod](https://riverpod.dev). Each object is declared once, as a *provider*. A *provider
container* builds them on demand, disposes them together, and lets a test override any one of them. In a Flutter app,
a `ProviderScope` widget creates the container. In a pure-Dart program you create it yourself, as a
`ProviderContainer`.

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

The providers get a file of their own, in a `config` folder under `lib/`. Create
`zenoh_sensors/apps/sensorctl/lib/config/providers.dart`:

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

There are two providers. `sessionSettingsProvider` says that this program is a collector. `zenohServiceProvider`
builds the service from it, and `ref.watch` reads another provider's value.
`ref.onDispose(service.dispose)` ties the session's life to the container's, so when the container is disposed, so is
the service, and the session closes.

**In a program, dispose the service through the provider, never by hand.**

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
`main`, which has no test yet. You checked `main` by its output, twice. The program gets its own tests in chapter 3,
when it has a view to test.

The program runs on the service, built by a container. Section 9 pins three more facts with tests.

## 9 — What the tests pin

Pin three more facts with tests. The four cycles drove the service into existence. The three new tests pass at once,
because they pin what the code already does. Their job is to keep something true as the code moves, and to say it to
whoever reads the file next. One is about zenoh, and two are rules of your own.

**A collector opens even when no sensor node is listening.** This is the pin about zenoh. In peer mode, `open()`
returning means the configuration was valid and the zenoh runtime started. Whether anyone is at the other end is a
separate question, and the list of peers answers it. So the test is of the same kind as the outer one, with real
zenoh at the service, and its name says what it pins.

**Two rules of the pattern.** The other two tests are of a new kind, and the file should show the difference, because
it matters when one of them fails. A *contract test* pins a promise your own code makes, a rule of this guide's
pattern that would hold whatever zenoh did.

Two of the service's promises have no test yet. Disposing a service is
safe before it opens, which `main`'s `try` relies on. Asking a service for its id before it opens is an error, and
the error says what was not done.

A promise made in prose is either backed by a test or withdrawn, so both get one. The dispose test also pins
disposing more than once, because the container disposes the service, and nothing may break if a test or a `finally`
already did.

**1. Add the three tests.** They pass at once, because they pin what the code already does. Add them to
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('a collector opens even when no sensor node is listening', () async {
    // A collector alone: nothing listens at its address.
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(collectorNode.dispose);

    await collectorNode.open();

    // The claim: the session opens, with no peers.
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

  test('asking an unopened service for its id is an error', () {
    // A rule of the pattern: an unopened service has no id to give.
    final sensorNode = ZenohService(SessionSettings.sensorNode());

    // The claim: asking throws a StateError.
    expect(() => sensorNode.zid, throwsStateError);
  });
}
```

The dispose test closes its own session, because that is its subject. It is the only test that opens a session
without `addTearDown`. `returnsNormally` and `throwsStateError` are matchers like `contains`. The first says a call
must not throw, and the second that it must, with that type. `peerIds` before `open()` would throw the same
`StateError` through the same guard, so one test of the guard is enough. Run the whole file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

8 tests pass. The first of the three new ones takes about half a second. That is `open()` waiting out
`scouting/delay` for a connection that cannot be made, while zenoh keeps retrying it in the background.

**What the tests guarantee:** a collector whose sensor node is absent still opens, with no peers, so in peer mode
`open()` succeeding does not mean "connected". Disposing a service is safe before it opens and more than once after,
so the container may dispose what a test or a `finally` already did. And a service asked for its id before `open()`
throws a `StateError` that names what was not done.

The file is now the chapter's test file, complete: one outer test that states the claim, four inner ones that drove
the code into existence, and three pins. Read top to bottom, it follows the chapter, so keep the tests in this order.

The tests are complete. Section 10 looks at what changed in the architecture.

## 10 — What changed in the architecture

This chapter builds the `ZenohService` layer:

```
view  →  view model  →  repository  →  codec  →  ZenohService  →  zenoh_dart
                                                 ─────────────
                                                  this chapter
```

`ZenohService` is the *service* of the pattern in Flutter's
[architecture guide](https://docs.flutter.dev/app-architecture/guide) [5], the class that wraps one data source's
API. `SessionSettings` sits beside it, the topology as a value, tested as data. Five rules start here.

**Of the code under `lib/`, only the service imports `zenoh_dart`.** In this chapter everything the service hands
upward is plain Dart: a `String` for an id, a `List<String>` for the peers. Nothing above it needs a type from
`zenoh_dart`.

**The service owns the session.** Flutter's guide says a service holds no state. This one holds a `Session`, because
the session is the one stateful thing between the program and zenoh, and it needs one owner with one lifecycle. That
owner is disposed through the provider, so the session lives as long as the container does.

**Real zenoh is used in two kinds of test.** The test of each chapter's claim and the service's own tests open real
sessions, two in one process, on the loopback. Every other test replaces the layer to its right with a fake, so those
tests are fast and run without a network.

**Both packages use `very_good_analysis`, with every rule on.** So every public name has a doc comment, imports
inside `lib/` use `package:`, and output goes through `stdout`.

**Providers wire the program.** Flutter's guide builds its objects with constructor injection and a
`ChangeNotifier`. Both of this guide's applications use Riverpod, because `ChangeNotifier` ships with Flutter and a
pure-Dart program cannot use it, and one mechanism serves both programs. `sensorctl` declares its providers in
`lib/config/providers.dart`, the container builds them, and a test replaces one with an override.

**Why the core package exists before the app that shares it.** In chapter 2 the phone app depends on `sensor_core`
as `sensorctl` does now, with the same `ZenohService`, opened from `SessionSettings.sensorNode()`, and the same tests,
run on the laptop.

**What comes next.** Chapter 2 builds the sensor node itself: a Flutter app on the Android emulator and then on a
phone, in this same workspace, depending on this same `sensor_core`. It opens a session from
`SessionSettings.sensorNode()`, puts the device's accelerometer behind a service of its own, and publishes readings on
`sensor/phone/accel`, which the package's `z_sub` receives on your laptop.

Chapter 3 builds `watch` and the collector's layers to the left of the service: a repository that owns the key
expressions, a view model, and the terminal as the view. Each is tested against a fake of the one to its right, and
`watch` replaces `z_sub`. Chapter 4 adds the phone's gyroscope, and `watch` shows both sensors.

## 11 — Files and versions at the end of this chapter

**1. Commit.** Commit everything this chapter made, in one commit:

```sh
# in zenoh_sensors
git add .
git commit -m "A session of your own: sensor_core, ZenohService and its tests"
```

The commit holds 18 files, and 19 with `.vscode/launch.json`:

- 12 are new: the core package, the providers, `z_info.dart` and the workspace's own `pubspec.yaml`.
- 3 changed, the program's lint rules among them, or 4 with `.vscode/launch.json`.
- 2 were deleted: the template's `lib/sensorctl.dart` and its test.
- 1 is the lock file, which git reports as moved from `apps/sensorctl/` to the top, because a workspace keeps one.

`.dart_tool/` stays out at every level, and so does `.fvm/`.

> **In VS Code.** **View › Source Control** lists the same changes. Choose the **+** on the **Changes** line to stage
> them all, type the message in the box above them, and choose **Commit**.

`zenoh_sensors` now holds this, in three commits, the third of them
`A session of your own: sensor_core, ZenohService and its tests`.

```
zenoh_sensors/
├── .dart_tool/
├── .fvm/
├── .fvmrc
├── .git/
├── .gitignore
├── .vscode/                    if you use VS Code
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

Two packages are new in this chapter, checked with these versions:

| what | version |
|---|---|
| `riverpod` | 3.4.3 |
| `very_good_analysis` | 11.0.0 |

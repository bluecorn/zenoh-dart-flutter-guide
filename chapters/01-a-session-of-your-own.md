# 1 — A session of your own

This chapter follows chapter 4 of the zenoh book, *Sessions and Configuration*, and starts from a program shipped with
the `zenoh_dart` package, `z_info`.

## 1 — What you build, and what you will see

By the end of this chapter `sensorctl` is your program, rather than a folder with someone else's examples in it. It
opens a zenoh session with a configuration you wrote, says who it is and who it found, and closes. On the way the top
folder becomes a **workspace** with a second package, `sensor_core`, and the code that touches zenoh moves there behind
one class, `ZenohService` — which stays, for the whole guide, the only file that imports `zenoh_dart`. A test drives
that move. It is the first test you write, and the first one you watch fail.

What you will see, with the package's `z_sub` running in a second terminal as it was in chapter 0:

```
sensorctl is 8d2ee6a92269c47d3b9fd2897f946465
connected to 1 peer:
  8278f066730daedae9902fc949189943
```

Both ids will be different on your machine, and different again every time you run it: a session picks a new one each
time it opens. The first is `sensorctl`'s own. The second is `z_sub`'s, and it is the point of the chapter — **nothing
discovered it.** You will tell `sensorctl` exactly where to look, and in chapter 0 you told `z_sub` exactly where to
listen. Zenoh found no one on your behalf, and that is deliberate: from here to the last chapter, every program in this
guide says where it is and who it talks to.

> **If you already know zenoh.** This is `z_info` with two changes. The configuration is written in code rather than
> taken from flags, because the topology is fixed from here on — every program a peer, multicast scouting and gossip
> off, the sensor node listening on the loopback and the collectors connecting to it. And the zenoh calls go behind
> one class
> immediately, because the Flutter app in Part 2 shares that class rather than reimplementing it.

> **If you have not used a pub workspace, or Riverpod outside Flutter.** Both arrive in this chapter with the smallest
> example that needs them, and both are explained where they appear: a workspace is one `pubspec.yaml` at the top that
> resolves the dependencies of every package below it, and a `ProviderContainer` is Riverpod without a widget tree —
> the same providers the Flutter app will use in Part 2.

## 2 — What to read

Two books sit behind this guide, and they do different jobs. Read a little of each before you start.

**[The Zenoh Book](https://corsaro.me/zenoh/book/core-concepts/sessions/), *Core Concepts → Sessions*, for the idea.**
What a session actually is — your program's one connection to everything zenoh does — what the three modes mean and
what each costs you, what closing one entails, and what it means to open more than one in a single process. That last
point is not a curiosity here: the test you write in this chapter opens two.

**[Zenoh Programming in Rust](https://kydos.github.io/zenoh-book/chapter_04.html), chapter 4, *Sessions and
Configuration*, for the shape of the API.** Opening a session; the default configuration and what it assumes;
configuration from a file against configuration built in code, which is what you will do; session info; and graceful
shutdown. Skip *Runtime Configuration via Admin Space* — it changes a running router's settings, and this guide has no
router until the last chapter.

Both are written in Rust. That is not a problem — you are reading them for the ideas and the shapes, not to copy — but
five things look different by the time they reach Dart:

| in the books | in `zenoh_dart` |
|---|---|
| `zenoh::open(config).await.unwrap()` | `await Session.open(config: config)` — a failure throws rather than returning a `Result` |
| `config.insert_json5("mode", r#""peer""#)` | `config.insertJson5('mode', '"peer"')` — same paths, same `/` between nested keys, same JSON5 values |
| `session.info().zid().await` | `session.zid` — and `.toHexString()` to print it |
| `routers_zid()`, `peers_zid()` return async streams you `.collect()` | `routersZid()` and `peersZid()` return plain lists |
| `session.close().await.unwrap()` | `session.close()` — nothing to await, nothing returned, and safe to call twice |

And one point from chapter 4 matters more than its length suggests: **opening a session completes the handshake before
it returns.** Scouting, binding, and any connection the configuration asked for are all finished by the time you have a
session in your hand. This chapter leans on that from its first test to its last line.

> **A note on versions.** *Zenoh Programming in Rust* is written against Zenoh 1.4.0 and `zenoh_dart` 1.0.0-rc.1 is
> built on 1.8.0, so an occasional detail there will have moved on. Where this guide states something about the Dart
> API, it has been read in the package itself.

## 3 — The program that opens a session

Chapter 0 ran two programs that came with the package. This section writes the first one that is yours. It starts by
running a third example, `z_info`, because `z_info` does what `sensorctl` is about to do — opens a session, says who it
is and who it found, and closes — and it is easier to write a program once you have watched the thing it has to do.

**1. Go to the program's folder.** Chapter 0 ended in the top folder.

```sh
# in zenoh_sensors
cd apps/sensorctl
```

**2. Copy one more example.** The same two lines as chapter 0: read the package's folder out of
`.dart_tool/package_config.json`, then copy.

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

Two marks in that block come back in every expected output. `⋮` stands for lines the guide does not show — here,
what pub prints before the program speaks: `Running build hooks...`, and sometimes `Building package executable...`
and `Built …`, on one line or several, depending on what it had to do. `…` stands for the part of a line that
differs on your machine, such as an identity or a timestamp. Everything else is expected exactly as printed.

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

Four things to take from that.

**The id is the session.** `own id` is sixteen bytes printed as thirty-two hexadecimal characters, and it is new every
time a session opens — yours differs from the one above and from the one you get next time. Zenoh uses it in timestamps
and in the names of things in its admin space; in this guide it is how you tell one running program from another.

**`routers ids:` is empty, and will stay empty.** A router is a separate program, `zenohd`, that sessions connect
through instead of connecting to each other. This guide does not use one until the last chapter, so the line is not a
problem to fix.

**`peers ids:` is `z_sub`.** It is the only other zenoh program on your machine, and it is there because you told both
programs where to be. The three options are the ones chapter 0 gave `z_put`: `-e` connects to the subscriber,
`--no-multicast-scouting` stops this program announcing itself on your network, and `--cfg 'listen/endpoints:[]'` stops
it listening — without that last one a peer also listens on every interface, on a port picked at random. Nothing
searched, and nothing was discovered.

**The red `ERROR` line is the one chapter 0 warned about**, zenoh 1.8.0 complaining as a connected session closes.
Nothing failed.

**5. Write your own program.** Replace the whole content of `zenoh_sensors/apps/sensorctl/bin/sensorctl.dart` — the
template's `Hello world` — with this:

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

> **In VS Code.** Open the file from the Explorer on the left — `apps` › `sensorctl` › `bin` › `sensorctl.dart` — and
> replace what is in it.

`Zenoh.initLog('error')` turns on zenoh's own logging, at the level that prints errors and nothing else. It is the
first line of `main` for a reason: a session that fails to open throws an exception carrying one code for nearly every
cause, and the sentence that says what actually went wrong is in that log. Without this line, a wrong endpoint and an
unreachable machine look identical.

The five settings are this guide's topology, written down once. **`mode: peer`** because there is no router to be a
client of. **Multicast scouting and gossip off** because both are ways of finding sessions you were not told about, and
nothing here should be found by accident. **`listen/endpoints: []`** because `sensorctl` is a collector here: it
starts
conversations and never has to accept one, so it needs no address of its own — and a peer that is given no listen
endpoints listens on every interface instead, which is exactly what you do not want. **`connect/endpoints`** is the one
address it does need: the node, on the loopback, where `z_sub` is waiting.

The rest is what `z_info` did. `Session.open` is a `Future` because the connection happens while you wait: when it
returns, the session is open and the connection it was told to make has already been made, or already failed.
`session.zid` is the id, and `peersZid()` is the list you saw. `close()` sits in a `finally` so that it runs even if
printing throws, which is the habit chapter 10 builds on when it handles signals.

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

Your program found the subscriber, and printed the same id `z_info` printed.

**7. Give it an entry in Run and Debug.** Chapter 0 wrote entries for the two examples; your own program deserves one
too. Replace the whole content of `zenoh_sensors/.vscode/launch.json` with this — the two entries you already have, and
a third:

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

`sensorctl` needs no `args`: everything the examples took from the command line is in its own code now.

> **In VS Code.** With `z_sub` still running, choose **sensorctl** at the top of the Run and Debug view and press ▶.
> The Debug Console prints what the terminal printed. Nothing here is new except the entry itself — and the `cwd` line,
> which says the program starts in the folder holding its own `pubspec.yaml`, because that is where the package
> unpacked zenoh's native library in chapter 0. **The next section moves that folder, and all three of these entries
> change with it.** That is not a mistake in this file; it is what a workspace does, and you will see it happen.

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

**The session still opened.** Nothing was listening at `tcp/127.0.0.1:7447`, `sensorctl` spent about half a second
trying, and then carried on with no peers and no complaint — and no `ERROR` line either, because a session that was
never connected has nothing to complain about as it closes.

This is worth holding on to, because it is the first thing that will mislead you: **a session that opens is not a
session that is connected.** `open()` succeeding means your configuration was valid and zenoh is running, not that
anyone is on the other end. Later in this chapter a test pins that down, so it stays true as the code moves.

**9. Throw away the template's leftovers.** When `dart create` made this project in chapter 0 it wrote a small library,
`lib/sensorctl.dart`, holding a `calculate()` function, and a test for it — and the program it wrote imported that
library and printed `Hello world: 42!`. That is what proved the toolchain worked. The program you have just written
imports nothing of the sort, so both files are now dead, and a test that tests dead code is worse than no test at all:

```sh
# in zenoh_sensors/apps/sensorctl
rm lib/sensorctl.dart test/sensorctl_test.dart
```

`lib/` and `test/` come back, with things of your own in them. `lib/` returns at the end of this chapter, holding the
providers that wire the program together; its view models and its terminal view arrive in chapter 3, when
`bin/sensorctl.dart` shrinks to the few lines that start them. `test/` holds this program's first test in chapter 3,
when there is something of its own to test. The rule the deletion follows is worth keeping:
**code that nothing uses goes the moment it stops being used**, not at some tidying-up later.

## 4 — A workspace, and a package to share

The zenoh code you are about to write is not only `sensorctl`'s. In Part 2 a Flutter app on a phone opens a session of
its own, with the same class, the same settings and the same tests — so it belongs in a package both programs can
depend on, not inside one of them. This section makes that package and the arrangement that lets two programs share it:
a **pub workspace**, one folder at the top that resolves the dependencies of everything below it.

It changes where you run things from. That is the part worth watching, because it is not obvious and it is permanent.

**1. Go back to the top folder.**

```sh
# in zenoh_sensors/apps/sensorctl
cd ../..
```

**2. Make the core package.** `dart create` will not make the folder above the one it creates, so `packages` comes
first:

```sh
# in zenoh_sensors
mkdir packages
fvm dart create --template=package packages/sensor_core
```

You now have a second Dart package with its own `pubspec.yaml`, its own `lib/`, and its own `test/` holding one
passing test. Nothing connects it to `sensorctl` yet.

`dart create` also writes an `example/` folder demonstrating the library it invented. Nothing here will use it, so it
goes the same way the last section's leftovers went:

```sh
# in zenoh_sensors
rm -r packages/sensor_core/example
```

What the template left in `lib/` and `test/` stays for now — the next two sections replace both with real files, and
until then the package resolves and its test passes, which is a better place to work from than an empty folder.

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
        │   └── sensor_core_test.dart
        └── pubspec.yaml
```

**3. Write the workspace's own `pubspec.yaml`.** Create `zenoh_sensors/pubspec.yaml` with this content:

```yaml
name: zenoh_sensors
publish_to: none

environment:
  sdk: ^3.13.2

workspace:
  - apps/sensorctl
  - packages/sensor_core
```

This package holds no code and is never published — `publish_to: none` says so. It exists to name the members and to
own the one resolution they share.

**4. Tell each package that it belongs to the workspace.** Add one line, `resolution: workspace`, to each member's
`pubspec.yaml`. `zenoh_sensors/apps/sensorctl/pubspec.yaml` becomes:

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
  lints: ^6.0.0
  test: ^1.25.6
```

Three of `dart create`'s lines go at the same time. The description was `A sample command-line application.`, which is
true of a sample and not of this; the commented-out `repository:` points at `my_org/my_repo`; and `path` was never
imported by anything here. **A dependency nothing imports is dead in the same way an unused file is** — it still has to
resolve, still pins a version, and still has to be explained to whoever reads the file next.

and `zenoh_sensors/packages/sensor_core/pubspec.yaml` becomes:

```yaml
name: sensor_core
description: The zenoh data layer that sensorctl and the phone app share.
version: 1.0.0

environment:
  sdk: ^3.13.2

resolution: workspace

dev_dependencies:
  lints: ^6.0.0
  test: ^1.25.6
```

with the same three lines removed, and no `dependencies:` at all yet — this package depends on nothing until section 6
gives it `zenoh_dart`.

**5. Resolve once, from the top.**

```sh
# in zenoh_sensors
fvm dart pub get
```

It prints two or three lines about deleting an old lock file and an old package config, and points at a page
explaining them. Nothing is wrong: until now each package resolved its own dependencies and kept its own
`pubspec.lock`, and a workspace has one of each for everything. From here there is a single `pubspec.lock` and a single
`.dart_tool/` beside the top `pubspec.yaml`, and the per-package ones are gone.

**6. Throw away what chapter 0 left behind.**

```sh
# in zenoh_sensors
rm -rf apps/sensorctl/.dart_tool
```

This one matters more than it looks. `zenoh_dart`'s build hook stages zenoh's native libraries into the `.dart_tool/`
of whatever is being resolved, and it has just staged them at the top. The copy chapter 0 staged inside
`apps/sensorctl/.dart_tool/` is still there, and `pub get` does not remove it — it only removed the package config
beside it. A program started from inside `apps/sensorctl` finds that old copy first and runs happily on it, which looks
like good news and is not: it is a second, ageing copy of the library that no longer gets updated. Delete it, and from
here there is exactly one.

**7. Run the program from the top folder.** This is the change:

```sh
# in zenoh_sensors
fvm dart run sensorctl:sensorctl
```

```
⋮
…sensorctl is …
connected to 0 peers:
```

`sensorctl:sensorctl` is *package name*, then *program name* — the file `bin/sensorctl.dart` inside the package
`sensorctl`. Zero peers is right: nothing is listening now, which is the end of the last section.

**From here, programs and tests are run from `zenoh_sensors`,** not from inside a package. `fvm dart run
sensorctl:sensorctl` for the program, `fvm dart test packages/sensor_core` for the core's tests. The reason is the
library those build hooks staged: it is at the top now, and a program started from inside `apps/sensorctl` cannot find
it — you just deleted the copy that was hiding that. The next sections lean on this, and so does every chapter after.

**8. Move the editor with it.** All three entries in `zenoh_sensors/.vscode/launch.json` name a folder that programs
can no longer start in. Replace the whole file:

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

One line changed in each: `cwd` is now the folder VS Code has open, the top of the workspace. This is the same rule as
the terminal, written where the editor can read it.

> **In VS Code.** Press ▶ on **sensorctl** again. It prints what it printed a moment ago. Had you not changed this file,
> it would have failed to start with an error about not finding `libzenoh_dart.so` — the editor was starting it in a
> folder that no longer holds the library.

## 5 — The test that says what you are building

From here the chapter works the way the rest of the guide works: the test is written before the thing it tests. Not as
a ritual. A test is the first place the intention has to be put in words precise enough to run, and **if you cannot
write the assertion, you do not yet know what you are building.** It is also the first caller of code that does not
exist, so the way it reads is the shape that code will have.

**1. Say what this chapter claims, in one sentence.**

> A sensor node and a collector, configured the way this guide configures them, find each other on the loopback.

That sentence is the test's name, and everything the rest of the chapter writes exists to make it true.

**2. Clear the template out of the core package, and make room.** `dart create` left a library called `Awesome` and a
test for it. Neither is yours:

```sh
# in zenoh_sensors
rm packages/sensor_core/test/sensor_core_test.dart packages/sensor_core/lib/src/sensor_core_base.dart
mkdir -p packages/sensor_core/lib/src/services packages/sensor_core/test/services
```

`test/` mirrors `lib/` — a file's test sits at the same path under `test/` as the file does under `lib/src/`. That is
the arrangement for the whole guide, and it is easier to start it than to impose it later.

**3. Write the test.** Create `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:test/test.dart';

void main() {
  test('a sensor node and a collector find each other on loopback', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    await sensorNode.open();
    await collectorNode.open();

    expect(collectorNode.peerIds, contains(sensorNode.zid));
    expect(sensorNode.peerIds, contains(collectorNode.zid));
    expect(collectorNode.zid, isNot(sensorNode.zid));
  });
}
```

Two sessions, in one Dart process, doing to each other exactly what `sensorctl` and `z_sub` did in two terminals. This
is the only kind of test in the guide that opens real zenoh sessions: zenoh is the subject, so the claim about zenoh is
checked against zenoh, once, here. Everything built on top of this in later chapters is tested against stand-ins.

**The sensor node opens first.** A collector needs something to connect to, and its settings say where. Swap the two
lines and it starts before anything is listening — the case the last section ended on.

**`addTearDown`** hands the closing to the test runner, so the sessions close even when an expectation fails. Without
it, a failing test leaves port 7447 held and the next run fails for a reason that has nothing to do with your code.

**The three expectations are the sentence, in order.** The first two are the claim itself: each one's list of peers
holds the other's identity. The third is the guard that makes the first two mean something — if both sessions somehow
reported the same id, `contains` would pass while nothing had been found at all.

**4. Write just enough for it to compile.** Three files, none of which do anything yet. First,
`zenoh_sensors/packages/sensor_core/lib/src/services/session_settings.dart`:

```dart
class SessionSettings {
  const SessionSettings._();

  factory SessionSettings.sensorNode() => const SessionSettings._();
  factory SessionSettings.collectorNode() => const SessionSettings._();
}
```

Then `zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'session_settings.dart';

class ZenohService {
  ZenohService(this.settings);

  final SessionSettings settings;

  Future<void> open() async {}
  String get zid => '';
  List<String> get peerIds => const [];
  void dispose() {}
}
```

And replace `zenoh_sensors/packages/sensor_core/lib/sensor_core.dart`, which still exports the template, with the two
files that are now the package:

```dart
/// The zenoh data layer that `sensorctl` and the phone app share.
library;

export 'src/services/session_settings.dart';
export 'src/services/zenoh_service.dart';
```

**These are the emptiest classes that will compile** — they answer every question, with nothing. That is deliberate.
Watching them fail is the check that the test is worth keeping: if something this empty could satisfy it, the test
would be about the shape of a class rather than about zenoh, and it would go on passing long after the code stopped
working.

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

That is the right failure. It is not "you have not written this yet" — it is the claim, in the test runner's own
words: the collector's list of peers should hold the sensor node's identity, and it is empty.

**6. Tell the editor the same thing.** The terminal runs tests from the top folder; the editor has to be told to.
Replace `zenoh_sensors/.vscode/launch.json` with this — the three entries you have, and a fourth with no program in it:

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

An entry with `templateFor` and no `program` is a template: the editor takes its `cwd` for every test it runs from
the Testing view, and for the **Run** and **Debug** links it shows above a test. Without it a test would start inside
`packages/sensor_core`, where there is no copy of zenoh's library, and fail before it began.

> **In VS Code.** Open the Testing view — the flask in the Activity Bar — and press ▶ beside the test, or click **Run**
> above it in the editor. It fails the same way.

**This test now stays red until the chapter is finished**, and that is deliberate. It is the outer of two loops. The
outer one holds the chapter's promise and goes green once, at the end. The inner one is where the work happens: small
cycles, each with its own test, each red then green, each leaving the code a little less empty than it was. The next
section is four of them.

## 6 — Two cycles: an identity

The outer test stays red for the rest of the chapter. Inside it the work happens in **cycles**, and a cycle is small:
one test, run it, watch it fail, do the least that makes it pass, run it again. Four of them build the service — two
here and two in the next section.

Run one test at a time while you are inside a cycle, by a piece of its name:

**Cycle 1 — a service has an identity once it is open.** Add a second test to
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`, so the file reads:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:test/test.dart';

void main() {
  test('a sensor node and a collector find each other on loopback', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    await sensorNode.open();
    await collectorNode.open();

    expect(collectorNode.peerIds, contains(sensorNode.zid));
    expect(sensorNode.peerIds, contains(collectorNode.zid));
    expect(collectorNode.zid, isNot(sensorNode.zid));
  });

  test('a service has an identity once it is open', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);

    await sensorNode.open();

    expect(sensorNode.zid, isNotEmpty);
  });
}
```

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

**Now do the least that makes it pass.** Not the right thing — the least thing. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart` with this, which is the same skeleton with one
constant in it:

```dart
import 'session_settings.dart';

class ZenohService {
  ZenohService(this.settings);

  final SessionSettings settings;

  Future<void> open() async {}
  String get zid => 'the-sensor-node';
  List<String> get peerIds => const [];
  void dispose() {}
}
```

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'has an identity'
```

It passes. `dart test` says so in one line, `All tests passed!`, and that is all a passing run ever prints — so from
here the guide shows nothing for one, and only the lines that matter for a failure.

That is a real technique and it has a name: **fake it**. The test asked for an identity that is not empty, and a
constant is an identity that is not empty. It is obviously not the answer, and writing it anyway is the point — you now
have a passing test and a lie, and the next test's job is to kill the lie. Faking it is only wrong as the *last* thing
you do.

**Cycle 2 — two services have different identities.** This is the test that kills it. Add a third to
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`, so the whole file reads:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:test/test.dart';

void main() {
  test('a sensor node and a collector find each other on loopback', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    await sensorNode.open();
    await collectorNode.open();

    expect(collectorNode.peerIds, contains(sensorNode.zid));
    expect(sensorNode.peerIds, contains(collectorNode.zid));
    expect(collectorNode.zid, isNot(sensorNode.zid));
  });

  test('a service has an identity once it is open', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);

    await sensorNode.open();

    expect(sensorNode.zid, isNotEmpty);
  });

  test('two services have different identities', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    await sensorNode.open();
    await collectorNode.open();

    expect(collectorNode.zid, isNot(sensorNode.zid));
  });
}
```

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

**A constant cannot differ from itself.** No cleverness will save the fake here: the only way to two different
identities is two real sessions. That move — adding a second case the shortcut cannot satisfy — is called
**triangulation**, and it is how a test forces code into existence rather than merely checking it afterwards.

Two real sessions need the package, and this is the moment the core package starts to depend on it — the one section
4 put off. Replace `zenoh_sensors/packages/sensor_core/pubspec.yaml` with:

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
  lints: ^6.0.0
  test: ^1.25.6
```

and resolve, from the top as always:

```sh
# in zenoh_sensors
fvm dart pub get
```

> **In VS Code.** Open `packages` › `sensor_core` › `pubspec.yaml`, add the two lines, and save. The Dart extension
> runs `pub get` for you whenever a `pubspec.yaml` is saved.

Then replace `zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart` with one that opens a session:

```dart
import 'package:zenoh_dart/zenoh.dart';

import 'session_settings.dart';

class ZenohService {
  ZenohService(this.settings);

  final SessionSettings settings;

  Session? _session;

  Future<void> open() async {
    _session = await Session.open(config: Config());
  }

  String get zid => _opened.zid.toHexString();
  List<String> get peerIds => const [];

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

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'identit'
```

Both pass. `-n 'identit'` matches both test names, which is the quickest way to check that the new one did not break
the old one.

Three things arrived with that file, and only one of them was asked for by the test.

**The session, which the test demanded.** `Session.open` is awaited, the handle is kept, and `zid` reports it as
thirty-two hexadecimal characters — the same id `z_info` printed in section 3.

**`_opened`, which turns a mistake into a sentence.** Asking a service for its id before opening it is a programming
error, so it throws `StateError` naming what was not done, instead of a null error from somewhere inside the package.

**And a configuration that is wrong on purpose.** `Session.open(config: Config())` is zenoh's *default* — the one
section 3 warned about, which listens on every interface and announces itself to the network. It is here because it is
the least thing that opened a session, and nothing has yet asked for better. **The next section asks.**

## 7 — Two cycles: a topology, and a connection

The outer test is still red, and the service still opens zenoh's default configuration. Two cycles remain. The first
takes away the two ways a session has of finding company it was never told about; the second gives each side its
address, and the outer test goes green.

That order is not a matter of taste. A green is worth exactly as much as the red before it. Had the addresses gone in
first, while scouting was still on, the outer test could have gone green for the wrong reason — two sessions found by
the network rather than by the addresses you wrote — and nothing in its output would say which. With scouting off
first, the addresses are the only way left, and the green that follows means what it says.

**Cycle 3 — neither side announces itself on the network.** Section 3 wrote five settings into `bin/sensorctl.dart`.
Three of them never vary, whichever side a program is on: every session in this guide is a `peer`, and it neither
scouts nor gossips. Scouting is how zenoh finds sessions nobody configured — a new session calls out on a multicast
address, and every session that hears it answers with where it can be reached. The Zenoh Book's
[Multicast Scouting](https://corsaro.me/zenoh/book/routing/scouting/) page has that exchange in one picture. Gossip is
the second-hand version: a session passes on to the sessions it knows the addresses of the others it has met. Both are
off here, for the whole guide, because nothing in it should be found by accident.

Where this cycle's test looks is its whole point. It does not open a session. What these three settings do is make
something *not* happen, and a test that opens two sessions can watch them find each other but cannot watch them not
find anyone else — there is no one else in the test to find. You can prove that once the chapter is finished: delete
the two `scouting` entries from the settings, and the outer test still passes. A setting whose entire effect is an
absence has to be checked the one way it can be: read back as data, and compared with what it should say.

So the test reads the settings as data, through a property they do not have yet — `asJson5`, the settings as the
entries `insertJson5` takes, a map from a key path to a JSON5 value, the same pairs section 3 wrote by hand. Replace
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart` with this, which adds a fourth test:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:test/test.dart';

void main() {
  test('a sensor node and a collector find each other on loopback', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    await sensorNode.open();
    await collectorNode.open();

    expect(collectorNode.peerIds, contains(sensorNode.zid));
    expect(sensorNode.peerIds, contains(collectorNode.zid));
    expect(collectorNode.zid, isNot(sensorNode.zid));
  });

  test('a service has an identity once it is open', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);

    await sensorNode.open();

    expect(sensorNode.zid, isNotEmpty);
  });

  test('two services have different identities', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    await sensorNode.open();
    await collectorNode.open();

    expect(collectorNode.zid, isNot(sensorNode.zid));
  });

  test('neither side announces itself on the network', () {
    final sensorSettings = SessionSettings.sensorNode().asJson5;
    final collectorSettings = SessionSettings.collectorNode().asJson5;

    for (final settings in [sensorSettings, collectorSettings]) {
      expect(settings, containsPair('mode', '"peer"'));
      expect(settings, containsPair('scouting/multicast/enabled', 'false'));
      expect(settings, containsPair('scouting/gossip/enabled', 'false'));
    }
  });
}
```

And give `SessionSettings` the emptiest `asJson5` that compiles, so that the test fails on its claim and not on a
missing name. Replace `zenoh_sensors/packages/sensor_core/lib/src/services/session_settings.dart`:

```dart
class SessionSettings {
  const SessionSettings._();

  factory SessionSettings.sensorNode() => const SessionSettings._();
  factory SessionSettings.collectorNode() => const SessionSettings._();

  Map<String, String> get asJson5 => const {};
}
```

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

Then write the real thing. The three ways to green have names — fake it, triangulate, and this one, **obvious
implementation**: when you know what to write, write it. There is no fake to make here anyway. When the thing under
test is data, the least thing that passes and the real thing are the same three lines, which is what makes a test like
this look like a restatement of the code — and it is one, on purpose. Its job is to make those three lines impossible
to delete without a test going red, which no other test in this chapter can do. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/session_settings.dart`:

```dart
class SessionSettings {
  const SessionSettings._();

  factory SessionSettings.sensorNode() => const SessionSettings._();
  factory SessionSettings.collectorNode() => const SessionSettings._();

  Map<String, String> get asJson5 => const {
    'mode': '"peer"',
    'scouting/multicast/enabled': 'false',
    'scouting/gossip/enabled': 'false',
  };
}
```

And make the service build its configuration from it. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'package:zenoh_dart/zenoh.dart';

import 'session_settings.dart';

class ZenohService {
  ZenohService(this.settings);

  final SessionSettings settings;

  Session? _session;

  Future<void> open() async {
    _session = await Session.open(config: _config());
  }

  String get zid => _opened.zid.toHexString();
  List<String> get peerIds => const [];

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

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'announces'
```

It passes. Run the two identity tests as well, with `-n 'identit'`, and they still pass: two sessions still open, now
as peers that scout nothing, and each still gets an id.

`_config()` is the one thing in that file no test asked for. `Config()` is zenoh's default, and
`forEach(config.insertJson5)` hands every key and value of the map to `insertJson5` in turn, so the map is the whole
of the difference between the default and what you open. No test in this guide watches that line do its work — an
absence, again — so it is the one line in this chapter you take on trust. It is one line.

**What the test guarantees:** every session this guide opens is a peer that neither goes looking for other sessions
nor passes on word of the ones it has met. Nothing is found by accident.

**Cycle 4 — and yet they find each other.** Before you run the outer test again, take out the last of section 5's
neutral answers. `peerIds` still returns an empty constant, and a red produced by a stub says nothing about zenoh.
Replace `zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
import 'package:zenoh_dart/zenoh.dart';

import 'session_settings.dart';

class ZenohService {
  ZenohService(this.settings);

  final SessionSettings settings;

  Session? _session;

  Future<void> open() async {
    _session = await Session.open(config: _config());
  }

  String get zid => _opened.zid.toHexString();
  List<String> get peerIds =>
      _opened.peersZid().map((id) => id.toHexString()).toList();

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

`peersZid()` is the list `z_info` printed under `peers ids:`. As with `zid`, the service turns each id into its
thirty-two characters before handing it up, so that nothing above the service needs a type from the package.

Now the outer test:

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

Compare it with the red at the end of section 5. The id is real now — thirty-two characters of it. And the empty list
is real too, zenoh's own answer and not the skeleton's: two sessions are open in one process, each a peer, each with
scouting off, and neither has been told where the other is. That is exactly what cycle 3 asked for, and it is the
situation every program in this guide starts from. Nothing finds anything until it is given an address.

The sensor node waits at an address; the collector goes to it. That is data again, so the test is of the same kind as
the last one. Replace `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart` with this, which adds
a fifth test:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:test/test.dart';

void main() {
  test('a sensor node and a collector find each other on loopback', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    await sensorNode.open();
    await collectorNode.open();

    expect(collectorNode.peerIds, contains(sensorNode.zid));
    expect(sensorNode.peerIds, contains(collectorNode.zid));
    expect(collectorNode.zid, isNot(sensorNode.zid));
  });

  test('a service has an identity once it is open', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);

    await sensorNode.open();

    expect(sensorNode.zid, isNotEmpty);
  });

  test('two services have different identities', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    await sensorNode.open();
    await collectorNode.open();

    expect(collectorNode.zid, isNot(sensorNode.zid));
  });

  test('neither side announces itself on the network', () {
    final sensorSettings = SessionSettings.sensorNode().asJson5;
    final collectorSettings = SessionSettings.collectorNode().asJson5;

    for (final settings in [sensorSettings, collectorSettings]) {
      expect(settings, containsPair('mode', '"peer"'));
      expect(settings, containsPair('scouting/multicast/enabled', 'false'));
      expect(settings, containsPair('scouting/gossip/enabled', 'false'));
    }
  });

  test('the collector connects to where the sensor node listens', () {
    const address = '["tcp/127.0.0.1:7447"]';
    final sensorSettings = SessionSettings.sensorNode().asJson5;
    final collectorSettings = SessionSettings.collectorNode().asJson5;

    expect(sensorSettings, containsPair('listen/endpoints', address));
    expect(sensorSettings, containsPair('connect/endpoints', '[]'));
    expect(collectorSettings, containsPair('listen/endpoints', '[]'));
    expect(collectorSettings, containsPair('connect/endpoints', address));
  });
}
```

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

Obvious implementation again. The two lists are the only thing that differs between the two sides, so they become the
value's two fields, and the two factories fill them in opposite ways. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/session_settings.dart`:

```dart
class SessionSettings {
  const SessionSettings._({
    required this.listenEndpoints,
    required this.connectEndpoints,
  });

  factory SessionSettings.sensorNode() => const SessionSettings._(
    listenEndpoints: [nodeEndpoint],
    connectEndpoints: [],
  );

  factory SessionSettings.collectorNode() => const SessionSettings._(
    listenEndpoints: [],
    connectEndpoints: [nodeEndpoint],
  );

  static const nodeEndpoint = 'tcp/127.0.0.1:7447';

  final List<String> listenEndpoints;
  final List<String> connectEndpoints;

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

`nodeEndpoint` is the one address in Part 1: the loopback, port 7447, where `z_sub` waited in chapter 0 and where the
stand-in sensor node will wait from chapter 2. `listen/endpoints: []` on the collector is the setting section 3
explained: without it, a peer listens on every interface, on a port picked at random. And `_json5List` writes a Dart
list as the JSON5 text `insertJson5` wants — `["tcp/127.0.0.1:7447"]`, or `[]`.

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'connects to where'
```

It passes. Now the outer test, for the last time:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'find each other'
```

It passes. The chapter's promise is true, and you watched every part of it become true. Run the whole file once, the
way it will be run from here on:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

Five tests, all passing.

**What the outer test guarantees:** a sensor node and a collector, configured as this guide configures them, find each
other on the loopback — and have already done so when the collector's `open()` returns. The test asserts straight after
the second `open()`, with no waiting and no retry, and it holds because opening a session completes the handshake
before it returns: the point from chapter 4 of *Zenoh Programming in Rust* that the start of this chapter flagged.

The service is finished. `bin/sensorctl.dart` has not noticed: it still carries its own five settings and opens its own
session. The next section moves it onto the service — the refactor — with these five tests green before and after.

## 8 — The refactor, and a provider container

`bin/sensorctl.dart` still opens zenoh by itself, with its own copy of the five settings the service now carries and
tests. This section moves it onto the service, and then onto a *provider container*, which is how every program in
this guide is put together from here on. It is the chapter's refactor: the behaviour does not change, the tests are
green before and after, and the program prints exactly what it printed in section 3.

**1. Green before.** A refactor starts from green, so that anything red on the way is yours:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

Five tests pass.

**2. Let the program depend on the core.** Replace `zenoh_sensors/apps/sensorctl/pubspec.yaml` — one line is new:

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
  lints: ^6.0.0
  test: ^1.25.6
```

and resolve, from the top:

```sh
# in zenoh_sensors
fvm dart pub get
```

A workspace member depends on another by name — no path to write, and the version is the one
`packages/sensor_core/pubspec.yaml` declares; pub resolves it to the folder next door. `zenoh_dart` stays, although
nothing of yours in this package imports it after this section: the three examples in `apps/sensorctl/example/` are
the package's own programs, and `z_sub` stands in for the sensor node until chapter 2 gives you one of your own.

> **In VS Code.** Open `apps` › `sensorctl` › `pubspec.yaml`, add the line, and save; the extension runs `pub get`.

**3. Logging, from the core.** Section 3's program started zenoh's logging with `Zenoh.initLog('error')`. That call
lives in the package, which `bin/sensorctl.dart` is about to stop importing, so the core offers it as a function of
its own. Replace `zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart` — the class is unchanged; one
line is added above it:

```dart
import 'package:zenoh_dart/zenoh.dart';

import 'session_settings.dart';

void initZenohLogging(String level) => Zenoh.initLog(level);

class ZenohService {
  ZenohService(this.settings);

  final SessionSettings settings;

  Session? _session;

  Future<void> open() async {
    _session = await Session.open(config: _config());
  }

  String get zid => _opened.zid.toHexString();
  List<String> get peerIds =>
      _opened.peersZid().map((id) => id.toHexString()).toList();

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

It is a function, not a method of the service, on purpose. Logging belongs to the process, not to a session: the
package starts it once, and a later call — from a second service, say — is silently ignored. So it is called once,
from `main`, before anything opens, as the package's examples do. The level you pass is a fallback: the `RUST_LOG`
environment variable wins when it is set, which is how you turn on more detail without editing the program.
Misspell the level — `'errod'`, say — and you get no log at all, and no complaint: zenoh reads the word as a filter
that matches nothing.

**4. The refactor.** Replace `zenoh_sensors/apps/sensorctl/bin/sensorctl.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';

Future<void> main() async {
  initZenohLogging('error');

  final service = ZenohService(SessionSettings.collectorNode());
  try {
    await service.open();
    print('sensorctl is ${service.zid}');
    final peers = service.peerIds;
    final noun = peers.length == 1 ? 'peer' : 'peers';
    print('connected to ${peers.length} $noun:');
    for (final peer in peers) {
      print('  $peer');
    }
  } finally {
    service.dispose();
  }
}
```

Read it against section 3's. The five settings are gone: they are `SessionSettings.collectorNode()` now, and they are
tested. `Config` and `Session` went with them — the program imports `sensor_core` and nothing else, and no longer
knows that `zenoh_dart` exists. What is left is what the program is *for*: say who I am, say who I found. One thing
moved: `open()` is inside the `try` now, which it could not be in section 3, because a service exists before its
session does, and disposing one that never opened is safe — a rule the next section pins with a test.

**5. Start the subscriber, and run it.** In a terminal, start `z_sub` from the top folder — everything starts from
the top now, the examples included — and leave it running:

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

The three lines of section 3, and the same `ERROR` after them — zenoh 1.8.0's complaint as a connected session
closes, still nothing to fix. The program does what it did. Only its shape changed, and that is the whole of what a
refactor is.

**6. A container to build from.** The program made its service by hand, `ZenohService(SessionSettings.collectorNode())`,
in `main`. For one object that is fine. From chapter 3 the program has view models, which need a repository, which
needs a codec, which needs the service — and a test has to be able to swap any one of them for a stand-in. The guide's
way to build that graph is [Riverpod](https://riverpod.dev): every object is declared once, as a *provider*, next to
the others in one file, and a *provider container* builds them on demand, disposes them together, and lets a test
override any one of them. In the Flutter app of Part 2 the same providers live under a `ProviderScope` widget; a
`ProviderContainer` is that without a widget tree, which is why a pure-Dart program can use it.

Replace `zenoh_sensors/apps/sensorctl/pubspec.yaml` once more — `riverpod` is new:

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
  lints: ^6.0.0
  test: ^1.25.6
```

```sh
# in zenoh_sensors
fvm dart pub get
```

Then create `zenoh_sensors/apps/sensorctl/lib/config/providers.dart` — `lib/` is back, with the folder chapter 3
fills:

```dart
import 'package:riverpod/riverpod.dart';
import 'package:sensor_core/sensor_core.dart';

final sessionSettingsProvider = Provider<SessionSettings>(
  (ref) => SessionSettings.collectorNode(),
);

final zenohServiceProvider = Provider<ZenohService>((ref) {
  final service = ZenohService(ref.watch(sessionSettingsProvider));
  ref.onDispose(service.dispose);
  return service;
});
```

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

Two providers, and the split between them is deliberate. `sessionSettingsProvider` says which side of the topology
this program is on: a collector. It is a provider of its own rather than a constant inside the next one because it is
the entry that gets overridden — by a test, and by chapter 2's `simulate`, which is a sensor node inside the same
program. `zenohServiceProvider` builds the service from it — `ref.watch` reads another provider's value — and
`ref.onDispose(service.dispose)` ties the session's life to the container's: when the container is disposed, so is
the service, and the session closes. That is the rule from here to the end of the guide: **the service is disposed
through the provider**, never by hand.

Replace `zenoh_sensors/apps/sensorctl/bin/sensorctl.dart`:

```dart
import 'package:riverpod/riverpod.dart';
import 'package:sensor_core/sensor_core.dart';
import 'package:sensorctl/config/providers.dart';

Future<void> main() async {
  initZenohLogging('error');

  final container = ProviderContainer();
  try {
    final service = container.read(zenohServiceProvider);
    await service.open();
    print('sensorctl is ${service.zid}');
    final peers = service.peerIds;
    final noun = peers.length == 1 ? 'peer' : 'peers';
    print('connected to ${peers.length} $noun:');
    for (final peer in peers) {
      print('  $peer');
    }
  } finally {
    container.dispose();
  }
}
```

`container.read(zenohServiceProvider)` asks the container for the service; the container builds it, and the settings
it depends on, the first time it is asked, and hands back the same one after that. `container.dispose()` in the
`finally` is the only closing left in the program.

> **In VS Code.** With `z_sub` still running, press ▶ on **sensorctl**. The entry has not changed: the program is
> still `bin/sensorctl.dart`, started from the top folder.

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

Unchanged.

**7. Stop the subscriber, and green after.** Press Ctrl-C in the first terminal, then run the tests once more:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

Five tests pass. Notice what they did and did not say. They said nothing about `main`, which has no test of its own —
its check was its output, twice. They said the service still does what it did, which is what a refactor is allowed to
lean on. The program gets a test of its own in chapter 3, when it has a view worth testing.

## 9 — What the tests pin

The four cycles drove the service into existence, and every line in it has a test that asked for it. Three more
tests go in now, and they go in green: nothing is left to drive, so their job is the other half of what a test is
for — to keep something true as the code moves, and to say it to whoever reads the file next. One is about zenoh.
Two are about rules of your own.

**The second thing about zenoh worth pinning.** Section 3 showed it with the subscriber stopped: a session that opens
is not a session that is connected. `open()` returning means the configuration was valid and zenoh is running;
whether anyone is at the other end is a separate question, and its answer is the list of peers. This is the
misunderstanding most likely to cost you an afternoon later, so it gets a test of the same kind as the outer one —
real zenoh, at the service — named for what it says: *a collector opens even when no sensor node is listening*, and
finds no one.

**Two rules of the pattern, not facts about zenoh.** The other two tests are of a kind you have not written yet, and
the file should make the difference visible, because it matters when one of them fails. A *contract test* pins a
promise your own code makes — a rule of this guide's pattern that would hold whatever zenoh did. Section 8 leaned on
one without a test: `open()` moved inside the `try` because disposing a service that never opened is safe. Section 6
made another: asking a service for its identity before opening it is a programming error, and the error says so. A
promise made in prose is either backed by a test or withdrawn, so both get one. The dispose test also pins the half
section 8 did not need yet — disposing more than once — because from now on the container disposes the service, and
nothing may break if something else already did.

Replace `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart` with this, which adds the three:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:test/test.dart';

void main() {
  test('a sensor node and a collector find each other on loopback', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    await sensorNode.open();
    await collectorNode.open();

    expect(collectorNode.peerIds, contains(sensorNode.zid));
    expect(sensorNode.peerIds, contains(collectorNode.zid));
    expect(collectorNode.zid, isNot(sensorNode.zid));
  });

  test('a service has an identity once it is open', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);

    await sensorNode.open();

    expect(sensorNode.zid, isNotEmpty);
  });

  test('two services have different identities', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(sensorNode.dispose);
    addTearDown(collectorNode.dispose);

    await sensorNode.open();
    await collectorNode.open();

    expect(collectorNode.zid, isNot(sensorNode.zid));
  });

  test('neither side announces itself on the network', () {
    final sensorSettings = SessionSettings.sensorNode().asJson5;
    final collectorSettings = SessionSettings.collectorNode().asJson5;

    for (final settings in [sensorSettings, collectorSettings]) {
      expect(settings, containsPair('mode', '"peer"'));
      expect(settings, containsPair('scouting/multicast/enabled', 'false'));
      expect(settings, containsPair('scouting/gossip/enabled', 'false'));
    }
  });

  test('the collector connects to where the sensor node listens', () {
    const address = '["tcp/127.0.0.1:7447"]';
    final sensorSettings = SessionSettings.sensorNode().asJson5;
    final collectorSettings = SessionSettings.collectorNode().asJson5;

    expect(sensorSettings, containsPair('listen/endpoints', address));
    expect(sensorSettings, containsPair('connect/endpoints', '[]'));
    expect(collectorSettings, containsPair('listen/endpoints', '[]'));
    expect(collectorSettings, containsPair('connect/endpoints', address));
  });

  test('a collector opens even when no sensor node is listening', () async {
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(collectorNode.dispose);

    await collectorNode.open();

    expect(collectorNode.peerIds, isEmpty);
  });

  test('disposing is safe before open, and more than once after', () async {
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    expect(sensorNode.dispose, returnsNormally);

    await sensorNode.open();
    sensorNode.dispose();

    expect(sensorNode.dispose, returnsNormally);
  });

  test('asking an unopened service for its identity is an error', () {
    final sensorNode = ZenohService(SessionSettings.sensorNode());

    expect(() => sensorNode.zid, throwsStateError);
  });
}
```

The dispose test closes its own session — that is its subject — so it is the one test in the file without
`addTearDown`. `returnsNormally` and `throwsStateError` are matchers like `contains`: the first says a call must not
throw, the second that it must, with that type. `peerIds` before `open()` would throw the same `StateError` through
the same guard, so one test of the guard is enough.

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

Eight tests pass. The first of the three new ones takes about half a second: the time zenoh 1.8.0 spends trying an
address where nothing answers before it gives up, the same half second `sensorctl` spent in section 3.

**What they guarantee.** A collector whose sensor node is absent still opens, with no peers — so `open()` succeeding
never means "connected". Disposing a service is safe before it opens and more than once after, so the container may
dispose what a test or a `finally` already did. And a service asked for its identity before `open()` throws a
`StateError` that names what was not done, rather than a null error from inside the package.

The file is the chapter's test file, complete: one outer test that states the claim, four inner ones that drove the
code into existence, and three pins. Read top to bottom it is the story of the chapter, which is why the tests stay
in this order rather than grouped by class or by method.

## 10 — What changed in the architecture

Chapter 0 drew the layers and had nothing in them. This chapter filled in the one on the right:

```
view  →  view model  →  repository  →  codec  →  ZenohService  →  zenoh_dart
                                                 ─────────────
                                                  this chapter
```

`ZenohService` is the *service* of the pattern Flutter's
[architecture guide](https://docs.flutter.dev/app-architecture/guide) describes: one class per data source, wrapping
its API and handing plain values up. `SessionSettings` sits beside it — the topology as a value, tested as data. Three
rules began here, and they hold for the rest of the guide.

**Only the service imports the package.** Everything it hands upward is plain Dart: a `String` for an identity, a
`List<String>` for the peers. Nothing above it needs a type from `zenoh_dart`, and nothing above it needs zenoh to be
tested. The examples you copied in chapter 0 import the package too, but they are the package's programs, not yours.

**The service owns the session.** Flutter's guide says a service holds no state. This one holds a `Session`, on
purpose: the session *is* the connection, the one stateful thing in the transport, and it needs exactly one owner
with one lifecycle. That owner is disposed through the provider, so the session lives as long as the container does.

**Real zenoh is used in the service's tests and nowhere else.** Two sessions in one process, on the loopback, is where
the guide checks each zenoh claim it makes — once, at the layer that touches it. Everything to the left of the service
will be tested against a hand-written stand-in for the layer to its right, which is what makes those tests fast and
what lets them run without a network.

**And the program is wired by providers.** Flutter's guide builds its objects with constructor injection and a
`ChangeNotifier`; this guide uses Riverpod for both of its applications, because `ChangeNotifier` ships with Flutter
and a pure-Dart program cannot use it, and one mechanism for both programs is worth more than the default. From now
on `lib/config/providers.dart` is the one place a program's objects are declared, the container is what builds them,
and an override is how a test replaces one.

**Why the core package exists before the app that will share it.** In Part 2 the phone app depends on `sensor_core`
exactly as `sensorctl` does now — the same `ZenohService`, opened from `SessionSettings.sensorNode()`, and the same
tests, run on the laptop. The package boundary is also what keeps the first rule enforceable: `zenoh_dart` is
`sensor_core`'s dependency, and a program that wants zenoh gets the service.

**What comes next.** Chapter 2 gives `sensorctl` its second role: `simulate`, the stand-in sensor node, built from
`SessionSettings.sensorNode()` in the same program, publishing made-up readings that the package's `z_sub` can watch.
Chapter 3 builds `watch` and the layers to the left of the service — a repository that owns the key expressions, a
view model, and the terminal as the view — each tested against a stand-in for the one to its right; and `z_sub`
retires.

## 11 — Files and versions at the end of this chapter

**1. Commit.** Everything this chapter made, in one commit:

```sh
# in zenoh_sensors
git add .
git commit -m "A session of your own: sensor_core, ZenohService and its tests"
```

It takes eighteen files. Twelve are new: the core package, the providers, `z_info.dart` and the workspace's own
`pubspec.yaml`. Three changed, two were deleted in section 3, and the lock file git reports as moved from
`apps/sensorctl/` to the top, because a workspace keeps one. `.dart_tool/` stays out at every level, and so does
`.fvm/`, as before.

> **In VS Code.** **View › Source Control** lists the same eighteen changes. Choose the **+** on the **Changes** line
> to stage them all, type the message in the box above them, and choose **Commit**.

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

This chapter was checked with these versions. Newer ones are expected to work; if something does not, these are the
ones to go back to.

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
| `lints` | 6.1.0 |
| `test` | 1.32.0 |

---

*The code listings in this chapter are licensed under the Apache License 2.0. The text is © 2026 Hugo Alberto Garcia,
all rights reserved — see [COPYRIGHT](../COPYRIGHT).*

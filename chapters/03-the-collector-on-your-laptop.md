# 3 — The collector on your laptop

## 1 — What you build, and what you will see

By the end of this chapter, `sensorctl` has its first command, `watch`. It connects to the sensor node, subscribes to
`sensor/phone/accel`, and shows each reading as it arrives: the three values, and how many readings there have been.
With the node on the emulator, `watch` shows one line that changes 15 to 20 times a second:

```
Press Ctrl-C to stop.
x   0.000  y   9.776  z   0.812  153 readings
```

Both ends are now your code. The node puts through your `ZenohService`, and `watch` receives through another:

```
 the emulator or a phone                    the laptop
┌───────────────────────────────┐          ┌─────────────────────────────────┐
│ sensor_node                   │          │ sensorctl watch                 │
│                               │          │                                 │
│ SensorNodeRepository          │          │ WatchView, the terminal         │
│   │ each reading as x,y,z     │          │   ▲                             │
│   ▼                           │          │ WatchViewModel                  │
│ ZenohService + Publication    │          │   ▲                             │
│   │                           │          │ ReadingsRepository              │
│   ▼                           │   adb    │   ▲ each payload as a reading   │
│ put on sensor/phone/accel ════╪══════════╪═► ZenohService + Subscription   │
└───────────────────────────────┘ forward  └─────────────────────────────────┘
```

You build it in two passes, each led by a test. First the data side: a subscription in the service, and a repository
for the collector, until the chapter's claim holds against real zenoh. Then `sensorctl`, from the terminal in: the
line it shows, a view model that keeps the state, and the providers and the command that connect them to the core.

You run `watch` as `fvm dart run sensorctl:sensorctl watch`, and it runs until you press Ctrl-C. In the code, `watch`
is `WatchCommand`, at the top of this stack, and each layer has its section:

```
sensorctl watch                      what you type
  WatchCommand                       section 8: runs until Ctrl-C, then closes what it opened
    WatchView                        section 6: draws the one line
    WatchViewModel                   section 7: keeps the latest reading and the count
      ReadingsRepository             section 5: turns each payload into a reading
        ZenohService + Subscription  section 4: hands up each payload
```

You check the claim about a device and a laptop by running `watch` against the node, in section 9.

> **If you already know zenoh.** `watch` is `z_sub` on one key, with its subscriber behind your own service. Sections 3
> to 5 hold the zenoh code, and sections 6 to 8 build the program around it.

> **If you already know Flutter.** This chapter has no Flutter. `sensorctl` is plain Dart, with Riverpod and no
> widgets, and the app does not change.

## 2 — What to read

| | page | what to take from it |
|---|---|---|
| [1] zenoh.io | [*Abstractions*](https://zenoh.io/docs/manual/abstractions/), *Subscriber* | a subscriber registers interest in every put or delete on the keys its expression matches |
| [2] *The Zenoh Book* | [*Core Concepts → Publishers & Subscribers*](https://corsaro.me/zenoh/book/core-concepts/pub-sub/), *Subscribers* and *Sample* | the two ways to receive, a loop or a callback, and what a sample carries |
| [3] *Zenoh Programming in Rust* | [chapter 6, *Subscribers*](https://kydos.github.io/zenoh-book/chapter_06.html) | declaring a subscriber, a sample's fields, and how long a subscriber lives. Skip *Wildcard Subscriptions*, which is chapter 4 of this guide, and *Pull Model: Ring Channel* |
| `args` | [its `CommandRunner`](https://pub.dev/documentation/args/latest/command_runner/CommandRunner-class.html) | a program with commands, and the usage it prints, from section 8 |
| `dart_console` | [its documentation](https://pub.dev/packages/dart_console) | the cursor and the erase that redraw a line in place, from section 8 |

## 3 — The test of the chapter's claim

Write the test that states the chapter's claim, and watch it fail.

**1. State the claim in one sentence.** The test's name repeats it.

> A reading the phone publishes reaches the collector.

**2. Read how `z_sub` subscribes.** `watch` takes the place of the package's `z_sub`, so start from what `z_sub`
does. Open `zenoh_sensors/apps/sensorctl/example/z_sub.dart`. After it parses its arguments and opens its session, it
does four things:

- `session.declareSubscriber(keyExpr)` declares a subscriber on a key expression, and returns it at once.
- `subscriber.stream.listen(…)` receives the samples, and prints each one's kind, key and payload.
- Two handlers wait for SIGINT or SIGTERM, the signals that Ctrl-C and a plain `kill` send.
- Then it cancels the listening, and closes the subscriber and the session.

Your collector splits the same work. The service declares the subscriber and hands up each payload. A repository owns
the key and turns each payload back into a reading. `watch` waits for Ctrl-C and closes what it opened.

**3. Write the test.** The test has two ends, and both are your code. The node's end publishes one reading from a fake
sensor, through `SensorNodeRepository`. The laptop's end is the code under test: a new repository,
`ReadingsRepository`, which receives through a collector's `ZenohService`. Create
`zenoh_sensors/packages/sensor_core/test/repositories/readings_repository_test.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:test/test.dart';

import '../support/collector.dart';
import '../support/fakes.dart';

void main() {
  test('a reading the phone publishes reaches the collector', () async {
    // The node's end: its session, from the settings that listen.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);
    await sensorNode.open();

    // The laptop's end: a collector's session, from the settings that
    // connect.
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(collectorNode.dispose);
    await collectorNode.open();

    // The code to implement: a repository that receives the phone's
    // readings, collected into a list.
    final repository = ReadingsRepository(collectorNode, nodeName: 'phone');
    final received = <Reading>[];
    final listening = repository.readings().listen(received.add);
    addTearDown(listening.cancel);
    // The declaration travels to the node, so give it time to arrive.
    await Future<void>.delayed(delivery);

    // The node's code publishes one reading from a fake sensor.
    const reading = Reading(x: 0, y: 9.776, z: 0.812);
    final sensor = FakeSensorService(Stream.value(reading));
    final node = SensorNodeRepository(sensorNode, sensor, nodeName: 'phone');
    await node.publish().toList();
    // A put returns before the sample arrives, so give it time to cross.
    await Future<void>.delayed(delivery);

    // The claim: one reading arrives, with the three values it left with.
    final values = received.map((r) => [r.x, r.y, r.z]).toList();
    expect(values, [
      [0, 9.776, 0.812],
    ]);
  });
}
```

The node's session opens first, and listens. The collector's session connects to it. The repository starts receiving
when something listens to its stream, and the test collects every reading into a list.

The test waits twice. Declaring a subscription sends a declaration to the node, and a put made before it arrives can
be lost. So the test waits for the declaration to arrive before the node publishes, and then for the sample to cross.

The expectation compares the three values, because `Reading` defines no equality. The reading that arrives is a new
object, made from the text the node sent.

**4. Write just enough for it to compile.** The repository starts empty: the constructor the test calls, and a stream
with nothing in it. Create `zenoh_sensors/packages/sensor_core/lib/src/repositories/readings_repository.dart`:

```dart
import 'package:sensor_core/src/domain/reading.dart';
import 'package:sensor_core/src/services/zenoh_service.dart';

/// The collector's side of the readings: it owns the key expression and turns
/// what arrives back into readings.
class ReadingsRepository {
  /// A repository receiving, through [zenoh], the readings of the node called
  /// [nodeName].
  new(this.zenoh, {required this.nodeName});

  /// The service the readings arrive through.
  final ZenohService zenoh;

  /// The node's name, the middle segment of its key expressions.
  final String nodeName;

  /// The readings that arrive from the node.
  Stream<Reading> readings() => const Stream.empty();
}
```

The core's exports gain the new file. Replace `zenoh_sensors/packages/sensor_core/lib/sensor_core.dart`:

```dart
/// The zenoh data layer that `sensorctl` and the phone app share.
library;

export 'src/domain/reading.dart';
export 'src/repositories/readings_repository.dart';
export 'src/repositories/sensor_node_repository.dart';
export 'src/services/sensor_service.dart';
export 'src/services/session_settings.dart';
export 'src/services/zenoh_service.dart';
```

**5. Run it, and read the failure.**

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'reaches the collector'
```

```
⋮
  Expected: [[0, 9.776, 0.812]]
    Actual: []
     Which: at location [0] is [] which shorter than expected
⋮
```

No reading arrived. The sessions are real and connected, and the repository's stream is empty. **This test stays red
until section 5.** Each part of its expectation is made true by a smaller test, in the next two sections:

```
a reading the phone publishes reaches the collector
 ├─ one reading arrives         section 4: a subscription receives the text a publication puts
 │                              section 5, cycles 1 and 2: the repository subscribes on the phone's key
 └─ with the same three values  section 5, cycle 3: the text becomes a reading
```

Section 5's fourth cycle makes cancelling the stream close the subscription. `watch` needs that in section 8, and this
test does not check it.

> **In VS Code.** The Testing view lists the new file under `sensor_core`, and ▶ on the test fails it the same way.

The outer test is red. Section 4 gives the service its subscription.

## 4 — The subscriber, behind the service

Give the service a subscription, a class of its own around the package's subscriber, as `Publication` is around the
publisher. It goes in the same file.

**Cycle 1 — a subscription receives the text a publication puts.** The node's end puts through a publication. The
laptop's end is the new contract. Add a test to
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('a subscription receives the text a publication puts', () async {
    // The node's end: its session, from the settings that listen.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);
    await sensorNode.open();

    // The laptop's end: a collector's session, from the settings that
    // connect.
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(collectorNode.dispose);
    await collectorNode.open();

    // The new contract to implement: declare a subscription on a key, then
    // collect the text that arrives through it.
    final subscription = collectorNode.declareSubscription(
      'sensor/phone/accel',
    );
    final received = <String>[];
    subscription.payloads.listen(received.add);
    // The declaration travels to the node, so give it time to arrive.
    await Future<void>.delayed(delivery);

    // The node puts text through a publication.
    sensorNode
        .declarePublication('sensor/phone/accel')
        .put('0.000,9.776,0.812');
    // A put returns before the sample arrives, so give it time to cross.
    await Future<void>.delayed(delivery);

    // The claim: the same text arrives.
    expect(received, ['0.000,9.776,0.812']);
  });
}
```

The call in the test sets the shape of the subscription: `declareSubscription` with a key, then `payloads`, a stream
of strings.

Neither exists yet. Write a subscription that compiles and delivers nothing: a second class beside `Publication`, and
one method on the service that returns it. Replace
`zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

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

/// A declared subscriber on one key expression, as the service hands it out:
/// the payload of each sample, as text.
class Subscription {
  new _();

  /// The payload of every sample that arrives, as text, in order of arrival.
  Stream<String> get payloads => const Stream.empty();

  /// Undeclares the subscriber. Safe to call twice.
  void close() {}
}

/// The owner of the zenoh session. It hands upward only plain Dart values and
/// its own [Publication]s and [Subscription]s.
class ZenohService {
  /// A service for one role's [settings]. Nothing opens until [open].
  new(this.settings);

  /// Which side of the topology this session is on.
  final SessionSettings settings;

  Session? _session;
  final _publications = <Publication>[];

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

  /// Declares a publisher on [keyExpr]. The service closes it on [dispose].
  Publication declarePublication(String keyExpr) {
    final publication = Publication._(_opened.declarePublisher(keyExpr));
    _publications.add(publication);
    return publication;
  }

  /// Declares a subscriber on [keyExpr]. The service closes it on [dispose].
  Subscription declareSubscription(String keyExpr) => Subscription._();

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

The fake service in the tests says `implements ZenohService`, so it must have the new method too, or the tests that
import it do not compile. Its subscriptions record their key and whether they were closed, and a test plays payloads
into one through `arrivals`. Section 5's tests use them. Replace
`zenoh_sensors/packages/sensor_core/test/support/fakes.dart`:

```dart
import 'dart:async';

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

class FakeSubscription implements Subscription {
  new(this.keyExpr);

  final String keyExpr;
  final arrivals = StreamController<String>();
  bool isClosed = false;

  @override
  Stream<String> get payloads => arrivals.stream;

  @override
  void close() => isClosed = true;
}

class FakeZenohService implements ZenohService {
  final publications = <FakePublication>[];
  final subscriptions = <FakeSubscription>[];

  @override
  Publication declarePublication(String keyExpr) {
    final publication = FakePublication(keyExpr);
    publications.add(publication);
    return publication;
  }

  @override
  Subscription declareSubscription(String keyExpr) {
    final subscription = FakeSubscription(keyExpr);
    subscriptions.add(subscription);
    return subscription;
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

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'a subscription receives'
```

```
⋮
  Expected: ['0.000,9.776,0.812']
    Actual: []
     Which: at location [0] is [] which shorter than expected
⋮
```

Nothing arrived, because the skeleton's stream is empty.

**Write the obvious implementation.** A constant stream would pass this test, but the package's subscriber already
delivers every sample, so wrap it. Replace `zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

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

/// A declared subscriber on one key expression, as the service hands it out:
/// the payload of each sample, as text.
class Subscription {
  new _(this._subscriber);

  final Subscriber _subscriber;

  /// The payload of every sample that arrives, as text, in order of arrival.
  Stream<String> get payloads =>
      _subscriber.stream.map((sample) => sample.payload);

  /// Undeclares the subscriber. Safe to call twice.
  void close() => _subscriber.close();
}

/// The owner of the zenoh session. It hands upward only plain Dart values and
/// its own [Publication]s and [Subscription]s.
class ZenohService {
  /// A service for one role's [settings]. Nothing opens until [open].
  new(this.settings);

  /// Which side of the topology this session is on.
  final SessionSettings settings;

  Session? _session;
  final _publications = <Publication>[];
  final _subscriptions = <Subscription>[];

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

  /// Declares a publisher on [keyExpr]. The service closes it on [dispose].
  Publication declarePublication(String keyExpr) {
    final publication = Publication._(_opened.declarePublisher(keyExpr));
    _publications.add(publication);
    return publication;
  }

  /// Declares a subscriber on [keyExpr]. The service closes it on [dispose].
  Subscription declareSubscription(String keyExpr) {
    final subscription = Subscription._(_opened.declareSubscriber(keyExpr));
    _subscriptions.add(subscription);
    return subscription;
  }

  /// Closes the publications, the subscriptions and the session. Safe before
  /// [open], and more than once.
  void dispose() {
    for (final publication in _publications) {
      publication.close();
    }
    _publications.clear();
    for (final subscription in _subscriptions) {
      subscription.close();
    }
    _subscriptions.clear();
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
fvm dart test packages/sensor_core -n 'a subscription receives'
```

It passes.

**`payloads` hands up each sample's payload as text**, so nothing above the service holds a `Sample`.

**The service closes what it declared**, its subscriptions as well as its publications, in `dispose()`. Section 10
pins that, and that closing a subscription twice is safe.

**`openCollector` stays.** Three tests check what the node puts against the package's own subscriber, a witness that
shares no code with what it checks. Its doc comment said there was no collector-side service, and now there is.
Replace `zenoh_sensors/packages/sensor_core/test/support/collector.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:zenoh_dart/zenoh.dart';

/// A collector opened with the package directly, as `z_sub` is on the laptop:
/// a witness that does not go through the service it checks.
Future<Session> openCollector() {
  final config = Config();
  SessionSettings.collectorNode().asJson5.forEach(config.insertJson5);
  return Session.open(config: config);
}

/// Long enough for a sample, or a declaration, to cross the loopback, which
/// takes milliseconds.
const delivery = Duration(milliseconds: 500);
```

**What the test guarantees:** a string put on a key reaches a subscription to that key unchanged, once the
subscription's declaration has reached the node.

The service can subscribe. Section 5 builds the repository that uses it, and the outer test goes green.

## 5 — The repository, and the outer test goes green

Build the collector's repository in four cycles, and the outer test goes green. The repository owns the key
expression `sensor/<node>/accel`, built from the node's name, and turns each payload back into a reading. Each cycle
runs against the fake service.

**Cycle 1 — the collector subscribes to `sensor/phone/accel`.** Add a second test to
`zenoh_sensors/packages/sensor_core/test/repositories/readings_repository_test.dart`:

```dart
⋮

  test('the collector subscribes to sensor/phone/accel', () async {
    // A fake: a service that records what is declared on it.
    final zenoh = FakeZenohService();

    // The code to implement: listening declares the subscription.
    final repository = ReadingsRepository(zenoh, nodeName: 'phone');
    final listening = repository.readings().listen((_) {});
    addTearDown(listening.cancel);

    // The claim: one subscription, on the phone's key.
    final keys = zenoh.subscriptions.map((s) => s.keyExpr).toList();
    expect(keys, ['sensor/phone/accel']);
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'the collector subscribes'
```

```
⋮
  Expected: ['sensor/phone/accel']
    Actual: []
     Which: at location [0] is [] which shorter than expected
⋮
```

No subscription was declared, because the skeleton's `readings()` declares nothing.

**Fake it.** Declare the subscription on a constant key. Replace
`zenoh_sensors/packages/sensor_core/lib/src/repositories/readings_repository.dart`:

```dart
import 'package:sensor_core/src/domain/reading.dart';
import 'package:sensor_core/src/services/zenoh_service.dart';

/// The collector's side of the readings: it owns the key expression and turns
/// what arrives back into readings.
class ReadingsRepository {
  /// A repository receiving, through [zenoh], the readings of the node called
  /// [nodeName].
  new(this.zenoh, {required this.nodeName});

  /// The service the readings arrive through.
  final ZenohService zenoh;

  /// The node's name, the middle segment of its key expressions.
  final String nodeName;

  /// The key expression the readings arrive on.
  String get keyExpr => 'sensor/phone/accel';

  /// The readings that arrive from the node.
  Stream<Reading> readings() {
    zenoh.declareSubscription(keyExpr);
    return const Stream.empty();
  }
}
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'the collector subscribes'
```

It passes. The key is a constant, and the node's name is ignored, so the next cycle names a second node.

**Cycle 2 — a collector of the node `sim` subscribes to `sensor/sim/accel`.** Add a third test to
`zenoh_sensors/packages/sensor_core/test/repositories/readings_repository_test.dart`:

```dart
⋮

  test('a collector of the node sim subscribes to sensor/sim/accel', () async {
    // A fake: a service that records what is declared on it.
    final zenoh = FakeZenohService();

    // The code to implement: the key built from the node's name.
    final repository = ReadingsRepository(zenoh, nodeName: 'sim');
    final listening = repository.readings().listen((_) {});
    addTearDown(listening.cancel);

    // The claim: one subscription, on the sim node's key.
    final keys = zenoh.subscriptions.map((s) => s.keyExpr).toList();
    expect(keys, ['sensor/sim/accel']);
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'node sim'
```

```
⋮
  Expected: ['sensor/sim/accel']
    Actual: ['sensor/phone/accel']
     Which: at location [0] is 'sensor/phone/accel' instead of 'sensor/sim/accel'
⋮
```

The constant gives every collector the phone's key.

**Triangulate.** Build the key from the node's name, because no constant satisfies both tests. Replace
`zenoh_sensors/packages/sensor_core/lib/src/repositories/readings_repository.dart`:

```dart
import 'package:sensor_core/src/domain/reading.dart';
import 'package:sensor_core/src/services/zenoh_service.dart';

/// The collector's side of the readings: it owns the key expression and turns
/// what arrives back into readings.
class ReadingsRepository {
  /// A repository receiving, through [zenoh], the readings of the node called
  /// [nodeName].
  new(this.zenoh, {required String nodeName})
    : keyExpr = 'sensor/$nodeName/accel';

  /// The service the readings arrive through.
  final ZenohService zenoh;

  /// The key expression the readings arrive on.
  final String keyExpr;

  /// The readings that arrive from the node.
  Stream<Reading> readings() {
    zenoh.declareSubscription(keyExpr);
    return const Stream.empty();
  }
}
```

Run both:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'subscribes to'
```

Both pass.

**What the tests guarantee:** the collector's repository subscribes on `sensor/<node>/accel`, built from the name of
the node it watches.

**Cycle 3 — a payload `x,y,z` arrives as a reading.** Add a fourth test to
`zenoh_sensors/packages/sensor_core/test/repositories/readings_repository_test.dart`:

```dart
⋮

  test('a payload x,y,z arrives as a reading', () async {
    // A fake: a service whose subscription plays what the test puts into it.
    final zenoh = FakeZenohService();
    final repository = ReadingsRepository(zenoh, nodeName: 'phone');
    final received = <Reading>[];
    final listening = repository.readings().listen(received.add);
    addTearDown(listening.cancel);

    // The code to implement: each payload parsed back into a reading.
    zenoh.subscriptions.single.arrivals.add('0.000,9.776,0.812');
    await pumpEventQueue();

    // The claim: one reading, with the three values of the text.
    final values = received.map((r) => [r.x, r.y, r.z]).toList();
    expect(values, [
      [0, 9.776, 0.812],
    ]);
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'arrives as a reading'
```

```
⋮
  Expected: [[0, 9.776, 0.812]]
    Actual: []
     Which: at location [0] is [] which shorter than expected
⋮
```

Nothing arrived, because `readings()` declares the subscription and returns an empty stream.

**Write the obvious implementation.** The node puts one shape of text, `x,y,z`, so parse it. Replace
`zenoh_sensors/packages/sensor_core/lib/src/repositories/readings_repository.dart`:

```dart
import 'dart:async';

import 'package:sensor_core/src/domain/reading.dart';
import 'package:sensor_core/src/services/zenoh_service.dart';

/// The collector's side of the readings: it owns the key expression and turns
/// what arrives back into readings.
class ReadingsRepository {
  /// A repository receiving, through [zenoh], the readings of the node called
  /// [nodeName].
  new(this.zenoh, {required String nodeName})
    : keyExpr = 'sensor/$nodeName/accel';

  /// The service the readings arrive through.
  final ZenohService zenoh;

  /// The key expression the readings arrive on.
  final String keyExpr;

  /// Every reading that arrives, parsed from `x,y,z`. Listening declares the
  /// subscription.
  Stream<Reading> readings() {
    final controller = StreamController<Reading>();
    controller.onListen = () {
      zenoh
          .declareSubscription(keyExpr)
          .payloads
          .listen((text) => controller.add(_asReading(text)));
    };
    return controller.stream;
  }

  static Reading _asReading(String text) {
    final [x, y, z] = text.split(',').map(double.parse).toList();
    return Reading(x: x, y: y, z: z);
  }
}
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'arrives as a reading'
```

It passes, and so do the first two cycles' tests, `-n 'subscribes to'`. `readings()` returns a stream that does
nothing until someone listens. When someone does, it declares the subscription, and turns each payload into a reading
as it arrives.

`_asReading` reads the wire format back. It splits the text at the commas and parses each part as a number, and the
pattern `[x, y, z]` takes the three numbers. Text in another shape throws inside the repository's listener, and nothing
catches the exception. The wire format stays in the repository until chapter 6 moves it into a codec.

**What the test guarantees:** each payload `x,y,z` arrives on the repository's stream as a reading with the same three
values.

**Cycle 4 — cancelling the stream closes the subscription.** A collector that stops watching should tell zenoh, and
here it stops when the stream's listener cancels. Add a fifth test to
`zenoh_sensors/packages/sensor_core/test/repositories/readings_repository_test.dart`:

```dart
⋮

  test('cancelling the stream closes the subscription', () async {
    // A fake: a service that records what is closed, whose subscription
    // stays quiet, as a real one does between readings.
    final zenoh = FakeZenohService();
    final repository = ReadingsRepository(zenoh, nodeName: 'phone');

    // The code to implement: a cancel that closes the subscription, even
    // while nothing arrives.
    final listening = repository.readings().listen((_) {});
    await listening.cancel();

    // The claim: the subscription the repository declared is closed.
    expect(zenoh.subscriptions.single.isClosed, isTrue);
  });
}
```

Nothing arrives, so the listener cancels while the collector waits for a payload, which is when `watch` is cancelled
too. Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'closes the subscription'
```

```
⋮
  Expected: true
    Actual: <false>
⋮
```

The subscription was declared and never closed.

**Write the obvious implementation.** Give the controller a cancel callback that closes the subscription. Replace
`zenoh_sensors/packages/sensor_core/lib/src/repositories/readings_repository.dart`:

```dart
import 'dart:async';

import 'package:sensor_core/src/domain/reading.dart';
import 'package:sensor_core/src/services/zenoh_service.dart';

/// The collector's side of the readings: it owns the key expression and turns
/// what arrives back into readings.
class ReadingsRepository {
  /// A repository receiving, through [zenoh], the readings of the node called
  /// [nodeName].
  new(this.zenoh, {required String nodeName})
    : keyExpr = 'sensor/$nodeName/accel';

  /// The service the readings arrive through.
  final ZenohService zenoh;

  /// The key expression the readings arrive on.
  final String keyExpr;

  /// Every reading that arrives, parsed from `x,y,z`. Listening declares the
  /// subscription; cancelling closes it.
  Stream<Reading> readings() {
    late final Subscription subscription;
    late final StreamSubscription<String> listening;
    final controller = StreamController<Reading>();
    controller.onListen = () {
      subscription = zenoh.declareSubscription(keyExpr);
      listening = subscription.payloads.listen(
        (text) => controller.add(_asReading(text)),
      );
    };
    controller.onCancel = () async {
      await listening.cancel();
      subscription.close();
    };
    return controller.stream;
  }

  static Reading _asReading(String text) {
    final [x, y, z] = text.split(',').map(double.parse).toList();
    return Reading(x: x, y: y, z: z);
  }
}
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'closes the subscription'
```

It passes.

**What the test guarantees:** cancelling the collector's stream closes its subscription, even while nothing arrives.

**Run the outer test again.** Every part of its expectation has its code now:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'reaches the collector'
```

It passes. Then run the whole core:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

23 tests pass.

**What the outer test guarantees:** a reading the phone publishes reaches the collector's repository with the same
three values.

The claim holds on the laptop. Sections 6 to 8 build `watch` around it, and section 9 runs it against the node.

## 6 — The line `watch` shows

Build `sensorctl` from the terminal in. Start from what its user sees, and add each layer below when the one above
needs it. Make room for the program's first view and its tests:

```sh
# in zenoh_sensors
mkdir -p apps/sensorctl/lib/ui/watch apps/sensorctl/test/ui/watch
```

`sensorctl` now looks like this:

```
zenoh_sensors/apps/sensorctl/
├── bin/
├── example/
├── lib/
│   ├── config/
│   └── ui/
│       └── watch/
└── test/
    └── ui/
        └── watch/
```

`ui/watch/` holds the `watch` command, its view and its view model, and `test/` mirrors it.

**Cycle 1 — before the first reading the line says so.** The least `watch` shows is one line: the latest reading, and
how many there have been. Before the first reading arrives there is nothing to show, and the line says so. Create
`zenoh_sensors/apps/sensorctl/test/ui/watch/watch_view_test.dart`:

```dart
import 'package:sensorctl/ui/watch/watch_view.dart';
import 'package:sensorctl/ui/watch/watch_view_model.dart';
import 'package:test/test.dart';

void main() {
  test('before the first reading the line says so', () {
    // The state the program starts with: no reading yet.
    const state = WatchState();

    // The code to implement: the line for that state.
    final line = watchLine(state);

    // The claim: the line says that nothing has arrived.
    expect(line, 'no readings yet');
  });
}
```

The test needs two things that do not exist yet: the state that the line is made from, and the function that makes
it. The state belongs to the view model, so it goes in the view model's file. This test needs only a state with no
reading yet. Create `zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view_model.dart`:

```dart
/// What `watch` shows.
class WatchState {
  /// A state with no reading yet.
  const new();
}
```

Then the function, which compiles and makes nothing yet. Create
`zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view.dart`:

```dart
import 'package:sensorctl/ui/watch/watch_view_model.dart';

/// The line `watch` shows for [state].
String watchLine(WatchState state) => '';
```

Run it:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl -n 'says so'
```

```
⋮
  Expected: 'no readings yet'
    Actual: ''
⋮
```

The line is empty.

> **In VS Code.** The Testing view lists `sensorctl` beside `sensor_core` and `sensor_node`, and ▶ on the test fails it
> the same way.

**Fake it.** Return the line the test asks for. Replace `zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view.dart`:

```dart
import 'package:sensorctl/ui/watch/watch_view_model.dart';

/// The line `watch` shows for [state].
String watchLine(WatchState state) => 'no readings yet';
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl -n 'says so'
```

It passes. The line ignores the state, so the next cycle gives the state a reading.

**Cycle 2 — the line shows the latest reading and the count.** Add a second test to
`zenoh_sensors/apps/sensorctl/test/ui/watch/watch_view_test.dart`, and the import it needs at the top:

```dart
import 'package:sensor_core/sensor_core.dart';
import 'package:sensorctl/ui/watch/watch_view.dart';
import 'package:sensorctl/ui/watch/watch_view_model.dart';
import 'package:test/test.dart';

⋮

  test('the line shows the latest reading and the count', () {
    // A state with a reading and a count, made by the test.
    const state = WatchState(
      latest: Reading(x: 0.1, y: 9.776, z: 0.812),
      count: 42,
    );

    // The code to implement: the line for that state.
    final line = watchLine(state);

    // The claim: each value to three decimals, and the count.
    expect(line, 'x   0.100  y   9.776  z   0.812  42 readings');
  });
}
```

The test sets a reading and a count, which the state does not hold yet. Replace
`zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view_model.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';

/// What `watch` shows: the latest reading, and how many there were.
class WatchState {
  /// A state with [latest] as the newest reading and [count] readings so far.
  const new({this.latest, this.count = 0});

  /// The newest reading, or null before the first.
  final Reading? latest;

  /// How many readings have arrived.
  final int count;
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl -n 'latest reading and the count'
```

```
⋮
  Expected: 'x   0.100  y   9.776  z   0.812  42 readings'
    Actual: 'no readings yet'
⋮
```

The constant shows nothing of the state.

**Triangulate.** Make the line from the state, because no constant satisfies both tests. Replace
`zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view.dart`:

```dart
import 'package:sensorctl/ui/watch/watch_view_model.dart';

/// The line `watch` shows for [state]: the latest reading, each value to three
/// decimals, and the count.
String watchLine(WatchState state) => switch (state.latest) {
  null => 'no readings yet',
  final latest =>
    'x ${_fixed(latest.x)}  y ${_fixed(latest.y)}  '
        'z ${_fixed(latest.z)}  ${state.count} readings',
};

/// [value] to three decimals, seven characters wide, so that the columns stay
/// in place when a value turns negative.
String _fixed(double value) => value.toStringAsFixed(3).padLeft(7);
```

Run both:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl
```

Both pass. The `switch` takes the state's latest reading: `null` before the first, and the reading after it.

**What the tests guarantee:** the line shows the latest reading's three values to three decimals and the count, and
says so before the first reading.

The line is made from a state the test built. Section 7 gives `watch` a view model that builds the state from the
readings.

## 7 — The view model

Give `watch` a view model that keeps the state: the latest reading, and the count.

**Cycle 1 — the view model keeps the latest reading and counts them.** The view model listens to `readingsProvider`,
the program's way in to the readings. Create `zenoh_sensors/apps/sensorctl/test/ui/watch/watch_view_model_test.dart`:

```dart
import 'package:riverpod/riverpod.dart';
import 'package:sensor_core/sensor_core.dart';
import 'package:sensorctl/config/providers.dart';
import 'package:sensorctl/ui/watch/watch_view_model.dart';
import 'package:test/test.dart';

void main() {
  test('the view model keeps the latest reading and counts them', () async {
    // A fake: the readings provider, overridden with two readings, so
    // nothing below the view model is built.
    const first = Reading(x: 0, y: 9.776, z: 0.812);
    const second = Reading(x: 0, y: 0, z: 9.81);
    final container = ProviderContainer.test(
      overrides: [
        readingsProvider.overrideWith(
          (ref) => Stream.fromIterable([first, second]),
        ),
      ],
    )..listen(watchViewModelProvider, (_, _) {});

    // Let both readings flow through before reading the state.
    await pumpEventQueue();

    // The code to implement: the view model's state, built from the
    // readings it heard.
    final state = container.read(watchViewModelProvider);

    // The claim: two readings counted, and the second one kept.
    expect(state.count, 2);
    expect(state.latest, second);
  });
}
```

The test overrides `readingsProvider`, which does not exist yet. Declare it with no readings, so the test has
something to replace. Replace `zenoh_sensors/apps/sensorctl/lib/config/providers.dart`:

```dart
import 'package:riverpod/riverpod.dart';
import 'package:sensor_core/sensor_core.dart';

/// Which side of the topology this program is on: a collector.
final sessionSettingsProvider = Provider<SessionSettings>(
  (ref) => SessionSettings.collectorNode(),
);

/// The program's one zenoh service, disposed with the container.
final zenohServiceProvider = Provider<ZenohService>((ref) {
  final service = ZenohService(ref.watch(sessionSettingsProvider));
  ref.onDispose(service.dispose);
  return service;
});

/// The readings that arrive. This first version yields none.
final readingsProvider = StreamProvider<Reading>((ref) => const Stream.empty());
```

Then the view model, which holds the state it starts with and never changes it. Replace
`zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view_model.dart`:

```dart
import 'package:riverpod/riverpod.dart';
import 'package:sensor_core/sensor_core.dart';

/// What `watch` shows: the latest reading, and how many there were.
class WatchState {
  /// A state with [latest] as the newest reading and [count] readings so far.
  const new({this.latest, this.count = 0});

  /// The newest reading, or null before the first.
  final Reading? latest;

  /// How many readings have arrived.
  final int count;
}

/// The view model behind `watch`, which keeps the latest reading and counts the
/// readings.
class WatchViewModel extends Notifier<WatchState> {
  @override
  WatchState build() => const WatchState();
}

/// The view model of `watch`.
final watchViewModelProvider = NotifierProvider<WatchViewModel, WatchState>(
  WatchViewModel.new,
);
```

Run it:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl -n 'counts them'
```

```
⋮
  Expected: <2>
    Actual: <0>
⋮
```

The count is 0, because the skeleton never listens to the readings.

**Write the obvious implementation.** The view model listens to the readings and replaces its state on each one.
Replace `zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view_model.dart`:

```dart
import 'package:riverpod/riverpod.dart';
import 'package:sensor_core/sensor_core.dart';
import 'package:sensorctl/config/providers.dart';

/// What `watch` shows: the latest reading, and how many there were.
class WatchState {
  /// A state with [latest] as the newest reading and [count] readings so far.
  const new({this.latest, this.count = 0});

  /// The newest reading, or null before the first.
  final Reading? latest;

  /// How many readings have arrived.
  final int count;
}

/// The view model behind `watch`, which keeps the latest reading and counts the
/// readings.
class WatchViewModel extends Notifier<WatchState> {
  @override
  WatchState build() {
    ref.listen(readingsProvider, (_, next) {
      if (next case AsyncData(:final value)) {
        state = WatchState(latest: value, count: state.count + 1);
      }
    });
    return const WatchState();
  }
}

/// The view model of `watch`.
final watchViewModelProvider = NotifierProvider<WatchViewModel, WatchState>(
  WatchViewModel.new,
);
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl -n 'counts them'
```

It passes.

A view model's provider is declared in the view model's file, beside the class it creates. `providers.dart` holds the
data side's providers. The view model imports it to listen to the readings, so the imports run one way, from `ui/` to
`config/`:

```
ui/watch/watch_view_model.dart     WatchViewModel, watchViewModelProvider
        │ imports (to listen to readingsProvider)
        ▼
config/providers.dart              sessionSettingsProvider, zenohServiceProvider, readingsProvider
        │ imports
        ▼
sensor_core                        ZenohService, ReadingsRepository, …
```

**What the test guarantees:** the view model counts every reading that arrives, and keeps the latest.

The view model keeps the state, from readings that the test made. Section 8 connects it to the core.

## 8 — The providers, the command, and `main`

Connect the view model to the core, give `sensorctl` its first command, and run it.

**1. Wire the readings to the core.** `readingsProvider` yields nothing yet. On the collector's side, the readings
pass through two links: `ReadingsRepository` receives each payload as a reading, and `readingsProvider` brings that
stream into the program once the session is open. Replace `zenoh_sensors/apps/sensorctl/lib/config/providers.dart`:

```dart
import 'package:riverpod/riverpod.dart';
import 'package:sensor_core/sensor_core.dart';

/// Which side of the topology this program is on: a collector.
final sessionSettingsProvider = Provider<SessionSettings>(
  (ref) => SessionSettings.collectorNode(),
);

/// The program's one zenoh service, disposed with the container.
final zenohServiceProvider = Provider<ZenohService>((ref) {
  final service = ZenohService(ref.watch(sessionSettingsProvider));
  ref.onDispose(service.dispose);
  return service;
});

/// The collector's repository: the phone's readings, on `sensor/phone/accel`.
final readingsRepositoryProvider = Provider<ReadingsRepository>(
  (ref) =>
      ReadingsRepository(ref.watch(zenohServiceProvider), nodeName: 'phone'),
);

/// The session, opened once; what depends on it waits for this.
final sessionProvider = FutureProvider<void>(
  (ref) => ref.watch(zenohServiceProvider).open(),
);

/// The readings that arrive, once the session is open.
final readingsProvider = StreamProvider<Reading>((ref) async* {
  await ref.watch(sessionProvider.future);
  yield* ref.watch(readingsRepositoryProvider).readings();
});
```

The repository gets the service, and the name of the node it watches, `'phone'`. Listening to the view model starts
the chain: the session opens, the subscription is declared on `sensor/phone/accel`, and the readings arrive.

`async*` makes `readingsProvider`'s function return a stream at once and run its body when something listens. The
body waits for the session. `yield*` then passes on every reading of the repository's stream, and a pause or a cancel
of the provider's stream reaches the repository's.

**2. Add `dart_console`.** The view redraws one line in place on a terminal, which needs control of the cursor and
a way to erase the line. `dart_console` has both. Replace `zenoh_sensors/apps/sensorctl/pubspec.yaml`:

```yaml
name: sensorctl
description: Watch, query and command the zenoh sensor network from a terminal.
version: 1.0.0

environment:
  sdk: ^3.13.2

resolution: workspace

dependencies:
  args: ^2.7.0
  dart_console: ^5.1.0
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

`pub get` adds `dart_console`, and three packages that it needs.

**3. Write the view.** `watchLine` makes the text, and a class decides how the terminal shows it. On a terminal, the
line is redrawn in place. When the output goes to a file or a pipe, each line is printed once, so a program that reads
the output gets one line for each state. Replace `zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view.dart`:

```dart
import 'dart:io';

import 'package:dart_console/dart_console.dart';
import 'package:sensorctl/ui/watch/watch_view_model.dart';

/// The line `watch` shows for [state]: the latest reading, each value to three
/// decimals, and the count.
String watchLine(WatchState state) => switch (state.latest) {
  null => 'no readings yet',
  final latest =>
    'x ${_fixed(latest.x)}  y ${_fixed(latest.y)}  '
        'z ${_fixed(latest.z)}  ${state.count} readings',
};

/// [value] to three decimals, seven characters wide, so that the columns stay
/// in place when a value turns negative.
String _fixed(double value) => value.toStringAsFixed(3).padLeft(7);

/// The terminal as the view: the line for each state, redrawn in place on a
/// terminal, and printed once for each state when the output is not one.
class WatchView {
  /// A view on [console].
  new(this.console);

  /// The terminal it draws on.
  final Console console;

  /// Says how to stop, and hides the cursor on a terminal.
  void open() {
    stdout.writeln('Press Ctrl-C to stop.');
    if (console.hasTerminal) console.hideCursor();
  }

  /// Shows [state].
  void show(WatchState state) {
    if (console.hasTerminal) {
      console.eraseLine();
      stdout.write('\r${watchLine(state)}');
    } else {
      stdout.writeln(watchLine(state));
    }
  }

  /// Leaves the terminal ready for the next command: the cursor shown, on a
  /// new line.
  void close() {
    if (console.hasTerminal) {
      console.showCursor();
      stdout.writeln();
    }
  }
}
```

`console.hasTerminal` is true when the output is a terminal. `open` says how to stop, and hides the cursor so it does
not blink on the line. `eraseLine` clears the line, and the carriage return, `\r`, moves the cursor to its start, so
each state overwrites the last. `close` shows the cursor again and ends the line, so the terminal is ready for your next
command when `watch` ends.

Only `watchLine` has a test. What the class adds is the terminal itself, which you check by running it.

**4. Write the command.** `CommandRunner`, from `args`, gives a program commands, each a class with a name, a
description and a `run` method. Create `zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_command.dart`:

```dart
import 'dart:async';
import 'dart:io';

import 'package:args/command_runner.dart';
import 'package:dart_console/dart_console.dart';
import 'package:riverpod/riverpod.dart';
import 'package:sensorctl/ui/watch/watch_view.dart';
import 'package:sensorctl/ui/watch/watch_view_model.dart';

/// `sensorctl watch`: the phone's readings as they arrive, until SIGINT or
/// SIGTERM.
class WatchCommand extends Command<void> {
  @override
  String get name => 'watch';

  @override
  String get description => "Show the phone's readings as they arrive.";

  @override
  Future<void> run() async {
    final container = ProviderContainer(retry: (retryCount, error) => null);
    final view = WatchView(Console())..open();
    final stop = Completer<void>();
    final signals = [
      for (final signal in [ProcessSignal.sigint, ProcessSignal.sigterm])
        signal.watch().listen((_) {
          if (!stop.isCompleted) stop.complete();
        }),
    ];
    try {
      container.listen(
        watchViewModelProvider,
        (_, state) => view.show(state),
        fireImmediately: true,
      );
      await stop.future;
    } finally {
      for (final signal in signals) {
        await signal.cancel();
      }
      view.close();
      container.dispose();
    }
  }
}
```

`run` builds the program's provider container, and listens to the view model, which starts the chain.
`fireImmediately` shows the state the view model starts with, before any reading. `retry` turns off Riverpod's
retries of a failed provider.

Then `run` waits for SIGINT or SIGTERM, as `z_sub` does. The first one to arrive completes `stop`, and a second is
ignored. The `finally` then stops listening for signals, restores the terminal and disposes the container. A listener
left on a signal would keep the program from ending. Disposing cancels the readings and disposes the service, and
between them they close the subscription and the session.

If text that is not `x,y,z` arrives on `sensor/phone/accel`, `_asReading` throws, and nothing catches the exception.
`watch` then ends with `Unhandled exception:` and exit code 255, and the `finally` does not run, so the cursor stays
hidden. The core's tests put `before` and `after` on that key, so `watch` can end this way if it runs while they do.

**5. Make `main` run the commands.** The program's first output, its id and its peers, gives way to its commands.
Replace `zenoh_sensors/apps/sensorctl/bin/sensorctl.dart`:

```dart
import 'dart:io';

import 'package:args/command_runner.dart';
import 'package:sensor_core/sensor_core.dart';
import 'package:sensorctl/ui/watch/watch_command.dart';

Future<void> main(List<String> arguments) async {
  initZenohLogging('error');

  final runner = CommandRunner<void>(
    'sensorctl',
    'Watch, query and command the zenoh sensor network from a terminal.',
  )..addCommand(WatchCommand());
  try {
    await runner.run(arguments);
  } on UsageException catch (error) {
    stderr.writeln(error);
    exitCode = 64;
  }
}
```

`CommandRunner` takes the program's name and description, and `addCommand` gives it `watch`. `run` picks the command
from the arguments. An unknown command is a `UsageException`, which `main` prints, and the program ends with exit code
64, which `args`'s documentation gives for a usage error. The service keeps `zid` and `peerIds`, which its tests use.

**6. Run it.** With no command, `sensorctl` prints its usage:

```sh
# in zenoh_sensors
fvm dart run sensorctl:sensorctl
```

```
⋮
…Watch, query and command the zenoh sensor network from a terminal.

Usage: sensorctl <command> [arguments]

Global options:
-h, --help    Print this usage information.

Available commands:
  watch   Show the phone's readings as they arrive.

Run "sensorctl help <command>" for more information about a command.
```

Then start `watch`. Nothing listens on port 7447 yet, so no reading arrives:

```sh
# in zenoh_sensors
fvm dart run sensorctl:sensorctl watch
```

```
⋮
…Press Ctrl-C to stop.
no readings yet
```

The session opened with no peers, and zenoh keeps trying to connect in the background. `watch` waits.

**7. Run the program's tests.** Press Ctrl-C to stop `watch`. Then run the tests, from the top:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl
```

3 tests pass. They are the program's first, and none of them opens a session. From the top folder, `fvm dart analyze`
reports no issues.

> **In VS Code.** The `sensorctl` entry in `launch.json` runs the program with no arguments, which now prints the
> usage. Rename it, and give it `watch`. Replace `zenoh_sensors/.vscode/launch.json`:
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
>       "args": [
>         "-e", "tcp/127.0.0.1:7447", "--no-multicast-scouting", "--cfg", "listen/endpoints:[]",
>         "-k", "sensor/**"
>       ]
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
>       "name": "sensorctl watch",
>       "type": "dart",
>       "request": "launch",
>       "program": "${workspaceFolder}/apps/sensorctl/bin/sensorctl.dart",
>       "cwd": "${workspaceFolder}",
>       "args": ["watch"]
>     },
>     {
>       "name": "sensor_node",
>       "type": "dart",
>       "request": "launch",
>       "program": "${workspaceFolder}/apps/sensor_node/lib/main.dart",
>       "cwd": "${workspaceFolder}/apps/sensor_node"
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
> Run **sensorctl watch** from Run and Debug. Its output goes to the Debug Console, which is not a terminal, so it
> prints one line for each state. The red square on the toolbar stops it with SIGTERM, which runs the same `finally`
> as Ctrl-C.

`watch` runs, and waits for readings. Section 9 starts the node.

## 9 — On the emulator

Run the node on the emulator, and watch its readings with `sensorctl`. You need three terminals: the node in the
first, the forward and `watch` in the second, and the device's moves in the third. In the diagram, each row is one
action, in order, in the column where you take it, and the emulator's column also shows what the emulator does. `┃`
marks a terminal that a running program holds, and `◉` marks where you read the result.

```
   first terminal        second terminal        third terminal         emulator
   ────────────────────  ─────────────────────  ─────────────────────  ──────────────────
1  start the emulator                                                  boots
   cd apps/sensor_node
   flutter run                                                         runs the app
2  ┃                     adb forward
   ┃                     watch
   ┃                     ◉ the resting pose
3  ┃                     ┃                      adb emu: lay it flat   lies flat
   ┃                     ◉ z reads 9.810                               ◉ z reads 9.810
   ┃                     ┃                      adb emu: put it back   stands as before
4  ┃                     Ctrl-C
   ┃                     adb forward --remove
   q                                                                   the app stops
   cd ../..
5                                                                      you click its ×
```

**1. Start the node.** With your phone unplugged, start the emulator in the first terminal by its id, from the list
that `fvm flutter emulators` prints:

```sh
# in zenoh_sensors
fvm flutter emulators --launch <the id from the list>
```

When the device has booted, go into the app's folder and run the app:

```sh
# in zenoh_sensors
cd apps/sensor_node
```

```sh
# in zenoh_sensors/apps/sensor_node
fvm flutter run
```

The emulator shows the latest reading and the count.

**2. Forward the port, and watch.** Open a second terminal at the top folder, and forward the laptop's port 7447 to
the device's:

```sh
# in zenoh_sensors
adb forward tcp:7447 tcp:7447
```

In the same terminal, start `watch`, and leave it running. Its output below shows only the readings:

```sh
# in zenoh_sensors
fvm dart run sensorctl:sensorctl watch
```

```
⋮
x   0.000  y   9.776  z   0.812  … readings
```

The line changes in place 15 to 20 times a second. The three values stay those of the emulator's resting
pose, and the count climbs.

> **In VS Code.** Pick the emulator in the status bar, and run **sensor_node** from Run and Debug in place of
> `flutter run`. After the forward, run **sensorctl watch** from the same list in place of `watch`.

**3. Move the device.** Open a third terminal at the top folder, and lay the device flat on its back:

```sh
# in zenoh_sensors
adb emu sensor set acceleration 0:0:9.81
```

In the emulator and in the second terminal, within a second, gravity moves to `z`:

```
x   0.000  y   0.000  z   9.810  … readings
```

In the third terminal, put the device back where it started:

```sh
# in zenoh_sensors
adb emu sensor set acceleration 0:9.77631:0.812349
```

**4. Stop.** In the second terminal, press Ctrl-C to stop `watch`. The cursor comes back on a new line, and zenoh
1.8.0 prints its `ERROR` line, because the session was connected when it closed. In the same terminal, remove the
forward:

```sh
# in zenoh_sensors
adb forward --remove tcp:7447
```

In the first terminal, press `q` to stop the app, and go back to the top folder:

```sh
# in zenoh_sensors/apps/sensor_node
cd ../..
```

The emulator keeps running.

**5. Close the emulator.** Click the × in the panel on the right of the emulator's screen.

> **On a phone.** With the emulator closed, connect your phone by its cable. Follow steps 1, 2 and 4 in the first
> two terminals, without starting the emulator. In place of step 3, tilt the phone. The values are your own, and
> they change as you tilt it.

`watch` shows what the node publishes. Section 10 pins two rules.

## 10 — What the tests pin, and what changed in the architecture

Pin two rules with tests, and look at what the chapter built.

**1. Add two more tests.** They pass at once, because they pin what the code already does. Both are rules of your own.
Add them to `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('disposing the service closes its subscriptions', () async {
    // A rule of the pattern: dispose closes what the service declared.
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    await collectorNode.open();
    final subscription = collectorNode.declareSubscription(
      'sensor/phone/accel',
    );

    collectorNode.dispose();

    // The claim: the subscription's stream ends, because the subscriber
    // underneath is closed.
    expect(subscription.payloads, emitsDone);
  });

  test('closing a subscription twice is safe', () async {
    // A rule of the pattern, which the repository's cancel and the
    // service's dispose both rely on.
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(collectorNode.dispose);
    await collectorNode.open();
    final subscription = collectorNode.declareSubscription('sensor/phone/accel')
      ..close();

    // The claim: the second close returns normally.
    expect(subscription.close, returnsNormally);
  });
}
```

Run the whole core:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

25 tests pass.

**What the tests guarantee:** `dispose()` ends the stream of every subscription the service declared, and a second
`close()` is safe. `watch` relies on both, because disposing its container closes the subscription twice, from the
repository's cancel and from the service's `dispose()`.

**2. Look at what changed in the architecture.** `sensorctl` has every layer of the pattern but the codec, each with
the least that makes the claim true:

```
WatchView  →  WatchViewModel  →  ReadingsRepository  →  ZenohService + Subscription  →  zenoh_dart
  view          view model          repository             service                       package
```

Two rules start here.

**Each command draws its view the same way.** A function makes the text from the state, and the view shows it in
place on a terminal, and line by line anywhere else.

**Each command owns its container.** It builds its providers when it starts and disposes them when it stops, which
closes what each layer opened.

**What comes next.** Chapter 4 adds `simulate`, a sensor node inside `sensorctl`, for when you would rather not start a
device. It also lets `watch` take a key expression, so that it can follow two nodes at once. Chapter 5 takes the phone
off the cable and onto your Wi-Fi.

The chapter's code is done. Section 11 commits it.

## 11 — Files and versions at the end of this chapter

**1. Commit.** Commit everything this chapter made, in one commit:

```sh
# in zenoh_sensors
git add .
git commit -m "The collector on your laptop: sensorctl watch, and readings from sensor/phone/accel"
```

The commit holds 16 files, 7 of them new, and 17 with `.vscode/launch.json`:

- 8 are `sensorctl`'s: the 5 new files under `lib/ui/watch/` and `test/ui/watch/`, and `main`, the providers and
  `pubspec.yaml`, changed.
- 7 are the core's: the repository and its test, new, and 5 changed.
- The last is `pubspec.lock`, at the top.

> **In VS Code.** **View › Source Control** lists the same changes. Stage them with the **+** on the **Changes** line,
> type the message, and choose **Commit**.

`zenoh_sensors` now holds this, in five commits:

```
zenoh_sensors/
├── .dart_tool/
├── .fvm/
├── .fvmrc
├── .git/
├── .gitignore
├── .vscode/                        if you use VS Code
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
│       │   ├── config/
│       │   │   └── providers.dart
│       │   └── ui/
│       │       └── watch/
│       │           ├── watch_command.dart
│       │           ├── watch_view.dart
│       │           └── watch_view_model.dart
│       ├── pubspec.yaml
│       ├── README.md
│       └── test/
│           └── ui/
│               └── watch/
│                   ├── watch_view_model_test.dart
│                   └── watch_view_test.dart
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
│       │       │   ├── readings_repository.dart
│       │       │   └── sensor_node_repository.dart
│       │       └── services/
│       │           ├── sensor_service.dart
│       │           ├── session_settings.dart
│       │           └── zenoh_service.dart
│       ├── pubspec.yaml
│       ├── README.md
│       └── test/
│           ├── repositories/
│           │   ├── readings_repository_test.dart
│           │   └── sensor_node_repository_test.dart
│           ├── services/
│           │   └── zenoh_service_test.dart
│           └── support/
│               ├── collector.dart
│               └── fakes.dart
├── pubspec.lock
└── pubspec.yaml
```

One package is new in this chapter. Two others resolved to newer versions:

| what | version |
|---|---|
| `dart_console` | 5.1.0 |
| Flutter, and the Dart it carries | 3.47.6, Dart 3.13.5 |
| `sensors_plus` | 7.1.1 |

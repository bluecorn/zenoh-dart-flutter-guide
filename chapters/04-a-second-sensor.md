# 4 — A second sensor

## 1 — What you build, and what you will see

By the end of this chapter, the node publishes a second sensor. The phone's gyroscope goes out on `sensor/phone/gyro`,
beside the accelerometer on `sensor/phone/accel`. `watch` asks for both with one key expression, `sensor/phone/*`, and
shows a line for each key:

```
Press Ctrl-C to stop.
sensor/phone/accel  x   0.000  y   9.776  z   0.812  153 readings
sensor/phone/gyro   x   0.000  y   0.000  z   0.000  51 readings
```

Each reading now travels with its key, from the put on the node to the line on your laptop:

```
 the emulator or a phone                      the laptop
┌─────────────────────────────────┐          ┌──────────────────────────────────┐
│ sensor_node                     │          │ sensorctl watch                  │
│                                 │          │                                  │
│ SensorNodeRepository            │          │ WatchView, a line for each key   │
│   │ each sensor on its own key  │          │   ▲                              │
│   ▼                             │          │ WatchViewModel                   │
│ put on sensor/phone/accel ══╗   │   adb    │   ▲ each reading with its key    │
│ put on sensor/phone/gyro  ══╬═══╪══════════╪═► ReadingsRepository             │
│                                 │ forward  │     subscribed to sensor/phone/* │
└─────────────────────────────────┘          └──────────────────────────────────┘
```

You build it in two passes, each led by a test. First the data side, until the chapter's claim holds against real
zenoh. The node's repository publishes each sensor on its own key, the service hands up the key of each sample, and
the collector's repository subscribes to every sensor of its node. Then each program from its view in: the phone's
screen, and the lines `watch` shows.

You run the node with `fvm flutter run` in `apps/sensor_node`, and `watch` with `fvm dart run sensorctl:sensorctl
watch`. Each layer that changes has its section:

```
sensor_node                          the app on the device
  NodeScreen                         section 8: a block for each key
  NodeViewModel                      section 8: the latest reading and the count, for each key
    SensorNodeRepository             section 5: publishes each sensor on its own key, and hands each on with it
      DeviceSensorService            section 8: reads the gyroscope

sensorctl watch                      what you type
  WatchView                          section 9: a line for each key
  WatchViewModel                     section 9: the latest reading and the count, for each key
    ReadingsRepository               section 7: subscribes to every sensor of its node
      ZenohService + Subscription    section 6: hands up the key of each sample
```

You check the claim about a device and a laptop by running both programs, in section 10.

> **Zenoh guidance.** The chapter uses one wildcard, `*`, in a subscriber's key expression. Section 3
> shows what it matches, with the package's examples. Sections 4 to 7 hold the zenoh code.

> **Flutter guidance.** The app changes in section 8: a second stream from `sensors_plus`, and a state that
> holds the latest reading and the count for each key.

## 2 — What to read

| | page | what to take from it |
|---|---|---|
| [1] zenoh.io | [*Abstractions*](https://zenoh.io/docs/manual/abstractions/), *Key Expression* | the three wildcards, and the canon form, the one spelling zenoh allows for each set of keys. It calls a segment of a key a chunk |
| [2] *The Zenoh Book* | [*Core Concepts → Key Expressions*](https://corsaro.me/zenoh/book/core-concepts/key-expressions/) | intersection and inclusion, the two tests on key expressions that match subscriptions with publications |
| [3] *Zenoh Programming in Rust* | [chapter 3, *The Zenoh Data Model*](https://kydos.github.io/zenoh-book/chapter_03.html), *Key Expressions* | each wildcard with keys it matches and keys it does not. It says zenoh puts a key expression into canon form for you. `zenoh_dart` refuses one that is not in canon form, and its `KeyExpr.canonize` rewrites one |
| [3] *Zenoh Programming in Rust* | [chapter 6, *Subscribers*](https://kydos.github.io/zenoh-book/chapter_06.html), *Wildcard Subscriptions* | a sample carries the key it was put on, never the subscriber's expression. Its `+` is not a wildcard in zenoh, and zenoh.io [1] lists the three there are |

## 3 — A wildcard, with `z_sub` and `z_put`

Run the package's examples with a wildcard before you write one. `z_sub` subscribes to any key expression, and `z_put`
puts on any key.

**1. Start the subscriber.** In a terminal at the top folder, start `z_sub` on `demo/phone/*`, listening on the
loopback, and leave it running:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_sub.dart \
  -l tcp/127.0.0.1:7447 --no-multicast-scouting -k 'demo/phone/*'
```

```
⋮
…Opening session...
Declaring Subscriber on 'demo/phone/*'...
Press CTRL-C to quit...
```

The quotes keep your shell from reading the `*` as a pattern for file names.

**2. Put on three keys.** Open a second terminal at the top folder, and put a value on `demo/phone/accel`:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_put.dart \
  -e tcp/127.0.0.1:7447 --no-multicast-scouting --cfg 'listen/endpoints:[]' \
  -k demo/phone/accel -p '0.000,9.776,0.812'
```

```
⋮
…Opening session...
Putting Data ('demo/phone/accel': '0.000,9.776,0.812')...
… ERROR ThreadId(…) zenoh::api::admin: Unable to publish transport event: session closed
```

Then put a value on `demo/phone/gyro`:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_put.dart \
  -e tcp/127.0.0.1:7447 --no-multicast-scouting --cfg 'listen/endpoints:[]' \
  -k demo/phone/gyro -p '0.000,0.000,0.500'
```

Then put a value on a key one segment longer, `demo/phone/gyro/raw`:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_put.dart \
  -e tcp/127.0.0.1:7447 --no-multicast-scouting --cfg 'listen/endpoints:[]' \
  -k demo/phone/gyro/raw -p 'one segment too many'
```

**3. Read what arrived.** In the first terminal, `z_sub` printed two lines:

```
>> [Subscriber] Received PUT ('demo/phone/accel': '0.000,9.776,0.812')
>> [Subscriber] Received PUT ('demo/phone/gyro': '0.000,0.000,0.500')
```

**`*` stands for one segment of a key.** The segments of a key are the parts between its slashes. `demo/phone/*`
matches every key of three segments that starts with `demo/phone`. `demo/phone/gyro/raw` has four segments, so the
third put did not arrive.

**Each sample carries its own key.** The subscriber named a set of keys, and `z_sub` printed the key that each value
was put on.

**Zenoh has two more wildcards.** `**` stands for any number of segments, so all three puts match `demo/phone/**`.
`$*` stands for part of a segment, as in `demo/ph$*/accel`. The page on zenoh.io [1] advises keys that do not need
`$*`, because matching it is slower.

**4. Stop the subscriber** with Ctrl-C in the first terminal.

The node needs one key for each sensor, and the collector one `*` to receive them all. Section 4 writes that as a test.

## 4 — The chapter's claim, as a test

Write the chapter's claim as one test, and leave it red. Sections 5 to 7 make it pass, one part each.

**1. State the claim in one sentence.** The test's name repeats it.

> The collector receives each of the phone's sensors under its own key.

**2. Write the test.** The test opens both ends on real zenoh, the node's session and a collector's session, in one
process on the loopback. The node's end publishes one reading of each sensor from a fake sensor, and the laptop's
end receives them through the collector's repository. The test goes at the end of the collector repository's test
file, after the five tests already there. Add a test to
`zenoh_sensors/packages/sensor_core/test/repositories/readings_repository_test.dart`:

```dart
⋮

  test(
    "the collector receives each of the phone's sensors under its own key",
    () async {
      // The node's end: its session, from the settings that listen.
      final sensorNode = ZenohService(SessionSettings.sensorNode());
      addTearDown(sensorNode.dispose);
      await sensorNode.open();

      // The laptop's end: a collector's session, from the settings that
      // connect.
      final collectorNode = ZenohService(SessionSettings.collectorNode());
      addTearDown(collectorNode.dispose);
      await collectorNode.open();

      // The code to implement: a repository that receives every sensor of the
      // phone, each reading with the key it arrived on, collected into a list.
      final repository = ReadingsRepository(collectorNode, nodeName: 'phone');
      final received = <KeyedReading>[];
      final listening = repository.readings().listen(received.add);
      addTearDown(listening.cancel);
      // The declaration travels to the node, so give it time to arrive.
      await Future<void>.delayed(delivery);

      // The node's code publishes one reading of each sensor from a fake
      // sensor.
      const accel = Reading(x: 0, y: 9.776, z: 0.812);
      const gyro = Reading(x: 0, y: 0, z: 0.5);
      final sensor = FakeSensorService(
        Stream.value(accel),
        gyroscope: Stream.value(gyro),
      );
      final node = SensorNodeRepository(sensorNode, sensor, nodeName: 'phone');
      await node.publish().toList();
      // A put returns before the sample arrives, so give it time to cross.
      await Future<void>.delayed(delivery);

      // The claim: each sensor's reading arrives under that sensor's key.
      final byKey = {
        for (final (:keyExpr, :reading) in received)
          keyExpr: [reading.x, reading.y, reading.z],
      };
      expect(byKey, {
        'sensor/phone/accel': [0, 9.776, 0.812],
        'sensor/phone/gyro': [0, 0, 0.5],
      });
    },
  );
}
```

The claim is a map from each key to the values that arrived on it, so the test says nothing about the order in which
the two samples arrive. The two samples are put on different keys, and the collector tells them apart by the key.

The test asks for two things that do not exist yet:

- `KeyedReading`, the type the repository hands on, a reading and the key it arrived on.
- A gyroscope on `FakeSensorService`, beside its accelerometer.

The test does not compile until steps 3 and 4 define them, with values that leave the claim false. Steps 5 to 7
carry the pair's type through the repository, the provider of `watch` and the two tests that read the stream.

**3. Name the pair.** `KeyedReading` is a domain type, and it goes beside `Reading`, after it, because it names it.
Add it to `zenoh_sensors/packages/sensor_core/lib/src/domain/reading.dart`:

```dart
⋮

/// A reading and the key expression it travels on.
typedef KeyedReading = ({String keyExpr, Reading reading});
```

> **Dart guidance.** `({String keyExpr, Reading reading})` is a record type, a value with two named fields and no
> class. `(keyExpr: 'sensor/phone/accel', reading: reading)` makes one, `.keyExpr` reads one field, and
> `final (:keyExpr, :reading) = pair` takes both out at once, as the test's loop does. The `typedef` gives the type
> a name.

**Numbered lines.** A listing of numbered lines gives the lines to change, each by its number in the file after the
change. Work from the top. Replace the line that has that number. Insert a line marked `+` so that it gets that
number. Remove a line marked `-`, which is shown as it is now. A `⋮` between two numbers stands for the lines the
listing does not show.

**4. Give the fake a gyroscope.** `FakeSensorService` was built for one sensor. The change here is the smallest that
lets the test compile with two sensors. The gyroscope is a named parameter, with an empty stream for its default, so
the six tests that give it only an accelerometer stay as they are. The change serves this test, and the refactor that
follows makes the fake fulfill the contract of a service for two sensors. Change
`zenoh_sensors/packages/sensor_core/test/support/fakes.dart`:

```dart
   6     new(this.readings, {this.gyroscope = const Stream.empty()});
   ⋮
   9+    final Stream<Reading> gyroscope;
```

`SensorService` does not change. The test hands the fake a gyroscope, and nothing reads it yet. The test in section 5
that puts a gyroscope reading on its key writes the method that reads it.

**5. Hand each reading on with a key.** The repository's stream carries a `KeyedReading` now. The key it hands on is
`''`, because the subscription hands up only the payload. Change
`zenoh_sensors/packages/sensor_core/lib/src/repositories/readings_repository.dart`:

```dart
  20     /// Every reading that arrives, parsed from `x,y,z`, with the key it arrived
  21     /// on. Listening declares the subscription; cancelling closes it.
  22     Stream<KeyedReading> readings() {
   ⋮
  25       final controller = StreamController<KeyedReading>();
   ⋮
  29           (text) => controller.add((keyExpr: '', reading: _asReading(text))),
```

**6. Keep `watch` compiling.** The readings provider of `watch` yields a `Reading` for its view model. It takes the
reading out of each pair, until section 9 shows the key. Change
`zenoh_sensors/apps/sensorctl/lib/config/providers.dart`:

```dart
  27   /// The readings that arrive, once the session is open, without their keys.
   ⋮
  30     yield* ref
  31+        .watch(readingsRepositoryProvider)
  32+        .readings()
  33+        .map((keyed) => keyed.reading);
```

**7. Refactor two tests for the stream's new type.** The two tests that collect what the repository hands on
declared a list of `Reading`. They collect a `KeyedReading` each now, and read the three values from the pair's
reading. This refactors the two tests, and their claims do not change:

1. The first test, `a reading the phone publishes reaches the collector`, at lines 23 and 38.
2. The fourth test, `a payload x,y,z arrives as a reading`, at lines 78 and 87.

Change `zenoh_sensors/packages/sensor_core/test/repositories/readings_repository_test.dart`:

```dart
  23       final received = <KeyedReading>[];
   ⋮
  38       final values = received
  39+          .map((r) => [r.reading.x, r.reading.y, r.reading.z])
  40+          .toList();
   ⋮
  78       final received = <KeyedReading>[];
   ⋮
  87       final values = received
  88+          .map((r) => [r.reading.x, r.reading.y, r.reading.z])
  89+          .toList();
```

**8. Run it, and read the failure.**

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'under its own key'
```

```
⋮
  Expected: {'sensor/phone/accel': [0, 9.776, 0.812], 'sensor/phone/gyro': [0, 0, 0.5]}
    Actual: {'': [0.0, 9.776, 0.812]}
     Which: has different length and is missing map key 'sensor/phone/accel'
⋮
```

The accelerometer's reading arrived with no key, and the gyroscope's did not arrive, because the node reads only the
accelerometer. Then run the file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core/test/repositories/readings_repository_test.dart
```

```
⋮
  Expected: {'sensor/phone/accel': [0, 9.776, 0.812], 'sensor/phone/gyro': [0, 0, 0.5]}
    Actual: {'': [0.0, 9.776, 0.812]}
     Which: has different length and is missing map key 'sensor/phone/accel'
⋮
```

5 tests pass, and the outer test fails on the same assertion. **This test stays red until section 7.** Each part of
its expectation is made true by a smaller test, in the next three sections:

```
the collector receives each of the phone's sensors under its own key
 ├─ the accelerometer's reading    section 6: the service's subscription hands up the key of each sample, in two
 │  under sensor/phone/accel                  cycles, the second subscribing with the wildcard
 │                                 section 7: the repository hands each reading on with the key the subscription
 │                                            gave it, in the cycle that reads the key from a fake subscription
 └─ the gyroscope's reading        section 5, cycle 1: a second publication, on sensor/phone/gyro
    under sensor/phone/gyro        section 5, cycle 2: the gyroscope read and put through it, which writes
                                              gyroscope() on SensorService
                                   section 7: the collector's repository subscribes to sensor/phone/*, in the
                                              cycle that checks its key expression
```

Section 5's third cycle hands each reading on with its key. The node's screen needs that in section 8, and this test
does not check it. Section 8 reads the device's gyroscope through `sensors_plus`, and this test does not check that
either, because its sensor is a fake.

The outer test is red. Section 5 makes the node publish both sensors.

## 5 — The node's repository, a key for each sensor

Build the node's repository in three cycles. It owns two key expressions now, `sensor/<node>/accel` and
`sensor/<node>/gyro`, both built from the node's name. It reads each sensor, puts each reading through that sensor's
publication as text, and hands each reading on with its key, for the screen.

These are layer tests, each against a fake service. The chapter's outer test runs the repository against real zenoh,
and you end by running it.

You write three tests:

- the phone publishes on `sensor/phone/accel` and `sensor/phone/gyro`
- a gyroscope reading is put on `sensor/phone/gyro` and handed on
- each reading is handed on with its key

> **Zenoh guidance.** A publisher is declared on one key expression, and every put through it goes on that key. A
> node with two sensors on two keys declares two publishers on its one session.

**Listings with `⋮`.** A declaration or a test shown in full takes the place of the one with that name, or is new.
Its neighbors are shown closed, their first line, an indented `⋮` and their last line, and they stay as they are.
Two neighbors with nothing shown between them say that what stood there is gone. Every line shown is a line of the
file.

**Cycle 1 — the phone publishes on `sensor/phone/accel` and `sensor/phone/gyro`.** The test that said the phone
publishes on one key made a narrower claim than this one. Two changes:

1. Remove the test `the phone publishes on sensor/phone/accel`, the second in the file. The first test and the
   `sim` test are shown closed with nothing between them, where it stood.
2. Add this claim's test at the end of the file.

Change `zenoh_sensors/packages/sensor_core/test/repositories/sensor_node_repository_test.dart`:

```dart
⋮

  test('a sensor reading reaches a subscriber on sensor/phone/accel', () async {
    ⋮
  });

  test('a node named sim publishes on sensor/sim/accel', () async {
    ⋮
  });

⋮

  test(
    'the phone publishes on sensor/phone/accel and sensor/phone/gyro',
    () async {
      // Fakes: a service that records what is declared on it, and a
      // sensor with nothing to deliver.
      final zenoh = FakeZenohService();
      final sensor = FakeSensorService(const Stream.empty());

      // The code to implement: the repository declares one publication for
      // each sensor.
      final repository = SensorNodeRepository(zenoh, sensor, nodeName: 'phone');
      await repository.publish().toList();

      // The claim: two publications, one on each of the phone's keys.
      final keys = zenoh.publications.map((p) => p.keyExpr).toList();
      expect(keys, ['sensor/phone/accel', 'sensor/phone/gyro']);
    },
  );
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'and sensor/phone/gyro'
```

```
⋮
  Expected: ['sensor/phone/accel', 'sensor/phone/gyro']
    Actual: ['sensor/phone/accel']
     Which: at location [1] is ['sensor/phone/accel'] which shorter than expected
⋮
```

One publication was declared, on the accelerometer's key.

**Write the obvious implementation.** The gyroscope's key is built from the node's name as the accelerometer's is, so
a second key and a second publication are a known step. Three changes:

1. A second key expression, `gyroKeyExpr`, built from the node's name beside `keyExpr`. The class comment says key
   expressions.
2. A second publication, `gyro`, declared when the first is.
3. The second publication closed when the first is, in `onCancel`.

Change `zenoh_sensors/packages/sensor_core/lib/src/repositories/sensor_node_repository.dart`:

```dart
   7   /// The node's side of the readings: it owns the key expressions and publishes
   8   /// what the sensors deliver.
   ⋮
  11     /// expressions of the node called [nodeName].
   ⋮
  13       : keyExpr = 'sensor/$nodeName/accel',
  14+        gyroKeyExpr = 'sensor/$nodeName/gyro';
   ⋮
  22     /// The key expression the accelerometer's readings are published on.
   ⋮
  25+    /// The key expression the gyroscope's readings are published on.
  26+    final String gyroKeyExpr;
  27+
   ⋮
  29     /// Listening declares the publications; cancelling closes them.
   ⋮
  31       late final Publication accel;
  32+      late final Publication gyro;
   ⋮
  36         accel = zenoh.declarePublication(keyExpr);
  37+        gyro = zenoh.declarePublication(gyroKeyExpr);
   ⋮
  39           accel.put(_asText(reading));
   ⋮
  45         accel.close();
  46+        gyro.close();
```

Run it again. It passes. Then run the file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core/test/repositories/sensor_node_repository_test.dart
```

```
⋮
  Expected: ['sensor/sim/accel']
    Actual: ['sensor/sim/accel', 'sensor/sim/gyro']
     Which: at location [1] is ['sensor/sim/accel', 'sensor/sim/gyro'] which longer than expected
⋮
```

3 tests fail. The `sim` test expected one key and got two. The put test and the cancel test take `.single` of the
publications, and there are two, so each ends on `Bad state: Too many elements` before its claim.

The `sim` test's claim is narrower too, one key where there are two, and the put test and the cancel test say what
holds for two publications. Four changes, in two listings:

1. Remove the test `a node named sim publishes on sensor/sim/accel`, the second in the file. The first test and
   the put test are shown closed with nothing between them, where it stood.
2. Add the test `a node named sim publishes on sensor/sim/accel and sensor/sim/gyro` at the end of the file, after
   the phone's two-key test.

Change `zenoh_sensors/packages/sensor_core/test/repositories/sensor_node_repository_test.dart`:

```dart
⋮

  test('a sensor reading reaches a subscriber on sensor/phone/accel', () async {
    ⋮
  });

  test('a reading is put as x,y,z to three decimals and handed on', () async {
    ⋮
  });

⋮

  test(
    'a node named sim publishes on sensor/sim/accel and sensor/sim/gyro',
    () async {
      // Fakes: a service that records what is declared on it, and a
      // sensor with nothing to deliver.
      final zenoh = FakeZenohService();
      final sensor = FakeSensorService(const Stream.empty());

      // The code to implement: both keys built from the node's name.
      final repository = SensorNodeRepository(zenoh, sensor, nodeName: 'sim');
      await repository.publish().toList();

      // The claim: two publications, one on each of the sim node's keys.
      final keys = zenoh.publications.map((p) => p.keyExpr).toList();
      expect(keys, ['sensor/sim/accel', 'sensor/sim/gyro']);
    },
  );
}
```

3. Change the put test, `a reading is put as x,y,z to three decimals and handed on`, in place. It reads the puts
   by key, and finds nothing on the gyroscope's.
4. Change the cancel test, `cancelling the stream closes the publication`, in place. It expects both publications
   closed.

Change `zenoh_sensors/packages/sensor_core/test/repositories/sensor_node_repository_test.dart`:

```dart
  57       // The claim: the text put through the accelerometer's publication, nothing
  58       // through the gyroscope's, and the same reading handed on.
  59       final puts = {for (final p in zenoh.publications) p.keyExpr: p.puts};
  60+      expect(puts, {
  61+        'sensor/phone/accel': ['0.000,9.776,0.812'],
  62+        'sensor/phone/gyro': <String>[],
  63+      });
   ⋮
  75       // The code to implement: a cancel that closes the publications, even
   ⋮
  80       // The claim: both publications the repository declared are closed.
  81       final closed = zenoh.publications.map((p) => p.isClosed).toList();
  82+      expect(closed, [true, true]);
```

Run the file again. 5 tests pass.

**Refactor.** `keyExpr` names one key of two, so it becomes `accelKeyExpr`. Nothing outside the repository reads it.
Change `zenoh_sensors/packages/sensor_core/lib/src/repositories/sensor_node_repository.dart`:

```dart
  13       : accelKeyExpr = 'sensor/$nodeName/accel',
   ⋮
  23     final String accelKeyExpr;
   ⋮
  36         accel = zenoh.declarePublication(accelKeyExpr);
```

The tests pass before and after.

**What the test guarantees:** a repository declares one publication for each of the node's two sensors, on
`sensor/<node>/accel` and `sensor/<node>/gyro`, both built from the node's name.

**Cycle 2 — a gyroscope reading is put on `sensor/phone/gyro` and handed on.** The test gives the fake a gyroscope
that delivers one reading and an accelerometer that delivers nothing. Add it to
`zenoh_sensors/packages/sensor_core/test/repositories/sensor_node_repository_test.dart`:

```dart
⋮

  test(
    'a gyroscope reading is put on sensor/phone/gyro and handed on',
    () async {
      // Fakes: a service that records what is put through it, and a sensor
      // whose gyroscope delivers one reading while its accelerometer is quiet.
      final zenoh = FakeZenohService();
      const reading = Reading(x: 0, y: 0, z: 0.5);
      final sensor = FakeSensorService(
        const Stream.empty(),
        gyroscope: Stream.value(reading),
      );

      // The new contract to implement: the gyroscope read through the sensor
      // service. The code to implement: each of its readings put as text on
      // the gyroscope's key, then handed on.
      final repository = SensorNodeRepository(zenoh, sensor, nodeName: 'phone');
      final published = await repository.publish().toList();

      // The claim: the text put through the gyroscope's publication, nothing
      // through the accelerometer's, and the same reading handed on.
      final puts = {for (final p in zenoh.publications) p.keyExpr: p.puts};
      expect(puts, {
        'sensor/phone/accel': <String>[],
        'sensor/phone/gyro': ['0.000,0.000,0.500'],
      });
      expect(published, [reading]);
    },
  );
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'gyroscope reading'
```

```
⋮
  Expected: {'sensor/phone/accel': [], 'sensor/phone/gyro': ['0.000,0.000,0.500']}
    Actual: {'sensor/phone/accel': [], 'sensor/phone/gyro': []}
     Which: at location ['sensor/phone/gyro'][0] is [] which shorter than expected
⋮
```

Nothing was put on the gyroscope's key, because the repository reads only the accelerometer.

**Write the obvious implementation.** The gyroscope is read the way the accelerometer is, so the step is known. The
contract needs a gyroscope, and five files change.

`Reading` is a reading of either sensor now, so its comment says so. Change
`zenoh_sensors/packages/sensor_core/lib/src/domain/reading.dart`:

```dart
   1   /// One reading of a sensor: its value on each of the device's three axes.
   2-  /// gravity included.
   ⋮
   6     /// On the device's x axis, to the right.
   ⋮
   9     /// On the device's y axis, towards the top of the screen.
   ⋮
  12     /// On the device's z axis, out of the screen.
```

The new contract is `gyroscope()` beside `accelerometer()`. Two changes:

1. The class comment describes the gyroscope's readings with the accelerometer's.
2. `gyroscope()`, after `accelerometer()`.

Change `zenoh_sensors/packages/sensor_core/lib/src/services/sensor_service.dart`:

```dart
   7   /// gyroscope reports radians per second about the same three axes, so a device
   8   /// at rest reads 0 on each. The axes are the device's own, with the screen in
   9+  /// its natural orientation: x to the right, y towards the top, z out of the
  10+  /// screen.
   ⋮
  14+
  15+    /// The gyroscope's readings, as the device delivers them.
  16+    Stream<Reading> gyroscope();
```

The fake implements it with the stream the test hands in. Its field is private, because the public name `gyroscope`
is the method's. Two changes:

1. The constructor's parameter and the field become `_gyroscope`.
2. `gyroscope()` returns the field.

Change `zenoh_sensors/packages/sensor_core/test/support/fakes.dart`:

```dart
   6     new(this.readings, {this._gyroscope = const Stream.empty()});
   ⋮
   9     final Stream<Reading> _gyroscope;
   ⋮
  13+
  14+    @override
  15+    Stream<Reading> gyroscope() => _gyroscope;
```

> **Dart guidance.** `{this._gyroscope = const Stream.empty()}` is an initializing formal for a private field. A
> caller names the parameter `gyroscope`, without the underscore, and the value lands in `_gyroscope`.

The device service implements it too, so that the app compiles. It answers with an empty stream until section 8
reads the gyroscope through `sensors_plus`. Change
`zenoh_sensors/apps/sensor_node/lib/data/services/device_sensor_service.dart`:

```dart
  20+
  21+    @override
  22+    Stream<Reading> gyroscope() => const Stream.empty();
```

The repository reads the gyroscope, puts each reading on the gyroscope's key, and hands it on. Its stream ends when
both sensors have ended, because the test's accelerometer ends at once and the gyroscope's reading arrives after
that. A stream that ended with the accelerometer would take no reading from the gyroscope. Four changes:

1. The comment of `publish` says each sensor's key, and when the stream ends.
2. A subscription to the gyroscope beside the accelerometer's, `gyroReadings`, which puts each reading through
   `gyro` and hands it on, and is cancelled with the accelerometer's.
3. A count of the sensors still delivering, `delivering`, starting at 2.
4. A local function, `ended`, which each sensor's `onDone` calls, and which closes the stream when the count
   reaches 0.

Change `zenoh_sensors/packages/sensor_core/lib/src/repositories/sensor_node_repository.dart`:

```dart
  28     /// Publishes every reading of each sensor as `x,y,z` to three decimals, on
  29     /// that sensor's key, and hands each on. Listening declares the
  30+    /// publications; cancelling closes them. The stream ends when both sensors
  31+    /// have ended.
   ⋮
  35       late final StreamSubscription<Reading> accelReadings;
  36+      late final StreamSubscription<Reading> gyroReadings;
  37+      var delivering = 2;
   ⋮
  39+      void ended() {
  40+        delivering -= 1;
  41+        if (delivering == 0) {
  42+          unawaited(controller.close());
  43+        }
  44+      }
  45+
   ⋮
  49         accelReadings = sensor.accelerometer().listen((reading) {
   ⋮
  52         }, onDone: ended);
  53+        gyroReadings = sensor.gyroscope().listen((reading) {
  54+          gyro.put(_asText(reading));
  55+          controller.add(reading);
  56+        }, onDone: ended);
   ⋮
  59         await accelReadings.cancel();
  60+        await gyroReadings.cancel();
```

`close()` returns a future, and `unawaited` marks it as one that nothing waits for, which the lint set asks for
where a future is dropped.

Run it again. It passes. Then run the file. 6 tests pass.

**Refactor.** The two listen blocks are alike line for line. Three changes:

1. One local function, `publishOn`, declares the publication for a key, listens to a sensor's readings, and
   counts the sensor as delivering. It is called once for each sensor, and `ended` goes into it.
2. The publications and the subscriptions go into two lists, which the cancel walks.
3. The two `late final` publications and subscriptions go, and the count starts at 0.

Change `zenoh_sensors/packages/sensor_core/lib/src/repositories/sensor_node_repository.dart`:

```dart
  33       final publications = <Publication>[];
  34       final subscriptions = <StreamSubscription<Reading>>[];
  35       var delivering = 0;
  36-      late final StreamSubscription<Reading> gyroReadings;
  36-      var delivering = 2;
  37 
  38       void publishOn(String keyExpr, Stream<Reading> readings) {
  39         final publication = zenoh.declarePublication(keyExpr);
  40         publications.add(publication);
  41         delivering += 1;
  42+        subscriptions.add(
  43+          readings.listen(
  44+            (reading) {
  45+              publication.put(_asText(reading));
  46+              controller.add(reading);
  47+            },
  48+            onDone: () {
  49+              delivering -= 1;
  50+              if (delivering == 0) {
  51+                unawaited(controller.close());
  52+              }
  53+            },
  54+          ),
  55+        );
   ⋮
  58       controller
  59         ..onListen = () {
  60           publishOn(accelKeyExpr, sensor.accelerometer());
  61           publishOn(gyroKeyExpr, sensor.gyroscope());
  62         }
  63         ..onCancel = () async {
  64           for (final subscription in subscriptions) {
  65             await subscription.cancel();
  66           }
  67           for (final publication in publications) {
  68             publication.close();
  69           }
  70         };
  71-        await accelReadings.cancel();
  71-        await gyroReadings.cancel();
  71-        accel.close();
  71-        gyro.close();
  71-      };
```

The tests pass before and after. The `..` before `onListen` and `onCancel` is a cascade, two assignments on one
receiver, which the lint set asks for when two statements in a row address the same object.

**What the test guarantees:** a reading of the gyroscope goes on the wire on `sensor/<node>/gyro`, as `x,y,z` to
three decimals, and nothing goes on the accelerometer's key for it.

**Cycle 3 — each reading is handed on with its key.** The node's screen shows a block for each key in section 8, so
the repository hands each reading on with the key it was put on. The test delivers one reading of each sensor. Add it to
`zenoh_sensors/packages/sensor_core/test/repositories/sensor_node_repository_test.dart`:

```dart
⋮

  test('each reading is handed on with its key', () async {
    // Fakes: a service that records what is put through it, and a sensor
    // that delivers one reading of each kind.
    final zenoh = FakeZenohService();
    const accel = Reading(x: 0, y: 9.776, z: 0.812);
    const gyro = Reading(x: 0, y: 0, z: 0.5);
    final sensor = FakeSensorService(
      Stream.value(accel),
      gyroscope: Stream.value(gyro),
    );

    // The code to implement: each reading handed on as a pair, with the key
    // it was put on.
    final repository = SensorNodeRepository(zenoh, sensor, nodeName: 'phone');
    final published = await repository.publish().toList();

    // The claim: each reading handed on under the key it was put on.
    final byKey = {
      for (final (:keyExpr, :reading) in published)
        keyExpr: [reading.x, reading.y, reading.z],
    };
    expect(byKey, {
      'sensor/phone/accel': [0, 9.776, 0.812],
      'sensor/phone/gyro': [0, 0, 0.5],
    });
  });
}
```

The test reads `published` as pairs of a key and a reading, and `publish()` hands on readings, so the test does not
compile. The stream carries a `KeyedReading` now, with `''` for the key. Change
`zenoh_sensors/packages/sensor_core/lib/src/repositories/sensor_node_repository.dart`:

```dart
  29     /// that sensor's key, and hands each on with its key. Listening declares
  30     /// the publications; cancelling closes them. The stream ends when both
  31     /// sensors have ended.
  32     Stream<KeyedReading> publish() {
   ⋮
  36       final controller = StreamController<KeyedReading>();
   ⋮
  46               controller.add((keyExpr: '', reading: reading));
```

The app's readings provider yields a `Reading` for its view model. It takes the reading out of each pair, until
section 8 shows a block for each key. Change `zenoh_sensors/apps/sensor_node/lib/config/providers.dart`:

```dart
  22   /// The node's repository: this phone's sensors, each on its own key.
   ⋮
  36   /// The readings the node publishes, once the session is open, without their
  37+  /// keys.
   ⋮
  40     yield* ref
  41+        .watch(sensorNodeRepositoryProvider)
  42+        .publish()
  43+        .map((keyed) => keyed.reading);
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'with its key'
```

```
⋮
  Expected: {'sensor/phone/accel': [0, 9.776, 0.812], 'sensor/phone/gyro': [0, 0, 0.5]}
    Actual: {'': [0.0, 0.0, 0.5]}
     Which: has different length and is missing map key 'sensor/phone/accel'
⋮
```

Both readings were handed on under the empty key, so the map kept one.

**Write the obvious implementation.** `publishOn` has the key, and the pair carries it. Change
`zenoh_sensors/packages/sensor_core/lib/src/repositories/sensor_node_repository.dart`:

```dart
  46               controller.add((keyExpr: keyExpr, reading: reading));
```

Run it again. It passes. Then run the file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core/test/repositories/sensor_node_repository_test.dart
```

```
⋮
  Expected: [Instance of 'Reading']
    Actual: [
              ({String keyExpr, Reading reading}):(keyExpr: sensor/phone/accel, reading: Instance of 'Reading')
            ]
     Which: at location [0] is ({String keyExpr, Reading reading}):<(keyExpr: sensor/phone/accel, reading: Instance of 'Reading')> instead of <Instance of 'Reading'>
⋮
```

3 tests fail on the reading they expect handed on, because each expects a reading and gets a pair. Each expects
the pair now, the reading with its key, and their claims do not change:

1. The first test, `a sensor reading reaches a subscriber on sensor/phone/accel`, with `sensor/phone/accel`.
2. The put test, `a reading is put as x,y,z to three decimals and handed on`, with `sensor/phone/accel`.
3. The gyroscope test, `a gyroscope reading is put on sensor/phone/gyro and handed on`, with `sensor/phone/gyro`.

Change `zenoh_sensors/packages/sensor_core/test/repositories/sensor_node_repository_test.dart`:

```dart
  38       // and the reading handed on for the screen with that key.
   ⋮
  43       expect(published, [(keyExpr: 'sensor/phone/accel', reading: reading)]);
   ⋮
  58       // through the gyroscope's, and the same reading handed on with its key.
   ⋮
  64       expect(published, [(keyExpr: 'sensor/phone/accel', reading: reading)]);
   ⋮
 141         // through the accelerometer's, and the same reading handed on with its
 142+        // key.
   ⋮
 148         expect(published, [(keyExpr: 'sensor/phone/gyro', reading: reading)]);
```

Run the file again. 7 tests pass.

**Refactor.** Nothing to change. The key reaches the pair through `publishOn`'s parameter, and the two sensors still
share that one function.

**What the test guarantees:** each reading the node hands on carries the key it was put on, so the screen tells the
sensors apart by the key.

**Run the outer test again.**

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'under its own key'
```

```
⋮
  Expected: {'sensor/phone/accel': [0, 9.776, 0.812], 'sensor/phone/gyro': [0, 0, 0.5]}
    Actual: {'': [0.0, 9.776, 0.812]}
     Which: has different length and is missing map key 'sensor/phone/accel'
⋮
```

The node puts the gyroscope's reading on `sensor/phone/gyro` now, and the collector does not receive it, because its
repository subscribes to `sensor/phone/accel`. The accelerometer's reading arrives with no key, because the
subscription hands up only the payload. Section 6 makes the subscription hand up the key of each sample, and section
7 makes the collector subscribe to `sensor/phone/*`.

**What the tests guarantee:** a node named `phone` declares one publication for each of its sensors, on
`sensor/phone/accel` and `sensor/phone/gyro`. Each reading goes on the wire on its sensor's key, as `x,y,z` to three
decimals, and is handed on with that key. Cancelling the node's stream closes both publications, and the stream ends
only when both sensors have ended.

## 6 — The service hands up the key of each sample

Build the service's part in two cycles. A subscription hands up each sample as one value, the key it was put on
and its payload as text. Section 7's collector repository tells the sensors apart by that key.

These are the service's own tests, and they open two sessions on real zenoh, the node's and a collector's, in one
process on the loopback. You end by running the chapter's outer test.

You write two tests:

- a sample arrives with the key it was put on
- a subscriber to `sensor/phone/*` receives the key of the put

> **Zenoh guidance.** A sample is what a subscriber receives for one put: the key the put was made on, the
> payload, and its encoding. `z_sub` prints the key from each sample it receives, and the service reads it from the
> same place.

**Cycle 1 — a sample arrives with the key it was put on.** The test subscribes on `sensor/phone/accel`, puts text on
that key, and collects what the subscription hands up. Add a test to
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('a sample arrives with the key it was put on', () async {
    // The node's end: its session, from the settings that listen.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);
    await sensorNode.open();

    // The laptop's end: a collector's session, from the settings that
    // connect.
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(collectorNode.dispose);
    await collectorNode.open();

    // The new contract to implement: the samples a subscription hands up,
    // each the key it was put on and its payload as text.
    final subscription = collectorNode.declareSubscription(
      'sensor/phone/accel',
    );
    final received = <KeyedPayload>[];
    subscription.samples.listen(received.add);
    // The declaration travels to the node, so give it time to arrive.
    await Future<void>.delayed(delivery);

    // The node puts text through a publication on that key.
    sensorNode
        .declarePublication('sensor/phone/accel')
        .put('0.000,9.776,0.812');
    // A put returns before the sample arrives, so give it time to cross.
    await Future<void>.delayed(delivery);

    // The claim: the sample arrives with the key it was put on.
    expect(received, [
      (keyExpr: 'sensor/phone/accel', payload: '0.000,9.776,0.812'),
    ]);
  });
}
```

The test asks for two things that do not exist yet:

- `KeyedPayload`, the value the subscription hands up for each sample, the key it was put on and its payload as text.
- `samples` on `Subscription`, the stream of those values.

`KeyedPayload` is a record type, and it goes before `Subscription`, because `Subscription` names it. Both of its fields
are strings, so the service still hands nothing of the package upward. `samples` answers with an empty stream, so
that the test compiles and reaches its claim. Change
`zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
  30+  /// A sample as the service hands it up: the key expression it was put on, and
  31+  /// its payload as text.
  32+  typedef KeyedPayload = ({String keyExpr, String payload});
  33+
   ⋮
  44+
  45+    /// Every sample that arrives, with its key, in order of arrival.
  46+    Stream<KeyedPayload> get samples => const Stream.empty();
```

The fake subscription promises every member of `Subscription`, so it gets `samples` too. It answers with an empty
stream, because no test of this section reads it. Change `zenoh_sensors/packages/sensor_core/test/support/fakes.dart`:

```dart
  43+    Stream<KeyedPayload> get samples => const Stream.empty();
  44+
  45+    @override
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'key it was put on'
```

```
⋮
  Expected: [
              ({String keyExpr, String payload}):(keyExpr: sensor/phone/accel, payload: 0.000,9.776,0.812)
            ]
    Actual: []
     Which: at location [0] is [] which shorter than expected
⋮
```

Nothing arrived, because the skeleton's `samples` is an empty stream.

> **Dart guidance.** Two records with the same fields and the same values are equal, so `expect` compares the
> sample that arrived with the one written in the test, field by field.

**Fake it.** Map each sample of the subscriber's stream to its payload beside the key as a constant. Change
`zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
  46     Stream<KeyedPayload> get samples => _subscriber.stream.map(
  47+      (sample) => (keyExpr: 'sensor/phone/accel', payload: sample.payload),
  48+    );
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'key it was put on'
```

It passes. Then run the file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core/test/services/zenoh_service_test.dart
```

16 tests pass. The key is a constant, and the key the sample was put on is ignored, so the next cycle puts on
another key.

**Refactor.** Nothing to change. The constant is what the next cycle removes.

**Cycle 2 — a subscriber to `sensor/phone/*` receives the key of the put.** The subscription's expression names a
set of keys now, and the put goes on one of them, `sensor/phone/gyro`. Add a test to
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('a subscriber to sensor/phone/* receives the key of the put', () async {
    // The node's end: its session, from the settings that listen.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);
    await sensorNode.open();

    // The laptop's end: a collector's session, from the settings that
    // connect.
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(collectorNode.dispose);
    await collectorNode.open();

    // The code to implement: the key of each sample read from the sample.
    // The subscription's own expression names a set of keys.
    final subscription = collectorNode.declareSubscription('sensor/phone/*');
    final received = <KeyedPayload>[];
    subscription.samples.listen(received.add);
    // The declaration travels to the node, so give it time to arrive.
    await Future<void>.delayed(delivery);

    // The node puts text on one key of that set.
    sensorNode.declarePublication('sensor/phone/gyro').put('0.000,0.000,0.500');
    // A put returns before the sample arrives, so give it time to cross.
    await Future<void>.delayed(delivery);

    // The claim: the sample arrives with the key of the put, not with the
    // subscription's expression.
    expect(received, [
      (keyExpr: 'sensor/phone/gyro', payload: '0.000,0.000,0.500'),
    ]);
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'key of the put'
```

```
⋮
  Expected: [
              ({String keyExpr, String payload}):(keyExpr: sensor/phone/gyro, payload: 0.000,0.000,0.500)
            ]
    Actual: [
              ({String keyExpr, String payload}):(keyExpr: sensor/phone/accel, payload: 0.000,0.000,0.500)
            ]
     Which: at location [0] is ({String keyExpr, String payload}):<(keyExpr: sensor/phone/accel, payload: 0.000,0.000,0.500)> instead of ({String keyExpr, String payload}):<(keyExpr: sensor/phone/gyro, payload: 0.000,0.000,0.500)>
⋮
```

The constant gives every sample the accelerometer's key.

**Triangulate.** Read the key from the sample, because no constant satisfies both tests. Change
`zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
  47       (sample) => (keyExpr: sample.keyExpr, payload: sample.payload),
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'key of the put'
```

It passes. Then run the file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core/test/services/zenoh_service_test.dart
```

17 tests pass. The subscription was declared on `sensor/phone/*`, and the key that arrived is `sensor/phone/gyro`,
the key of the put. A sample carries the key it was put on, and the subscriber's expression only chooses which
samples arrive.

**Refactor.** `samples` and `payloads` each map the subscriber's stream, and `payloads` is `samples` without the
keys. Three changes:

1. The class comment of `Subscription` says what it hands out now, each sample as its key and its payload.
2. `payloads` maps `samples` to the payload, so the subscriber's stream is mapped once.
3. `samples` goes before `payloads`, because `payloads` uses it.

Change `zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
  35   /// each sample as the key it was put on and its payload as text.
   ⋮
  41-    /// The payload of every sample that arrives, as text, in order of arrival.
  41-    Stream<String> get payloads =>
  41-        _subscriber.stream.map((sample) => sample.payload);
  41-
   ⋮
  45+
  46+    /// The payload of every sample that arrives, as text, in order of arrival.
  47+    Stream<String> get payloads => samples.map((sample) => sample.payload);
```

The tests pass before and after.

**What the tests guarantee:** a subscription hands up each sample with the key it was put on and its payload as
text. The key is one of the set its expression names, so a subscriber to a set of keys tells them apart.

**Run the outer test again.**

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'under its own key'
```

```
⋮
  Expected: {'sensor/phone/accel': [0, 9.776, 0.812], 'sensor/phone/gyro': [0, 0, 0.5]}
    Actual: {'': [0.0, 9.776, 0.812]}
     Which: has different length and is missing map key 'sensor/phone/accel'
⋮
```

The service hands up the key of each sample now, and the collector's repository still reads `payloads`, so the
accelerometer's reading arrives with no key. The gyroscope's reading does not arrive, because the repository
subscribes to `sensor/phone/accel`. Section 7 makes the repository read `samples`, hand each reading on with its
key, and subscribe to `sensor/phone/*`.

## 7 — The collector's repository, every sensor of its node

Build the collector's repository in two cycles, and the outer test goes green. The repository reads each sample from
the service with its key, and hands each reading on with that key. It subscribes to every sensor of its node with
one key expression, `sensor/<node>/*`.

These are layer tests, each against a fake service. The chapter's outer test runs the repository against real zenoh,
and you end by running it.

You write two tests:

- a reading is handed on with the key it arrived on
- the collector subscribes to `sensor/phone/*`

> **Zenoh guidance.** The matching is zenoh's. The collector names a set of keys once, when it declares the
> subscription, and compares no key itself. The key it hands on is the one the sample carries.

**Cycle 1 — a reading is handed on with the key it arrived on.** The test's fake subscription plays one sample, on
`sensor/phone/gyro`, while the repository still subscribes to `sensor/phone/accel`. A fake plays what the test puts
into it, whatever expression it was declared on. The sample's key differs from the subscription's expression, so the
test tells a key read from the sample from the repository's own. Add a test to
`zenoh_sensors/packages/sensor_core/test/repositories/readings_repository_test.dart`:

```dart
⋮

  test('a reading is handed on with the key it arrived on', () async {
    // A fake: a service whose subscription plays what the test puts into it,
    // a sample with its key.
    final zenoh = FakeZenohService();
    final repository = ReadingsRepository(zenoh, nodeName: 'phone');
    final received = <KeyedReading>[];
    final listening = repository.readings().listen(received.add);
    addTearDown(listening.cancel);

    // The code to implement: the key read from the sample, and handed on with
    // the reading.
    zenoh.subscriptions.single.arrivals.add((
      keyExpr: 'sensor/phone/gyro',
      payload: '0.000,0.000,0.500',
    ));
    await pumpEventQueue();

    // The claim: the reading arrives under the key of the sample.
    final byKey = {
      for (final (:keyExpr, :reading) in received)
        keyExpr: [reading.x, reading.y, reading.z],
    };
    expect(byKey, {
      'sensor/phone/gyro': [0, 0, 0.5],
    });
  });
}
```

The test puts a sample with its key into the fake subscription, and `arrivals` takes text. The fake mirrors the
service's `Subscription` now. Three changes:

1. `arrivals` carries keyed samples.
2. `samples` is the stream of `arrivals`.
3. `payloads` maps `samples` to the payload, so `samples` goes before `payloads`.

Change `zenoh_sensors/packages/sensor_core/test/support/fakes.dart`:

```dart
  36     final arrivals = StreamController<KeyedPayload>();
   ⋮
  40     Stream<KeyedPayload> get samples => arrivals.stream;
   ⋮
  43     Stream<String> get payloads => samples.map((sample) => sample.payload);
```

The payload test, `a payload x,y,z arrives as a reading`, puts text into `arrivals`, and `arrivals` takes a sample
now. Its claim stands, and its body feeds a sample on `sensor/phone/accel`. Change
`zenoh_sensors/packages/sensor_core/test/repositories/readings_repository_test.dart`:

```dart
  83       zenoh.subscriptions.single.arrivals.add((
  84+        keyExpr: 'sensor/phone/accel',
  85+        payload: '0.000,9.776,0.812',
  86+      ));
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'key it arrived on'
```

```
⋮
  Expected: {'sensor/phone/gyro': [0, 0, 0.5]}
    Actual: {'': [0.0, 0.0, 0.5]}
     Which: is missing map key 'sensor/phone/gyro'
⋮
```

The reading arrived under the empty key, because the repository reads `payloads` and hands on `''`.

**Write the obvious implementation.** The sample carries its key, and the repository reads it there. Two changes:

1. `listening` listens to `samples`, so it is a subscription to `KeyedPayload`.
2. Each sample is handed on as the pair of its key and the reading parsed from its payload.

Change `zenoh_sensors/packages/sensor_core/lib/src/repositories/readings_repository.dart`:

```dart
  24       late final StreamSubscription<KeyedPayload> listening;
   ⋮
  28         listening = subscription.samples.listen(
  29           (sample) => controller.add((
  30+            keyExpr: sample.keyExpr,
  31+            reading: _asReading(sample.payload),
  32+          )),
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'key it arrived on'
```

It passes. Then run the file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core/test/repositories/readings_repository_test.dart
```

```
⋮
  Expected: {'sensor/phone/accel': [0, 9.776, 0.812], 'sensor/phone/gyro': [0, 0, 0.5]}
    Actual: {'sensor/phone/accel': [0.0, 9.776, 0.812]}
     Which: has different length and is missing map key 'sensor/phone/gyro'
⋮
```

6 tests pass, and the outer test fails on its assertion. The accelerometer's reading arrives under its key now. The
gyroscope's reading does not arrive, because the repository subscribes to `sensor/phone/accel`. Cycle 2 widens the
subscription.

**Refactor.** Nothing to change. The parse stays one function, and the key goes from the sample to the pair in the
one listen block.

**What the test guarantees:** each reading the collector's repository hands on carries the key of the sample it came
from, so the collector tells the sensors apart by the key.

**Cycle 2 — the collector subscribes to `sensor/phone/*`.** The test that said the collector subscribes to
`sensor/phone/accel` made a narrower claim than this one. Two changes:

1. Remove the test `the collector subscribes to sensor/phone/accel`, the second in the file.
2. Add this claim's test at the end of the file.

Change `zenoh_sensors/packages/sensor_core/test/repositories/readings_repository_test.dart`:

```dart
⋮

  test('a reading the phone publishes reaches the collector', () async {
    ⋮
  });

  test('a collector of the node sim subscribes to sensor/sim/accel', () async {
    ⋮
  });

⋮

  test('the collector subscribes to sensor/phone/*', () async {
    // A fake: a service that records what is declared on it.
    final zenoh = FakeZenohService();

    // The code to implement: one subscription for every sensor of the phone.
    final repository = ReadingsRepository(zenoh, nodeName: 'phone');
    final listening = repository.readings().listen((_) {});
    addTearDown(listening.cancel);

    // The claim: one subscription, on the expression that matches each of the
    // phone's keys.
    final keys = zenoh.subscriptions.map((s) => s.keyExpr).toList();
    expect(keys, ['sensor/phone/*']);
  });
}
```

Run it by the first words of its name, because `-n` takes a regular expression, in which `*` has a meaning of its
own:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'collector subscribes'
```

```
⋮
  Expected: ['sensor/phone/*']
    Actual: ['sensor/phone/accel']
     Which: at location [0] is 'sensor/phone/accel' instead of 'sensor/phone/*'
⋮
```

The subscription is on the accelerometer's key.

**Write the obvious implementation.** The expression keeps the node's name and takes `*` for the sensor's segment, a
known step. Three changes:

1. The constructor builds `sensor/<node>/*`, and its comment says every sensor.
2. The comment of `keyExpr` says what the expression matches.
3. The comment of the repository's provider in `sensorctl` names the expression.

Change `zenoh_sensors/packages/sensor_core/lib/src/repositories/readings_repository.dart`:

```dart
   9     /// A repository receiving, through [zenoh], the readings of every sensor of
  10     /// the node called [nodeName].
  11     new(this.zenoh, {required String nodeName}) : keyExpr = 'sensor/$nodeName/*';
  12-      : keyExpr = 'sensor/$nodeName/accel';
   ⋮
  16     /// The key expression the subscription is declared on: one segment for the
  17+    /// sensor, so it matches each of the node's keys.
```

Change `zenoh_sensors/apps/sensorctl/lib/config/providers.dart`:

```dart
  16   /// The collector's repository: the phone's readings, on `sensor/phone/*`.
```

Run it again:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'collector subscribes'
```

It passes. Then run the file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core/test/repositories/readings_repository_test.dart
```

```
⋮
  Expected: ['sensor/sim/accel']
    Actual: ['sensor/sim/*']
     Which: at location [0] is 'sensor/sim/*' instead of 'sensor/sim/accel'
⋮
```

6 tests pass, the outer test among them, and the `sim` test fails. Its claim is narrower too, one sensor's key
where the expression matches every sensor's. Two changes:

1. Remove the test `a collector of the node sim subscribes to sensor/sim/accel`, the second in the file.
2. Add the test `a collector of the node sim subscribes to sensor/sim/*` at the end of the file.

Change `zenoh_sensors/packages/sensor_core/test/repositories/readings_repository_test.dart`:

```dart
⋮

  test('a reading the phone publishes reaches the collector', () async {
    ⋮
  });

  test('a payload x,y,z arrives as a reading', () async {
    ⋮
  });

⋮

  test('a collector of the node sim subscribes to sensor/sim/*', () async {
    // A fake: a service that records what is declared on it.
    final zenoh = FakeZenohService();

    // The code to implement: the expression built from the node's name.
    final repository = ReadingsRepository(zenoh, nodeName: 'sim');
    final listening = repository.readings().listen((_) {});
    addTearDown(listening.cancel);

    // The claim: one subscription, on the expression that matches each of the
    // sim node's keys.
    final keys = zenoh.subscriptions.map((s) => s.keyExpr).toList();
    expect(keys, ['sensor/sim/*']);
  });
}
```

Run the file again. 7 tests pass.

**Refactor.** Nothing to change. One line of the repository changed, the expression, and nothing outside the
repository reads `keyExpr`.

**What the tests guarantee:** a collector of the node `<node>` declares one subscription, on `sensor/<node>/*`, built
from the node's name. That expression matches every key of three segments under `sensor/<node>`.

**Run the outer test again.**

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'under its own key'
```

It passes. Then run the whole core:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

31 tests pass, the three files one after the other.

**What the outer test guarantees:** a reading from each of the phone's two sensors is published on that sensor's key.
A collector subscribed to `sensor/phone/*` receives both. Each reading arrives with the key it was put on, so the
collector tells the sensors apart by the key.

The claim holds on the laptop, with a fake sensor. The data side's pass is done. Section 8 builds the app from its
screen in, and reads the device's gyroscope.

## 8 — The app from its screen in

Build the app from its screen in, in three cycles. The screen shows a block for each key, and the view model keeps
the latest reading and the count for each key. The device service reads the gyroscope, a second stream from
`sensors_plus`.

These are layer tests, each against a fake of the layer below. Nothing in them opens a session or reads a sensor. You
end by running the chapter's outer test.

You write three tests:

- the screen shows a block for each key
- the view model keeps the latest reading and the count for each key
- a gyroscope event becomes a reading, field for field

**Cycle 1 — the screen shows a block for each key.** The test that said the screen shows the latest reading and the
count made a claim for one sensor. It goes, and this claim's test takes its place. The fake view model holds a
reading and a count on each of the phone's two keys, and the screen is built once from that state. Replace
`zenoh_sensors/apps/sensor_node/test/ui/node/node_screen_test.dart`:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:sensor_core/sensor_core.dart';
import 'package:sensor_node/ui/node/node_screen.dart';
import 'package:sensor_node/ui/node/node_view_model.dart';

import '../../support/fakes.dart';

void main() {
  testWidgets('the screen shows a block for each key', (tester) async {
    // A fake: a view model that holds a fixed state, with a reading and a
    // count on each of the phone's two keys.
    const accel = Reading(x: 0.1, y: 9.776, z: 0.812);
    const gyro = Reading(x: 0, y: 0, z: 0.5);
    final container = ProviderContainer.test(
      overrides: [
        nodeViewModelProvider.overrideWith(
          () => FakeNodeViewModel(
            const NodeState(
              sensors: {
                'sensor/phone/accel': SensorState(latest: accel, count: 42),
                'sensor/phone/gyro': SensorState(latest: gyro, count: 7),
              },
            ),
          ),
        ),
      ],
    );

    // The code to implement: the screen, built once from that state, with a
    // block for each key.
    await tester.pumpWidget(
      UncontrolledProviderScope(
        container: container,
        child: const MaterialApp(home: NodeScreen()),
      ),
    );

    // The claim: each key, its reading to three decimals, and its count.
    expect(find.text('sensor/phone/accel'), findsOneWidget);
    expect(find.text('x 0.100'), findsOneWidget);
    expect(find.text('y 9.776'), findsOneWidget);
    expect(find.text('z 0.812'), findsOneWidget);
    expect(find.text('42 readings'), findsOneWidget);
    expect(find.text('sensor/phone/gyro'), findsOneWidget);
    expect(find.text('x 0.000'), findsOneWidget);
    expect(find.text('y 0.000'), findsOneWidget);
    expect(find.text('z 0.500'), findsOneWidget);
    expect(find.text('7 readings'), findsOneWidget);
  });
}
```

The test asks for two things that do not exist yet:

- `SensorState`, what the screen shows for one key: its latest reading, and how many there were.
- `sensors` on `NodeState`, a map from each key to its `SensorState`.

`SensorState` goes before `NodeState`, because `NodeState` names it. Its `latest` is never null, because a key gets
a state when its first reading arrives. `NodeState` keeps `latest` and `count` for this cycle, because the view model
and its test still read them. Change `zenoh_sensors/apps/sensor_node/lib/ui/node/node_view_model.dart`:

```dart
   5   /// What the node's screen shows for one key: the latest reading on it, and how
   6+  /// many there were.
   7+  class SensorState {
   8+    /// A state with [latest] as the newest reading and [count] readings so far.
   9+    const new({required this.latest, required this.count});
  10+
  11+    /// The newest reading on the key.
  12+    final Reading latest;
  13+
  14+    /// How many readings the node has published on the key.
  15+    final int count;
  16+  }
  17+
  18+  /// What the node's screen shows: the latest reading and how many there were,
  19+  /// and the state of each key.
   ⋮
  21     /// A state with [latest] as the newest reading, [count] readings so far, and
  22     /// [sensors] as the state of each key.
  23+    const new({this.latest, this.count = 0, this.sensors = const {}});
   ⋮
  30+
  31+    /// The state of each key the node has published on.
  32+    final Map<String, SensorState> sensors;
```

Run it by name. `flutter test` takes the name after `--name`, and has no `-n`:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node --name 'block for each key'
```

```
⋮
Expected: exactly one matching candidate
  Actual: _TextWidgetFinder:<Found 0 widgets with text "sensor/phone/accel": []>
   Which: means none were found but one was expected
⋮
```

The screen shows no key, because it shows one reading and one count.

**Write the obvious implementation.** The screen builds one block for each entry of the map, a known step. Each
block is a column of the key, the three values and the count, and the blocks go in a column of their own. Change
`zenoh_sensors/apps/sensor_node/lib/ui/node/node_screen.dart`:

```dart
   5   /// The node's one screen: a block for each key, with the key, its latest
   6+  /// reading to three decimals, and its count.
   ⋮
  14-      final latest = node.latest;
   ⋮
  20+              spacing: 24,
   ⋮
  22                 for (final MapEntry(key: keyExpr, value: sensor)
  23                     in node.sensors.entries)
  24                   Column(
  25                     mainAxisSize: MainAxisSize.min,
  26+                    children: [
  27+                      Text(keyExpr),
  28+                      Text('x ${_fixed(sensor.latest.x)}'),
  29+                      Text('y ${_fixed(sensor.latest.y)}'),
  30+                      Text('z ${_fixed(sensor.latest.z)}'),
  31+                      Text('${sensor.count} readings'),
  32+                    ],
  33+                  ),
   ⋮
  41     static String _fixed(double value) => value.toStringAsFixed(3);
```

> **Dart guidance.** `for (final MapEntry(key: keyExpr, value: sensor) in node.sensors.entries)` is a collection
> `for`, which puts one element into the list for each entry of the map. `MapEntry(key: keyExpr, value: sensor)` is
> an object pattern, which takes the entry's key and value out under two names of your own.

`spacing: 24` puts 24 logical pixels between the blocks. Before the first reading the screen is empty, because the
state has no key yet.

Run it again:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node --name 'block for each key'
```

It passes.

**Refactor.** `aReading` in the fakes has no reader, because the screen's test defines its own two readings. It goes.
Change `zenoh_sensors/apps/sensor_node/test/support/fakes.dart`:

```dart
   1-  import 'package:sensor_core/sensor_core.dart';
   ⋮
  11-
  11-  const aReading = Reading(x: 0.1, y: 9.776, z: 0.812);
```

The tests pass before and after.

**What the test guarantees:** the screen shows, for each key the node has published on, the key, its latest reading
to three decimals, and its count.

**Cycle 2 — the view model keeps the latest reading and the count for each key.** The test that said the view model
keeps the latest reading and counts them made a claim for one sensor, and its readings came without keys. It goes,
and this claim's test takes its place. The readings provider delivers three readings on two keys, each with its key.
Replace `zenoh_sensors/apps/sensor_node/test/ui/node/node_view_model_test.dart`:

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:sensor_core/sensor_core.dart';
import 'package:sensor_node/config/providers.dart';
import 'package:sensor_node/ui/node/node_view_model.dart';

void main() {
  test(
    'the view model keeps the latest reading and the count for each key',
    () async {
      // A fake: the readings provider, overridden with three readings on two
      // keys, so nothing below the view model is built.
      const first = Reading(x: 0, y: 9.776, z: 0.812);
      const second = Reading(x: 0, y: 0, z: 0.5);
      const third = Reading(x: 0, y: 0, z: 9.81);
      final container = ProviderContainer.test(
        overrides: [
          readingsProvider.overrideWith(
            (ref) => Stream.fromIterable([
              (keyExpr: 'sensor/phone/accel', reading: first),
              (keyExpr: 'sensor/phone/gyro', reading: second),
              (keyExpr: 'sensor/phone/accel', reading: third),
            ]),
          ),
        ],
      )..listen(nodeViewModelProvider, (_, _) {});

      // Let the three readings flow through before reading the state.
      await pumpEventQueue();

      // The code to implement: the view model's state, a latest reading and
      // a count for each key it heard.
      final state = container.read(nodeViewModelProvider);

      // The claim: two readings counted on the accelerometer's key and the
      // third kept, one on the gyroscope's.
      final sensors = {
        for (final MapEntry(key: keyExpr, value: sensor)
            in state.sensors.entries)
          keyExpr: (
            latest: (sensor.latest.x, sensor.latest.y, sensor.latest.z),
            count: sensor.count,
          ),
      };
      expect(sensors, {
        'sensor/phone/accel': (latest: (0, 0, 9.81), count: 2),
        'sensor/phone/gyro': (latest: (0, 0, 0.5), count: 1),
      });
    },
  );
}
```

The claim is a map from each key to its latest reading and its count. The reading goes in as a record of its three
values, because a `Reading` has no equality of its own.

The test overrides `readingsProvider` with a stream of pairs, and the provider yields a `Reading`. Two changes let the
test compile and leave the claim false:

1. `readingsProvider` yields each pair the repository hands on, key included.
2. The view model takes the reading out of each pair, as the provider did.

Change `zenoh_sensors/apps/sensor_node/lib/config/providers.dart`:

```dart
  36   /// The readings the node publishes, each with its key, once the session is
  37   /// open.
  38   final readingsProvider = StreamProvider<KeyedReading>((ref) async* {
   ⋮
  40     yield* ref.watch(sensorNodeRepositoryProvider).publish();
  41-        .watch(sensorNodeRepositoryProvider)
  41-        .publish()
  41-        .map((keyed) => keyed.reading);
```

Change `zenoh_sensors/apps/sensor_node/lib/ui/node/node_view_model.dart`:

```dart
  42           state = NodeState(latest: value.reading, count: state.count + 1);
```

Run it:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node --name 'count for each key'
```

```
⋮
  Expected: {
              'sensor/phone/accel': ({int count, (int, int, double) latest}):(count: 2, latest: (0, 0, 9.81)),
              'sensor/phone/gyro': ({int count, (int, int, double) latest}):(count: 1, latest: (0, 0, 0.5))
            }
    Actual: {}
     Which: has different length and is missing map key 'sensor/phone/accel'
⋮
```

The state has no key, because the view model keeps one latest reading and one count.

**Write the obvious implementation.** The pair carries the key, and the state keeps a `SensorState` under it. On each
reading, the view model takes the key and the reading out of the pair. It reads the count on that key so far, or 0
for a key it has not seen, and replaces its state with a new map. Change
`zenoh_sensors/apps/sensor_node/lib/ui/node/node_view_model.dart`:

```dart
  36   /// counts the readings, for each key.
   ⋮
  42           final (:keyExpr, :reading) = value;
  43+          final count = state.sensors[keyExpr]?.count ?? 0;
  44+          state = NodeState(
  45+            sensors: {
  46+              ...state.sensors,
  47+              keyExpr: SensorState(latest: reading, count: count + 1),
  48+            },
  49+          );
```

`{...state.sensors, keyExpr: …}` is a new map with every entry of the old one, and the key's new state in place of
its old one. A key already in the map keeps its place, so the blocks on the screen stay in the order in which the
keys first arrived.

Run it again:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node --name 'count for each key'
```

It passes.

**Refactor.** `latest` and `count` on `NodeState` have no reader now. They go, with the two comments that named
them. Change `zenoh_sensors/apps/sensor_node/lib/ui/node/node_view_model.dart`:

```dart
  18   /// What the node's screen shows: the state of each key.
  19-  /// and the state of each key.
  20     /// A state with [sensors] as the state of each key.
  21     const new({this.sensors = const {}});
  22-    const new({this.latest, this.count = 0, this.sensors = const {}});
  23     /// The state of each key the node has published on, in the order the keys
  24     /// first arrived.
  25-
  25-    /// How many readings the node has published.
  25-    final int count;
  25-
  25-    /// The state of each key the node has published on.
```

The tests pass before and after.

**What the test guarantees:** the view model counts the readings on each key the node publishes on, and keeps the
latest reading of each.

**Cycle 3 — a gyroscope event becomes a reading, field for field.** `DeviceSensorService.gyroscope()` answers an
empty stream. The test hands the service a gyroscope function in place of the plugin's, as the accelerometer's test
does, with one event, and reads it through `gyroscope()`. The test goes at the end of the file, after the
accelerometer's. Change `zenoh_sensors/apps/sensor_node/test/data/services/device_sensor_service_test.dart`:

```dart
⋮

  test('a gyroscope event becomes a reading, field for field', () async {
    // A fake of the plugin's gyroscope function: one event, made by the test.
    final event = GyroscopeEvent(0.1, 0.2, 0.3, DateTime(2026, 9, 23, 12));
    final service = DeviceSensorService(
      gyroscopeEvents: ({samplingPeriod = SensorInterval.normalInterval}) =>
          Stream.value(event),
    );

    // The code to implement: each gyroscope event mapped to a reading.
    final readings = await service.gyroscope().toList();

    // The claim: one reading, with the event's values on the three axes.
    final fields = readings.map((r) => (r.x, r.y, r.z)).toList();
    expect(fields, [(0.1, 0.2, 0.3)]);
  });
}
```

The test names a parameter the constructor does not take, `gyroscopeEvents`. The service takes it, in a field of the
gyroscope function's shape, with the plugin's `gyroscopeEventStream` as its default, and `gyroscope()` still answers
an empty stream. Change `zenoh_sensors/apps/sensor_node/lib/data/services/device_sensor_service.dart`:

```dart
  10+  /// The shape of the plugin's gyroscope function, so that a test can hand in
  11+  /// another.
  12+  typedef GyroscopeEvents = Stream<GyroscopeEvent> Function({
  13+    Duration samplingPeriod,
  14+  });
  15+
   ⋮
  18     /// A service over the plugin's accelerometer and gyroscope, or over what a
  19     /// test hands in.
  20+    new({
  21+      this._events = accelerometerEventStream,
  22+      this._gyroscopeEvents = gyroscopeEventStream,
  23+    });
   ⋮
  26+    final GyroscopeEvents _gyroscopeEvents;
```

The analyzer warns that `_gyroscopeEvents` is unused, until the next step reads it.

Run it:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node --name 'gyroscope event'
```

```
⋮
  Expected: [(double, double, double):(0.1, 0.2, 0.3)]
    Actual: []
     Which: at location [0] is [] which shorter than expected
⋮
```

No reading arrived, because `gyroscope()` answers an empty stream.

**Write the obvious implementation.** The gyroscope's events are mapped to readings as the accelerometer's are, a
known step. Change `zenoh_sensors/apps/sensor_node/lib/data/services/device_sensor_service.dart`:

```dart
  33     Stream<Reading> gyroscope() => _gyroscopeEvents().map(
  34+      (event) => Reading(x: event.x, y: event.y, z: event.z),
  35+    );
```

`gyroscopeEventStream` is the plugin's gyroscope function, a broadcast stream of `GyroscopeEvent`s, each with the
rate of rotation about the three axes.

Run it again:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node --name 'gyroscope event'
```

It passes. Then run the file:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node/test/data/services/device_sensor_service_test.dart
```

2 tests pass.

**Refactor.** `_events` names one of two functions now, and the two typedefs differ only in the event's type. Three
changes:

1. One typedef, `SensorEvents<E>`, the shape of the plugin's sensor functions for any event type, takes the place of
   the two.
2. `_events` becomes `_accelerometerEvents`, and the accelerometer's test names the parameter `accelerometerEvents`.
3. The two `map` calls stay alike, because `AccelerometerEvent` and `GyroscopeEvent` share no type with the three
   axes on it.

Change `zenoh_sensors/apps/sensor_node/lib/data/services/device_sensor_service.dart`:

```dart
   4   /// The shape of the plugin's sensor functions, each a stream of its events,
   5   /// so that a test can hand in another.
   6   typedef SensorEvents<E> = Stream<E> Function({Duration samplingPeriod});
   7-    Duration samplingPeriod,
   7-  });
   7-
   7-  /// The shape of the plugin's gyroscope function, so that a test can hand in
   7-  /// another.
   7-  typedef GyroscopeEvents = Stream<GyroscopeEvent> Function({
   7-    Duration samplingPeriod,
   7-  });
   ⋮
  13       this._accelerometerEvents = accelerometerEventStream,
   ⋮
  17     final SensorEvents<AccelerometerEvent> _accelerometerEvents;
  18     final SensorEvents<GyroscopeEvent> _gyroscopeEvents;
   ⋮
  21     Stream<Reading> accelerometer() => _accelerometerEvents().map(
  22       (event) => Reading(x: event.x, y: event.y, z: event.z),
  23+    );
```

The accelerometer's test changes in place, and its claim stands. Change
`zenoh_sensors/apps/sensor_node/test/data/services/device_sensor_service_test.dart`:

```dart
  10         accelerometerEvents: ({samplingPeriod = SensorInterval.normalInterval}) =>
```

The tests pass before and after.

> **Dart guidance.** `typedef SensorEvents<E> = Stream<E> Function({Duration samplingPeriod});` is a generic
> typedef, a name for a function type with a type parameter. `SensorEvents<GyroscopeEvent>` is the type of a
> function that takes a named `samplingPeriod` and returns a stream of gyroscope events, which is what the plugin's
> `gyroscopeEventStream` is.

**What the test guarantees:** a gyroscope event from `sensors_plus` becomes a reading with the event's values on the
three axes, as an accelerometer event does.

**Run the outer test again.**

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'under its own key'
```

It passes. The app's pass changed nothing in the core. Then run every test of the app:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node
```

4 tests pass. Nothing in them opens a session or reads a sensor.

**What the tests guarantee:** the node's screen shows a block for each key the node publishes on, with the key, its
latest reading to three decimals, and its count. The view model counts the readings on each key and keeps the latest
of each, in the order the keys first arrived. The device service reads the phone's gyroscope through `sensors_plus`,
event for event, as it reads the accelerometer.

The app publishes both sensors and shows both. Section 9 builds `watch` from its lines in.

## 9 — `watch` from its lines in

Build `watch` from its lines in, in two cycles. The view shows a line for each key, with the key, its latest reading
and its count. The view model keeps the latest reading and the count for each key.

These are layer tests, each against a state the test builds or a fake of the readings provider. Nothing in them
opens a session. You end by running the chapter's outer test.

You write two tests:

- `watch` shows a line for each key
- the view model keeps the latest reading and the count for each key

**Cycle 1 — `watch` shows a line for each key.** The test that said the line shows the latest reading and the count
made a claim for one sensor. It goes, and this claim's test takes its place, at the end of the file. The state holds
a reading and a count on each of the phone's two keys, and the lines are made from it. Change
`zenoh_sensors/apps/sensorctl/test/ui/watch/watch_view_test.dart`:

```dart
⋮

  test('before the first reading the line says so', () {
    ⋮
  });

  test('watch shows a line for each key', () {
    // A state with a reading and a count on each of the phone's two keys,
    // made by the test.
    const accel = Reading(x: 0.1, y: 9.776, z: 0.812);
    const gyro = Reading(x: 0, y: 0, z: 0.5);
    const state = WatchState(
      sensors: {
        'sensor/phone/accel': SensorState(latest: accel, count: 42),
        'sensor/phone/gyro': SensorState(latest: gyro, count: 7),
      },
    );

    // The code to implement: the lines for that state, one for each key.
    final lines = watchLines(state);

    // The claim: each key, its reading to three decimals, and its count, the
    // keys padded so that the readings line up.
    expect(lines, [
      'sensor/phone/accel  x   0.100  y   9.776  z   0.812  42 readings',
      'sensor/phone/gyro   x   0.000  y   0.000  z   0.500  7 readings',
    ]);
  });
}
```

The new test reads a list of lines from `watchLines`, and the view has one function for its lines from here on. The
test that says the line before the first reading says so reads the list too. Its claim stands, and its body changes
in place. Change `zenoh_sensors/apps/sensorctl/test/ui/watch/watch_view_test.dart`:

```dart
  11       // The code to implement: the lines for that state.
  12       final lines = watchLines(state);
   ⋮
  14       // The claim: one line, which says that nothing has arrived.
  15       expect(lines, ['no readings yet']);
```

The tests ask for three things that do not exist yet:

- `SensorState`, what `watch` shows for one key: its latest reading, and how many there were.
- `sensors` on `WatchState`, a map from each key to its `SensorState`.
- `watchLines`, the lines for a state, in place of `watchLine`.

`SensorState` goes before `WatchState`, because `WatchState` names it. Its `latest` is never null, because a key gets
a state when its first reading arrives. `WatchState` keeps `latest` and `count` for this cycle, because the view
model and its test still read them. Change `zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view_model.dart`:

```dart
   5   /// What `watch` shows for one key: the latest reading on it, and how many
   6+  /// there were.
   7+  class SensorState {
   8+    /// A state with [latest] as the newest reading and [count] readings so far.
   9+    const new({required this.latest, required this.count});
  10+
  11+    /// The newest reading on the key.
  12+    final Reading latest;
  13+
  14+    /// How many readings have arrived on the key.
  15+    final int count;
  16+  }
  17+
  18+  /// What `watch` shows: the latest reading and how many there were, and the
  19+  /// state of each key.
   ⋮
  21     /// A state with [latest] as the newest reading, [count] readings so far, and
  22     /// [sensors] as the state of each key.
  23+    const new({this.latest, this.count = 0, this.sensors = const {}});
   ⋮
  30+
  31+    /// The state of each key a reading has arrived on.
  32+    final Map<String, SensorState> sensors;
```

`watchLines` answers the no-readings line for a state with no key, and nothing yet for the keys.

`show` draws the list it answers. On a terminal, it keeps how many lines the last state took and moves the cursor up
over them. Then it draws each line of the new state in their place, each on a line of its own. When the output is not
a terminal, it prints each line. The cursor ends below the last line after each state, so `close` only shows it
again.

The drawing has no test, and section 10 runs it on your terminal. Change `zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view.dart`:

```dart
   6   /// The lines `watch` shows for [state]: one for each key, or one that says so
   7   /// before the first reading.
   8   List<String> watchLines(WatchState state) =>
   9       state.sensors.isEmpty ? const ['no readings yet'] : const [];
  10-    final latest =>
  10-      'x ${_fixed(latest.x)}  y ${_fixed(latest.y)}  '
  10-          'z ${_fixed(latest.z)}  ${state.count} readings',
  10-  };
   ⋮
  15   /// The terminal as the view: the lines for each state, redrawn in place on a
   ⋮
  24+    /// How many lines the last state took on the terminal.
  25+    int _shown = 0;
  26+
   ⋮
  33     /// Shows [state]: on a terminal, each line over the one it replaces.
   ⋮
  35+      final lines = watchLines(state);
   ⋮
  37         for (var i = 0; i < _shown; i++) {
  38           console.cursorUp();
  39+        }
  40+        for (final line in lines) {
  41+          console.eraseLine();
  42+          stdout.writeln(line);
  43+        }
  44+        _shown = lines.length;
   ⋮
  46         lines.forEach(stdout.writeln);
   ⋮
  50     /// Leaves the terminal ready for the next command, with the cursor shown.
  51-    /// new line.
  52       if (console.hasTerminal) console.showCursor();
  53-        console.showCursor();
  53-        stdout.writeln();
  53-      }
```

`cursorUp` and `eraseLine` are `dart_console`'s moves: one line up, and the current line cleared. The analyzer warns
that `_fixed` is unused, until the next step reads it.

Run it:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl -n 'line for each key'
```

```
⋮
  Expected: [
              'sensor/phone/accel  x   0.100  y   9.776  z   0.812  42 readings',
              'sensor/phone/gyro   x   0.000  y   0.000  z   0.500  7 readings'
            ]
    Actual: []
     Which: at location [0] is [] which shorter than expected
⋮
```

No line was made, because `watchLines` answers nothing for the keys.

**Write the obvious implementation.** One line for each entry of the map, a known step. The key comes first, padded
to the longest key in the state so that the readings line up, then the reading and the count as the line showed
them. Change `zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view.dart`:

```dart
   2+  import 'dart:math';
   ⋮
   7   /// The lines `watch` shows for [state]: one for each key, with the key, its
   8   /// latest reading to three decimals and its count, or one that says so before
   9   /// the first reading.
  10   List<String> watchLines(WatchState state) {
  11+    if (state.sensors.isEmpty) return const ['no readings yet'];
  12+    final width = state.sensors.keys.map((key) => key.length).reduce(max);
  13+    return [
  14+      for (final MapEntry(key: keyExpr, value: sensor) in state.sensors.entries)
  15+        _line(keyExpr.padRight(width), sensor),
  16+    ];
  17+  }
  18+
  19+  /// The line for one key: [key], its latest reading and its count.
  20+  String _line(String key, SensorState sensor) =>
  21+      '$key  x ${_fixed(sensor.latest.x)}  y ${_fixed(sensor.latest.y)}  '
  22+      'z ${_fixed(sensor.latest.z)}  ${sensor.count} readings';
```

`state.sensors.keys.map((key) => key.length).reduce(max)` is the length of the longest key, with `max` from
`dart:math`, and `padRight(width)` fills each key to that length with spaces. The line for one key is a function of
its own, `_line`, because the lint set refuses two adjacent string literals inside a list.

Run it again:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl -n 'line for each key'
```

It passes. Then run the file:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl/test/ui/watch/watch_view_test.dart
```

2 tests pass.

**Refactor.** Nothing to change. The line for one key is one function, the width is computed once, and the view
draws the list in one place.

**What the test guarantees:** `watch` shows, for each key a reading has arrived on, the key, its latest reading to
three decimals, and its count. The keys are padded so that the readings line up.

**Cycle 2 — the view model keeps the latest reading and the count for each key.** The test that said the view model
keeps the latest reading and counts them made a claim for one sensor, and its readings came without keys. It goes,
and this claim's test takes its place. The readings provider delivers three readings on two keys, each with its key.
Replace `zenoh_sensors/apps/sensorctl/test/ui/watch/watch_view_model_test.dart`:

```dart
import 'package:riverpod/riverpod.dart';
import 'package:sensor_core/sensor_core.dart';
import 'package:sensorctl/config/providers.dart';
import 'package:sensorctl/ui/watch/watch_view_model.dart';
import 'package:test/test.dart';

void main() {
  test(
    'the view model keeps the latest reading and the count for each key',
    () async {
      // A fake: the readings provider, overridden with three readings on two
      // keys, so nothing below the view model is built.
      const first = Reading(x: 0, y: 9.776, z: 0.812);
      const second = Reading(x: 0, y: 0, z: 0.5);
      const third = Reading(x: 0, y: 0, z: 9.81);
      final container = ProviderContainer.test(
        overrides: [
          readingsProvider.overrideWith(
            (ref) => Stream.fromIterable([
              (keyExpr: 'sensor/phone/accel', reading: first),
              (keyExpr: 'sensor/phone/gyro', reading: second),
              (keyExpr: 'sensor/phone/accel', reading: third),
            ]),
          ),
        ],
      )..listen(watchViewModelProvider, (_, _) {});

      // Let the three readings flow through before reading the state.
      await pumpEventQueue();

      // The code to implement: the view model's state, a latest reading and
      // a count for each key it heard.
      final state = container.read(watchViewModelProvider);

      // The claim: two readings counted on the accelerometer's key and the
      // third kept, one on the gyroscope's.
      final sensors = {
        for (final MapEntry(key: keyExpr, value: sensor)
            in state.sensors.entries)
          keyExpr: (
            latest: (sensor.latest.x, sensor.latest.y, sensor.latest.z),
            count: sensor.count,
          ),
      };
      expect(sensors, {
        'sensor/phone/accel': (latest: (0, 0, 9.81), count: 2),
        'sensor/phone/gyro': (latest: (0, 0, 0.5), count: 1),
      });
    },
  );
}
```

The claim is a map from each key to its latest reading and its count, the reading as a record of its three values.

The test overrides `readingsProvider` with a stream of pairs, and the provider yields a `Reading`. Two changes let the
test compile and leave the claim false:

1. `readingsProvider` yields each pair the repository hands on, key included.
2. The view model takes the reading out of each pair, as the provider did.

Change `zenoh_sensors/apps/sensorctl/lib/config/providers.dart`:

```dart
  27   /// The readings that arrive, each with its key, once the session is open.
  28   final readingsProvider = StreamProvider<KeyedReading>((ref) async* {
   ⋮
  30     yield* ref.watch(readingsRepositoryProvider).readings();
  31-        .watch(readingsRepositoryProvider)
  31-        .readings()
  31-        .map((keyed) => keyed.reading);
```

Change `zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view_model.dart`:

```dart
  42           state = WatchState(latest: value.reading, count: state.count + 1);
```

Run it:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl -n 'count for each key'
```

```
⋮
  Expected: {
              'sensor/phone/accel': ({int count, (int, int, double) latest}):(count: 2, latest: (0, 0, 9.81)),
              'sensor/phone/gyro': ({int count, (int, int, double) latest}):(count: 1, latest: (0, 0, 0.5))
            }
    Actual: {}
     Which: has different length and is missing map key 'sensor/phone/accel'
⋮
```

The state has no key, because the view model keeps one latest reading and one count.

**Write the obvious implementation.** The pair carries the key, and the state keeps a `SensorState` under it, as the
node's view model does. On each reading, the view model takes the key and the reading out of the pair. It reads the
count on that key so far, or 0 for a key it has not seen, and replaces its state with a new map. Change
`zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view_model.dart`:

```dart
  36   /// readings, for each key.
   ⋮
  42           final (:keyExpr, :reading) = value;
  43+          final count = state.sensors[keyExpr]?.count ?? 0;
  44+          state = WatchState(
  45+            sensors: {
  46+              ...state.sensors,
  47+              keyExpr: SensorState(latest: reading, count: count + 1),
  48+            },
  49+          );
```

A key already in the map keeps its place, so the lines stay in the order in which the keys first arrived.

Run it again:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl -n 'count for each key'
```

It passes.

**Refactor.** `latest` and `count` on `WatchState` have no reader now. They go, with the two comments that named
them. Change `zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_view_model.dart`:

```dart
  18   /// What `watch` shows: the state of each key.
  19-  /// state of each key.
  20     /// A state with [sensors] as the state of each key.
  21     const new({this.sensors = const {}});
  22-    const new({this.latest, this.count = 0, this.sensors = const {}});
  23     /// The state of each key a reading has arrived on, in the order the keys
  24     /// first arrived.
  25-
  25-    /// How many readings have arrived.
  25-    final int count;
  25-
  25-    /// The state of each key a reading has arrived on.
```

The tests pass before and after. `WatchCommand` does not change, because it hands each state to `show` as before.

**What the test guarantees:** the view model counts the readings on each key the collector receives, and keeps the
latest reading of each, in the order the keys first arrived.

**Run the outer test again.**

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'under its own key'
```

It passes. The program's pass changed nothing in the core. Then run every test of `sensorctl`:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl
```

3 tests pass. Nothing in them opens a session.

**What the tests guarantee:** `watch` shows a line for each key a reading has arrived on, with the key, its latest
reading to three decimals, and its count. Before the first reading it says so. The view model counts the readings
on each key and keeps the latest of each, in the order the keys first arrived, from the pairs the collector's
repository hands on.

`watch` shows both sensors, each under its key. Section 10 runs both programs, the node on the emulator and `watch`
on your laptop.

## 10 — On the emulator

Run the node on the emulator, and watch both sensors with `sensorctl`. You need three terminals: the node in the
first, the forward and `watch` in the second, and the device's moves in the third.

```
   first terminal        second terminal        third terminal           emulator
   ────────────────────  ─────────────────────  ───────────────────────  ─────────────────────
1  start the emulator                                                    boots
   cd apps/sensor_node
   flutter run                                                           runs the app
2  ┃                     adb forward
   ┃                     watch
   ┃                     ◉ two lines                                     ◉ two blocks
3  ┃                     ┃                      adb emu: turn it         its gyroscope turns
   ┃                     ◉ gyro z reads 0.500                            ◉ gyro z reads 0.500
   ┃                     ┃                      adb emu: hold it still   its gyroscope rests
4  ┃                     Ctrl-C
   ┃                     adb forward --remove
   q                                                                     the app stops
   cd ../..
5                                                                        you click its ×
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

The emulator shows two blocks, `sensor/phone/accel` and `sensor/phone/gyro`, each with its latest reading and its
count.

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
sensor/phone/accel  x   0.000  y   9.776  z   0.812  … readings
sensor/phone/gyro   x   0.000  y   0.000  z   0.000  … readings
```

Both lines change in place. The accelerometer's count climbs about 15 times a second. The gyroscope's climbs 5
times a second, the plugin's own period, because nothing else on the emulator asks the gyroscope for more.

> **In VS Code.** Pick the emulator in the status bar, and run **sensor_node** from Run and Debug in place of
> `flutter run`. After the forward, run **sensorctl watch** from the same list in place of `watch`.

**3. Turn the device.** Open a third terminal at the top folder, and tell the emulator that the device turns about
its z axis, half a radian a second:

```sh
# in zenoh_sensors
adb emu sensor set gyroscope 0:0:0.5
```

In the emulator and in the second terminal, within a second, the gyroscope's `z` changes, and the accelerometer's
line stays as it was:

```
sensor/phone/gyro   x   0.000  y   0.000  z   0.500  … readings
```

The emulator holds a value until the next `set`. In the third terminal, hold the device still again:

```sh
# in zenoh_sensors
adb emu sensor set gyroscope 0:0:0
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
> two terminals, without starting the emulator. In place of step 3, turn the phone in your hand. The gyroscope's
> values rise while it turns, and fall back to about 0 when you hold it still.

`watch` shows both sensors of the node. Section 11 pins one fact about the wildcard.

## 11 — What the tests pin, and what changed in the architecture

Pin one fact with a test, and look at what the chapter changed.

**1. Add one more test.** It passes at once, because it pins what zenoh already does. Add it to
`zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('a wildcard stands for one segment of a key', () async {
    // A pin about zenoh: what a subscription on sensor/phone/* leaves out.
    final sensorNode = ZenohService(SessionSettings.sensorNode());
    addTearDown(sensorNode.dispose);
    await sensorNode.open();
    final collectorNode = ZenohService(SessionSettings.collectorNode());
    addTearDown(collectorNode.dispose);
    await collectorNode.open();
    final subscription = collectorNode.declareSubscription('sensor/phone/*');
    final received = <String>[];
    subscription.samples.listen((sample) => received.add(sample.keyExpr));
    // The declaration travels to the node, so give it time to arrive.
    await Future<void>.delayed(delivery);

    // The node puts on a key of three segments, and on one of four.
    sensorNode.declarePublication('sensor/phone/gyro').put('in');
    sensorNode.declarePublication('sensor/phone/gyro/raw').put('too deep');
    // A put returns before the sample arrives, so give it time to cross.
    await Future<void>.delayed(delivery);

    // The claim: only the key of three segments arrives.
    expect(received, ['sensor/phone/gyro']);
  });
}
```

Run the whole core:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

32 tests pass.

**What the test guarantees:** a subscription on `sensor/phone/*` receives a key of three segments, and nothing put on
a longer key below it. A sensor that publishes on a longer key needs another key expression.

**2. Look at what changed in the architecture.** No layer is new. A second sensor went through every layer there
was:

```
NodeScreen  →  NodeViewModel  →  SensorNodeRepository  →  ZenohService + Publication  →  zenoh_dart
  a block        a state            a publication       ↘
  for each key   for each key       for each sensor       SensorService: accelerometer, gyroscope

WatchView   →  WatchViewModel  →  ReadingsRepository   →  ZenohService + Subscription  →  zenoh_dart
  a line         a state            one key expression     the key of each sample
  for each key   for each key       with a wildcard
```

Three rules start here.

**A sensor is a key.** The node gives each sensor its own key, `sensor/<node>/<sensor>`. The collector names none of
them, and `sensor/<node>/*` matches each one.

**The key travels with the reading.** The service hands up the key of each sample, both repositories hand on a
`KeyedReading`, and each view shows the key as zenoh delivered it.

**A contract that changes takes its tests first.** In sections 4 to 9 you wrote each test before the code it
describes, watched it fail, and then wrote the code.

**What comes next.** Chapter 5 takes the phone off the cable and onto your Wi-Fi.

The chapter's code is done. Section 12 commits it.

## 12 — Files and versions at the end of this chapter

**1. Commit.** Commit everything this chapter changed, in one commit:

```sh
# in zenoh_sensors
git add .
git commit -m "A second sensor: the gyroscope on sensor/phone/gyro, and watch on sensor/phone/*"
```

The commit holds 22 files, all of them changed and none new:

- 9 are the core's: the reading, the sensor's contract, the service, both repositories, and 4 files of tests.
- 8 are the app's: its providers, the device's service, the view model and the screen, and 4 files of tests.
- 5 are `sensorctl`'s: its providers, the view and the view model, and 2 tests.

> **In VS Code.** **View › Source Control** lists the same changes. Stage them with the **+** on the **Changes** line,
> type the message, and choose **Commit**.

`zenoh_sensors` now holds this, in six commits:

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

No tool or package is new, and no version changed.

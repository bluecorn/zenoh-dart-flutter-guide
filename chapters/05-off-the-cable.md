# 5 — Off the cable

## 1 — What you build, and what you will see

By the end of this chapter, the phone publishes over your Wi-Fi network, and nothing of zenoh goes through a cable.
Each program reads its session's settings from a file, and the file names the topology the program is in. You write
the files one topology at a time, and `watch` shows the same two lines under each:

```
Press Ctrl-C to stop.
sensor/phone/accel  x   0.000  y   9.776  z   0.812  153 readings
sensor/phone/gyro   x   0.000  y   0.000  z   0.000  51 readings
```

**The cable is for development.** A node that is deployed connects to a router: as a peer, when the other peers can
reach it, and as a client, when they cannot. Which one it is, and where the router is, are lines in the node's file.
You take the phone off the cable in three steps, each a pair of files:

1. **Peer to peer over Wi-Fi.** The node listens on every interface of the phone, and `watch` connects to the phone's
   address. Only the addresses change.
2. **Two peers that a router introduces.** `zenohd`, zenoh's router, runs on the laptop. The node and `watch` both
   connect to it, and neither knows the other's address. The router tells each about the other, they connect to each
   other, and the readings go over that connection. This is the shape the two programs keep.
3. **A client of the router.** The node connects to the router and listens nowhere. The router carries its readings
   to `watch`. This is the shape for a phone that nothing can reach.

```
 the phone, on your Wi-Fi                           the laptop
┌────────────────────────────────┐              ┌──────────────────────────────────────────┐
│ sensor_node                    │              │ zenohd                 sensorctl watch   │
│   a peer, listening on 7447    │              │   a router, on 7447      a peer          │
│                                │   connects   │     ▲                      │ connects   │
│   session ─────────────────────┼──────────────┼─────┴──────────────────────┘            │
│     ▲                          │              │     tells each about the other          │
│     │  connects to the phone   │              │                                          │
│     └──────────────────────────┼──────────────┼──────────────────── ReadingsRepository   │
│   put on sensor/phone/accel ═══╪══════════════╪══════════════════► subscribed to         │
│   put on sensor/phone/gyro     │   readings   │                    sensor/phone/*        │
└────────────────────────────────┘              └──────────────────────────────────────────┘
```

You build it in three parts. First the settings leave the code: both programs read a file, and the files hold what
the code held, so that every test of chapters 1 to 4 stays green. Then the app keeps the phone's screen on. Then the
three topologies, each with its files and its run on the phone.

You run `watch` with `fvm dart run sensorctl:sensorctl --config <file> watch`, and the node with `fvm flutter run -d
<your phone> --dart-define=ZENOH_CONFIG=<name>`. With no `--config` and no define, each program reads its
development file, which is the cable. Each layer that changes has its section:

```
sensor_node                          the app on the device
  main                               section 7: keeps the screen on
  providers                          section 6: the file's text into the service
    SettingsAssetService             section 6: reads the file the build names
      SessionSettings                section 4: holds the file's text
      ZenohService                   section 4: opens the session from the text

sensorctl watch                      what you type
  main, WatchCommand                 section 5: --config
  providers                          section 5: the file's text into the service
    SettingsFileService              section 5: reads the file --config names
      SessionSettings                section 4
      ZenohService                   section 4

config/ in each program              sections 5 and 6: development
                                     sections 8, 9 and 10: wifi-peer, router-peer, router-client
```

The chapter's claim is a phone and a laptop as two peers that a router introduces. You run it by hand, in sections
8 to 10, because `zenohd` is a program, and a test opens no program. The tests are of what you write: the files, the
services that read them, and what `mode` changes.

> **Zenoh guidance.** The chapter uses two of zenoh's three modes, `peer` and `client`, and runs the third, `router`,
> as `zenohd`. Section 3 shows the three with the package's examples. Sections 4 and 8 to 10 hold the zenoh
> settings, as files.

> **Flutter guidance.** The app changes in sections 6 and 7: a service that reads an asset, a build-time value that
> names which, and a plugin that keeps the screen on. From section 8 on, `flutter run` reaches the phone over Wi-Fi.

## 2 — What to read

| | page | what to take from it |
|---|---|---|
| [1] zenoh.io | [*Deployment*](https://zenoh.io/docs/getting-started/deployment/) | peer to peer, brokered and routed, with the configuration of each, and the two ways nodes find each other, multicast scouting and gossip scouting. It calls the connection between two nodes a session |
| [1] zenoh.io | [*Configuration*](https://zenoh.io/docs/manual/configuration/) | a configuration file is JSON5 or YAML, every key is optional, and `--cfg` changes one key of `zenohd` |
| zenoh's default configuration | [`DEFAULT_CONFIG.json5` at 1.8.0](https://github.com/eclipse-zenoh/zenoh/blob/1.8.0/DEFAULT_CONFIG.json5) | every key with its default and its comment. The files in this chapter name only the keys they change, and this file is the reference for the rest |
| [2] *The Zenoh Book* | [*Routing & Topology*](https://corsaro.me/zenoh/book/routing/): *Peer Mode*, *Client Mode*, *Router Mode* | when each mode fits, and what a client gives up against a peer |
| [2] *The Zenoh Book* | [*Getting Started → The Zenoh Router*](https://corsaro.me/zenoh/book/getting-started/router/) | `zenohd` for peers that cannot reach each other, and where it listens by default |
| [3] *Zenoh Programming in Rust* | [chapter 11, *Routing and Topology*](https://kydos.github.io/zenoh-book/chapter_11.html) | the three modes in one table, and two warnings: a client fails to open when none of its routers answers, and a router connected to no other router is an island. Skip *Multicast Scouting*, which is chapter 14 of this guide |
| Android's `adb` | [*Connect to a device over Wi-Fi*](https://developer.android.com/tools/adb#connect-to-a-device-over-wi-fi) | pairing a phone on Android 11 or newer with a code, then connecting to it, with no cable |
| `wakelock_plus` | [its documentation](https://pub.dev/packages/wakelock_plus) | one call keeps the screen on while the app shows |

## 3 — A router on your laptop

Run a router before you write a file for one. `zenohd` is zenoh's router, a program, and the package's examples
connect to it as they connect to each other. This guide was checked with `zenohd` 1.8.0, the version of zenoh the
package runs, and 1.8.0 or newer serves. Download the standalone archive for your machine from zenoh's release page,
[release 1.8.0](https://github.com/eclipse-zenoh/zenoh/releases/tag/1.8.0), unpack it, and put its folder on your
PATH.

**1. Start the router.** In a terminal at the top folder, start `zenohd` on the loopback with multicast scouting
off, and leave it running:

```sh
# in zenoh_sensors
zenohd --no-multicast-scouting -l tcp/127.0.0.1:7447
```

```
… INFO main ThreadId(01) zenohd: zenohd v1.8.0 built with rustc …
… INFO main ThreadId(01) zenohd: Initial conf: {…
… INFO main ThreadId(01) zenoh::net::runtime: Using ZID: …
… INFO main ThreadId(01) zenoh::net::runtime::orchestrator: Zenoh can be reached at: tcp/127.0.0.1:7447
```

`-l` and `--no-multicast-scouting` mean on `zenohd` what they mean on the examples. The second line is the router's
whole configuration, every key with its value, on one line. The third is its id. The last says where it listens.

**2. Ask who is there.** Open a second terminal at the top folder, and run `z_info` connecting to the router:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_info.dart \
  -e tcp/127.0.0.1:7447 --no-multicast-scouting --cfg 'listen/endpoints:[]'
```

```
⋮
…Opening session...
own id: …
routers ids:
…
peers ids:
… ERROR ThreadId(…) zenoh::api::admin: Unable to publish transport event: session closed
```

**`routers ids:` has one id.** It is the one `zenohd` printed after `Using ZID`. A router is a session in router
mode, and `z_info` connected to it as it connects to a peer. `peers ids:` is empty, because nothing else is
connected to the router.

**3. Two peers, introduced by the router.** In the second terminal, start `z_sub` as a peer that listens on port
7448 of the loopback, connects to the router, and has gossip scouting on, and leave it running:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_sub.dart \
  -l tcp/127.0.0.1:7448 -e tcp/127.0.0.1:7447 \
  --no-multicast-scouting --cfg 'scouting/gossip/enabled:true'
```

```
⋮
…Opening session...
Declaring Subscriber on 'demo/example/**'...
Press CTRL-C to quit...
```

Open a third terminal at the top folder, and start `z_pub` the same way, listening on port 7449, and leave it
running:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_pub.dart \
  -l tcp/127.0.0.1:7449 -e tcp/127.0.0.1:7447 \
  --no-multicast-scouting --cfg 'scouting/gossip/enabled:true'
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

In the second terminal, `z_sub` receives each put:

```
>> [Subscriber] Received PUT ('demo/example/zenoh-dart-pub': '[   0] Pub from Dart!')
>> [Subscriber] Received PUT ('demo/example/zenoh-dart-pub': '[   1] Pub from Dart!')
⋮
```

**Neither peer was told the other's address, and the puts arrive.** Each peer connected to the router alone. With
gossip scouting on, a node tells the nodes it is connected to about the nodes it knows. So the router told each
peer about the other, and each peer connected to the other's port. The puts go over that connection, from `z_pub` to
`z_sub`. The router introduced the two peers and carries none of their data.

Stop `z_pub` with Ctrl-C in the third terminal, and `z_sub` with Ctrl-C in the second.

**4. Start both again without gossip scouting.** The same two commands without their `--cfg` show what the
router does on its own. In the second terminal, start `z_sub` as a peer on port 7448, connected to the router, and
leave it running:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_sub.dart \
  -l tcp/127.0.0.1:7448 -e tcp/127.0.0.1:7447 --no-multicast-scouting
```

```
⋮
…Opening session...
Declaring Subscriber on 'demo/example/**'...
Press CTRL-C to quit...
```

`z_sub` is connected to the router, as its first run was. In the third terminal, start `z_pub` as a peer on port
7449, connected to the router, and leave it running:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_pub.dart \
  -l tcp/127.0.0.1:7449 -e tcp/127.0.0.1:7447 --no-multicast-scouting
```

```
⋮
…Opening session...
Declaring Publisher on 'demo/example/zenoh-dart-pub'...
Press CTRL-C to quit...
Putting Data ('demo/example/zenoh-dart-pub': '[   0] Pub from Dart!')...
⋮
```

**Nothing arrives at `z_sub`.** Both peers are connected to the router, and the router tells neither about the
other. A router does not carry data between two peers. Stop `z_pub` with Ctrl-C in the third terminal.

**5. Put through the router as a client.** In the third terminal, start `z_pub` as a client of the router:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_pub.dart \
  -m client -e tcp/127.0.0.1:7447 --no-multicast-scouting
```

```
⋮
…Opening session...
Declaring Publisher on 'demo/example/zenoh-dart-pub'...
Press CTRL-C to quit...
Putting Data ('demo/example/zenoh-dart-pub': '[   0] Pub from Dart!')...
⋮
```

In the second terminal, `z_sub` receives each put again:

```
>> [Subscriber] Received PUT ('demo/example/zenoh-dart-pub': '[   0] Pub from Dart!')
⋮
```

**A client's data goes through the router.** `-m client` sets `mode`, which every program in this guide has left at
`peer`. A client listens nowhere, so `--cfg 'listen/endpoints:[]'` is not needed, and it keeps one connection, to
the router. The router carries what the client puts to the peer that subscribed. `z_sub` kept gossip scouting off and
still received, because the router delivered the puts.

**6. Stop everything.** Stop `z_pub` with Ctrl-C in the third terminal, `z_sub` with Ctrl-C in the second, and
`zenohd` with Ctrl-C in the first.

The files you write in section 4 describe what every program in this guide has been so far, a peer and a peer on the
loopback. The files of sections 8 to 10 describe what this section ran.

## 4 — The settings leave the code

Move the settings out of the code in four cycles. `SessionSettings` holds the text of a zenoh configuration file, in
JSON5, and `ZenohService` opens its session from that text with the package's `Config.fromStr`. The core's tests read
their two configurations from two files under `packages/sensor_core/test/config/`, one for the sensor node and one
for a collector.

Each file names its mode and the keys that differ from zenoh's defaults, with a comment on each key. Every test of
chapters 1 to 4 stays green, so the change is a refactoring.

The tests, in the order you write them:

1. *neither side announces itself on the network*, read from the files
2. *the collector connects to where the sensor node listens*, read from the files
3. *a sensor node and a collector find each other on loopback*, from the files
4. *a client whose router does not answer does not open*, a new test

The first three stand in the service's test file with their claims, and each one's body changes. The fourth goes at
the end of the file.

**Cycle 1 — neither side announces itself on the network, read from the files.** The claim stands, and the test
reads each side's settings from a file. Zenoh parses the text, with `Config.fromStr`, and `get` reads one key back as
JSON text, so a string comes back in its quotes. Change `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
   6+  import '../support/settings.dart';
   ⋮
  56       // The code to implement: both sides' files, each parsed by zenoh and read
  57       // back as data. An absence cannot be watched, so it is checked as the
  58       // value behind it.
  59       final sensorConfig = Config.fromStr(sensorNodeSettings().json5);
  60+      final collectorConfig = Config.fromStr(collectorSettings().json5);
  61+      addTearDown(sensorConfig.dispose);
  62+      addTearDown(collectorConfig.dispose);
   ⋮
  65       for (final config in [sensorConfig, collectorConfig]) {
  66         expect(config.get('mode'), '"peer"');
  67         expect(config.get('scouting/multicast/enabled'), 'false');
  68         expect(config.get('scouting/gossip/enabled'), 'false');
```

A `Config` holds native memory, and one that no session consumes is disposed by hand, so each one is disposed when
the test ends.

The test asks for three things that do not exist yet:

- a constructor `SessionSettings(json5)`, which holds a file's text
- `sensorNodeSettings()` and `collectorSettings()`, which read the two files
- the two files

Give the settings the constructor, beside the two factories, with no endpoints, so that the test compiles. Change
`zenoh_sensors/packages/sensor_core/lib/src/services/session_settings.dart`:

```dart
   4     /// Settings read from a configuration file, whose text is [json5].
   5+    const new(this.json5)
   6+      : listenEndpoints = const [],
   7+        connectEndpoints = const [];
   8+
   9+    const new _({required this.listenEndpoints, required this.connectEndpoints})
  10+      : json5 = '';
   ⋮
  35+    /// The text of the session's configuration file, in JSON5.
  36+    final String json5;
  37+
```

The two readers are test support, because the core reads no file. `sensorctl` reads the file `--config` names in
section 5, and the app reads an asset in section 6. The tests run from the top folder, so the path starts at
`packages/`. Create `zenoh_sensors/packages/sensor_core/test/support/settings.dart`:

```dart
import 'dart:io';

import 'package:sensor_core/sensor_core.dart';

/// The sensor node's settings, read from its file, as the app reads its own.
SessionSettings sensorNodeSettings() => _read('sensor_node');

/// A collector's settings, read from its file, as sensorctl reads its own.
SessionSettings collectorSettings() => _read('collector');

/// The file called [name] under the core's test configuration. The path is
/// from the top folder, where the tests run.
SessionSettings _read(String name) => SessionSettings(
  File('packages/sensor_core/test/config/$name.json5').readAsStringSync(),
);
```

Create each file with an empty object, so that the test reaches its assertion. Create `zenoh_sensors/packages/sensor_core/test/config/sensor_node.json5`:

```json5
{}
```

Create `zenoh_sensors/packages/sensor_core/test/config/collector.json5`:

```json5
{}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'neither side announces'
```

```
⋮
  Expected: '"peer"'
    Actual: 'null'
⋮
```

The key reads back as `null`, because the file sets nothing. At open, zenoh takes its default for a key the file
leaves out: `peer` for the mode, and scouting on.

**Write the obvious implementation.** A configuration file is JSON5, so a key needs no quotes, a list or an object
can end in a comma, and a line can be a comment. Each file names its mode, although `peer` is the default. The mode
is the line that says which topology the file is in, and a reader who opens the file reads it there.

Each file names the two scouting keys, nested as zenoh's configuration nests them, and a comment above each key says
what the key does for this session. Replace `zenoh_sensors/packages/sensor_core/test/config/sensor_node.json5`:

```json5
// The sensor node's session.
{
  // A peer: it listens, and it connects to other peers itself.
  mode: "peer",
  scouting: {
    // No multicast scouting: the session asks the network for no one.
    multicast: { enabled: false },
    // No gossip scouting: no node tells this one about the nodes it knows.
    gossip: { enabled: false },
  },
}
```

Replace `zenoh_sensors/packages/sensor_core/test/config/collector.json5`:

```json5
// A collector's session.
{
  // A peer: it listens, and it connects to other peers itself.
  mode: "peer",
  scouting: {
    // No multicast scouting: the session asks the network for no one.
    multicast: { enabled: false },
    // No gossip scouting: no node tells this one about the nodes it knows.
    gossip: { enabled: false },
  },
}
```

Run it again. It passes. Then run the file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core/test/services/zenoh_service_test.dart
```

18 tests pass. The other 17 still take their settings from the factories.

**Refactor.** Nothing to change. The service still builds its configuration from the map, and cycle 3 moves it to
the text.

**What the test guarantees:** the two files make a peer of each side, with multicast scouting and gossip scouting
off, as zenoh reads them.

**Cycle 2 — the collector connects to where the sensor node listens, read from the files.** The claim stands, and
the test reads the endpoints back from the files. Change `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
  73       // The code to implement: the endpoints of both files, each parsed by
  74+      // zenoh and read back as data.
   ⋮
  76       final sensorConfig = Config.fromStr(sensorNodeSettings().json5);
  77       final collectorConfig = Config.fromStr(collectorSettings().json5);
  78+      addTearDown(sensorConfig.dispose);
  79+      addTearDown(collectorConfig.dispose);
   ⋮
  83       expect(sensorConfig.get('listen/endpoints'), address);
  84       expect(sensorConfig.get('connect/endpoints'), '[]');
  85       expect(collectorConfig.get('listen/endpoints'), '[]');
  86       expect(collectorConfig.get('connect/endpoints'), address);
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'connects to where'
```

```
⋮
  Expected: '["tcp/127.0.0.1:7447"]'
    Actual: '{"router":["tcp/[::]:7447"],"peer":["tcp/[::]:0"]}'
⋮
```

The sensor node's file names no `listen` key, so zenoh's default comes back, one list per mode. A router listens on
port 7447, and a peer on a port the system picks, both on every interface.

**Write the obvious implementation.** The sensor node listens on the loopback, and its file names no `connect` key,
because an empty list is zenoh's default. A collector listens nowhere, which is a key of its own, because a peer
listens by default, and it connects to the sensor node. Each new key gets its comment, and the head comment names
the whole. Replace `zenoh_sensors/packages/sensor_core/test/config/sensor_node.json5`:

```json5
// The sensor node's session: it listens on the loopback and connects nowhere.
{
  // A peer: it listens, and it connects to other peers itself.
  mode: "peer",
  scouting: {
    // No multicast scouting: the session asks the network for no one.
    multicast: { enabled: false },
    // No gossip scouting: no node tells this one about the nodes it knows.
    gossip: { enabled: false },
  },
  // Where the session listens: the loopback, port 7447.
  listen: { endpoints: ["tcp/127.0.0.1:7447"] },
}
```

Replace `zenoh_sensors/packages/sensor_core/test/config/collector.json5`:

```json5
// A collector's session: it listens nowhere and connects to the sensor node.
{
  // A peer: it listens, and it connects to other peers itself.
  mode: "peer",
  scouting: {
    // No multicast scouting: the session asks the network for no one.
    multicast: { enabled: false },
    // No gossip scouting: no node tells this one about the nodes it knows.
    gossip: { enabled: false },
  },
  // Where the session listens: nowhere, where a peer listens on every
  // interface by default.
  listen: { endpoints: [] },
  // Where the session connects: the sensor node, on the loopback.
  connect: { endpoints: ["tcp/127.0.0.1:7447"] },
}
```

Run it again. It passes. Then run the file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core/test/services/zenoh_service_test.dart
```

18 tests pass.

**Refactor.** Nothing to change. The two files now hold everything the map holds.

**What the test guarantees:** the sensor node's file listens at `tcp/127.0.0.1:7447` and connects nowhere, and the
collector's listens nowhere and connects to that address, as zenoh reads them.

**Cycle 3 — a sensor node and a collector find each other on loopback, from the files.** The claim stands, and the
two services open from the files' settings. Change `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
  12       final sensorNode = ZenohService(sensorNodeSettings());
  13       final collectorNode = ZenohService(collectorSettings());
   ⋮
  17       // The code to implement: two sessions opened from the files' text.
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'find each other on loopback'
```

```
⋮
  Expected: contains '…'
    Actual: []
⋮
```

The collector has no peers, because the service builds its configuration from the map, and settings read from a
file hold no endpoints in it.

**Write the obvious implementation.** The service hands the text to `Config.fromStr`, and the map is read nowhere.
Change `zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
 109     Config _config() => Config.fromStr(settings.json5);
 110-      final config = Config();
 110-      settings.asJson5.forEach(config.insertJson5);
 110-      return config;
 110-    }
```

Run it again. It passes. Then run the file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core/test/services/zenoh_service_test.dart
```

```
⋮
  ZenohException: Failed to create config from string (code: -2) [Z_EPARSE]
⋮
```

14 tests fail at the parse, with no assertion reached, because settings from a factory hold no text. Every test
that takes its settings from a factory now takes them from the files. That is 20 calls in the service's file and
5 in the two repository files, whose claim tests open real sessions. Change `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
  30       final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
  42       final sensorNode = ZenohService(sensorNodeSettings());
  43       final collectorNode = ZenohService(collectorSettings());
   ⋮
  91       final collectorNode = ZenohService(collectorSettings());
   ⋮
 102       final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
 114       final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
 122       final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
 154         final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
 186       final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
 200       final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
 212       final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
 218       final collectorNode = ZenohService(collectorSettings());
   ⋮
 245       final collectorNode = ZenohService(collectorSettings());
   ⋮
 261       final collectorNode = ZenohService(collectorSettings());
   ⋮
 273       final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
 279       final collectorNode = ZenohService(collectorSettings());
   ⋮
 308       final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
 314       final collectorNode = ZenohService(collectorSettings());
   ⋮
 340       final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
 343       final collectorNode = ZenohService(collectorSettings());
```

Change `zenoh_sensors/packages/sensor_core/test/repositories/readings_repository_test.dart`:

```dart
   6+  import '../support/settings.dart';
   ⋮
  11       final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
  17       final collectorNode = ZenohService(collectorSettings());
   ⋮
  90         final sensorNode = ZenohService(sensorNodeSettings());
   ⋮
  96         final collectorNode = ZenohService(collectorSettings());
```

Change `zenoh_sensors/packages/sensor_core/test/repositories/sensor_node_repository_test.dart`:

```dart
   9+  import '../support/settings.dart';
   ⋮
  14       final zenoh = ZenohService(sensorNodeSettings());
```

Run the core:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

32 tests pass.

**Refactor.** `SessionSettings` has callers for the text and none for the factories, the lists or the map, so it
keeps the text and nothing else. Change `zenoh_sensors/packages/sensor_core/lib/src/services/session_settings.dart`:

```dart
   1   /// A session's settings: the text of its zenoh configuration file, in JSON5.
   2   /// The file holds the keys that differ from zenoh's defaults, among them
   3+  /// where the session listens and where it connects, and its mode.
   ⋮
   6     const new(this.json5);
   7-      : listenEndpoints = const [],
   7-        connectEndpoints = const [];
   7-
   7-    const new _({required this.listenEndpoints, required this.connectEndpoints})
   7-      : json5 = '';
   7-
   7-    /// The sensor node's settings: it listens on the loopback and connects
   7-    /// nowhere.
   7-    factory sensorNode() => const SessionSettings._(
   7-      listenEndpoints: [nodeEndpoint],
   7-      connectEndpoints: [],
   7-    );
   7-
   7-    /// A collector's settings: it listens nowhere and connects to the sensor
   7-    /// node.
   7-    factory collectorNode() => const SessionSettings._(
   7-      listenEndpoints: [],
   7-      connectEndpoints: [nodeEndpoint],
   7-    );
   7-
   7-    /// Where the sensor node listens: the loopback, port 7447.
   7-    static const nodeEndpoint = 'tcp/127.0.0.1:7447';
   7-
   7-    /// The endpoints the session listens on.
   7-    final List<String> listenEndpoints;
   7-
   7-    /// The endpoints the session connects to.
   7-    final List<String> connectEndpoints;
   ⋮
  10-
  10-    /// The settings as the entries `Config.insertJson5` takes: a key path and a
  10-    /// JSON5 value each.
  10-    Map<String, String> get asJson5 => {
  10-      'mode': '"peer"',
  10-      'scouting/multicast/enabled': 'false',
  10-      'scouting/gossip/enabled': 'false',
  10-      'listen/endpoints': _json5List(listenEndpoints),
  10-      'connect/endpoints': _json5List(connectEndpoints),
  10-    };
  10-
  10-    static String _json5List(List<String> items) =>
  10-        '[${items.map((item) => '"$item"').join(', ')}]';
```

The collector witness opens from the collector's file. Change `zenoh_sensors/packages/sensor_core/test/support/collector.dart`:

```dart
   1-  import 'package:sensor_core/sensor_core.dart';
   2+
   3+  import 'settings.dart';
   ⋮
   7   Future<Session> openCollector() =>
   8       Session.open(config: Config.fromStr(collectorSettings().json5));
   9-    SessionSettings.collectorNode().asJson5.forEach(config.insertJson5);
   9-    return Session.open(config: config);
   9-  }
```

The fake service answers with settings that say nothing, as a fake should. Change `zenoh_sensors/packages/sensor_core/test/support/fakes.dart`:

```dart
  68     SessionSettings get settings => const SessionSettings('{}');
```

> **Dart guidance.** A string between `'''` runs over several lines and keeps its line breaks, so the text of a file
> can stand in Dart as it stands in the file.

Both programs called the factories, so each holds its text in its providers, the same keys as its file in the
core's tests. Section 5 moves `sensorctl`'s into a file, and section 6 the app's. Change `zenoh_sensors/apps/sensorctl/lib/config/providers.dart`:

```dart
   4   /// Which side of the topology this program is on: a collector, which listens
   5+  /// nowhere and connects to the sensor node on the loopback.
   ⋮
   7     (ref) => const SessionSettings('''
   8+  {
   9+    mode: "peer",
  10+    scouting: {
  11+      multicast: { enabled: false },
  12+      gossip: { enabled: false },
  13+    },
  14+    listen: { endpoints: [] },
  15+    connect: { endpoints: ["tcp/127.0.0.1:7447"] },
  16+  }
  17+  '''),
```

Change `zenoh_sensors/apps/sensor_node/lib/config/providers.dart`:

```dart
   5   /// Which side of the topology this app is on: the sensor node, which listens
   6+  /// on the loopback and connects nowhere.
   ⋮
   8     (ref) => const SessionSettings('''
   9+  {
  10+    mode: "peer",
  11+    scouting: {
  12+      multicast: { enabled: false },
  13+      gossip: { enabled: false },
  14+    },
  15+    listen: { endpoints: ["tcp/127.0.0.1:7447"] },
  16+  }
  17+  '''),
```

Run the core, and both programs' tests:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl
```

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node
```

32, 3 and 4 tests pass. The tests pass before and after. The core now knows no topology, and each program says
where it is.

**What the test guarantees:** a service opens its session from the text of a configuration file, and two sessions
opened from the two files find each other on the loopback.

**Cycle 4 — a client whose router does not answer does not open.** The test's text names three keys:

- `mode`, which every file so far has set to `peer`
- `connect`, where the router is, and nothing listens there, as nothing listens for the collector in *a collector
  opens even when no sensor node is listening*
- multicast scouting off, so that the client asks nothing but that address

Add the test at the end of the file. Change `zenoh_sensors/packages/sensor_core/test/services/zenoh_service_test.dart`:

```dart
⋮

  test('a client whose router does not answer does not open', () async {
    // A client of a router that is not there: nothing listens at the
    // address, as nothing listens for the collector that opens alone.
    final client = ZenohService(
      const SessionSettings('''
{
  mode: "client",
  scouting: { multicast: { enabled: false } },
  connect: { endpoints: ["tcp/127.0.0.1:7447"] },
}
'''),
    );
    addTearDown(client.dispose);

    // The claim: opening fails, where a peer's open returns with no peers.
    await expectLater(client.open(), throwsA(isA<ZenohException>()));
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'whose router does not answer'
```

It passes at once, because it pins what zenoh does with `mode`. A client gives up when its router does not answer,
and `open()` throws the package's `ZenohException`. A peer waits for `scouting/delay`, returns with no peers, and
zenoh keeps trying its connection. Then run the file:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core/test/services/zenoh_service_test.dart
```

19 tests pass.

The comment on `open()` says that every connection is retried, which is true of a peer. Make it say what a client
does. Change `zenoh_sensors/packages/sensor_core/lib/src/services/zenoh_service.dart`:

```dart
  66     /// Opens the session with the settings. For a peer, when this returns, each
  67     /// connection they ask for is made, or the wait set by `scouting/delay` has
  68     /// run out, and zenoh keeps retrying the connections not made yet. For a
  69+    /// client, this throws a `ZenohException` when no router answers.
```

**Refactor.** Nothing to change. The code of `open()` did not change.

**What the test guarantees:** a session in client mode opens only when its router answers, and the service lets the
failure through as a `ZenohException`.

**1. Run the loopback pair again.** The chapter's claim is run by hand in sections 8 to 10, so this section's
proof is the pair from chapter 1, green with the settings read from files:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core -n 'find each other on loopback'
```

It passes. Then run the whole core:

```sh
# in zenoh_sensors
fvm dart test packages/sensor_core
```

33 tests pass. The core holds no address, no port and no mode. Each program holds its text in its providers, and
section 5 takes `sensorctl`'s to a file that `--config` names.

**What the tests guarantee:** a session's settings are the text of a zenoh configuration file, and zenoh parses
that text. The file names its mode and the keys that differ from zenoh's defaults, and a key it leaves out takes the
default.

The core's two files make a peer of each side with both kinds of scouting off, the sensor node listening at
`tcp/127.0.0.1:7447` and the collector connecting to it. Two sessions opened from them find each other on the
loopback. A peer opens whether or not its connection is made. A client opens only when its router answers.

## 5 — `sensorctl` reads its file

Move `sensorctl`'s settings into a file in three cycles and one step. A service in the program's data layer,
`SettingsFileService`, reads a zenoh configuration file by its path and hands up its `SessionSettings`. The providers
take the settings from that service. The program takes one option before its command, `--config <path>`, and with
none it reads the file it ships, `apps/sensorctl/config/development.json5`, which holds the keys the providers held.

The tests, in the order you write them:

1. *the settings are the text of the file at the path*, the service's layer test
2. *the development file is a peer that connects to the sensor node*, the program's test of the file it ships
3. *every file sensorctl ships is a configuration zenoh accepts*, a pin on the folder, for the files of sections 8
   to 10

After the second test, the providers read the file, and the program gets `--config`. You add four files to
`apps/sensorctl`:

```
apps/sensorctl
├── config
│   └── development.json5
├── lib
│   └── data
│       └── services
│           └── settings_file_service.dart
└── test
    ├── config
    │   └── config_files_test.dart
    └── data
        └── services
            └── settings_file_service_test.dart
```

**Cycle 1 — the settings are the text of the file at the path.** The test writes a file of its own, in a folder of
its own under the system's temporary folder, and reads it back through the service. The file has a comment and a key,
as a configuration file has, so that the claim covers the whole text. Create
`zenoh_sensors/apps/sensorctl/test/data/services/settings_file_service_test.dart`:

```dart
import 'dart:io';

import 'package:sensorctl/data/services/settings_file_service.dart';
import 'package:test/test.dart';

void main() {
  test('the settings are the text of the file at the path', () {
    // A file made by the test, in a folder of its own that goes when the
    // test ends, with a comment and a key, as a configuration file has.
    final folder = Directory.systemTemp.createTempSync('sensorctl');
    addTearDown(() => folder.deleteSync(recursive: true));
    const text = '''
// A client of a router.
{ mode: "client" }
''';
    final file = File('${folder.path}/client.json5')..writeAsStringSync(text);

    // The code to implement: the file read by its path.
    final settings = SettingsFileService().read(file.path);

    // The claim: the settings hold the file's text, whole.
    expect(settings.json5, text);
  });
}
```

The test asks for one thing that does not exist yet, the service. Give it a `read` that answers with empty settings,
so that the test compiles and reaches its assertion. Create
`zenoh_sensors/apps/sensorctl/lib/data/services/settings_file_service.dart`:

```dart
import 'package:sensor_core/sensor_core.dart';

/// Reads a session's settings from a zenoh configuration file: the file
/// `--config` names, or the development file sensorctl ships.
class SettingsFileService {
  /// The settings in the file at [path]. A relative path starts at the folder
  /// the program runs in.
  SessionSettings read(String path) => const SessionSettings('');
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl -n 'text of the file'
```

```
⋮
  Expected: '// A client of a router.\n'
              '{ mode: "client" }\n'
              ''
    Actual: ''
⋮
```

The settings are empty, because the skeleton reads nothing.

**Write the obvious implementation.** The service reads the file at the path and hands its text up, untouched.
Change `zenoh_sensors/apps/sensorctl/lib/data/services/settings_file_service.dart`:

```dart
   1+  import 'dart:io';
   2+
   ⋮
  10     SessionSettings read(String path) =>
  11+        SessionSettings(File(path).readAsStringSync());
```

Run it again. It passes, and it is the file's only test.

**Refactor.** Nothing to change. The service is one line, and the core holds the text.

**What the test guarantees:** `sensorctl`'s settings are the text of the file at the path, whole, so what zenoh
parses is what the file says, comments included.

**Cycle 2 — the development file is a peer that connects to the sensor node.** The program's file holds the keys the
providers held. The test reads it through the service, as the program will, at the path the program reads with no
`--config`, and zenoh reads the two keys back. Create
`zenoh_sensors/apps/sensorctl/test/config/config_files_test.dart`:

```dart
import 'package:riverpod/riverpod.dart';
import 'package:sensorctl/config/providers.dart';
import 'package:sensorctl/data/services/settings_file_service.dart';
import 'package:test/test.dart';
import 'package:zenoh_dart/zenoh.dart';

void main() {
  test('the development file is a peer that connects to the sensor node', () {
    // The file the program reads with no --config, read as the program reads
    // it and parsed by zenoh.
    final path = ProviderContainer.test().read(configPathProvider);
    final config = Config.fromStr(SettingsFileService().read(path).json5);
    addTearDown(config.dispose);

    // The claim: a peer, which connects to the sensor node on the loopback.
    expect(config.get('mode'), '"peer"');
    expect(config.get('connect/endpoints'), '["tcp/127.0.0.1:7447"]');
  });
}
```

The test asks for two things that do not exist yet:

- `configPathProvider`, the path of the file the program reads, which is the development file's until `--config`
  names another
- the development file

Declare the provider with the path, above the settings. The program's tests run from the top folder, as its commands
do, so the path starts at `apps/`. Change `zenoh_sensors/apps/sensorctl/lib/config/providers.dart`:

```dart
   3+
   4+  /// The path of the session's configuration file, from the top folder: the
   5+  /// development file, unless `--config` names another.
   6+  final configPathProvider = Provider<String>(
   7+    (ref) => 'apps/sensorctl/config/development.json5',
   8+  );
```

Create the file with an empty object, so that the test reaches its assertion. Create
`zenoh_sensors/apps/sensorctl/config/development.json5`:

```json5
{}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl -n 'development file is a peer'
```

```
⋮
  Expected: '"peer"'
    Actual: 'null'
⋮
```

The file sets nothing.

**Write the obvious implementation.** The file holds the keys the provider holds, each with its comment. Its head
comment says what the program is while it develops: a collector on the laptop, with the sensor node on the cable.
Replace `zenoh_sensors/apps/sensorctl/config/development.json5`:

```json5
// sensorctl's session for development: a collector on the laptop, which
// listens nowhere and connects to the sensor node over the cable, on the
// loopback.
{
  // A peer: it listens, and it connects to other peers itself.
  mode: "peer",
  scouting: {
    // No multicast scouting: the session asks the network for no one.
    multicast: { enabled: false },
    // No gossip scouting: no node tells this one about the nodes it knows.
    gossip: { enabled: false },
  },
  // Where the session listens: nowhere, where a peer listens on every
  // interface by default.
  listen: { endpoints: [] },
  // Where the session connects: the sensor node, on the loopback.
  connect: { endpoints: ["tcp/127.0.0.1:7447"] },
}
```

Run it again. It passes, and it is the file's only test.

**Refactor.** The providers read the file. A provider holds the service, and the settings provider reads the file at
the path through it, so that the text stands in no Dart file. Change
`zenoh_sensors/apps/sensorctl/lib/config/providers.dart`:

```dart
   3+  import 'package:sensorctl/data/services/settings_file_service.dart';
   ⋮
  11   /// The service that reads the configuration file.
  12   final settingsFileServiceProvider = Provider<SettingsFileService>(
  13+    (ref) => SettingsFileService(),
  14+  );
  15+
  16+  /// Which side of the topology this program is on, as its configuration file
  17+  /// says.
   ⋮
  19     (ref) => ref
  20         .watch(settingsFileServiceProvider)
  21         .read(ref.watch(configPathProvider)),
  22-    scouting: {
  22-      multicast: { enabled: false },
  22-      gossip: { enabled: false },
  22-    },
  22-    listen: { endpoints: [] },
  22-    connect: { endpoints: ["tcp/127.0.0.1:7447"] },
  22-  }
  22-  '''),
```

Run the program's tests:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl
```

5 tests pass, before and after. They do not cover the providers, which have no test. Step 2 shows the program
reading a file.

**What the test guarantees:** the file `sensorctl` ships for development makes it a peer that connects to the sensor
node at `tcp/127.0.0.1:7447`, as zenoh reads it.

**1. Give the program `--config`.** A global option is one the runner takes before the command, as in `sensorctl
--config <path> watch`, and the `args` package keeps what it parsed in the command's `globalResults`. Add the option
to the runner. Change `zenoh_sensors/apps/sensorctl/bin/sensorctl.dart`:

```dart
  14+    runner.argParser.addOption(
  15+      'config',
  16+      help:
  17+          'The zenoh configuration file of the session, in place of the '
  18+          'development file.',
  19+      valueHelp: 'path',
  20+    );
```

The command reads the option, which is `null` when it was not given, and builds its container with the path in
place of the provider's. `overrideWithValue` replaces a provider's value for one container, as the tests' overrides
do. Change `zenoh_sensors/apps/sensorctl/lib/ui/watch/watch_command.dart`:

```dart
   7+  import 'package:sensorctl/config/providers.dart';
   ⋮
  22       final path = globalResults?.option('config');
  23+      final container = ProviderContainer(
  24+        overrides: [if (path != null) configPathProvider.overrideWithValue(path)],
  25+        retry: (retryCount, error) => null,
  26+      );
```

Ask the program for its usage:

```sh
# in zenoh_sensors
fvm dart run sensorctl:sensorctl --help
```

The global options list `--config=<path>` with its help, above the `watch` command.

**2. Run it with a file that listens.** `--config` is read if a file changes what the program is. The core's test
file for the sensor node makes a peer that listens on the loopback and connects nowhere. So `watch` with that file
waits to be found, and `z_info` finds it. In a terminal at the top folder, start `watch` with that file, and leave
it running:

```sh
# in zenoh_sensors
fvm dart run sensorctl:sensorctl \
  --config packages/sensor_core/test/config/sensor_node.json5 watch
```

```
⋮
…Press Ctrl-C to stop.
no readings yet
```

Open a second terminal at the top folder, and run `z_info` connecting to the port the file names:

```sh
# in zenoh_sensors
fvm dart run apps/sensorctl/example/z_info.dart \
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

**`peers ids:` has one id.** It is `watch`, listening where the file says. Stop `watch` with Ctrl-C in the first
terminal.

**Cycle 3 — every file sensorctl ships is a configuration zenoh accepts.** Sections 8 to 10 add three files to
`apps/sensorctl/config/`, and a file with a key out of place fails at open, under the program. The test lists the
folder and hands each file to zenoh. `returnsNormally` is the claim that a call throws nothing, and the `reason`
puts the file's path under the failure.

Add the test at the end of the file, and `dart:io` to its imports. Change
`zenoh_sensors/apps/sensorctl/test/config/config_files_test.dart`:

```dart
import 'dart:io';

import 'package:riverpod/riverpod.dart';
import 'package:sensorctl/config/providers.dart';
import 'package:sensorctl/data/services/settings_file_service.dart';
import 'package:test/test.dart';
import 'package:zenoh_dart/zenoh.dart';

⋮

  test('every file sensorctl ships is a configuration zenoh accepts', () {
    // The files in the folder the program ships, each read as the program
    // reads it. A folder with no file would pass for the wrong reason.
    final files = Directory('apps/sensorctl/config')
        .listSync()
        .whereType<File>();
    expect(files, isNotEmpty);

    // The claim: zenoh parses each one. A failure names its file.
    for (final file in files) {
      final settings = SettingsFileService().read(file.path);
      expect(
        () => Config.fromStr(settings.json5).dispose(),
        returnsNormally,
        reason: file.path,
      );
    }
  });
}
```

Run it:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl -n 'every file sensorctl ships'
```

It passes at once, because it pins what the folder holds, one file that cycle 2 made good. Then run the file:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl/test/config/config_files_test.dart
```

2 tests pass.

**Refactor.** Nothing to change. Both tests read a file through the service and hand its text to zenoh, and each
stays complete on its own, which is how a test is read.

**What the test guarantees:** every file in `apps/sensorctl/config/` is a configuration zenoh accepts, and a file it
rejects fails the test by its path.

**3. Run the program's tests.** Run them from the top folder:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl
```

6 tests pass. `sensorctl` holds no address, no port and no mode, and the app still holds its text in its providers.
Section 6 takes the app's to an asset that the build names.

**What the tests guarantee:** `sensorctl` reads its session's settings from a file, the one `--config` names or the
development file it ships. The text reaches zenoh as the file holds it. The development file makes a peer that
connects to the sensor node at `tcp/127.0.0.1:7447`, and every file in the program's folder is one zenoh accepts.

## 6 — The app reads its asset

Move the app's settings into a file in two cycles. A service in the app's data layer, `SettingsAssetService`, reads a
zenoh configuration file the app ships as an asset under `apps/sensor_node/config/` and hands up its
`SessionSettings`. Which file is a build-time value, `ZENOH_CONFIG`, and with no define the app reads
`development.json5`, which holds the keys the providers hold. The providers take the settings from the service.

The tests, in the order you write them:

1. *the settings are the text of the asset with the name*, the service's layer test
2. *each file the app ships is accepted, and development listens on loopback*, the app's test of the files it ships

After the second test, the providers read the asset. You add four files to `apps/sensor_node`:

```
apps/sensor_node
├── config
│   └── development.json5
├── lib
│   └── data
│       └── services
│           └── settings_asset_service.dart
└── test
    ├── config
    │   └── config_files_test.dart
    └── data
        └── services
            └── settings_asset_service_test.dart
```

> **Flutter guidance.** An asset is a file the build packs into the app, listed under `flutter: assets:` in the
> app's `pubspec.yaml`. A folder name ending in `/` ships every file in that folder. `rootBundle` is the bundle the
> app reads its assets from, by the path the list gives, and `loadString` returns a `Future`, because the bundle
> reads through the platform.

**Cycle 1 — the settings are the text of the asset with the name.** The test hands the service a bundle of its own,
with one asset under `config/`, and reads it back by its name. The asset has a comment and a key, as a configuration
file has, so that the claim covers the whole text.

A fake of the bundle goes in the fakes file. `AssetBundle` is Flutter's class, and `load` is the one member it
leaves to a subclass, so the fake answers `load` from a map, and `loadString` decodes what `load` returns. Change
`zenoh_sensors/apps/sensor_node/test/support/fakes.dart`:

```dart
   1+  import 'dart:convert';
   2+
   3+  import 'package:flutter/services.dart';
   ⋮
  14+
  15+  class FakeAssetBundle extends AssetBundle {
  16+    new(this.assets);
  17+
  18+    final Map<String, String> assets;
  19+
  20+    @override
  21+    Future<ByteData> load(String key) async =>
  22+        ByteData.sublistView(utf8.encode(assets[key]!));
  23+  }
```

Create `zenoh_sensors/apps/sensor_node/test/data/services/settings_asset_service_test.dart`:

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:sensor_node/data/services/settings_asset_service.dart';

import '../../support/fakes.dart';

void main() {
  test('the settings are the text of the asset with the name', () async {
    // A fake of the bundle: one asset under config/, with a comment and a
    // key, as a configuration file has.
    const text = '''
// A client of a router.
{ mode: "client" }
''';
    final bundle = FakeAssetBundle({'config/client.json5': text});

    // The code to implement: the asset read by its name.
    final settings = await SettingsAssetService(bundle: bundle).read('client');

    // The claim: the settings hold the asset's text, whole.
    expect(settings.json5, text);
  });
}
```

The test asks for one thing that does not exist yet, the service. Give it a `read` that answers with empty settings,
so that the test compiles and reaches its assertion. The service keeps the bundle it reads from, the app's unless a
test hands in another. Create `zenoh_sensors/apps/sensor_node/lib/data/services/settings_asset_service.dart`:

```dart
import 'package:flutter/services.dart';
import 'package:sensor_core/sensor_core.dart';

/// Reads a session's settings from a zenoh configuration file the app ships
/// as an asset under `config/`: the file the build names, or the development
/// file.
class SettingsAssetService {
  /// A service over the app's asset bundle, or over the bundle a test hands
  /// in.
  new({AssetBundle? bundle}) : bundle = bundle ?? rootBundle;

  /// The bundle the assets are read from.
  final AssetBundle bundle;

  /// The settings in the asset `config/<name>.json5`.
  Future<SessionSettings> read(String name) async => const SessionSettings('');
}
```

`flutter test` selects a test by `--name`. Run it:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node --name 'text of the asset'
```

```
⋮
  Expected: '// A client of a router.\n'
              '{ mode: "client" }\n'
              ''
    Actual: ''
⋮
```

The settings are empty, because the skeleton reads nothing.

**Write the obvious implementation.** The service loads the asset `config/<name>.json5` from its bundle and hands
the text up, untouched. Change `zenoh_sensors/apps/sensor_node/lib/data/services/settings_asset_service.dart`:

```dart
  16     Future<SessionSettings> read(String name) async =>
  17+        SessionSettings(await bundle.loadString('config/$name.json5'));
```

Run it again. It passes, and it is the file's only test.

**Refactor.** Nothing to change. The service is one line.

**What the test guarantees:** the app's settings are the text of the asset `config/<name>.json5`, whole, so what
zenoh parses is what the file says, comments included.

**Cycle 2 — each file the app ships is accepted, and development listens on loopback.** The app's file holds the
keys the providers hold. The test lists the folder, hands each file to zenoh, and reads two keys of the development
file back.

`flutter test` builds the assets of the project in the folder it runs in, the top folder. So the app's assets are not
in the test's bundle, and the test reads the folder from disk. `returnsNormally` is the claim that a call throws
nothing, and the `reason` puts the file's path under the failure. Create
`zenoh_sensors/apps/sensor_node/test/config/config_files_test.dart`:

```dart
import 'dart:io';

import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:sensor_node/config/providers.dart';
import 'package:zenoh_dart/zenoh.dart';

void main() {
  test(
    'each file the app ships is accepted, and development listens on loopback',
    () {
      // The files in the folder the app ships, read from disk, because a test
      // run from the top folder has no asset bundle of the app. A folder with
      // no file would pass for the wrong reason.
      final folder = Directory('apps/sensor_node/config');
      final files = folder.listSync().whereType<File>();
      expect(files, isNotEmpty);

      // The claim: zenoh parses each one. A failure names its file.
      for (final file in files) {
        expect(
          () => Config.fromStr(file.readAsStringSync()).dispose(),
          returnsNormally,
          reason: file.path,
        );
      }

      // The file the app reads with no define, parsed by zenoh.
      final name = ProviderContainer.test().read(configNameProvider);
      final text = File('${folder.path}/$name.json5').readAsStringSync();
      final config = Config.fromStr(text);
      addTearDown(config.dispose);

      // The claim: a peer, which listens on the loopback.
      expect(config.get('mode'), '"peer"');
      expect(config.get('listen/endpoints'), '["tcp/127.0.0.1:7447"]');
    },
  );
}
```

The test asks for three things that do not exist yet:

- `configNameProvider`, the name of the file the app reads, which is `development` until the build names another
- `zenoh_dart` among the app's dependencies, because the test hands each file to zenoh
- the development file

Declare the provider with the name, above the settings. `String.fromEnvironment` is a constant the build sets from
`--dart-define=ZENOH_CONFIG=<name>`, or the default when the build names none. Change
`zenoh_sensors/apps/sensor_node/lib/config/providers.dart`:

```dart
   4+
   5+  /// The name of the session's configuration file under `config/`: the build's
   6+  /// `ZENOH_CONFIG`, or `development` when the build names none.
   7+  final configNameProvider = Provider<String>(
   8+    (ref) =>
   9+        const String.fromEnvironment('ZENOH_CONFIG', defaultValue: 'development'),
  10+  );
```

`zenoh_dart` is a development dependency of the app, because only the test imports it. Replace
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
  zenoh_dart: ^1.0.0-rc.1

flutter:
  uses-material-design: true
```

Fetch it:

```sh
# in zenoh_sensors
fvm dart pub get
```

Create the file with an empty object, so that the test reaches its assertion. Create
`zenoh_sensors/apps/sensor_node/config/development.json5`:

```json5
{}
```

Run it:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node --name 'development listens'
```

```
⋮
  Expected: '"peer"'
    Actual: 'null'
⋮
```

Zenoh accepts the empty object, and the file sets nothing.

**Write the obvious implementation.** The file holds the keys the provider holds, each with its comment. Its head
comment says what the app is while it develops: the phone on the cable, listening on the loopback. Replace
`zenoh_sensors/apps/sensor_node/config/development.json5`:

```json5
// The sensor node's session for development: the phone on the cable, which
// listens on the loopback and connects nowhere.
{
  // A peer: it listens, and it connects to other peers itself.
  mode: "peer",
  scouting: {
    // No multicast scouting: the session asks the network for no one.
    multicast: { enabled: false },
    // No gossip scouting: no node tells this one about the nodes it knows.
    gossip: { enabled: false },
  },
  // Where the session listens: the loopback, port 7447.
  listen: { endpoints: ["tcp/127.0.0.1:7447"] },
}
```

Run it again. It passes, and it is the file's only test.

**Refactor.** The app ships the folder as assets, and the providers read the asset through the service, so that the
text stands in no Dart file. The folder goes under `flutter: assets:`. Replace
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
  zenoh_dart: ^1.0.0-rc.1

flutter:
  uses-material-design: true
  assets:
    - config/
```

A provider holds the service, and the settings provider reads the asset the name provider names through it. The
bundle answers with a `Future`, so the settings are a `FutureProvider`, and each provider that needs them awaits its
`future`. The service is made once the settings are read, the repository once the service is made, and the session
opens on the service. The readings await the session, then the repository. Change
`zenoh_sensors/apps/sensor_node/lib/config/providers.dart`:

```dart
   4+  import 'package:sensor_node/data/services/settings_asset_service.dart';
   ⋮
  13   /// The service that reads the configuration file.
  14   final settingsAssetServiceProvider = Provider<SettingsAssetService>(
  15     (ref) => SettingsAssetService(),
  16-    (ref) => const SessionSettings('''
  16-  {
  16-    mode: "peer",
  16-    scouting: {
  16-      multicast: { enabled: false },
  16-      gossip: { enabled: false },
  16-    },
  16-    listen: { endpoints: ["tcp/127.0.0.1:7447"] },
  16-  }
  16-  '''),
   ⋮
  18   /// Which side of the topology this app is on, as its configuration file says.
  19   final sessionSettingsProvider = FutureProvider<SessionSettings>(
  20     (ref) => ref
  21+        .watch(settingsAssetServiceProvider)
  22+        .read(ref.watch(configNameProvider)),
  23+  );
  24+
  25+  /// The app's one zenoh service, made once its settings are read, and disposed
  26+  /// with the container.
  27+  final zenohServiceProvider = FutureProvider<ZenohService>((ref) async {
  28+    final service = ZenohService(await ref.watch(sessionSettingsProvider.future));
   ⋮
  39   final sensorNodeRepositoryProvider = FutureProvider<SensorNodeRepository>(
  40     (ref) async => SensorNodeRepository(
  41       await ref.watch(zenohServiceProvider.future),
   ⋮
  48   final sessionProvider = FutureProvider<void>((ref) async {
  49     final service = await ref.watch(zenohServiceProvider.future);
  50     await service.open();
  51+  });
   ⋮
  57     yield* (await ref.watch(sensorNodeRepositoryProvider.future)).publish();
```

Run the app's tests:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node
```

6 tests pass, before and after. They do not cover the providers, which have no test. Sections 8 to 10 run the app
with the define, and the phone reads the file the build names.

**What the test guarantees:** the file the app ships for development makes it a peer that listens at
`tcp/127.0.0.1:7447`, as zenoh reads it. Every file in `apps/sensor_node/config/` is one zenoh accepts.

**1. Run the app's tests.** Run them from the top folder:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node
```

6 tests pass. The app holds no address, no port and no mode. Section 7 keeps the phone's screen on.

**What the tests guarantee:** the app reads its session's settings from an asset, the file the build names with
`ZENOH_CONFIG` or the development file it ships. The text reaches zenoh as the file holds it. The development file
makes a peer that listens at `tcp/127.0.0.1:7447`, and every file in the app's folder is one zenoh accepts.

## 7 — The screen stays on

Keep the phone's screen on while the app shows, in three steps and with no test. On the cable, the laptop reaches
the node on the phone's loopback, which Android leaves to an app that is off screen. From section 8 the readings go
over Wi-Fi, and Android takes the network from an app about five seconds after it stops being the visible app.

A phone that goes dark on the desk locks, its app leaves the screen, and the node stops publishing. The
`wakelock_plus` package keeps the screen on. It is a developer's convenience for the desk. No test sees the
screen, so you add the package, change `main`, and run the app's tests, which stay green.

**The lock has a limit.** When you lock the phone or switch to another app, the app leaves the screen, Android takes
its network, and the lock cannot keep it. Chapter 12's reconnect handles that.

**1. Add the package.** `wakelock_plus` is a plugin, a package with code on the Android side, and the app is its only
user. This guide was checked with 1.8.1. Replace `zenoh_sensors/apps/sensor_node/pubspec.yaml`:

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
  wakelock_plus: ^1.8.1

dev_dependencies:
  flutter_test:
    sdk: flutter
  very_good_analysis: ^11.0.0
  zenoh_dart: ^1.0.0-rc.1

flutter:
  uses-material-design: true
  assets:
    - config/
```

Fetch it:

```sh
# in zenoh_sensors
fvm dart pub get
```

The lock file gains the plugin and the packages it depends on.

**2. Enable the lock in `main`.** `WakelockPlus.enable()` asks the Android side to keep the screen on while the app
shows. A plugin call reaches the platform through Flutter's binding, and a call before `runApp` needs the binding
initialized first, so `WidgetsFlutterBinding.ensureInitialized()` comes before it. Change
`zenoh_sensors/apps/sensor_node/lib/main.dart`:

```dart
   1+  import 'dart:async';
   2+
   ⋮
   7+  import 'package:wakelock_plus/wakelock_plus.dart';
   ⋮
  10+    WidgetsFlutterBinding.ensureInitialized();
  11+    unawaited(WakelockPlus.enable());
```

`enable()` returns a `Future`, and `main` has nothing to do with its result, so the app starts while the platform
sets the flag. `unawaited`, from `dart:async`, says so to the analyzer, whose `discarded_futures` rule reports a
`Future` dropped without it.

> **Flutter guidance.** On Android the plugin sets the window's `FLAG_KEEP_SCREEN_ON`, which needs no permission.
> It remembers what it was asked, and sets the flag again when an activity attaches, so the lock survives the app's
> lifecycle transitions.

**3. Run the app's tests.** Run them from the top folder:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node
```

6 tests pass. They do not cover `main`, which has no test. The screen is the check, in section 8, where the phone
stays lit on the desk while `watch` counts its readings.

**What the tests guarantee:** nothing new. The app reads its settings, publishes and shows its readings as it did,
and the lock is one call in `main` before the app starts.

## 8 — Over Wi-Fi, peer to peer

Take the phone off the cable with two files. The node listens on every interface of the phone, and `watch` connects
to the phone's address on your Wi-Fi network. Nothing else changes.

**No VPN while the two are on Wi-Fi.** Turn off any VPN on the phone and on the laptop. On the network this chapter
was checked with, the laptop could not reach the phone while a VPN was active on the phone. Once the VPN was off, it
could.

**1. Pair `adb` over Wi-Fi.** `adb` reaches the phone over Wi-Fi from Android 11 on, paired once with a code. On the
phone, open **Settings › System › Developer options › Wireless debugging**, turn it on, and allow it on this network.
The screen shows the phone's address on the network and a port.

Tap **Pair device with pairing code**. The phone shows a second port, the pairing one, and a six-digit code. In a
terminal at the top folder, pair with that port:

```sh
# in zenoh_sensors
adb pair <the phone's address>:<the pairing port>
```

Type the code when `adb` asks. It prints `Successfully paired`. Then connect with the port the Wireless debugging
screen shows above the pairing dialog:

```sh
# in zenoh_sensors
adb connect <the phone's address>:<the port>
```

It prints `connected to` and the address. From here on `adb` and `flutter run` reach the phone with no cable, and
`fvm flutter devices` lists the phone by that address and port. The pairing is kept: next time, the phone and the
laptop connect on their own when both are on the network, and if they do not, `adb connect` alone does it.

**2. Make the node listen on every interface.** The sensor node's file for Wi-Fi differs from its development file
in one line. Create `zenoh_sensors/apps/sensor_node/config/wifi-peer.json5`:

```json5
// The sensor node's session on Wi-Fi, peer to peer: it listens on every
// interface of the phone, and the collector connects to it.
{
  // A peer: it listens, and it connects to other peers itself.
  mode: "peer",
  scouting: {
    // No multicast scouting: the session asks the network for no one.
    multicast: { enabled: false },
    // No gossip scouting: no node tells this one about the nodes it knows.
    gossip: { enabled: false },
  },
  // Where the session listens: port 7447 on every interface of the phone,
  // its Wi-Fi address included.
  listen: { endpoints: ["tcp/0.0.0.0:7447"] },
}
```

**What listening on every interface exposes.** `0.0.0.0` is every address the phone has, so anything on the
network can connect to the node's port, and nothing in this file asks who it is. The loopback allowed only the
phone itself. Chapter 14 is about who may connect.

**3. Point the collector at the phone.** The collector's file names the phone's address, the one the Wireless
debugging screen shows. The file ships with an address from the range kept for documentation, `192.0.2.20`, and
you replace it with yours. Create `zenoh_sensors/apps/sensorctl/config/wifi-peer.json5`:

```json5
// sensorctl's session on Wi-Fi, peer to peer: a collector on the laptop,
// which listens nowhere and connects to the phone's address.
{
  // A peer: it listens, and it connects to other peers itself.
  mode: "peer",
  scouting: {
    // No multicast scouting: the session asks the network for no one.
    multicast: { enabled: false },
    // No gossip scouting: no node tells this one about the nodes it knows.
    gossip: { enabled: false },
  },
  // Where the session listens: nowhere, where a peer listens on every
  // interface by default.
  listen: { endpoints: [] },
  // Where the session connects: the phone's address on your Wi-Fi network.
  // Replace 192.0.2.20 with the address the phone's Wireless debugging
  // screen shows.
  connect: { endpoints: ["tcp/192.0.2.20:7447"] },
}
```

Replace `192.0.2.20` with your phone's address, and save the file. Run both programs' tests:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl
```

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node
```

6 and 6 tests pass. The two tests of the folders read the new files, and zenoh accepts both.

**4. Two launch entries.** A define and a `--config` are arguments, and a Run and Debug entry carries them. Replace
`zenoh_sensors/.vscode/launch.json`:

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
      "name": "sensorctl watch",
      "type": "dart",
      "request": "launch",
      "program": "${workspaceFolder}/apps/sensorctl/bin/sensorctl.dart",
      "cwd": "${workspaceFolder}",
      "args": ["watch"]
    },
    {
      "name": "sensorctl watch (wifi-peer)",
      "type": "dart",
      "request": "launch",
      "program": "${workspaceFolder}/apps/sensorctl/bin/sensorctl.dart",
      "cwd": "${workspaceFolder}",
      "args": ["--config", "apps/sensorctl/config/wifi-peer.json5", "watch"]
    },
    {
      "name": "sensor_node",
      "type": "dart",
      "request": "launch",
      "program": "${workspaceFolder}/apps/sensor_node/lib/main.dart",
      "cwd": "${workspaceFolder}/apps/sensor_node"
    },
    {
      "name": "sensor_node (wifi-peer)",
      "type": "dart",
      "request": "launch",
      "program": "${workspaceFolder}/apps/sensor_node/lib/main.dart",
      "cwd": "${workspaceFolder}/apps/sensor_node",
      "toolArgs": ["--dart-define=ZENOH_CONFIG=wifi-peer"]
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

`toolArgs` are arguments for `flutter run` itself, where `args` are the program's. Sections 9 and 10 change one
word in each of the two new entries.

**5. Run the node over Wi-Fi.** You need two terminals: the node in the first, `watch` in the second. The phone
stays unlocked with the app on screen.

```
   first terminal                    second terminal                   phone
   ────────────────────────────────  ────────────────────────────────  ─────────────────────
1  cd apps/sensor_node
   flutter run -d <phone> wifi-peer                                    runs the app, lit
2  ┃                                 watch --config wifi-peer
   ┃                                 ◉ two lines
3  ┃                                 ◉ the values follow              you tilt it
4  ┃                                 Ctrl-C
   q                                                                   the app stops
   cd ../..
```

In the first terminal, go into the app's folder and run the app on the phone with the Wi-Fi file:

```sh
# in zenoh_sensors
cd apps/sensor_node
```

```sh
# in zenoh_sensors/apps/sensor_node
fvm flutter run -d <the phone's address>:<the port> --dart-define=ZENOH_CONFIG=wifi-peer
```

The app appears on the phone with your numbers, and the screen stays lit. Open a second terminal at the top folder,
and start `watch` with its Wi-Fi file, and leave it running. Its output below shows only the readings:

```sh
# in zenoh_sensors
fvm dart run sensorctl:sensorctl --config apps/sensorctl/config/wifi-peer.json5 watch
```

```
⋮
sensor/phone/accel  x   …  y   …  z   …  … readings
sensor/phone/gyro   x   …  y   …  z   …  … readings
```

Tilt the phone, and the values follow. The readings cross your Wi-Fi network from the phone's address to the laptop,
and no cable and no forward is in the way.

**6. Stop.** In the second terminal, press Ctrl-C to stop `watch`. Zenoh 1.8.0 prints its `ERROR` line, because the
session was connected when it closed. In the first terminal, press `q` to stop the app, and go back to the top
folder:

```sh
# in zenoh_sensors/apps/sensor_node
cd ../..
```

> **In VS Code.** The status bar lists the phone by its address once `adb` is connected to it. Pick it, and run
> **sensor_node (wifi-peer)** from Run and Debug in place of `flutter run`, then **sensorctl watch (wifi-peer)** in
> place of `watch`.

The phone publishes over Wi-Fi, and `watch` found it by its address. Section 9 puts a router between them, so that
`watch` types no address at all.

## 9 — Through a router, as peers

Run `zenohd` on the laptop, and connect both programs to it as peers. The router tells each about the other, they
connect to each other, and the readings go over that connection. `watch` names the router on its own laptop and
nothing else.

**1. Connect the node to the router.** The node's file names the router's address, the laptop's on your Wi-Fi
network, with a placeholder you replace. It turns gossip scouting on, so that the router can tell it about the
collector, and it listens on every interface still, so that the collector can connect to it. Create
`zenoh_sensors/apps/sensor_node/config/router-peer.json5`:

```json5
// The sensor node's session through a router, as a peer: it connects to the
// router on the laptop, which tells it about the collector, and it listens so
// that the collector can connect to it.
{
  // A peer: it listens, and it connects to other peers itself.
  mode: "peer",
  scouting: {
    // No multicast scouting: the session asks the network for no one.
    multicast: { enabled: false },
    // Gossip scouting on: the router tells this session about the peers it
    // knows, and this session connects to them.
    gossip: { enabled: true },
  },
  // Where the session listens: port 7447 on every interface of the phone.
  listen: { endpoints: ["tcp/0.0.0.0:7447"] },
  // Where the session connects: the router on the laptop. Replace 192.0.2.10
  // with your laptop's address on the Wi-Fi network.
  connect: { endpoints: ["tcp/192.0.2.10:7447"] },
}
```

Replace `192.0.2.10` with your laptop's address on the Wi-Fi network. `ip addr` prints it, on the `inet` line of
your Wi-Fi interface, and so do the network settings.

**2. Connect the collector to the router.** The collector's file connects to the laptop's own port 7447, where the
router listens, with gossip scouting on. It listens nowhere: the router tells it the phone's address, and it
connects to the phone. Create `zenoh_sensors/apps/sensorctl/config/router-peer.json5`:

```json5
// sensorctl's session through a router, as a peer: it connects to the router
// on this laptop, which tells it about the sensor node, and it connects to
// the node itself.
{
  // A peer: it listens, and it connects to other peers itself.
  mode: "peer",
  scouting: {
    // No multicast scouting: the session asks the network for no one.
    multicast: { enabled: false },
    // Gossip scouting on: the router tells this session about the peers it
    // knows, and this session connects to them.
    gossip: { enabled: true },
  },
  // Where the session listens: nowhere, where a peer listens on every
  // interface by default.
  listen: { endpoints: [] },
  // Where the session connects: the router, on this laptop.
  connect: { endpoints: ["tcp/127.0.0.1:7447"] },
}
```

Run both programs' tests:

```sh
# in zenoh_sensors
fvm dart test apps/sensorctl
```

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node
```

6 and 6 tests pass. In `zenoh_sensors/.vscode/launch.json`, change `wifi-peer` to `router-peer` in the two entries
that name it, in their names and in their arguments.

**3. Run the three programs.** You need three terminals: the router in the first, the node in the second, `watch`
in the third.

```
   first terminal        second terminal                   third terminal                     phone
   ────────────────────  ────────────────────────────────  ─────────────────────────────────  ──────────────────
1  zenohd
   ┃
2  ┃                     cd apps/sensor_node
   ┃                     flutter run -d <phone> router-peer                                    runs the app, lit
3  ┃                     ┃                                 watch --config router-peer
   ┃                     ┃                                 ◉ two lines
4  ┃                     ┃                                 ◉ the values follow                you tilt it
5  ┃                     ┃                                 Ctrl-C
   ┃                     q                                                                     the app stops
   ┃                     cd ../..
   Ctrl-C
```

In the first terminal, start the router on every interface of the laptop, with multicast scouting off, and leave it
running:

```sh
# in zenoh_sensors
zenohd --no-multicast-scouting -l tcp/0.0.0.0:7447
```

```
… INFO main ThreadId(01) zenohd: zenohd v1.8.0 built with rustc …
… INFO main ThreadId(01) zenohd: Initial conf: {…
… INFO main ThreadId(01) zenoh::net::runtime: Using ZID: …
… INFO main ThreadId(01) zenoh::net::runtime::orchestrator: Zenoh can be reached at: tcp/…:7447
⋮
```

The router listens on every address the laptop has, so anything on the network can connect to it, and it accepts
any peer or client. Chapter 14 is about who may connect.

In the second terminal, go into the app's folder and run the app on the phone with the router file:

```sh
# in zenoh_sensors
cd apps/sensor_node
```

```sh
# in zenoh_sensors/apps/sensor_node
fvm flutter run -d <the phone's address>:<the port> --dart-define=ZENOH_CONFIG=router-peer
```

In the first terminal, the router prints a line for the connection the phone opened. Open a third terminal at the
top folder, and start `watch` with its router file, and leave it running:

```sh
# in zenoh_sensors
fvm dart run sensorctl:sensorctl --config apps/sensorctl/config/router-peer.json5 watch
```

```
⋮
sensor/phone/accel  x   …  y   …  z   …  … readings
sensor/phone/gyro   x   …  y   …  z   …  … readings
```

Tilt the phone, and the values follow. `watch` typed no address of the phone. It connected to the router, the
router named the phone to it, and it connected to the phone. The readings go from the phone to `watch` over that
connection, and the router carries none of them.

**4. Stop.** In the third terminal, press Ctrl-C to stop `watch`. In the second, press `q` to stop the app, and go
back to the top folder:

```sh
# in zenoh_sensors/apps/sensor_node
cd ../..
```

In the first terminal, press Ctrl-C to stop the router.

> **In VS Code.** Start the router in a VS Code terminal. Then run **sensor_node (router-peer)** from Run and Debug
> in place of `flutter run`, and **sensorctl watch (router-peer)** in place of `watch`.

The two programs are peers that a router introduced, which is the shape they keep. Section 10 shows the one other
way through the router.

## 10 — As a client of the router

Make the node a client. A client keeps one connection, to the router, and the router carries its readings. The
phone listens nowhere, so nothing on the network needs to reach it, and gossip scouting plays no part, because a
client takes none.

**1. Write the node's client file.** It names the router's address, with the placeholder you replace, and no listen
key, because a client listens nowhere by default. Create `zenoh_sensors/apps/sensor_node/config/router-client.json5`:

```json5
// The sensor node's session as a client of the router on the laptop: one
// connection, to the router, which carries the readings; the phone listens
// nowhere, so nothing needs to reach it.
{
  // A client: it connects to a router, and the router routes for it.
  mode: "client",
  scouting: {
    // No multicast scouting: the session asks the network for no one.
    multicast: { enabled: false },
  },
  // Where the session connects: the router on the laptop. Replace 192.0.2.10
  // with your laptop's address on the Wi-Fi network.
  connect: { endpoints: ["tcp/192.0.2.10:7447"] },
}
```

Replace `192.0.2.10` with your laptop's address. The collector keeps its router file, because a router carries data
between a client and a peer. Run the app's tests:

```sh
# in zenoh_sensors
fvm flutter test apps/sensor_node
```

6 tests pass, and the new file is one zenoh accepts. In `zenoh_sensors/.vscode/launch.json`, change `router-peer` to
`router-client` in the **sensor_node** entry that names it.

**2. Run the three programs.** The same three terminals as in section 9, with the node's file changed. In the first
terminal, start the router and leave it running:

```sh
# in zenoh_sensors
zenohd --no-multicast-scouting -l tcp/0.0.0.0:7447
```

In the second terminal, run the app on the phone with the client file:

```sh
# in zenoh_sensors
cd apps/sensor_node
```

```sh
# in zenoh_sensors/apps/sensor_node
fvm flutter run -d <the phone's address>:<the port> --dart-define=ZENOH_CONFIG=router-client
```

In the third terminal, start `watch` with its router file, and leave it running:

```sh
# in zenoh_sensors
fvm dart run sensorctl:sensorctl --config apps/sensorctl/config/router-peer.json5 watch
```

```
⋮
sensor/phone/accel  x   …  y   …  z   …  … readings
sensor/phone/gyro   x   …  y   …  z   …  … readings
```

Tilt the phone, and the values follow. The readings go from the phone to the router and from the router to
`watch`. The phone's address appears nowhere, on the laptop or in the router.

**3. Stop.** Press Ctrl-C in the third terminal, `q` in the second, and Ctrl-C in the first, and go back to the top
folder:

```sh
# in zenoh_sensors/apps/sensor_node
cd ../..
```

**A client needs its router.** Start the app with the client file while the router is stopped, and the session
does not open. The app shows no readings, because the error reaches the session's provider and the screen shows
nothing of it. Chapter 12 changes that. A peer in the same place opens and waits.

> **In VS Code.** Run **sensor_node (router-client)** from Run and Debug in place of `flutter run`. The rest is as
> in section 9.

The three files are the three ways the phone talks to the laptop, and a file is all that changes between them.

## 11 — What changed in the architecture

One layer gained a class on each side, and one class in the core changed what it holds:

```
sensor_node                                     sensorctl
  providers                                       providers
    SettingsAssetService  →  rootBundle              SettingsFileService  →  dart:io
      config/<name>.json5, name from ZENOH_CONFIG      config/<file>.json5, path from --config
    SessionSettings: the file's text                 SessionSettings: the file's text
    ZenohService: Config.fromStr(text)               ZenohService: Config.fromStr(text)
```

Three rules start here.

**A session's settings are a file.** `SessionSettings` holds the text of a zenoh configuration file and nothing
else, and `ZenohService` hands that text to zenoh whole. The core names no address, no port and no mode.

**Each program reads its file in its data layer.** A service reads it, the app's from the asset bundle and
`sensorctl`'s from a path, and the provider graph hands the settings up. The view model and the view see no
configuration.

**A topology is a file, chosen at launch.** The app's `ZENOH_CONFIG` and `sensorctl`'s `--config` name the file,
and each program ships a development file that it reads when nothing names another. A new topology is a new file,
and no Dart changes.

**What comes next.** Chapter 6 puts a reading on the wire as bytes, with a sequence number, a timestamp and the three
values, in place of the text.

The chapter's code is done. Section 12 commits it.

## 12 — Files and versions at the end of this chapter

**1. Commit.** Commit everything this chapter changed, in one commit:

```sh
# in zenoh_sensors
git add .
git commit -m "Off the cable: the session's settings in files, and the phone over Wi-Fi, peer to peer and through zenohd"
```

The commit holds 32 files, 16 of them new:

- 10 are the core's: `SessionSettings` and the service changed, 2 configuration files and a reader for its tests,
  new, and 5 files of tests changed.
- 11 are the app's: a settings service and its test, a test of its files and 4 configuration files, new, and `main`,
  the providers, the fakes and the pubspec changed.
- 9 are `sensorctl`'s: a settings service and its test, a test of its files and 3 configuration files, new, and
  `main`, the command and the providers changed.
- `.vscode/launch.json`, with its two new entries, and the lock file.

> **In VS Code.** **View › Source Control** lists the same changes. Stage them with the **+** on the **Changes** line,
> type the message, and choose **Commit**.

`zenoh_sensors` now holds this, in seven commits:

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
│   │   ├── config/
│   │   │   ├── development.json5
│   │   │   ├── router-client.json5
│   │   │   ├── router-peer.json5
│   │   │   └── wifi-peer.json5
│   │   ├── lib/
│   │   │   ├── config/
│   │   │   │   └── providers.dart
│   │   │   ├── data/
│   │   │   │   └── services/
│   │   │   │       ├── device_sensor_service.dart
│   │   │   │       └── settings_asset_service.dart
│   │   │   ├── main.dart
│   │   │   └── ui/
│   │   │       └── node/
│   │   │           ├── node_screen.dart
│   │   │           └── node_view_model.dart
│   │   ├── pubspec.yaml
│   │   ├── README.md
│   │   ├── sensor_node.iml
│   │   └── test/
│   │       ├── config/
│   │       │   └── config_files_test.dart
│   │       ├── data/
│   │       │   └── services/
│   │       │       ├── device_sensor_service_test.dart
│   │       │       └── settings_asset_service_test.dart
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
│       ├── config/
│       │   ├── development.json5
│       │   ├── router-peer.json5
│       │   └── wifi-peer.json5
│       ├── example/
│       │   ├── common_args.dart
│       │   ├── z_info.dart
│       │   ├── z_pub.dart
│       │   ├── z_put.dart
│       │   └── z_sub.dart
│       ├── lib/
│       │   ├── config/
│       │   │   └── providers.dart
│       │   ├── data/
│       │   │   └── services/
│       │   │       └── settings_file_service.dart
│       │   └── ui/
│       │       └── watch/
│       │           ├── watch_command.dart
│       │           ├── watch_view.dart
│       │           └── watch_view_model.dart
│       ├── pubspec.yaml
│       ├── README.md
│       └── test/
│           ├── config/
│           │   └── config_files_test.dart
│           ├── data/
│           │   └── services/
│           │       └── settings_file_service_test.dart
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
│           ├── config/
│           │   ├── collector.json5
│           │   └── sensor_node.json5
│           ├── repositories/
│           │   ├── readings_repository_test.dart
│           │   └── sensor_node_repository_test.dart
│           ├── services/
│           │   └── zenoh_service_test.dart
│           └── support/
│               ├── collector.dart
│               ├── fakes.dart
│               └── settings.dart
├── pubspec.lock
└── pubspec.yaml
```

One tool and one package are new, and this is what resolved:

| | version |
|---|---|
| `zenohd` | 1.8.0 |
| `wakelock_plus` | 1.8.1, with `wakelock_plus_platform_interface` 1.7.0, `package_info_plus` 10.2.2, `package_info_plus_platform_interface` 4.1.0, `dbus` 0.8.0, `http` 1.6.0, `petitparser` 7.1.0 and `xml` 7.1.0 |

No other version changed.

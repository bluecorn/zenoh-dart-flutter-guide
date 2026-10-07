# 0 — Getting started

The system this guide builds:

```
  your phone                                         your laptop
 ┌──────────────────────────────┐                   ┌──────────────────────────────┐
 │ sensor_node, a Flutter app   │                   │ sensorctl, a Dart program    │
 │ reads the accelerometer      │                   │ shows the readings           │
 │ ┌──────────────────────────┐ │    connection     │ ┌──────────────────────────┐ │
 │ │         session          │◄┼───────────────────┼►│         session          │ │
 │ └──────────────────────────┘ │                   │ └──────────────────────────┘ │
 └──────────────────────────────┘                   └──────────────────────────────┘
            readings  ──────────────────────────────────────────────►
            ◄──────────────────────────────────────────────  queries and commands
```

Each program opens a session, and the two sessions are connected. In chapter 2 the connection runs through `adb`, to
the emulator and then over the phone's USB cable. From chapter 5 it runs over your Wi-Fi network.

## 1 — What you build, and what you will see

By the end of this chapter you have a working toolchain and a git repository with a Dart project that depends on
`zenoh_dart`. You run two of the package's example programs on your laptop, each in its own terminal. One subscribes
to a key expression, and the other publishes one message to it.

The zenoh code that runs here is the package's own, because the two programs are its examples, copied unchanged. Your
own first code, and its first test, come in chapter 1. This chapter pins one Flutter SDK for the whole project, adds
the package at a known version, and runs zenoh on your machine before any code of yours is involved.

At the end, the subscriber's terminal shows:

```
>> [Subscriber] Received PUT ('demo/example/test': 'Hello from the guide')
```

The publisher's terminal also shows a red line that says `ERROR`, printed after the message was delivered. It comes
from a known defect in zenoh 1.8.0, and nothing has failed.

> **Zenoh guidance.** The package's examples take zenoh-c's flags, and they behave like zenoh-c's, apart
> from two things. `z_put`'s default key and value name Dart, and `z_sub` closes its session when you press Ctrl-C.
> Most of this chapter is tooling. Section 6 is the one that runs zenoh.

> **Flutter guidance.** There is no app until chapter 2. The Flutter SDK is needed now because it carries
> the Dart SDK this guide uses everywhere, pinned per project with `fvm`. If you have not used `fvm`, these are the
> commands this guide uses: `fvm use` to pin a project's SDK, and `fvm dart` and `fvm flutter` to run its tools.

## 2 — What to read

| | page | what to take from it |
|---|---|---|
| [1] zenoh.io | [*What is Zenoh?*](https://zenoh.io/docs/overview/what-is-zenoh/) | what zenoh is: one protocol for publishing, subscribing and querying |
| [1] zenoh.io | [*Zenoh in action*](https://zenoh.io/docs/overview/zenoh-in-action/) | publish/subscribe and queries, in two short animations |
| [1] zenoh.io | [*Your first Zenoh app*](https://zenoh.io/docs/getting-started/first-app/), as far as *Store and Query* | a publisher and a subscriber on one key, in Python |
| [2] *The Zenoh Book* | [*Introduction*](https://corsaro.me/zenoh/book/introduction/) | what zenoh is, its design goals, and why it was built |
| [2] *The Zenoh Book* | [*Hello Zenoh*](https://corsaro.me/zenoh/book/getting-started/hello-zenoh/) | a publisher and a subscriber in Rust, run in two terminals |
| [3] *Zenoh Programming in Rust* | [chapter 1, *Introduction to Zenoh*](https://kydos.github.io/zenoh-book/chapter_01.html) | its *Key Terminology* table: session, key expression, publisher, subscriber, queryable, query, sample |
| [3] *Zenoh Programming in Rust* | [chapter 2, *Getting Started*](https://kydos.github.io/zenoh-book/chapter_02.html), as far as *First Pub/Sub Pair* | a publisher and a subscriber in Rust, after installing Rust and `zenohd`, which this chapter does not need |
| [4] `zenoh_dart` | [its README](https://pub.dev/packages/zenoh_dart) | the platforms it runs on, what zenoh's default configuration opens, and its known issue |
| `args` | [its documentation](https://pub.dev/packages/args) | how a program declares and parses its command-line options |

## 3 — The toolchain

**A Linux machine on x86_64, with glibc 2.34 or newer.** `zenoh_dart` ships zenoh's native library for Linux on
x86_64 and for Android, and for no other platform. So the laptop side of this guide cannot run on macOS or Windows.

Install these five tools yourself. Each is named with the version this guide was checked with, and tool 4, VS Code,
is optional.

**1. git.** fvm fetches Flutter with git, and the folder you build in is a git repository from its first minute. Most
Linux systems have git already. Check it:

```sh
git --version
```

This guide was checked with 2.53.0. Any version from 2.28 on works, because the guide creates repositories with
`git init -b`, which 2.28 added. If you have never committed with git on this machine, tell it once who you are. It
keeps this in your home folder and writes it into every commit you make:

```sh
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

**2. fvm**, the Flutter version manager. It keeps Flutter SDKs in your home directory, and each project names the one
it uses, so two projects on one machine can use different SDKs. Install it as
[its documentation](https://fvm.app/documentation/getting-started/installation) describes, then open a new terminal
and check it:

```sh
fvm --version
```

This guide was checked with 4.3.1, and a newer version is fine.

**3. The Flutter SDK, through fvm.** Flutter carries the Dart SDK. This guide uses Flutter's copy of Dart everywhere,
even in this chapter, where there is no app, so that the app changes nothing about the tools when it arrives. You
fetch no SDK here. Section 4 pins the project to Flutter's current stable release, and fvm downloads it then, more
than a gigabyte the first time.

`zenoh_dart` needs Flutter 3.47.1 or newer, the first release whose Dart the package accepts.

Do not put `flutter` or `dart` on your `PATH`, and do not use `fvm global`. Every command in this guide that touches
Dart or Flutter starts with `fvm`, as in `fvm dart run` and `fvm flutter test`, which runs the SDK the current project
names. A bare `dart` runs whichever SDK the `PATH` finds first, which may not be the project's.

**4. VS Code with the Flutter extension**, optional. Every step in this guide works from a terminal, and each VS Code
action is a note beside its step, marked **In VS Code.** Use the latest stable release of
[VS Code](https://code.visualstudio.com/) and of its
[Flutter extension](https://marketplace.visualstudio.com/items?itemName=Dart-Code.flutter), which brings the Dart
extension with it. The extensions update themselves.

> **Flutter guidance.** You may have a Flutter SDK on your `PATH` already. Leave it there if other projects
> need it.

**5. An Android emulator, an Android phone, and `adb`.** Chapter 2 is the first to use them. `adb` comes in the
Android SDK Platform Tools package, and the emulator is the SDK's Android Emulator component.

This guide was checked with adb 1.0.41 (37.0.1) and emulator 37.1.11, against a virtual device on API 37, and with a
Pixel 9a on Android 17.

**What Android needs.** `zenoh_dart` ships zenoh's library for Android on `arm64-v8a`, `armeabi-v7a` and `x86_64`,
each for API 24 or newer. The virtual device needs an **x86_64** system image, the kind that runs on an x86_64
machine. The package's `arm64-v8a` library has run on a phone, and its `armeabi-v7a` library has never been loaded on
any device. Android's documentation describes how to
[create a virtual device](https://developer.android.com/studio/run/managing-avds).

**About versions.** A version this guide gives for a tool or a package is the one a chapter was checked with, and a
newer one should work. The full table of versions is at the end of this chapter, and each later chapter ends with the
versions that changed.

If a listing ever stops compiling on your machine, pin those versions. `.fvmrc` already names one SDK version. A
package is pinned by replacing the caret in its constraint, `^1.0.0-rc.1`, with the exact version, `1.0.0-rc.1`, and
then running `fvm dart pub get`.

The toolchain is ready. Section 4 creates the project.

## 4 — The project

Create the project folder, pin its SDK, and create the program. Everything you build in this guide lives in one
folder, `zenoh_sensors`, which is a git repository from its first minute. In this chapter it holds one small
command-line program, `sensorctl`. From chapter 1 it also holds the code that program shares with the phone app, and
from chapter 2 the app itself.

Put `zenoh_sensors` wherever you keep your projects, such as your home folder. From here on, every command block
starts with a comment that names the folder it runs in. `zenoh_sensors/apps/sensorctl` is the program's folder,
wherever `zenoh_sensors` itself lives. When a block contains a `cd`, the next block names the folder it moved you to.

When this section is done, the top folder holds this:

```
zenoh_sensors/                  the top folder
├── .fvm/                       fvm's links to the pinned SDK, and its own files
├── .fvmrc                      the pin, written by fvm use
├── .git/                       the repository, created by git init
├── .gitignore                  written by fvm use, replaced in section 5
├── .vscode/                    if you use VS Code
│   └── settings.json           which SDK VS Code uses: written by fvm use, one line added by you
└── apps/
    └── sensorctl/              the program, created by fvm dart create
        ├── .dart_tool/         the Dart tools' working files; zenoh's libraries land here
        ├── .gitignore
        ├── analysis_options.yaml
        ├── bin/
        │   └── sensorctl.dart
        ├── CHANGELOG.md
        ├── lib/
        │   └── sensorctl.dart
        ├── pubspec.lock
        ├── pubspec.yaml
        ├── README.md
        └── test/
            └── sensorctl_test.dart
```

**1. Create the top folder.**

```sh
# in the folder where you keep your projects
mkdir zenoh_sensors
cd zenoh_sensors
```

> **In VS Code.** You can type every command of this guide in VS Code's own terminal. In place of the `cd` above,
> open the new folder: run `code zenoh_sensors`, or choose **File › Open Folder…** and pick `zenoh_sensors`. Then
> choose **Terminal › New Terminal**. The terminal starts in `zenoh_sensors`, the folder VS Code has open, so each
> command block below can be typed there as it stands.
>
> Keep VS Code on this folder for the whole guide. The extensions read their settings from `zenoh_sensors/.vscode`,
> and the paths in them count from the folder VS Code has open. Make that folder now, so that fvm writes the editor's
> setting into it in step 3:
>
> ```sh
> # in zenoh_sensors
> mkdir .vscode
> ```

**2. Make it a repository.**

```sh
# in zenoh_sensors
git init -b main
```

`-b main` names the first branch `main`. Without it, git picks a name from its own settings and prints a hint about
it.

> **In VS Code.** **View › Source Control** shows an **Initialize Repository** button while the folder is not yet a
> repository, and it does the same as the command. VS Code names the first branch `main` too, unless you have changed
> its `git.defaultBranchName` setting.

**3. Pin it.**

```sh
# in zenoh_sensors
mkdir apps
fvm use stable --pin --force
```

The pin comes first, because nothing else runs without it. You have no global SDK and no `dart` on your `PATH`, so
outside a pinned folder `fvm dart` has nothing to start and stops with `dart: not found`. Inside one, fvm finds the
pin in the folder you are in or in any folder above it, so everything under `zenoh_sensors` uses this one.

`stable --pin` asks for Flutter's current stable release and writes its number into the pin, so the project keeps
that release when a newer one comes out. When the release is not in fvm's cache yet, fvm downloads it first.

`fvm use` expects a folder that holds a `pubspec.yaml`, and without one it asks whether to continue. `--force` skips
the question. fvm then prints two warnings, both expected. One says that it skipped its version check because of
`--force`, and the other that it skipped `pub get`, because there is no `pubspec.yaml` yet.

`fvm use` wrote three things into `zenoh_sensors`, and a fourth if you made `.vscode`:

- `.fvmrc`, the pin itself. It names the release, which this guide shows as `…`:

  ```json
  {
    "flutter": "…"
  }
  ```

- `.fvm/`, with two links to the SDK in fvm's cache under your home folder, and three small files fvm keeps for
  itself.
- `.gitignore`, which lists `.fvm/`, because the links in `.fvm/` point into your home folder and mean nothing on
  another machine.
- `.vscode/settings.json`, but only when the `.vscode` folder exists.

> **In VS Code.** Point the editor at the same Dart. Open `zenoh_sensors/.vscode/settings.json`, from the Explorer on
> the left or with `code .vscode/settings.json` from `zenoh_sensors`. Add the second line, so that the file reads as
> below, with fvm's release number in place of `…`:
>
> ```json
> {
>   "dart.flutterSdkPath": ".fvm/versions/…",
>   "dart.sdkPath": ".fvm/flutter_sdk/bin/cache/dart-sdk"
> }
> ```
>
> The first line is fvm's, and names the Flutter SDK. For a plain Dart project, such as `sensorctl`, the Dart
> extension tries `dart.sdkPath` first, then the first `dart` on your `PATH`, then the Flutter SDK on the first line.
>
> With the second line, the editor runs the same Dart as `fvm dart`, whatever else is on your `PATH`.
> `.fvm/flutter_sdk` is a link that fvm keeps pointing at the pinned SDK, and `fvm use` keeps the line when it
> rewrites the file.

**4. Create the program.**

```sh
# in zenoh_sensors
cd apps
fvm dart create -t console sensorctl
```

`-t console` picks Dart's template for a command-line program. `bin/sensorctl.dart` holds `main`, which calls a
function in `lib/sensorctl.dart`, and `test/` holds a test of that function. Chapter 1 replaces `bin/sensorctl.dart`
and deletes the other two. When the tool finishes, it suggests `dart run`, which you run as `fvm dart run` in step 6.

> **In VS Code.** Type the command above in VS Code's terminal. Do not use the Command Palette's **Dart: New
> Project**. It runs the same template, but then it reopens VS Code on `zenoh_sensors/apps/sensorctl`, and VS Code must
> stay open on `zenoh_sensors`.

**5. Add the package.** Go into the program's folder first, because the next two steps run there.

```sh
# in zenoh_sensors/apps
cd sensorctl
```

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart pub add 'zenoh_dart:^1.0.0-rc.1'
```

The command names the version because pub prefers a stable version to a prerelease. The newest `zenoh_dart` it picks
by itself is `0.30.0`, the same code as `1.0.0-rc.1`, published below 1.0 so that a bare `pub add` gets it. A bare
`fvm dart pub add zenoh_dart` would write `^0.30.0`, which stops short of every 1.0 release candidate and of 1.0.0
itself. `^1.0.0-rc.1` means this release candidate or any later version below 2.0.0.

pub also notes that one package has a newer version it cannot use. That package is one `zenoh_dart` depends on, and it
needs nothing from you.

**6. Run it.**

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart run
```

The template's program prints `Hello world: 42!`, and nothing of zenoh runs yet. `dart run` prints
`Running build hooks...` in front of it, because `zenoh_dart` has a *build hook*, a small Dart program of its own
that `dart run` runs before yours. The hook has copied zenoh's native libraries into your project:

```sh
# in zenoh_sensors/apps/sensorctl
ls .dart_tool/lib
```

Two files are there. `libzenohc.so` is zenoh itself, zenoh-c 1.8.0, the C interface to zenoh's Rust core.
`libzenoh_dart.so` is the thin layer the package's Dart code calls zenoh through. `libzenoh_dart.so` loads
`libzenohc.so` from its own folder.

When the package comes from pub's cache, it looks for `libzenoh_dart.so` in `.dart_tool/lib` under the folder a
program is **started from**, and then asks the system's loader. The package's README lists this as a known issue of
this release candidate.

Until it is fixed, start your programs from `zenoh_sensors/apps/sensorctl`, a folder you own. A program started in a
folder someone else can write to could load a library placed there. Chapter 1 shows what changes when
`zenoh_sensors` becomes a workspace.

`Running build hooks...` comes before the output of every `fvm dart run` and `fvm dart test` from here on, sometimes
with a line or two more from pub. The guide shows those lines as `⋮` and starts an expected output at the program's
first words. `…` stands for the part of a line that differs on your machine.

> **Zenoh guidance.** `libzenohc.so` is the library a C program links. The package's examples work with
> zenoh-c's, and the README of its `example/` folder notes that a Dart `z_get` can query a C `z_queryable`.

> **In VS Code.** Open `apps/sensorctl/bin/sensorctl.dart` from the Explorer, and choose **Run** in the line of small
> links above `main`. VS Code starts the program in `zenoh_sensors/apps/sensorctl`, the nearest folder above it with a
> `pubspec.yaml`, which is the folder this step asks for. The Debug Console at the bottom of the window shows
> `Hello world: 42!`.

The project exists and runs. Section 5 decides what git keeps, and makes the first commit.

## 5 — What git keeps

`fvm use` and `fvm dart create` each wrote a `.gitignore`. The top folder's lists `.fvm/`, and the program's lists
`.dart_tool/`. Before the first commit, replace the top folder's with this project's own list, with a reason for
every line. Go back to the top folder first:

```sh
# in zenoh_sensors/apps/sensorctl
cd ../..
```

**1. Write the top folder's list.** Replace `zenoh_sensors/.gitignore`:

```gitignore
# What the Dart tools make, and make again when needed.
# The rules are dart.dev's, from "What not to commit".
.dart_tool/
build/
**/doc/api/
# fvm's links to the pinned SDK. They point into your home folder.
.fvm/
```

The rules for `.dart_tool/`, `build/` and `doc/api/` come from dart.dev's page
[What not to commit](https://dart.dev/tools/pub/private-files). `.dart_tool/` holds the Dart tools' working files,
zenoh's libraries among them. `build/` holds what `dart build` and `flutter build` make. `doc/api/` holds the
documentation `dart doc` writes.

A rule ending in `/` with no other slash applies in every folder below. `doc/api/` has a slash in the middle, so the
list writes it as `**/doc/api/`, which gives it the same reach. `.fvm/` is fvm's own line.

The list leaves out `pubspec.lock`, so git keeps it. The file records the exact version of every package pub chose.
For an application package, as dart.dev calls a program, dart.dev recommends committing it, because then every change
to a package your dependencies depend on shows in the lock file.

The list also leaves out the files your editor or operating system makes, such as IntelliJ's `.idea/` or macOS's
`.DS_Store`. For those, dart.dev suggests a global ignore file, one that applies to every repository on your machine.
git reads one from `~/.config/git/ignore`. GitHub's page [Ignoring
files](https://docs.github.com/en/get-started/git-basics/ignoring-files#configuring-ignored-files-for-all-repositories-on-your-computer)
explains how to set it up.

The program keeps the `.gitignore` that `fvm dart create` wrote. Its one rule, `.dart_tool/`, is also in the top
folder's list. A package created later keeps the list its own tool writes, in the same way.

> **In VS Code.** Open `.gitignore` from the Explorer to replace its content.

**2. Commit.** Commit everything the chapter has made so far:

```sh
# in zenoh_sensors
git add .
git commit -m "Pin Flutter and create sensorctl"
```

The commit takes 11 files, and 12 with `.vscode/settings.json`. It leaves out two folders, `.fvm/` and
`apps/sensorctl/.dart_tool/`, because both belong to this machine alone. `.fvm/` points into your home folder, and
the Dart tools rebuild `.dart_tool/` whenever they need it. When it exists, `.vscode/settings.json` is committed,
because its paths count from the top folder and are right on any machine.

> **In VS Code.** **View › Source Control** lists the same files under **Changes**. Choose the **+** on the
> **Changes** line to stage them all, type the message in the box above them, and choose **Commit**.

The first commit is made. Section 6 runs two of the package's programs against each other.

## 6 — Two programs, two terminals

Run two of the package's programs against each other. The package ships its examples as source, next to its own
code. `z_sub` subscribes to a key expression and prints whatever arrives, and `z_put` puts one value on a key and
ends. Steps 3 and 4 set them up like this:

```
  first terminal                                    second terminal
 ┌────────────────────────────────────┐            ┌────────────────────────────────────┐
 │ z_sub                              │            │ z_put                              │
 │ a session, listening on            │ connection │ a session, connecting to           │
 │   tcp/127.0.0.1:7447 ◄─────────────┼────────────┼──── tcp/127.0.0.1:7447             │
 │ a subscriber on demo/example/**    │◄── put ────┤ a put on demo/example/test         │
 └────────────────────────────────────┘            └────────────────────────────────────┘
```

**1. Copy the two examples.** They are in pub's cache under your home folder, in the folder pub unpacked the package
into, and your project's `.dart_tool/package_config.json` records which folder that is. The second block below reads
it into a shell variable, `pkg`. It then copies the two programs and `common_args.dart`, the command-line options they
share.

```sh
# in zenoh_sensors
cd apps/sensorctl
```

```sh
# in zenoh_sensors/apps/sensorctl
pkg=$(sed -n 's|.*"rootUri": "file://\(.*/zenoh_dart-[^/"]*\)".*|\1|p' .dart_tool/package_config.json)
mkdir example
cp "$pkg/example/common_args.dart" "$pkg/example/z_sub.dart" "$pkg/example/z_put.dart" example/
```

`ls "$pkg/example"` shows the rest: 28 programs, each named after a zenoh-c example, and `example.dart`, the
package's own showcase. Later chapters start from several of them.

**2. Add what they need.** The examples read their command-line options with the `args` package. Pub already has it,
because `zenoh_dart` depends on it. The template's lints ask a program to name every package it imports in its own
`pubspec.yaml`, with the rule `depend_on_referenced_packages`, so add it:

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart pub add args
```

**3. Start the subscriber, and leave it running.**

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

It waits there. `demo/example/**` is its default key expression, and `**` matches any number of levels below
`demo/example`, so it receives whatever is put anywhere under that prefix.

Both options limit who can reach the subscriber. `-l tcp/127.0.0.1:7447` makes it listen on port 7447 of the loopback
interface, which no other machine can reach. `--no-multicast-scouting` stops it announcing itself on your network.

Leave them out and it runs on zenoh's default configuration, which the package's README warns about. It listens for
TCP connections on every network interface, joins UDP multicast scouting, and accepts peers without authentication or
encryption. Every program stays on the loopback until chapter 5, which takes the phone onto your network. Chapter 14
is about security.

**4. Put a value.** Open a second terminal, go to the same folder, and put one value on a key under `demo/example`:

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart run example/z_put.dart \
  -e tcp/127.0.0.1:7447 --no-multicast-scouting --cfg 'listen/endpoints:[]' \
  -k demo/example/test -p 'Hello from the guide'
```

```
⋮
…Opening session...
Putting Data ('demo/example/test': 'Hello from the guide')...
… ERROR ThreadId(…) zenoh::api::admin: Unable to publish transport event: session closed
```

In the first terminal, one more line appears:

```
>> [Subscriber] Received PUT ('demo/example/test': 'Hello from the guide')
```

`-e tcp/127.0.0.1:7447` connects to the subscriber. `--cfg 'listen/endpoints:[]'` sets one entry of zenoh's
configuration directly, the list of endpoints to listen on, and empties it. By default a zenoh peer also listens while
it connects, on every interface, on a port picked at random. `-k` and `-p` give the key and the value.

The publisher's last line is red and says `ERROR`. The put worked. The subscriber has it, and `z_put` ended normally.

> **Note: a bug in zenoh 1.8.0.** This release of `zenoh_dart` is built on zenoh 1.8.0. With zenoh's log on, as the
> examples have it, zenoh 1.8.0 logs this error whenever a program closes its session while still connected to
> another. Nothing has failed. Zenoh fixed it in version 1.10.0.

**5. Put a value it does not match.**

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart run example/z_put.dart \
  -e tcp/127.0.0.1:7447 --no-multicast-scouting --cfg 'listen/endpoints:[]' \
  -k demo/other/test -p 'Not under demo/example'
```

The publisher prints the same three lines with the new key and value, and the subscriber prints nothing, because
`demo/other/test` is not under `demo/example`. Key expressions are the subject of chapter 2, and their wildcards of
chapter 4.

**6. Stop the subscriber** with Ctrl-C in the first terminal. The example catches it, closes its subscriber and its
session, and ends.

> **Zenoh guidance.** `common_args.dart` is the Dart translation of zenoh-c's `parse_args.h`. The flags
> are zenoh-c's, including `--cfg KEY:VALUE` with a JSON5 value.

> **In VS Code.** Open the second terminal with the **+** at the top right of the terminal panel. A new terminal
> starts in `zenoh_sensors`, so type `cd apps/sensorctl` in it first. **Terminal › Split Terminal** shows both
> terminals side by side, and starts the new one in the folder the first one is in. Ctrl-C works in VS Code's
> terminal as in any other.

> **In VS Code.** VS Code can also run the two programs from its **Run and Debug** view, with the same SDK and the
> same options, and show their output in its Debug Console. It needs one file that describes each program. Create
> `zenoh_sensors/.vscode/launch.json`, with `code .vscode/launch.json` from `zenoh_sensors`, and give it this content:
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
>     }
>   ]
> }
> ```
>
> Each entry names a program, the folder to start it in, and its options, one per string. `${workspaceFolder}` stands
> for the folder VS Code has open, `zenoh_sensors`. The `cwd` line starts the program in `apps/sensorctl`, where the
> package's libraries are. Without the line, VS Code picks the same folder, because it is the nearest one above the
> program with a `pubspec.yaml`. The line makes the choice visible, and chapter 1 changes the line.
>
> Open the view with **View › Run**. Choose **z_sub** in the list at its top and press the green ▶ beside it, or F5.
> The Debug Console shows what `z_sub` prints, and `z_sub` waits. Then choose **z_put** and press ▶ again.
> `z_put` prints its three lines and ends. A list at the top of the Debug Console switches between the two programs'
> output, and `z_sub`'s shows the line it received.
>
> To stop `z_sub`, select it in the small debug toolbar at the top of the window and press the red square, or Shift+F5.
> VS Code asks the program to stop, and the example closes its subscriber and its session. F5 starts a program with VS
> Code's debugger attached, so a breakpoint set in `example/z_sub.dart` stops it there. **Run › Run Without Debugging**,
> Ctrl+F5, starts it without the debugger.

**7. Commit.** In either terminal, go back to the top folder and commit what this chapter added:

```sh
# in zenoh_sensors/apps/sensorctl
cd ../..
```

```sh
# in zenoh_sensors
git add .
git commit -m "Run the package's z_sub and z_put examples"
```

The commit takes the three example files, and `pubspec.yaml` and `pubspec.lock`, which now name `args`. With VS Code,
it also takes `.vscode/launch.json`.

> **In VS Code.** **View › Source Control** lists the same files under **Changes**. Choose the **+** on the
> **Changes** line to stage them all, type the message in the box above them, and choose **Commit**.

The chapter's work is committed. Section 7 looks ahead to the architecture.

## 7 — What changed in the architecture

Nothing has changed yet. `sensorctl` is still the template's program, and the two programs that talked to each other
are the package's own.

The architecture starts in chapter 1. Both applications in this guide are built in layers, in the pattern
[Flutter's architecture guide](https://docs.flutter.dev/app-architecture/guide) [5] calls MVVM: model, view and view
model. Over the next chapters, `sensorctl` gains these layers, with the terminal as its view. It has all of them by
chapter 3, except the codec, which arrives in chapter 6:

```
view  →  view model  →  repository  →  codec  →  ZenohService  →  zenoh_dart
```

Each layer uses only the ones to its right. A layer's tests replace the layer to its right with a fake, except the
service's, which run against real zenoh.

Chapter 1 starts from another of the package's examples, `z_info`. You write it as one flat program that opens a session
on the loopback and prints its own id and the ids of its routers and peers. Then, with a test in place, you move its
zenoh code into the first layer, `ZenohService`. In the same chapter, `zenoh_sensors` becomes a pub workspace, with the
service in a package of its own that the phone app shares from chapter 2.

## 8 — Files and versions at the end of this chapter

`zenoh_sensors` now holds this, in two commits: `Pin Flutter and create sensorctl` and
`Run the package's z_sub and z_put examples`.

```
zenoh_sensors/
├── .fvm/
├── .fvmrc
├── .git/
├── .gitignore
├── .vscode/                    if you use VS Code
│   ├── launch.json
│   └── settings.json
└── apps/
    └── sensorctl/
        ├── .dart_tool/
        ├── .gitignore
        ├── analysis_options.yaml
        ├── bin/
        │   └── sensorctl.dart
        ├── CHANGELOG.md
        ├── example/
        │   ├── common_args.dart
        │   ├── z_put.dart
        │   └── z_sub.dart
        ├── lib/
        │   └── sensorctl.dart
        ├── pubspec.lock
        ├── pubspec.yaml
        ├── README.md
        └── test/
            └── sensorctl_test.dart
```

This chapter was checked with these versions:

| what | version |
|---|---|
| Linux | Ubuntu 26.04.1 on x86_64, glibc 2.43 |
| git | 2.53.0 |
| fvm | 4.3.1 |
| Flutter, and the Dart it carries | 3.47.5, Dart 3.13.4 |
| VS Code, and its Dart and Flutter extensions | the latest stable release |
| `zenoh_dart`, built on zenoh 1.8.0 | 1.0.0-rc.1 |
| `args` | 2.7.0 |
| `path` | 1.9.1 |
| `lints` | 6.1.0 |
| `test` | 1.32.0 |

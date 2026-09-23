# 0 — Getting started

This chapter follows chapters 1 and 2 of the zenoh book, *Introduction to Zenoh* and *Getting Started*, and starts from
two programs shipped with the `zenoh_dart` package, `z_sub` and `z_put`.

## 1 — What you build, and what you will see

By the end of this chapter you have a working toolchain, a git repository holding a Dart project that depends on
`zenoh_dart`, and two small programs from the package itself talking to each other on your laptop: one subscribes to a key expression, the other
publishes one message to it, each in its own terminal. Then you run the same two programs from VS Code.

Nothing here is yours yet. The two programs are copied from the package's examples, unchanged. The first code you write
comes in the next chapter, together with its first test. This chapter exists so that everything after it starts from a
known place: a Flutter version at or above the one this guide was checked with, the package at a known version, an
editor that runs that same SDK, and proof that zenoh runs on your machine before any of your own code is involved.

What you will see at the end, in the subscriber's terminal:

```
>> [Subscriber] Received PUT ('demo/example/test': 'Hello from the guide')
```

And in the publisher's terminal, a red line that says `ERROR` after the message has been delivered. It is harmless, and
this chapter explains where it comes from, so that it does not stop you the first time.

> **If you already know zenoh.** The package's examples mirror the zenoh-c examples flag for flag: `z_sub` and `z_put` do
> here what they do in C, with the same options. What is new to you is Dart's tooling, and this chapter is mostly that.

> **If you already know Flutter.** There is no app until chapter 2. The Flutter SDK is installed now only because it carries
> the Dart SDK the guide uses everywhere, pinned per project with `fvm`. If you have never used `fvm`, this chapter shows
> the three commands you need.

## 2 — What to read

Two books by Angelo Corsaro sit behind this guide, and they do different jobs. You will be sent to both, a little at a
time, at the point where each helps.

**[The Zenoh Book](https://corsaro.me/zenoh/book/) — for the idea.** What zenoh is, what it was built to solve, and
when it is the right answer. It is written above any particular release, so it ages slowly.

**[Zenoh Programming in Rust](https://kydos.github.io/zenoh-book/) — for the shape of the API.** The calls, their
options, and the ways they go wrong. It is a draft, and it is written against Zenoh 1.4.0 while the package this guide
uses is built on 1.8.0, so an occasional detail there will have moved on.

Both are in Rust, and that is fine: you are reading them for the ideas and the words, and typing nothing from them.
Where this guide states something about the Dart API, it has been read in the package itself.

For this chapter:

**[Introduction](https://corsaro.me/zenoh/book/introduction/), in The Zenoh Book, and
[chapter 1, Introduction to Zenoh](https://kydos.github.io/zenoh-book/chapter_01.html), in the other.** Read both;
between them they are short. Take the words the rest of this guide uses without explaining them again: key expression,
session, publisher and subscriber, queryable and query, and the three modes a session can run in — peer, client and
router. The case each makes for when to use zenoh is the argument for the application you are about to build: a sensor
on one device, a program watching it on another, and no server in between.

**[Getting Started](https://corsaro.me/zenoh/book/getting-started/) and
[chapter 2, Getting Started](https://kydos.github.io/zenoh-book/chapter_02.html).** Read as far as each one's first
publisher and subscriber, and skip the rest for now. Both install Rust and a zenoh router, `zenohd`, and end with a
subscriber receiving one message. This chapter does the same thing in Dart, with the package's `z_sub` and `z_put` in
place of the Rust programs, and two things are different on purpose. There is no Rust to install, because the package
ships zenoh's native library inside it. And there is no router: the books' programs find each other over the local
network, while here they connect over the loopback interface only, with the network left out until the last chapter.
The section *Two programs, two terminals* below says how, and why.

## 3 — The toolchain

**A Linux machine on x86_64.** That is the one hard requirement: the package ships zenoh's native library for Linux on
x86_64 and for Android, and for nothing else, so the laptop side of this guide cannot run on macOS or Windows. The
library needs glibc 2.34 or newer, which any distribution from the last few years has. Nothing from Android is needed
in this chapter or the next; from chapter 2 on, when the sensor node moves onto a phone, you also need the last two
tools below.

Five tools, in this order.

**1. git.** fvm fetches Flutter with git, and the folder you build in is a git repository from its first minute. Most
Linux systems have git already; check:

```sh
git --version
```

This guide was checked with 2.53.0. Any version from 2.28 on will do, because the guide creates repositories with
`git init -b`, which 2.28 added. If you have never committed with git on this machine, tell it once who you are; it keeps
this in your home folder and writes it into every commit you make:

```sh
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

**2. fvm**, the Flutter version manager. It keeps one or more Flutter SDKs in your home directory and lets each project
name the one it uses, so that two projects on one machine can disagree about the SDK and both be right. Install it with
its own script, which puts it under `~/fvm` and tells you the line to add to your shell's startup file:

```sh
curl -fsSL https://fvm.app/install.sh | bash
```

Open a new terminal, then check:

```sh
fvm --version
```

This guide was checked with 4.3.1; a newer version is fine.

**3. The Flutter SDK, through fvm.** Flutter carries the Dart SDK, and this guide uses Flutter's copy of Dart everywhere,
even in this chapter, where there is no app, so that the app changes nothing about the tools when it arrives. Install
the version this guide was checked with:

```sh
fvm install 3.47.2
```

On a fresh machine this clones the SDK and downloads what it needs to run, which takes several minutes and a few
gigabytes. Then:

`fvm list` now shows `3.47.2`, with Dart `3.13.2` beside it. If you would rather use a newer stable
release, install that one instead and use its number wherever this guide says `3.47.2`; the package needs Flutter 3.47.1
or newer, and the chapters were checked on 3.47.2.

Do not put `flutter` or `dart` on your `PATH`, and do not use `fvm global`. Every command in this guide that touches
Dart or Flutter starts with `fvm`, as in `fvm dart run` and `fvm flutter test`, which runs the SDK the current project
names. That is the whole point of fvm, and a bare `dart` on the `PATH` is how a project ends up built with the wrong SDK
without anyone noticing.

**4. VS Code with the Flutter extension.** Install VS Code from <https://code.visualstudio.com/>, then add the Flutter
extension from a terminal; it brings the Dart extension with it:

```sh
code --install-extension dart-code.flutter
```

This guide was checked with version 3.142.0 of both extensions; they update themselves, and newer is fine. The editor is
told which SDK to use per project, in the next section, so that what runs from a VS Code button is the same SDK as what
runs from `fvm`.

> **If you already know Flutter.** You may have a Flutter SDK on your `PATH` already. Leave it there if other projects
> need it; this guide never calls it, because every command goes through `fvm` and the project's own `.fvmrc`, and the
> next section points VS Code at the same SDK.

**5. An Android device or emulator, and `adb`.** Nothing in this chapter or the next uses them; from chapter 2 the
sensor node runs on a device. `adb` comes with the Android SDK's platform tools, and the emulator with the SDK itself —
this guide was checked with adb 1.0.41 (37.0.1) and emulator 37.1.11, against a virtual device on API 37. **The one
requirement is the architecture:** `zenoh_dart` ships zenoh's native library for Android on `arm64-v8a`, `armeabi-v7a`
and `x86_64`, so an emulator on this laptop needs an **x86_64** system image, and a phone on USB works as it is.
Installing the SDK and creating a virtual device are Android's own business, and
[its documentation](https://developer.android.com/studio/run/managing-avds) describes them; this guide only uses what
you have.

**About versions.** Every version number in this guide is the lowest the chapter was checked with, and a newer one is
expected to work. Each chapter ends with the exact versions it was checked with: the tools, the SDK, and the packages your
`pubspec.yaml` names. If a listing
ever stops compiling on your machine, the first thing to try is to pin those: `.fvmrc` already names one SDK version,
and a package is pinned by replacing the caret in its constraint, `^1.0.0-rc.1`, with the exact version the chapter
names, `1.0.0-rc.1`, followed by `fvm dart pub get`.

## 4 — The project

Everything you build in this guide lives in one folder, `zenoh_sensors`, and that folder is a git repository from its
first minute. In this chapter it holds one small command-line program, `sensorctl`; from the next chapter on it also holds the code that program shares with the phone
app, and from chapter 2 the app itself. Put it wherever you keep your projects; your home folder will do. From here on, this
guide names every folder from `zenoh_sensors` down: `zenoh_sensors/apps/sensorctl` is the program's folder, wherever
`zenoh_sensors` itself lives.

When this section is done, the top folder holds this:

```
zenoh_sensors/                  the top folder, the one VS Code opens
├── .fvm/                       links to the pinned SDK, written by fvm use
├── .fvmrc                      the pin, written by fvm use
├── .git/                       the repository, created by git init
├── .gitignore                  written by fvm use, replaced in the next section
├── .vscode/
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

Every command block from here on starts with a comment that names the folder it runs in. When a block contains a `cd`,
the next block names the folder it moved you to.

**1. Create the top folder.**

```sh
# in the folder where you keep your projects
mkdir zenoh_sensors
cd zenoh_sensors
```

> **In VS Code.** You can do the rest of this guide inside VS Code, typing every command in its own terminal. In place
> of the `cd` above, open the new folder: run `code zenoh_sensors`, or choose **File › Open Folder…** and pick
> `zenoh_sensors`. Then choose **Terminal › New Terminal**. The terminal starts in `zenoh_sensors`, the folder VS Code
> has open, so each command block below can be typed there as it stands. Keep VS Code on this folder for the whole
> guide: the settings the extensions read are in `zenoh_sensors/.vscode`, and the paths in them count from the folder VS
> Code has open.

**2. Make it a repository.**

```sh
# in zenoh_sensors
git init -b main
```

`-b main` names the first branch `main`; without it, git
picks a name from its own settings and prints a hint about it.

> **In VS Code.** **View › Source Control** shows an **Initialize Repository** button while the folder is not yet a
> repository, and it does the same as the command. VS Code names the first branch `main` too, unless you have changed its
> `git.defaultBranchName` setting.

**3. Pin it.**

```sh
# in zenoh_sensors
mkdir apps .vscode
fvm use 3.47.2 --force
```

The pin comes before anything else because nothing else can run without it. You have no global SDK and no `dart` on
your `PATH`, so outside a pinned folder `fvm dart` has nothing to start and stops with `dart: not found`. Inside one, fvm
finds the pin in the folder you are in or in any folder above it, so everything under `zenoh_sensors` uses this one.
fvm prints two warnings, both expected: `fvm use` is meant for a folder that already holds a Dart project, and asks
before pinning one that does not; `--force` answers yes.

`fvm use` wrote four things into `zenoh_sensors`:

- `.fvmrc`, the pin itself:

  ```json
  {
    "flutter": "3.47.2"
  }
  ```

- `.fvm/`, links from this folder to the SDK in fvm's cache under your home folder;
- `.gitignore`, which lists `.fvm/`, because those links mean nothing on another machine;
- `.vscode/settings.json`, because the `.vscode` folder existed. fvm writes the editor's setting only when it does.

**4. Point VS Code at the same Dart.** Open `zenoh_sensors/.vscode/settings.json` in VS Code, from the Explorer on the
left if VS Code has the folder open, or with `code .vscode/settings.json` from `zenoh_sensors`. Add the second line, so
that the file reads:

```json
{
  "dart.flutterSdkPath": ".fvm/versions/3.47.2",
  "dart.sdkPath": ".fvm/flutter_sdk/bin/cache/dart-sdk"
}
```

The first line is fvm's, and names the Flutter SDK. The second is yours, and everything you build depends on it. The
program you are about to create is plain Dart, and for a plain Dart project the Dart extension uses `dart.sdkPath` if it
is set, then the first `dart` on your `PATH`, and only after those the Flutter SDK named on the first line. With the
second line, the editor runs the same Dart as `fvm dart`, whatever else is on your `PATH`. `.fvm/flutter_sdk` is a link
that fvm keeps pointing at the pinned SDK, so the line stays right when you change the pin, and `fvm use` keeps it when
it rewrites the file.

**5. Create the program.**

```sh
# in zenoh_sensors
cd apps
fvm dart create -t console sensorctl
```

`-t console` picks Dart's template for a command-line program: `bin/sensorctl.dart` holds `main`, which calls a function
in `lib/sensorctl.dart`, and `test/` holds a test of that function. The next chapter replaces all three. When the tool
finishes it suggests `dart run`; this guide always writes `fvm dart run`.

> **In VS Code.** The Command Palette's **Dart: New Project** runs the same template, but afterwards it reopens VS Code
> on the new program's folder, `zenoh_sensors/apps/sensorctl`, and this guide keeps VS Code on `zenoh_sensors`. Type
> the command above in VS Code's terminal instead.

**6. Add the package.** Go into the program's folder first; the next two steps run there.

```sh
# in zenoh_sensors/apps
cd sensorctl
```

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart pub add 'zenoh_dart:^1.0.0-rc.1'
```

The version is written out on purpose. The newest `zenoh_dart` pub picks by itself is `0.30.0`: the same code as
`1.0.0-rc.1`, published under a number below 1.0 because pub never picks a release candidate on its own. A bare
`fvm dart pub add zenoh_dart` would therefore write `^0.30.0`, which stops short of every 1.0 release candidate and of
1.0.0 itself. `^1.0.0-rc.1` means this release candidate or any later version below 2.0.0. pub also notes that one
package has a newer version it cannot use; that package is one `zenoh_dart` depends on, and it needs nothing from you.

**7. Run it.**

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart run
```

The template's program prints `Hello world: 42!`. Nothing of zenoh runs yet, but the words `Running build hooks...` in
front of it are the package at work. `zenoh_dart` has a *build hook*, a small Dart program of its own that `dart run` runs before yours, and it has
copied zenoh's native libraries into your project:

```sh
# in zenoh_sensors/apps/sensorctl
ls .dart_tool/lib
```

Two files are there. `libzenohc.so` is zenoh itself: zenoh-c 1.8.0, the C interface to zenoh's Rust core.
`libzenoh_dart.so` is the thin layer the package's Dart code calls it through. The package looks for both in
`.dart_tool/lib` under the folder a program is **started from**, which its README lists as a known issue of this release
candidate. Until it is fixed, that gives you one rule: start your programs from `zenoh_sensors/apps/sensorctl`, a folder
you own. A program started in a folder someone else can write to could load a library placed there. The next chapter
shows what changes when `zenoh_sensors` becomes a workspace.

`Running build hooks...` will be in front of every program's output from here on, sometimes with a line or two more
from pub. The guide shows those lines as `⋮` and starts an expected output at the program's first words; `…` stands
for the part of a line that differs on your machine.

> **If you already know zenoh.** `libzenohc.so` is the library a C program links, and the package's examples are made
> to interoperate with zenoh-c's: its README notes that a Dart `z_get` can query a C `z_queryable`.

> **In VS Code.** Open `apps/sensorctl/bin/sensorctl.dart` from the Explorer and choose **Run** in the line of small
> links above `main`. VS Code starts the program in `zenoh_sensors/apps/sensorctl`, the nearest folder above it with a
> `pubspec.yaml`, which is the folder the rule above asks for. The Debug Console at the bottom of the window shows
> `Hello world: 42!`.

## 5 — What git keeps

`fvm use` and `fvm dart create` each wrote a `.gitignore`: `.fvm/` at the top, `.dart_tool/` in the program. Before the
first commit, the top folder's becomes the list this project keeps, with a reason for every line. Go back to the top folder
first:

```sh
# in zenoh_sensors/apps/sensorctl
cd ../..
```

**1. Write the top folder's list.** Replace the whole content of `zenoh_sensors/.gitignore` with this:

```gitignore
# What the Dart tools make, and make again when needed.
# The rules are dart.dev's, from "What not to commit".
.dart_tool/
build/
**/doc/api/
# fvm's links to the pinned SDK. They point into your home folder.
.fvm/
```

The first three rules are the Dart team's own, from dart.dev's page
[What not to commit](https://dart.dev/tools/pub/private-files). `.dart_tool/` holds the Dart tools' working files, zenoh's
libraries among them; `build/` holds what `dart build` and `flutter build` make; `doc/api/` holds the documentation
`dart doc` writes. A rule ending in `/` with no other slash applies in every folder below, and `**/` gives the third rule
the same reach. `.fvm/` is fvm's line from before.

What the list leaves out matters as much. `pubspec.lock` is not in it, so it is committed: it records the exact version of
every package pub chose, and for a program, an application in dart.dev's words, the Dart team recommends committing it,
so that every machine gets the same versions. Nor are the files your editor or operating system makes, such as IntelliJ's
`.idea/` or macOS's `.DS_Store`. dart.dev suggests keeping those in an ignore file of your own that applies to every
repository on your machine; git reads one from `~/.config/git/ignore`, as GitHub's page
[Ignoring files](https://docs.github.com/en/get-started/git-basics/ignoring-files#configuring-ignored-files-for-all-repositories-on-your-computer)
explains.

The program keeps the `.gitignore` that `fvm dart create` wrote. Its one rule, `.dart_tool/`, is also in the top folder's
list, and a package created later keeps the list its own tool writes in the same way.

> **In VS Code.** Open `.gitignore` from the Explorer to replace its content.

**2. Commit.** Commit everything the chapter has made so far:

```sh
# in zenoh_sensors
git add .
git commit -m "Pin Flutter 3.47.2 and create sensorctl"
```

The commit takes twelve files, and leaves two folders out on purpose: `.fvm/` and `apps/sensorctl/.dart_tool/`. Both
belong to this machine alone: `.fvm/` holds links into your home folder, and the Dart tools rebuild `.dart_tool/` whenever
they need it. `.vscode/settings.json` is in, because its paths count from the top folder and are right on any machine.

> **In VS Code.** **View › Source Control** lists the same twelve files under **Changes**. Choose the **+** on the
> **Changes** line to stage them all, type the message in the box above them, and choose **Commit**.

## 6 — Two programs, two terminals

The package ships its examples as source, next to its own code. This section copies two of them into `sensorctl` and
runs them against each other: `z_sub` subscribes to a key expression and prints whatever arrives, and `z_put` puts one
value on a key and ends. They are the pair the book's chapter 2 writes in Rust.

**1. Copy the two examples.** They are in pub's cache under your home folder, in the folder pub unpacked the package
into, and your project's `.dart_tool/package_config.json` records which folder that is. The second block below reads it
into a shell variable, `pkg`, then copies the two programs and `common_args.dart`, the command-line options they share.

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

`ls "$pkg/example"` shows the rest: some thirty programs, each named after the zenoh-c example it mirrors. Later chapters
start from several of them.

**2. Add what they need.** The examples read their command-line options with the `args` package. `zenoh_dart` depends on
it, so pub already has it, but a program may only import the packages its own `pubspec.yaml` names:

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart pub add args
```

**3. Start the subscriber.**

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

The two options decide who can reach it. `-l tcp/127.0.0.1:7447` makes it listen for zenoh connections on port 7447 of
the loopback interface, which no other machine can reach. `--no-multicast-scouting` stops it announcing itself on your
network. Without them it would run on zenoh's default configuration, which the package's README warns about: it listens
for TCP connections on every network interface, joins UDP multicast scouting, and accepts peers without authentication
or encryption. This guide keeps every program on the loopback until the last chapter, which takes it onto a real network
and secures it.

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
configuration directly, the list of endpoints to listen on, and empties it: by default a zenoh peer also listens while it
connects, on every interface, on a port picked at random. `-k` and `-p` give the key and the value.

The last line of the publisher's output is red and says `ERROR`, yet the put worked: the subscriber has it, and `z_put`
ended normally.

> **Note: a bug in zenoh 1.8.0.** This release of `zenoh_dart` is built on zenoh 1.8.0, which logs this error whenever a
> program closes its session while still connected to another. Nothing has failed. zenoh fixed it in version 1.10.0, and a
> later release of `zenoh_dart` built on that version will not print it. Until then you will see it after every put in
> this chapter, and again whenever one of your programs closes a connected session.

**5. Put a value it does not match.**

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart run example/z_put.dart \
  -e tcp/127.0.0.1:7447 --no-multicast-scouting --cfg 'listen/endpoints:[]' \
  -k demo/other/test -p 'Not under demo/example'
```

The publisher prints the same three lines with the new key and value, and the subscriber prints nothing, because
`demo/other/test` is not under `demo/example`. Key expressions and their wildcards are the subject of the book's chapter
3, which the guide follows next.

**6. Stop the subscriber** with Ctrl-C in the first terminal. The example catches it, closes its subscriber and its
session, and ends.

> **If you already know zenoh.** `common_args.dart` is the Dart translation of zenoh-c's `parse_args.h`, so these are the
> flags you know, down to `--cfg KEY:VALUE` taking a JSON5 value.

> **In VS Code.** Open the second terminal with the **+** at the top right of the terminal panel, or with **Terminal ›
> Split Terminal** to see both side by side. Every new terminal starts in `zenoh_sensors`, so type
> `cd apps/sensorctl` in it first. Ctrl-C works in VS Code's terminal as in any other.

## 7 — The same from VS Code

VS Code can start the two programs itself, from its Run and Debug view, with the same SDK and the same options, and show
their output in its Debug Console. It needs one file that describes each program: `zenoh_sensors/.vscode/launch.json`.

**1. Describe the two programs.** Open the file from `zenoh_sensors` with `code .vscode/launch.json`, which creates it when
you save, and give it this content:

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
    }
  ]
}
```

Each entry names a program, the folder to start it in, and its options, one per string: the options you typed in the
previous section. `${workspaceFolder}` stands for the folder VS Code has open, `zenoh_sensors`. The `cwd` line is the
project section's rule written down: start the program in `apps/sensorctl`, where the package's libraries are. VS Code
would pick that folder even without the line, as the nearest one above the program with a `pubspec.yaml`; the line makes
the choice visible, and the next chapter changes it.

**2. Start the subscriber.** Open the Run and Debug view with **View › Run**. At its top, choose **z_sub** in the list of
configurations and press the green ▶ beside it, or F5. The Debug Console at the bottom of the window shows the three lines
you saw in the terminal, and `z_sub` waits.

**3. Put a value.** Choose **z_put** in the same list and press ▶ again; the two now run side by side. `z_put` prints its
three lines, zenoh 1.8.0's ERROR line among them, and ends. A list at the top of the Debug Console switches between the
two programs' output, and `z_sub`'s shows the line it received.

**4. Stop the subscriber.** In the small debug toolbar at the top of the window, with `z_sub` selected, press the red
square, or Shift+F5. VS Code asks the program to stop, and the example closes its subscriber and its session as it did
for Ctrl-C.

F5 starts a program with VS Code's debugger attached, so a breakpoint set in `example/z_sub.dart` stops it there;
**Run › Run Without Debugging**, Ctrl+F5, starts it without the debugger. Both run the program the same way.

**5. Commit.** The chapter ends by committing what it added. In the terminal from the previous section, go back to the top
folder; a new terminal in VS Code already starts there.

```sh
# in zenoh_sensors/apps/sensorctl
cd ../..
```

```sh
# in zenoh_sensors
git add .
git commit -m "Run the package's z_sub and z_put examples"
```

The commit takes `launch.json`, the three example files, and `pubspec.yaml` and `pubspec.lock`, which now name `args`.

> **In VS Code.** **View › Source Control** lists the same six files under **Changes**. Choose the **+** on the
> **Changes** line to stage them all, type the message in the box above them, and choose **Commit**.

## 8 — What changed in the architecture

Nothing yet, on purpose. `sensorctl` is still the template's program, and the two programs that talked to each other are
the package's own, unchanged. This chapter laid the ground the rest stands on: a pinned toolchain, a repository, and proof
that zenoh runs on your machine.

The architecture starts in the next chapter. Both applications in this guide are built in layers, in the pattern
Flutter's [architecture guide](https://docs.flutter.dev/app-architecture/guide) calls MVVM, for model, view and view
model. By the end of chapter 3, `sensorctl` has these layers, with the terminal as its view:

```
view  →  view model  →  repository  →  codec  →  ZenohService  →  zenoh_dart
```

Each layer uses only the ones to its right. Only `ZenohService` imports `zenoh_dart`, and it hands plain Dart values to
the layers on its left, so that everything above it can be tested without a network.

Chapter 1 follows the book's chapter 4, [Sessions and Configuration](https://kydos.github.io/zenoh-book/chapter_04.html),
and starts from another of the package's examples, `z_info`. You write it as one flat program that opens a session on the
loopback, as the options did here, and prints who it is; then, with a test in place, you move its zenoh code into the
first layer, `ZenohService`. In the same chapter `zenoh_sensors` becomes a pub workspace, with the service in a package of
its own that the phone app will share in Part 2.

## 9 — Files and versions at the end of this chapter

`zenoh_sensors` now holds this, in two commits: `Pin Flutter 3.47.2 and create sensorctl` and `Run the package's z_sub
and z_put examples`.

```
zenoh_sensors/
├── .fvm/
├── .fvmrc
├── .git/
├── .gitignore
├── .vscode/
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

This chapter was checked with these versions. Newer ones are expected to work; if something does not, these are the ones
to go back to.

| what | version |
|---|---|
| Linux | Ubuntu 26.04.1 on x86_64, glibc 2.43 |
| git | 2.53.0 |
| fvm | 4.3.1 |
| Flutter, and the Dart it carries | 3.47.2, Dart 3.13.2 |
| VS Code | 1.138.0 |
| Dart and Flutter extensions | 3.142.0 |
| `zenoh_dart`, built on zenoh 1.8.0 | 1.0.0-rc.1 |
| `args` | 2.7.0 |
| `path` | 1.9.1 |
| `lints` | 6.1.0 |
| `test` | 1.32.0 |


---

*The code listings in this chapter are licensed under the Apache License 2.0. The text is © 2026 Hugo Alberto Garcia,
all rights reserved — see [COPYRIGHT](../COPYRIGHT).*

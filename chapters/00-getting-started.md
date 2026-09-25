# 0 — Getting started

This chapter follows chapters 1 and 2 of *Zenoh Programming in Rust*, *Introduction to Zenoh* and *Getting Started*,
and *The Zenoh Book*'s pages with the same names. It starts from two programs shipped with the `zenoh_dart` package,
`z_sub` and `z_put`.

## 1 — What you build, and what you will see

By the end of this chapter you have a working toolchain and a git repository with a Dart project that depends on
`zenoh_dart`. You run two of the package's example programs on your laptop, each in its own terminal: one subscribes to
a key expression, and the other publishes one message to it. Then you run the same two programs from VS Code.

Nothing here is yours yet. The two programs are the package's examples, copied unchanged. The first code you write
comes in the next chapter, with its first test. This chapter makes sure everything after it starts from a known place:
a Flutter version at or above the one this guide was checked with, the package at a known version, an editor that runs
the same SDK, and proof that zenoh runs on your machine before any code of yours is involved.

At the end, the subscriber's terminal shows:

```
>> [Subscriber] Received PUT ('demo/example/test': 'Hello from the guide')
```

The publisher's terminal shows a red line that says `ERROR`, after the message has been delivered. It is harmless, and
this chapter explains where it comes from, so that it does not stop you the first time.

> **If you already know zenoh.** The package's examples mirror the zenoh-c examples flag for flag, so `z_sub` and
> `z_put` do here what they do in C, with the same options. What is new to you is Dart's tooling, and this chapter is
> mostly that.

> **If you already know Flutter.** There is no app until chapter 2. The Flutter SDK is needed now only because it
> carries the Dart SDK this guide uses everywhere, pinned per project with `fvm`. If you have never used `fvm`, this
> chapter shows the three commands you need.

## 2 — What to read

This guide builds on two books by Angelo Corsaro, and each book does a different job. Each chapter points you to the
pages you need, at the step where they help.

**[The Zenoh Book](https://corsaro.me/zenoh/book/), for the idea.** It says what zenoh is, what it was built to
solve, and when it is the right answer. It is written above any particular release, so it ages slowly.

**[Zenoh Programming in Rust](https://kydos.github.io/zenoh-book/), for the shape of the API.** It covers the calls,
their options, and the ways they go wrong. It is a draft, written against Zenoh 1.4.0, and the package this guide uses
is built on 1.8.0, so a detail there may have changed since.

Both books use Rust. Read them for the ideas and the words, and type nothing from them. Every statement this guide
makes about the Dart API was read in the package itself.

For this chapter:

**[Introduction](https://corsaro.me/zenoh/book/introduction/), in The Zenoh Book, and
[chapter 1, Introduction to Zenoh](https://kydos.github.io/zenoh-book/chapter_01.html), in the other.** Read both,
because together they are short. Take the words the rest of this guide uses without explaining them again: key
expression, session, publisher and subscriber, queryable and query, and the three modes a session can run in, peer,
client and router. The case each book makes for when to use zenoh is the argument for the application you are about
to build: a sensor on one device, a program watching it on another, and no server in between.

**[Getting Started](https://corsaro.me/zenoh/book/getting-started/) and
[chapter 2, Getting Started](https://kydos.github.io/zenoh-book/chapter_02.html).** Read as far as each one's first
publisher and subscriber, and skip the rest for now. Both install Rust and a zenoh router, `zenohd`, and end with a
subscriber receiving one message. This chapter does the same in Dart, with the package's `z_sub` and `z_put` in place
of the Rust programs.

Two things differ from the books. There is no Rust to install, because the package ships zenoh's native library
inside it. And there is no router. The books' programs find each other over the local network, and here they connect
over the loopback interface only. Chapter 5 brings in the network and a router. Section 6, *Two programs, two
terminals*, says how and why.

**[`args`](https://pub.dev/packages/args), for more than a sentence on it.** The package's examples read their
command-line options with it, and you add it for them in section 6. For the rest, read its documentation.

## 3 — The toolchain

**A Linux machine on x86_64.** It is the one hard requirement. The package ships zenoh's native library for Linux on
x86_64 and for Android, and for nothing else, so the laptop side of this guide cannot run on macOS or Windows. The
library needs glibc 2.34 or newer, which any distribution from the last few years has. Nothing from Android is needed
in this chapter or the next. From chapter 2, when the sensor node moves onto a phone, you also need the Android tools
in item 5 below.

You need five tools. This guide names each one, with the version it was checked with, and installing them is
yours to do.

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
even in this chapter, where there is no app, so that the app changes nothing about the tools when it arrives. Fetch
the SDK this project uses:

```sh
fvm install 3.47.2
```

On a fresh machine this clones the SDK and downloads what it needs to run, which takes several minutes and a few
gigabytes. `fvm list` then shows `3.47.2`, with Dart `3.13.2` beside it. If you would rather use a newer stable
release, fetch that one instead, and use its number wherever this guide says `3.47.2`. The package needs Flutter
3.47.1 or newer, and the chapters were checked on 3.47.2.

Do not put `flutter` or `dart` on your `PATH`, and do not use `fvm global`. Every command in this guide that touches
Dart or Flutter starts with `fvm`, as in `fvm dart run` and `fvm flutter test`, which runs the SDK the current project
names. A bare `dart` on the `PATH` is how a project gets built with the wrong SDK without anyone noticing.

**4. VS Code with the Flutter extension.** This guide was checked with [VS Code](https://code.visualstudio.com/)
1.138.0 and version 3.142.0 of the
[Flutter extension](https://marketplace.visualstudio.com/items?itemName=Dart-Code.flutter), which brings the Dart
extension with it. The extensions update themselves, and newer is fine. Each project tells the editor which SDK to
use, in the next section, so that what runs from a VS Code button is the same SDK as what runs from `fvm`.

> **If you already know Flutter.** You may have a Flutter SDK on your `PATH` already. Leave it there if other projects
> need it. This guide never calls it, because every command goes through `fvm` and the project's own `.fvmrc`, and the
> next section points VS Code at the same SDK.

**5. An Android emulator, an Android phone, and `adb`.** Nothing in this chapter or the next uses them. From chapter 2
the sensor node runs on the emulator and then on a phone over its USB cable, and from chapter 5 the phone is on your
Wi-Fi network. `adb` comes with the Android SDK's platform tools, and the emulator with the SDK itself. This guide was
checked with adb 1.0.41 (37.0.1) and emulator 37.1.11, against a virtual device on API 37, and with a Pixel 9a on
Android 17.

**The one requirement is the architecture.** `zenoh_dart` ships zenoh's native library for Android on `arm64-v8a`,
`armeabi-v7a` and `x86_64`. So the virtual device needs an **x86_64** system image, the kind that runs on an x86_64
machine, and a phone works as it is. Installing the SDK, creating a virtual device and turning on a phone's developer
options are Android's own business, and [its documentation](https://developer.android.com/studio/run/managing-avds)
describes them. This guide only uses what you have.

**About versions.** Every version number in this guide is the lowest the chapter was checked with, and a newer one
should work. Each chapter ends with the exact versions it was checked with: the tools, the SDK, and the packages your
`pubspec.yaml` names. If a listing ever stops compiling on your machine, pin those first. `.fvmrc` already names one
SDK version. A package is pinned by replacing the caret in its constraint, `^1.0.0-rc.1`, with the exact version the
chapter names, `1.0.0-rc.1`, and then running `fvm dart pub get`.

The toolchain is ready. Section 4 creates the project.

## 4 — The project

Create the project folder, pin its SDK, and create the program. Everything you build in this guide lives in one
folder, `zenoh_sensors`, which is a git repository from its first minute. In this chapter it holds one small
command-line program, `sensorctl`. From the next chapter it also holds the code that program shares with the phone
app, and from chapter 2 the app itself. Put it wherever you keep your projects, such as your home folder. From here
on, this guide names every folder from `zenoh_sensors` down. For example, `zenoh_sensors/apps/sensorctl` is the
program's folder, wherever `zenoh_sensors` itself lives.

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

Every command block from here on starts with a comment that names the folder it runs in. When a block contains a
`cd`, the next block names the folder it moved you to.

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
> guide, because the settings the extensions read are in `zenoh_sensors/.vscode`, and the paths in them count from the
> folder VS Code has open.

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
mkdir apps .vscode
fvm use 3.47.2 --force
```

The pin comes first, because nothing else runs without it. You have no global SDK and no `dart` on your `PATH`, so
outside a pinned folder `fvm dart` has nothing to start and stops with `dart: not found`. Inside one, fvm finds the pin
in the folder you are in or in any folder above it, so everything under `zenoh_sensors` uses this one.

fvm prints two warnings, both expected. `fvm use` is meant for a folder that already holds a Dart project, and it asks
before pinning one that does not. `--force` answers yes.

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
left if VS Code has the folder open, or with `code .vscode/settings.json` from `zenoh_sensors`. Add the second line,
so that the file reads:

```json
{
  "dart.flutterSdkPath": ".fvm/versions/3.47.2",
  "dart.sdkPath": ".fvm/flutter_sdk/bin/cache/dart-sdk"
}
```

The first line is fvm's, and names the Flutter SDK. The second is yours, and everything you build depends on it. The
program you are about to create is plain Dart. For a plain Dart project, the Dart extension uses `dart.sdkPath` if it
is set, then the first `dart` on your `PATH`, and only then the Flutter SDK named on the first line. With the second
line, the editor runs the same Dart as `fvm dart`, whatever else is on your `PATH`. `.fvm/flutter_sdk` is a link that
fvm keeps pointing at the pinned SDK, so the line stays right when you change the pin, and `fvm use` keeps it when it
rewrites the file.

**5. Create the program.**

```sh
# in zenoh_sensors
cd apps
fvm dart create -t console sensorctl
```

`-t console` picks Dart's template for a command-line program. `bin/sensorctl.dart` holds `main`, which calls a
function in `lib/sensorctl.dart`, and `test/` holds a test of that function. The next chapter replaces all three. When
the tool finishes it suggests `dart run`, but this guide always writes `fvm dart run`.

> **In VS Code.** Type the command above in VS Code's terminal. Do not use the Command Palette's **Dart: New
> Project**. It runs the same template, but then it reopens VS Code on `zenoh_sensors/apps/sensorctl`, and VS Code must
> stay open on `zenoh_sensors`.

**6. Add the package.** Go into the program's folder first, because the next two steps run there.

```sh
# in zenoh_sensors/apps
cd sensorctl
```

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart pub add 'zenoh_dart:^1.0.0-rc.1'
```

The command names the version, because the newest `zenoh_dart` pub picks by itself is `0.30.0`. That is the same code
as `1.0.0-rc.1`, published under a number below 1.0, because pub never picks a release candidate on its own. A bare
`fvm dart pub add zenoh_dart` would write `^0.30.0`, which stops short of every 1.0 release candidate and of 1.0.0
itself. `^1.0.0-rc.1` means this release candidate or any later version below 2.0.0.

pub also notes that one package has a newer version it cannot use. That package is one `zenoh_dart` depends on, and it
needs nothing from you.

**7. Run it.**

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart run
```

The template's program prints `Hello world: 42!`. Nothing of zenoh runs yet, but the words `Running build hooks...` in
front of it are the package at work. `zenoh_dart` has a *build hook*, a small Dart program of its own that `dart run`
runs before yours. It has copied zenoh's native libraries into your project:

```sh
# in zenoh_sensors/apps/sensorctl
ls .dart_tool/lib
```

Two files are there. `libzenohc.so` is zenoh itself, zenoh-c 1.8.0, the C interface to zenoh's Rust core.
`libzenoh_dart.so` is the thin layer the package's Dart code calls it through.

The package looks for both in `.dart_tool/lib` under the folder a program is **started from**, which its README lists
as a known issue of this release candidate. Until it is fixed, follow one rule: start your programs from
`zenoh_sensors/apps/sensorctl`, a folder you own. A program started in a folder someone else can write to could load a
library placed there. The next chapter shows what changes when `zenoh_sensors` becomes a workspace.

`Running build hooks...` appears in front of every program's output from here on, sometimes with a line or two more
from pub. The guide shows those lines as `⋮` and starts an expected output at the program's first words. `…` stands
for the part of a line that differs on your machine.

> **If you already know zenoh.** `libzenohc.so` is the library a C program links. The package's examples work with
> zenoh-c's, and its README notes that a Dart `z_get` can query a C `z_queryable`.

> **In VS Code.** Open `apps/sensorctl/bin/sensorctl.dart` from the Explorer, and choose **Run** in the line of small
> links above `main`. VS Code starts the program in `zenoh_sensors/apps/sensorctl`, the nearest folder above it with a
> `pubspec.yaml`, which is the folder the rule above asks for. The Debug Console at the bottom of the window shows
> `Hello world: 42!`.

The project exists and runs. Section 5 decides what git keeps, and makes the first commit.

## 5 — What git keeps

`fvm use` and `fvm dart create` each wrote a `.gitignore`: the top folder's lists `.fvm/`, and the program's lists
`.dart_tool/`. Before the first commit, replace the top folder's with this project's own list, with a reason for every
line. Go back to the top folder first:

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

The first three rules are the Dart team's own, from dart.dev's page
[What not to commit](https://dart.dev/tools/pub/private-files). `.dart_tool/` holds the Dart tools' working files,
zenoh's libraries among them. `build/` holds what `dart build` and `flutter build` make. `doc/api/` holds the
documentation `dart doc` writes. A rule ending in `/` with no other slash applies in every folder below, and `**/`
gives the third rule the same reach. `.fvm/` is fvm's line from before.

Two things are left out of the list. `pubspec.lock` is not in it, so it is committed. It records the exact version of
every package pub chose. For a program, an application in dart.dev's words, the Dart team recommends committing it, so
that every machine gets the same versions.

The files your editor or operating system makes are not in it either, such as IntelliJ's `.idea/` or macOS's
`.DS_Store`. dart.dev suggests keeping those in an ignore file of your own that applies to every repository on your
machine. git reads one from `~/.config/git/ignore`, as GitHub's page
[Ignoring files](https://docs.github.com/en/get-started/git-basics/ignoring-files#configuring-ignored-files-for-all-repositories-on-your-computer)
explains.

The program keeps the `.gitignore` that `fvm dart create` wrote. Its one rule, `.dart_tool/`, is also in the top
folder's list. A package created later keeps the list its own tool writes, in the same way.

> **In VS Code.** Open `.gitignore` from the Explorer to replace its content.

**2. Commit.** Commit everything the chapter has made so far:

```sh
# in zenoh_sensors
git add .
git commit -m "Pin Flutter 3.47.2 and create sensorctl"
```

The commit takes 12 files, and leaves two folders out: `.fvm/` and `apps/sensorctl/.dart_tool/`. Both belong to this
machine alone. `.fvm/` holds links into your home folder, and the Dart tools rebuild `.dart_tool/` whenever they need
it. `.vscode/settings.json` is in, because its paths count from the top folder and are right on any machine.

> **In VS Code.** **View › Source Control** lists the same 12 files under **Changes**. Choose the **+** on the
> **Changes** line to stage them all, type the message in the box above them, and choose **Commit**.

The first commit is made. Section 6 runs two of the package's programs against each other.

## 6 — Two programs, two terminals

Run two of the package's programs against each other. The package ships its examples as source, next to its own
code. `z_sub` subscribes to a key expression and prints whatever arrives, and `z_put` puts one value on a key and ends.
They are the pair both books' getting-started pages write in Rust.

**1. Copy the two examples.** They are in pub's cache under your home folder, in the folder pub unpacked the package
into, and your project's `.dart_tool/package_config.json` records which folder that is. The second block below reads it
into a shell variable, `pkg`. It then copies the two programs and `common_args.dart`, the command-line options they
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

`ls "$pkg/example"` shows the rest, about 30 programs, each named after the zenoh-c example it mirrors. Later chapters
start from several of them.

**2. Add what they need.** The examples read their command-line options with the `args` package. Pub already has it,
because `zenoh_dart` depends on it. A program can only import the packages its own `pubspec.yaml` names, so add it:

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

Both options limit who can reach the subscriber. `-l tcp/127.0.0.1:7447` makes it listen on port 7447 of the loopback
interface, which no other machine can reach. `--no-multicast-scouting` stops it announcing itself on your network.

Leave them out and it runs on zenoh's default configuration, which the package's README warns about: it listens for TCP
connections on every network interface, joins UDP multicast scouting, and accepts peers without authentication or
encryption. This guide keeps every program on the loopback until chapter 5, which takes the phone onto your network.
The last chapter secures it.

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

The last line of the publisher's output is red and says `ERROR`, but the put worked. The subscriber has it, and
`z_put` ended normally.

> **Note: a bug in zenoh 1.8.0.** This release of `zenoh_dart` is built on zenoh 1.8.0, which logs this error
> whenever a program closes its session while still connected to another. Nothing has failed. Zenoh fixed it in
> version 1.10.0, and a later release of `zenoh_dart` built on that version will not print it. Until then you see it
> after every put in this chapter, and again whenever one of your programs closes a connected session.

**5. Put a value it does not match.**

```sh
# in zenoh_sensors/apps/sensorctl
fvm dart run example/z_put.dart \
  -e tcp/127.0.0.1:7447 --no-multicast-scouting --cfg 'listen/endpoints:[]' \
  -k demo/other/test -p 'Not under demo/example'
```

The publisher prints the same three lines with the new key and value, and the subscriber prints nothing, because
`demo/other/test` is not under `demo/example`. Key expressions and their wildcards are the subject of chapter 3 of
*Zenoh Programming in Rust*, which this guide's chapter 2 follows.

**6. Stop the subscriber** with Ctrl-C in the first terminal. The example catches it, closes its subscriber and its
session, and ends.

> **If you already know zenoh.** `common_args.dart` is the Dart translation of zenoh-c's `parse_args.h`. The flags
> are the ones you know, including `--cfg KEY:VALUE` with a JSON5 value.

> **In VS Code.** Open the second terminal with the **+** at the top right of the terminal panel, or with **Terminal ›
> Split Terminal** to see both side by side. Every new terminal starts in `zenoh_sensors`, so type
> `cd apps/sensorctl` in it first. Ctrl-C works in VS Code's terminal as in any other.

The two programs talked on your laptop. Section 7 runs them from VS Code.

## 7 — The same from VS Code

Run the two programs from VS Code's Run and Debug view, with the same SDK and the same options. VS Code shows their
output in its Debug Console. It needs one file that describes each program, `zenoh_sensors/.vscode/launch.json`.

**1. Describe the two programs.** From `zenoh_sensors`, open the file with `code .vscode/launch.json`. VS Code creates
it when you save. Give it this content:

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

Each entry names a program, the folder to start it in, and its options, one per string. The options are the ones you
typed in the previous section. `${workspaceFolder}` stands for the folder VS Code has open, `zenoh_sensors`.

The `cwd` line starts the program in `apps/sensorctl`, where the package's libraries are. Without the line VS Code
picks the same folder, because it is the nearest one above the program with a `pubspec.yaml`. The line makes the
choice visible, and the next chapter changes it.

**2. Start the subscriber.** Open the Run and Debug view with **View › Run**. At its top, choose **z_sub** in the list
of configurations and press the green ▶ beside it, or F5. The Debug Console at the bottom of the window shows the
three lines you saw in the terminal, and `z_sub` waits.

**3. Put a value.** Choose **z_put** in the same list and press ▶ again. The two now run side by side. `z_put` prints
its three lines, zenoh 1.8.0's `ERROR` line among them, and ends. A list at the top of the Debug Console switches
between the two programs' output, and `z_sub`'s shows the line it received.

**4. Stop the subscriber.** In the small debug toolbar at the top of the window, with `z_sub` selected, press the red
square, or Shift+F5. VS Code asks the program to stop, and the example closes its subscriber and its session, as it
did for Ctrl-C.

F5 starts a program with VS Code's debugger attached, so a breakpoint set in `example/z_sub.dart` stops it there.
**Run › Run Without Debugging**, Ctrl+F5, starts it without the debugger. Both run the program the same way.

**5. Commit.** Commit what this chapter added. In the terminal from the previous section, go back to the top folder. A
new VS Code terminal already starts there.

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

> **In VS Code.** **View › Source Control** lists the same 6 files under **Changes**. Choose the **+** on the
> **Changes** line to stage them all, type the message in the box above them, and choose **Commit**.

The chapter's work is committed. Section 8 looks ahead to the architecture.

## 8 — What changed in the architecture

Nothing has changed yet. `sensorctl` is still the template's program, and the two programs that talked to each other
are the package's own, unchanged. This chapter laid the ground the rest stands on: a pinned toolchain, a repository,
and proof that zenoh runs on your machine.

The architecture starts in the next chapter. Both applications in this guide are built in layers, in the pattern
Flutter's [architecture guide](https://docs.flutter.dev/app-architecture/guide) calls MVVM: model, view and view model.
Over the next chapters, `sensorctl` gains these layers, with the terminal as its view. It has all of them by chapter 3
except the codec, which arrives in chapter 6:

```
view  →  view model  →  repository  →  codec  →  ZenohService  →  zenoh_dart
```

Each layer uses only the ones to its right. Only `ZenohService` imports `zenoh_dart`, and it hands plain Dart values to
the layers on its left, so everything above it can be tested without a network.

Chapter 1 follows chapter 4 of *Zenoh Programming in Rust*,
[Sessions and Configuration](https://kydos.github.io/zenoh-book/chapter_04.html), and starts from another of the
package's examples, `z_info`. You write it as one flat program that opens a session on the loopback, as the options
did here, and prints who it is. Then, with a test in place, you move its zenoh code into the first layer,
`ZenohService`. In the same chapter, `zenoh_sensors` becomes a pub workspace, with the service in a package of its own
that the phone app shares from chapter 2.

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
| `args` | 2.7.0 |
| `path` | 1.9.1 |
| `lints` | 6.1.0 |
| `test` | 1.32.0 |


---

*The code listings in this chapter are licensed under the Apache License 2.0. The text is © 2026 Hugo Alberto Garcia,
all rights reserved — see [COPYRIGHT](../COPYRIGHT).*

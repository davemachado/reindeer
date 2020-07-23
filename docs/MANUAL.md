# Reindeer Care and Feeding

(In progress)

## Introduction

One of Rust’s strengths is its ecosystem of third-party crates. This ecosystem is built
around Cargo and crates.io. But what if you don't want to use
Cargo? What if you want to use a buildsystem like [Buck](https://buck.build)[^1]?

[^1]: Or potentially [Bazel](https://bazel.build), but nobody has tried that yet.

Reindeer is a tool intended to bridge that gap - it allows you to import Cargo packages
from crates.io, github, or whereever else they are stored, vendors them and automatically
generates Buck build rules so they can be built from source as part of your normal build
process.

This assumes your environment is:
1. A monorepo-like, in which all your sources are stored together and updated synchronously, and
2. You have a unified dependency graph in which all builds are managed by Buck.
This means you use Buck for both your home-grown code as well as third-party code.

(Neither of these are absolutely mandatory; they're just the environment the tool
was developed in. Your experience may vary depending on how close your environment
matches this.)

## An Example

See the [example directory](../example/README.md) as a starting point. This has:
- Your first-party code in "project" (though of course it can be anywhere and everywhere), and
- A [third-party](../example/third-party) directory which is managed by Reindeer
- A [`setup.sh`](../example/setup.sh) script to get you bootstrapped

Running `setup.sh` will build Reindeer (using Cargo) and then use it to vendor the small
number of third-party dependencies defined in
[`third-party/Cargo.toml`](../third-party/Cargo.toml) and generate build rules for them in `third-party/BUCK`.

I recommend using this as a starting template for your own project, at least until you're
familiar enough with how everything works to do it yourself.

(There's quite a lot else in there, but we'll get to that.)

## Basic Workflow

You're working away on your code, and you suddenly need to use something from crates.io.
What do you do?

1. Add the specification to `[dependencies]` in `third-party/Cargo.toml`,
as you would if this were a Cargo project. You can use all the usual options, from a
simple `foo = "1.2"` to adding features, defining a local name, and so on.
2. Run `reindeer --third-party-dir third-party vendor`. This will resolve the new
dependencies (creating or updating `Cargo.lock`), vendor all the new code in the
`third-party/vendor` directory (also deleting unused code)
3. Run `reindeer --third-party-dir buckify`. This will analyze the Cargo dependencies
and (re)generate the BUCK file accordingly. If this succeeds silently then there's a
good chance that nothing more is needed.
4. Do a test build with `buck build //third-party:new-package#check` to make sure it
is basically buildable.

Points to note:
- If any of the packages you're importing (either the ones you're explicitly importing
or their dependencies) has a `build.rs` build script, you'll need a `fixups.toml` file
for that package to tell Reindeer how to handle it. See [fixups](#Fixups) for details.
- The `vendor` directory is completely under Reindeer's control, and can be deleted and
regenerated at any time. Do not make any manual local changes in there.
See [below](#Local-Patches) for details on how to maintain local changes.
- Likewise `BUCK` is always completely regenerated by `reindeer buckify`. The generated
rules can be customized in the [reindeer configuration](#Configuring-Reindeer) or in the
[rule macros](#Buck-Macros).

## Vendoring and Managing Versions

Reindeer maintains a directory of all third-party sources used during a build.
The expectation is that they are committed along with your own code so that they
are all updated in lockstep. It also means the build process needs no network IO.

### Multiple versions of one package

This model of managing third-party code pushes heavily on the idea that there's
only one version of any given package in use. If you're doing an update and it
introduces some minor changes, I'd suggest updating the callsites to the new API
rather than introducing a new version.

Sometimes, however, it's impractical to update everything. When there's a large
API change - or even a whole ecosystem change - then its not practical to migrate
everything at once. In this case you can introduce a second (or more) version.

For example:
```
[dependencies]
tokio_old = { package = "tokio", version = "0.1" }
tokio = "0.2"
```

Note that even if you don't do this, you could have multiple versions of each package
vendored and part of you build dependency graph - when resolving versions, Cargo will
pull in as many versions as it needs to satisfy everyone's dependencies. They will almost
always happily co-exist.

### Specifying features

You can specify features in the normal way:
```
specialpackage = { version = "10.2", features = ["magic] }
```

### Importing from Git

You are not limited to just crates from crates.io - you can also use packages which
have not yet been published using `git` references.

Note that if you're switching a dependency between fetched from crates.io and git,
you'll need to manually delete the vendored code to avoid a bug in
[cargo vendor](https://github.com/rust-lang/cargo/issues/8181).
You can be indiscriminate about this, up to removing the entire content of the `vendor/`
dir.

### Dealing with merge conflicts

If two people are updating the third-party repo at once, there's the possibility of
collisions. The most likely place to get a collision is in `Cargo.toml`
(if the entries are near each other) and `Cargo.lock` (similarly). These can be resolved
in the normal way, but its worth re-running a vendor operation after updating Cargo.lock
to make sure the results are consistent.

Alternatively you can simply delete `Cargo.lock`, but that will cause everything to be
re-resolved (see [bulk updates](#bulk-updates)).

## Local Patches
(TODO)

## Bulk Updates
(TODO)

## Rustsec Auditing
(TODO)

## Configuring Reindeer
(TODO)

## Buckifying

In the best - and most common - case, generating Buck build rules is completely
automated. If the crate has no build script (and is therefore pure Rust), then the
chances are high that the generated rules will Just Work.

Even if they don't, most cases can be solved with a one or two line annotation.

## Fixups

Fixups are annotations to help Reindeer generate correct build rules for the Cargo
packages. They're generally only needed when the Cargo build does something that's not
precisely described by the Cargo metadata, such as the arbitrary actions of build scripts.

Fixups are defined in `fixups/<package name>/fixups.toml`. The package name is the base
name, not including any version information. The fixups directory also contains
other files as needed.

### Extra sources

By default Reindeer will simply add all `*.rs` files as the `srcs` for the rule.
If you're using the `precise_srcs` option then it will attempt to identify all the
sources by actually parsing the code. Both of these can fail from time to time -
such as by `include!()` of unexpected files, or when files or modules are introduced
by macros.

These extra sources can be added with
```
extra_srcs = [ ... ]
```
in `fixups.toml`, where
the extra sources are specified as one or more globs.

### Environment variables

Some packages use version and other information from Cargo via a set of environment
variables. If a build fails with a message about `CARGO_<something>` not being defined,
then you can add `cargo_env = True` to `fixups.toml`.

Sometimes they need an arbitrary environment variable to be defined. You can specify this
with
```
env = { "FOO" = "Value of FOO" }
```

### Build scripts
(TODO)

## Buck Macros
(TODO)

## Multi-platform Support
(TODO)
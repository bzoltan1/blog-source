---
title: "Reverse dependencies as a zypper plugin"
subtitle: "From a hackweek project to a zypper subcommand"
date: 2026-09-02T06:52:00+02:00
tags: ["openSUSE", "SLES", "SUSE", "Linux", "zypper", "Quality engineering"]
type: post
---

A few years ago I wrote a [blog post]({{< ref "2023-02-01-Reverse-dependencies.md" >}}) about a small hackweek project called `rdepends`. The idea was simple: given a package, find out what other packages depend on it, recursively. It lived in my home project on the build service and it was a useful but rough tool.

Since then a few things happened that made me revisit it.

### Zypper got the feature

Back when I started this project, zypper had no built-in way to ask "what depends on package X?". You could ask the other direction easily with `zypper info --requires`, but not the reverse. I raised this with the zypper developers and Benjamin Zeller implemented the `--requires-pkg` flag in zypper 1.14.33. So now you can do:

```
zypper se --requires-pkg bash
```

and get the list of packages that directly depend on `bash`. This is a really good example of upstream developers listening to their users. The feature request made sense, Benjamin agreed, and it got done. That kind of responsiveness is one of the things I genuinely appreciate about working with the zypper team.

### What rdepends still adds

The zypper command gives you one level. If you want to know everything that depends on a package, directly or indirectly, you need to run it repeatedly and follow the chain yourself. That is what `rdepends` automates.

Let me show a simple example. Say we want to know what depends on `suseconnect-ng`:

```
$ zypper rdepends suseconnect-ng
cockpit-subscriptions
libsuseconnect
mcp-server-suseconnect
rollback-helper
switch_sles_sle-hpc
```

Five packages directly depend on it. But if we want the full picture:

```
$ zypper rdepends -f suseconnect-ng
agama
cockpit
cockpit-subscriptions
cockpit_client
libsuseconnect
mcp-server-suseconnect
microos_cockpit
patterns-cockpit
patterns-cockpit-client
rollback-helper
suseconnect-ruby-bindings
switch_sles_sle-hpc
```

With the `--full-tree` flag, it walks the entire reverse dependency tree. Those 5 direct dependents turn into 12 packages when you follow the chain. For example, `cockpit-subscriptions` depends on `suseconnect-ng`, and `cockpit` depends on `cockpit-subscriptions`, so `cockpit` shows up in the full tree but not in the single-level output. The tool does this by calling `zypper se --requires-pkg` for each discovered package, tracking what it has already visited to avoid cycles, and collecting the full set.

An interesting side effect: because `rdepends` uses zypper's XML output mode (`--xmlout`) internally, it is actually about 3-5x faster than running `zypper se --requires-pkg` interactively. The XML renderer is much faster than the table formatter. So you get speed for free.

### Making it a zypper plugin

With the core logic being simpler now (thanks to the `--requires-pkg` flag), I cleaned up the code and decided to package it as a proper zypper subcommand. The zypper plugin mechanism is surprisingly simple. If you put an executable named `zypper-something` on the PATH, zypper automatically picks it up as `zypper something`. No registration, no config files, no plugin API. Just a naming convention.

So I renamed the script to `zypper-rdepends`, added a man page, wrote 37 unit tests (covering XML parsing, cycle detection, deduplication, and the recursive tree walk), and packaged it as `zypper-rdepends-plugin`. It follows the same pattern as the `zypper-changelog-plugin` that I maintain in Factory. After installing the package, `zypper subcommand` lists it and you can use it as `zypper rdepends`.

The package is available in openSUSE Tumbleweed from the default repos, no extra configuration needed:

```
zypper install zypper-rdepends-plugin
```

For SLES and Leap it is not available yet. Whether it lands there depends on whether customers and product managers see value in it.

### Why this matters for quality engineering

Here is the real point. When we update a package, we test the package itself. We run its own tests, we check the scenarios we know about. But what about the things we do not know about?

Let's say package B depends on package A. We release an update for A. The maintainer of A runs the tests for A, everything passes, the update ships. But nobody checked if B still works with the new A. Maybe A changed a default, maybe a function now returns a slightly different value, maybe a shared library symbol changed behavior. B breaks, and we find out from a bug report.

This is where reverse dependency information becomes directly useful. If we know that B depends on A, and B has a test package (`B-tests`), then we can run those tests as part of the validation for the A update. Not just the tests for A, but also the tests for everything that depends on A.

I wrote about this idea from the other side in my [previous post about packaging upstream functional tests]({{< ref "2026-07-25-Enabling-upstream-functional-tests.md" >}}). There I argued that we should ship installable test packages alongside production packages. People sometimes ask why bother publishing tests that already ran during the build. The answer is exactly this: because things break after the build. A package can pass all its tests at build time and still cause failures in dependent packages when it gets installed as an update.

The two pieces fit together. Reverse dependency information tells us what to test. Installable test packages give us something to test with. If package A is updated, `zypper rdepends -f A` gives us the list, and for every package on that list that ships a `*-tests` subpackage, we can run those tests against the updated system. That is a meaningful integration test.

One thing to keep in mind: for very core packages like `glibc` or `bash`, the reverse dependency tree can be enormous and listing it takes a long time. This is not a tool you blindly run on everything. It is most useful for packages in the middle of the dependency graph where the tree is large enough to matter but small enough to act on.

### What is next

The next step is to actually start using it in the openSUSE and SLES maintenance update validation process. When a maintenance update comes in, the QE workflow could automatically query the reverse dependencies and pull in the relevant test packages.

Since the initial release, the tool also gained a `--dot` flag that outputs a Graphviz dot file of the dependency graph. You can pipe it to `dot` to get a visual representation:

```
zypper rdepends --dot suseconnect-ng | dot -Tsvg -o rdepends.svg
```

The `--dot` flag implies `--full-tree` automatically. The root package is highlighted in the graph and all edges are shown, including cases where multiple packages depend on the same thing. This was suggested by one of the zypper developers and turned out to be a very natural fit.

Here is what the graph looks like for `suseconnect-ng`:

![Reverse dependency graph of suseconnect-ng](/rdepends-suseconnect-ng.svg)

The code is at [https://github.com/bzoltan1/rdepends](https://github.com/bzoltan1/rdepends) and the package is on OBS at [home:bzoltan1/zypper-rdepends-plugin](https://build.opensuse.org/package/show/home:bzoltan1/zypper-rdepends-plugin).

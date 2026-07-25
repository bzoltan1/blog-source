---
title : "Packaging and enabling upstream functional tests"
subtitle : "A lesson from Ubuntu/Debian, applied to openSUSE"
date : 2026-07-25T05:00:00+02:00
tags : ["openSUSE", "Quality engineering", "openQA","Ubuntu","OBS"]
type: post
---

Back when I was working at Canonical on the Ubuntu phone project, test-driven development was the default and natural way to work. One thing I noticed and genuinely liked was how Ubuntu and Debian ship functional tests alongside many of their packages. Not only build-time unit tests hidden in the CI pipeline, but installable test binaries that you can run against the actual packages on an actual system. They are named `*-tests` or `*-test`, and they live in the archive next to the production packages.

The idea is simple: the upstream developers wrote tests. Why not package those tests and make them available for post-install validation? It is a good discipline.

When I joined SUSE, I noticed that we do not do this at all. The packages are there, the upstream tests are there, but we do not distribute them and we do not run them as installed-tests. This seems like a missed opportunity.

### How big is the gap?

I decided to actually measure it instead of guessing.

Ubuntu Noble (24.04) has 142 binary packages named `*-test` or `*-tests` in its archive. Of the 37,685 source packages, 52.4% have some form of autopkgtest declaration. The other 47.6% have nothing.

Of those 142 Ubuntu test packages, 79 do not even exist in openSUSE:Factory (different scope, different packaging decisions, not a problem). But the remaining 63 exist in both distros. That is where the comparison gets interesting.

Of those 63 packages that exist in both:

- 5 are genuinely well covered on the SUSE side (systemd, mariadb, openvswitch, fwupd, cockpit; they have both `%check` in the spec and openQA modules)
- 29 have no `%check` at all, no openQA module, nothing
- The rest are somewhere in between

Out of curiosity, I also cross-referenced these packages against our Bugzilla and found about 301 matched bug reports. I want to be careful here, I am not claiming that better test coverage would have prevented all those bugs. But it is a concrete data point that there are real problems being reported against these packages. The top three are `libsoup` (100 bugs, 20 open, including two active CVEs from 2025), `pipewire` (97 bugs, 39 open, 40% open rate), and `bolt` (66 bugs).

### The missing piece

Before I could do anything, I hit an immediate wall. Ubuntu runs these tests with a tool called `gnome-desktop-testing-runner`. It is a tiny C binary that depends only on GLib and libsystemd, both of which we already have. It is the standard harness for GNOME installed-tests infrastructure. And it was not packaged in openSUSE at all.

I packaged it. It took few minutes. The spec is 66 lines. It builds fine on both openSUSE Tumbleweed and SLE 15-SP6.

```
osc -A https://api.opensuse.org co home:bzoltan1/gnome-desktop-testing
```

With that in place, the path was open for many of the listed packages.

### Enabling the tests

I focused on packages where the work was straightforward: the tests already exist upstream, can run in a container without hardware or a display, and the spec change is small. The average change across the 18 packages I touched was 29 lines, -2 lines. That is it. Most packages fit one of four patterns:

**Pattern A**: the meson build already supports installed-tests, it just needed one option flipped:

```diff
 %meson \
+    -Dinstalled_tests=true \
     %{nil}

+%package tests
+Summary: Installed tests for %{name}
+Requires: gnome-desktop-testing
+
+%description tests
+Installed tests for %{name}, compatible with gnome-desktop-testing-runner.
+
+%files tests
+%{_libexecdir}/installed-tests/%{name}/
+%{_datadir}/installed-tests/%{name}/
```


**Pattern B**: test files were already being installed but buried in `%files devel`. Moving them to a dedicated `tests` subpackage is near-zero net change.

**Pattern C**: GTK tests that require a display. Rather than skipping them entirely, I added wrapper scripts that check for `$DISPLAY` or `$WAYLAND_DISPLAY` and emit a proper TAP skip (`1..0 # SKIP no display available`, exit 0) when neither is set. On a desktop or in openQA, they run normally.

**Pattern D**: a couple of packages needed small fixes. `libfprint` generated `.test` files with hardcoded OBS build paths baked in (`/home/abuild/rpmbuild/BUILD/...`), so a sed pass in `%install` rewrites them to the correct installed paths.

The full list of improved packages, now all living in `home:bzoltan1` on `build.opensuse.org`:

| Package | Tests subpackage | How to run |
|---|---|---|
| bolt | `bolt-tests` | `gnome-desktop-testing-runner bolt` |
| geocode-glib | `geocode-glib-tests` | `gnome-desktop-testing-runner geocode-glib-2` |
| glib-networking | `glib-networking-tests` | `gnome-desktop-testing-runner glib-networking` |
| graphene | `graphene-tests` | `gnome-desktop-testing-runner graphene-1.0` |
| gspell | `gspell-tests` | `gnome-desktop-testing-runner gspell-1` |
| gtksourceview4 | `gtksourceview4-tests` | `gnome-desktop-testing-runner gtksourceview-4` |
| gtksourceview5 | `gtksourceview5-tests` | `gnome-desktop-testing-runner gtksourceview-5` |
| json-glib | `json-glib-tests` | `gnome-desktop-testing-runner json-glib-1.0` |
| libei | `libei-tests` | Direct binary execution |
| libfprint | `libfprint-tests` | `gnome-desktop-testing-runner libfprint-2` |
| libjcat | `libjcat-tests` | `gnome-desktop-testing-runner libjcat` |
| libsoup (3.x) | `libsoup-tests` | `gnome-desktop-testing-runner libsoup-3.0` |
| libsoup2 | `libsoup2-tests` | `gnome-desktop-testing-runner libsoup-2.4` |
| libxmlb | `libxmlb-tests` | `gnome-desktop-testing-runner libxmlb` |
| pipewire | `pipewire-tests` | `gnome-desktop-testing-runner -p 1 pipewire-0.3` |
| samtools | `samtools-test` | `perl ./test.pl` |
| xdg-dbus-proxy | `xdg-dbus-proxy-tests` | `gnome-desktop-testing-runner xdg-dbus-proxy` |

### Does it actually work?

I validated all of them in a plain openSUSE Tumbleweed container on 2026-07-25. No special setup, no emulators, no display. Pull the container, install the RPMs from the home project, run the runner. Here is a representative sample:

```
docker run --rm -it opensuse/tumbleweed:latest bash
zypper install gnome-desktop-testing libsoup-3_0-0 libsoup-tests
gnome-desktop-testing-runner --tap libsoup-3.0
```

```
# SUMMARY: total=36; passed=36; skipped=0; failed=0
```

For the packages that need a display (gspell, gtksourceview4/5), the same command without `$DISPLAY` set gives:

```
gnome-desktop-testing-runner --tap gspell-1
# 1..0 # SKIP no display available  (×6, all exit 0)
```

And with a display, or in openQA which provides one, they run fully.

The results across all 18 packages, briefly:

- **12 PASS**: tests run and pass outright (libsoup, libsoup2, json-glib, graphene, libxmlb, libjcat, geocode-glib, glib-networking, libfprint*, bolt, samtools, xdg-dbus-proxy)
- **3 SKIP**: tests skip gracefully without display, run fully with one (gspell, gtksourceview4, gtksourceview5)
- **1 PASS***: pipewire: 22/25 pass, 2 hardware-dependent tests skip correctly
- **1 PASS***: libfprint: 34/60 pass with virtual device; 26 need real fingerprint hardware
- **1 N/A**: gcab: build-time tests only, no upstream installed-tests infrastructure

The `*` cases are not failures. The hardware tests for libfprint require an actual fingerprint reader, which is fine, that is what openQA workers with real hardware are for. The same logic applies to pipewire's ALSA stress test.

### Next steps

The goal of this work is to get these packages into `openSUSE:Factory` via submit requests, and then add openQA modules that call `gnome-desktop-testing-runner <package>` as part of the standard test run on every Factory commit.

An openQA `console` module for any of these is genuinely simple:

```perl
zypper_call "in libsoup-tests gnome-desktop-testing";
assert_script_run "gnome-desktop-testing-runner --tap libsoup-3.0", 300;
```

Or actually we can use these tests in an openQA agnostic way if that is how we decide.

The display-dependent packages already work correctly for openQA because openQA provides a virtual framebuffer. The wrappers I added are designed exactly for this: they skip in headless environments and run normally when a display is present.

The code is all there. The packages build. The tests pass. The next step is the submit request and the openQA integration. If you maintain any of these packages and want to pick this up, the home project is at:

[https://build.opensuse.org/project/show/home:bzoltan1](https://build.opensuse.org/project/show/home:bzoltan1)


The spec changes are small. The upside is real: catching regressions in packages with combined hundreds of open bug reports before they reach users.

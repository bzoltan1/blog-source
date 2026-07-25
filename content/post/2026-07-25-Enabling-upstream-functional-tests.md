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

Out of curiosity, I also cross-referenced these packages against our Bugzilla and found about 324 matched bug reports. I want to be careful here, I am not claiming that better test coverage would have prevented all those bugs. But it is a concrete data point that there are real problems being reported against these packages. The top three are `libsoup` (100 bugs, 20 open, including two active CVEs from 2025), `pipewire` (97 bugs, 39 open, 40% open rate), and `bolt` (66 bugs).

### The missing piece

Before I could do anything, I hit an immediate wall. Ubuntu runs these tests with a tool called `gnome-desktop-testing-runner`. It is a tiny C binary that depends only on GLib and libsystemd, both of which we already have. It is the standard harness for GNOME installed-tests infrastructure. And it was not packaged in openSUSE at all.

I packaged it. It took few minutes. The spec is 66 lines. It builds fine on both openSUSE Tumbleweed and SLE 15-SP6.

```
osc -A https://api.opensuse.org co home:bzoltan1/gnome-desktop-testing
```

With that in place, the path was open for many of the listed packages.

### Enabling the tests

I focused on packages where the work was straightforward: the tests already exist upstream, can run in a container without hardware or a display, and the spec change is small. The average change across the 21 packages I touched was around 29 lines added, 2 lines removed. That is it. Most packages fit one or more of five patterns:

**Pattern A — flip a meson option**: the most common case. The upstream meson build already supports installed-tests but the option was disabled or missing in the spec. The option name is not standardized — it varies: `-Dinstalled_tests=true`, `-Dinstalled_tests=enabled`, `-Dinstall-tests=true`, `-Denable-installed-tests=true`, `-Dtests=true`. Check `meson_options.txt` upstream before assuming. The rest of the change is adding a `%package tests` block:

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
+%dir %{_libexecdir}/installed-tests
+%{_libexecdir}/installed-tests/%{name}/
+%dir %{_datadir}/installed-tests
+%{_datadir}/installed-tests/%{name}/
```

Note: the `%dir` entries for the parent directories are required, otherwise the build fails with "directories not owned by a package".

**Pattern B — move files already installed**: some packages were already building and installing the test binaries, but putting them in `%files devel` instead of a dedicated subpackage. `graphene`, `json-glib`, and `libxmlb` fell into this category. The change is near-zero net: remove lines from one `%files` section, add a `%package tests` block, done.

**Pattern C — display-dependent tests**: GTK tests that need a display. Rather than removing them, I added a wrapper script for each `.test` file that checks `$DISPLAY` or `$WAYLAND_DISPLAY` and emits `1..0 # SKIP no display available` (exit 0) when neither is set. On a desktop session or inside openQA, which provides a virtual framebuffer, they run normally. `gspell`, `gtksourceview4`, and `gtksourceview5` use this pattern, combined with Pattern A or B.

**Pattern D — fix hardcoded build paths**: meson generates `.test` metadata files at build time and bakes in the build directory path for environment variables like `G_TEST_SRCDIR` and `G_TEST_BUILDDIR`. After installation those paths no longer exist. `libfprint` had this problem, and `libxmlb` needed `G_TEST_SRCDIR` added to its `.test` file. A sed pass in `%install` rewrites the paths to the correct installed locations.

**Pattern E — autotools `check_PROGRAMS`**: `libcupsfilters` and `libppd` use autotools and their test binaries are declared as `check_PROGRAMS` — built only by `make check`, never installed by `make install`. There is an important RPM build order detail here: `%install` executes before `%check`, so the test binaries do not exist yet when `%install` runs. The fix: add `make check || :` at the end of `%build` to pre-build the test binaries, then install them explicitly from the `.libs/` subdirectory. That is where libtool places the real ELF binary — the file at the top level is a libtool shell wrapper that only works in-tree.

The full list of improved packages, now all living in `home:bzoltan1` on `build.opensuse.org`. The pattern column refers to the descriptions above; packages can use more than one:

| Package | Pattern | Tests subpackage | How to run |
|---|---|---|---|
| bolt | A + custom | `bolt-tests` | `gnome-desktop-testing-runner bolt` |
| gcab | A | — (build-time only) | `%meson_test` during build |
| geocode-glib | A | `geocode-glib-tests` | `gnome-desktop-testing-runner geocode-glib-2` |
| glib-networking | A | `glib-networking-tests` | `gnome-desktop-testing-runner glib-networking` |
| graphene | B | `graphene-tests` | `gnome-desktop-testing-runner graphene-1.0` |
| gspell | B + C | `gspell-tests` | `gnome-desktop-testing-runner gspell-1` |
| gtksourceview4 | A + C | `gtksourceview4-tests` | `gnome-desktop-testing-runner gtksourceview-4` |
| gtksourceview5 | A + C | `gtksourceview5-tests` | `gnome-desktop-testing-runner gtksourceview-5` |
| json-glib | B | `json-glib-tests` | `gnome-desktop-testing-runner json-glib-1.0` |
| libei | A + custom | `libei-tests` | Direct binary execution |
| libfprint | A + D | `libfprint-tests` | `gnome-desktop-testing-runner libfprint-2` |
| libjcat | A | `libjcat-tests` | `gnome-desktop-testing-runner libjcat` |
| libsoup (3.x) | A | `libsoup-tests` | `gnome-desktop-testing-runner libsoup-3.0` |
| libsoup2 | A | `libsoup2-tests` | `gnome-desktop-testing-runner libsoup-2.4` |
| libxmlb | B + D | `libxmlb-tests` | `gnome-desktop-testing-runner libxmlb` |
| pipewire | A + custom | `pipewire-tests` | `gnome-desktop-testing-runner -p 1 pipewire-0.3` |
| samtools | custom | `samtools-test` | `perl ./test.pl` |
| xdg-dbus-proxy | A | `xdg-dbus-proxy-tests` | `gnome-desktop-testing-runner xdg-dbus-proxy` |
| asciidoc | custom | `asciidoc-tests` | `/usr/libexec/asciidoc/test-generate-man` |
| libcupsfilters | E | `libcupsfilters-tests` | `testdither`, `testpdf1`, `testcmyk`, `testrgb` |
| libppd | E | `libppd-tests` | `cp -r /usr/share/ppd/testppd ppd && testppd` |

### Does it actually work?

I validated all of them in a plain openSUSE Tumbleweed container on 2026-07-25. Here are a few examples beyond the libsoup one shown below. No special setup, no emulators, no display. Pull the container, install the RPMs from the home project, run the runner. Here is a representative sample:

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

The results across all 21 packages, briefly:

- **15 PASS**: tests run and pass outright (libsoup, libsoup2, json-glib, graphene, libxmlb, libjcat, geocode-glib, glib-networking, bolt, samtools, xdg-dbus-proxy, asciidoc, libcupsfilters, libppd, libei)
- **3 SKIP**: tests skip gracefully without display, run fully with one (gspell, gtksourceview4, gtksourceview5)
- **1 PASS***: pipewire: 22/25 pass; 2 tests skip because they need hardware or are benchmarks, not unit tests
- **1 PASS***: libfprint: 34/60 pass with virtual device driver; 26 need real fingerprint hardware
- **1 build-time only**: gcab has `%check` enabled and the build-time tests pass, but upstream has no installed-tests infrastructure so there is no installable tests subpackage

A few concrete examples from the three new packages:

```
testdither > /tmp/out && file /tmp/out
/tmp/out: Netpbm image data, size = 512 x 512, rawbits, greymap
```

```
cp -r /usr/share/ppd/testppd ppd && testppd
ppdOpenFile("ppd/test.ppd"): PASS
ppdMarkDefaults: PASS
ppdEmitString (defaults): PASS
...
45 PASS / 0 FAIL
```

```
/usr/libexec/asciidoc/test-generate-man
asciidoc: All doctests passed
OK: asciidoc internal doctest passed
OK: asciidoc HTML generation works
```

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

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

- 4 are genuinely well covered on the SUSE side (systemd, mariadb, openvswitch, cockpit; they have both `%check` in the spec and openQA modules)
- 29 have no `%check` at all, no openQA module, nothing
- The rest are somewhere in between

Out of curiosity, I also cross-referenced these packages against our Bugzilla and found about 467 matched bug reports across the 24 packages I eventually improved. I want to be careful here, I am not claiming that better test coverage would have prevented all those bugs. But it is a concrete data point that there are real problems being reported against these packages. The top three are `libsoup` (100 bugs, 20 open, including two active CVEs from 2025), `pipewire` (97 bugs, 39 open, 40% open rate), and `bolt` (66 bugs).

### A quick look at Fedora

I also ran the same comparison against Fedora and found three more packages worth grabbing. `davix` was missed by the original Ubuntu-based analysis entirely — it has no `debian/tests/control`, so it was filtered out. But the CMake build already produces `davix-unit-tests` and `davix-tester` unconditionally; they just landed in the main package instead of a tests subpackage. No build flag changes, just a `%files` split. `evolution-data-server` simply needed `-DENABLE_INSTALLED_TESTS=ON` added to the `%cmake` call, which Fedora has had for years. And `fwupd` had a latent contradiction in the spec: `-Dtests=false` in `%build` next to `%meson_test` in `%check` — with no test targets registered, that `%meson_test` runs nothing. Changing it to `-Dtests=true` enables the full suite, including device emulation data so tests run without real hardware. All three now build in `home:bzoltan1`.

### The missing piece

Before I could do anything, I hit an immediate wall. Ubuntu runs these tests with a tool called `gnome-desktop-testing-runner`. It is a tiny C binary that depends only on GLib and libsystemd, both of which we already have. It is the standard harness for GNOME installed-tests infrastructure. And it was not packaged in openSUSE at all.

I packaged it. It took few minutes. The spec is 66 lines. It builds fine on both openSUSE Tumbleweed and SLE 15-SP6.

```
osc -A https://api.opensuse.org co home:bzoltan1/gnome-desktop-testing
```

With that in place, the path was open for many of the listed packages.

### Enabling the tests

I focused on packages where the work was straightforward: the tests already exist upstream, can run in a container without hardware or a display, and the spec change is small. The average change across the 24 packages I touched was around 29 lines added, 2 lines removed. That is it. Most packages fit one or more of five patterns:

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
| davix | B | `davix-tests` | `/usr/bin/davix-unit-tests` |
| evolution-data-server | A | `evolution-data-server-tests` | `gnome-desktop-testing-runner evolution-data-server` |
| fwupd | A | `fwupd-tests` | `gnome-desktop-testing-runner fwupd` |

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

The results across all 24 packages, briefly:

- **18 PASS**: tests run and pass outright (libsoup, libsoup2, json-glib, graphene, libxmlb, libjcat, geocode-glib, glib-networking, bolt, samtools, xdg-dbus-proxy, asciidoc, libcupsfilters, libppd, libei, davix, evolution-data-server, fwupd)
- **3 SKIP**: tests skip gracefully without display, run fully with one (gspell, gtksourceview4, gtksourceview5)
- **1 PASS***: pipewire: 22/25 pass; 2 tests skip because they need hardware or are benchmarks, not unit tests
- **1 PASS***: libfprint: 34/60 pass with virtual device driver; 26 need real fingerprint hardware
- **1 build-time only**: gcab has `%check` enabled and the build-time tests pass, but upstream has no installed-tests infrastructure so there is no installable tests subpackage

A few concrete examples from the cups, ppd and asciidoc packages:

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

### Why separating tests

Test binaries belong in a dedicated subpackage for the same reason documentation belongs in a `doc` subpackage: so users who do not need them do not pay for them, and so the people who do need them can install them explicitly.

Before these changes, several packages had already crossed that line in different ways. Four of them — `graphene`, `gspell`, `json-glib`, `libxmlb` — had their test binaries buried in `%files devel`. A developer pulling in headers to write code against a library does not want test executables; a QA engineer running post-install validation does not need headers. Two others — `davix`, `libppd` — shipped test binaries directly in `%{_bindir}` alongside production tools, putting executables with no end-user purpose into every installation. And `pipewire` shipped test SPA plugins inside the runtime `spa-plugins` subpackage, where the plugin scanner picks them up on every production system.

A dedicated `*-tests` subpackage makes the choice explicit and opt-in. Production systems install the package. CI workers and openQA runners additionally install the tests subpackage. That is the standard model in both Fedora and Debian, and it is the reason the installed-tests infrastructure exists at all.

### Next steps

The goal of this work is to get these packages into `openSUSE:Factory` via submit requests, and then add openQA modules that call `gnome-desktop-testing-runner <package>` as part of the standard test run on every Factory commit.

An openQA `console` module for any of these is genuinely simple:

```perl
zypper_call "in libsoup-tests gnome-desktop-testing";
assert_script_run "gnome-desktop-testing-runner --tap libsoup-3.0", 300;
```

Or actually we can use these tests in an openQA agnostic way if that is how we decide.

The display-dependent packages already work correctly for openQA because openQA provides a virtual framebuffer. The wrappers I added are designed exactly for this: they skip in headless environments and run normally when a display is present.

The code is all there. The packages build. The tests pass. The next step is shipping these changes to Factory and the openQA integration. If you are interested to see how the packages look, my home project is at:

[https://build.opensuse.org/project/show/home:bzoltan1](https://build.opensuse.org/project/show/home:bzoltan1)


The spec changes are small. The upside is real.

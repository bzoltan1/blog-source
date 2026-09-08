---
title : "Optimizing the sudo test"
subtitle : "From 9 minutes to 3 seconds with broader coverage"
date : 2026-09-08T08:00:00+02:00
tags : ["openSUSE", "SLES", "SUSE", "Quality engineering", "openQA", "sudo"]
type: post
---

The openQA test suite for openSUSE and SLE has a test module called
[`tests/console/sudo.pm`](https://github.com/os-autoinst/os-autoinst-distri-opensuse/blob/master/tests/console/sudo.pm).
It verifies that sudo works: passwords, shells, sudoers rules,
environment isolation. Basic stuff. It runs tens of thousands of
times per year and takes about 9 minutes each time. That adds up.

### Where the time goes

There is no single bottleneck. The test uses `expect` to interact
with password prompts. Every sudo call goes through credential cache
reset, process spawn, password entry, and result verification. It
does this 20 times because the test runs the full suite twice with
slightly different sudoers configurations.

The second iteration tests sudoers rules with and without a `(root)`
target user restriction. But the test body never runs
`sudo -u <nonroot>`, so both iterations exercise identical behavior.
Half the runtime is redundant.

The test used to be even slower. It used `sudo zypper -n in -f yast2`
as the heavy command that needs a password. Each forced yast2
reinstall took over a minute and ran 4 times per test. That was
replaced with `sudo visudo --check --strict` in 2025, which helped
a lot, but the structural overhead remained.

### Looking at what others do

Debian, Fedora, and Arch Linux all run `make check` during the sudo
package build. The [openSUSE sudo spec](https://build.opensuse.org/package/view_file/Base:System/sudo/sudo.spec)
never had a `%check` section.

Debian also has autopkgtests for sudo: shell scripts that create
real users, set real passwords, and test real sudo behavior against
the installed package. Four tests, about 340 lines of shell. They
run in seconds.

The openSUSE spec also had a `make -B` flag (unconditional rebuild)
that was added in 2018 to work around a parallel build race
condition (upstream sudo Bug #842). The upstream fix for that bug
landed in sudo 1.8.24, also in 2018, and a note on the original SR
said the workaround could be reverted after the next version update.
The revert did not happen, so it accumulated for 8 years.

### Three changes

**Enabling upstream tests during build.** A [change to the sudo
package](https://build.opensuse.org/request/show/1376260) drops the
`-B` workaround, adds `--with-devel` to the configure flags to build
the full upstream test suite including the `testsudoers` regression
binary, and adds a `%check` section. The `--with-devel` flag had a
parallel build issue because pre-generated parser files in the
tarball race against yacc regeneration. Removing those files before
the build forces proper dependency ordering. bison and flex were
added as BuildRequires. 1,594 upstream tests now run during every
build.

**Porting functional tests to Python.** Debian's autopkgtests were
ported to Python/pytest and combined with the test coverage from
the existing `sudo.pm`. The [agnostic test](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26609)
covers authentication (correct password, wrong password, non-member
rejection, credential caching), I/O redirection boundaries, shell
modes (sudo -i vs sudo -s), environment variable isolation,
NOPASSWD/PASSWD mixed rules, group-based sudoers access, DNS
resilience, and sudoers.d file parsing.

A shipped-config validator was also added -- something neither the
openQA test nor Debian had. It reads the real `/usr/etc/sudoers`
without replacing it and checks that the syntax is valid, permissions
are correct, env_reset is enabled, secure_path is set, includedir
directives exist, and the targetpw/ALL-ALL rule pair is consistent.

**Packaging as openqa-agnostic.** The test follows the openqa-agnostic
pattern so it runs both inside openQA via the agnosticTestRunner and
standalone on any SUSE system with pytest installed.

### Why look at bugzilla

When developing tests, it is natural to ask how troublesome the
subject is. Without falling into survivorship bias, we want to know
if our tests would have caught any issues that were reported in the
past. One purpose of quality engineering is to take load off the
engineers who deal with customer feedback, bug reports, and
regressions. Good tests save money because they catch problems
before customers do, and they prevent already-fixed bugs from coming
back.

A review of sudo bugs in the SUSE bugzilla helped calibrate what the
tests should focus on. The largest single category of real-world
complaints was targetpw misconfiguration: users locked out because
sudo asks for the wrong password after an update. These are not
sudo code bugs but shipped-configuration problems. A functional
test that replaces the config to test in isolation cannot detect
them. That is why the shipped-config validator was added as a
separate test class -- it reads the real sudoers file and checks
that the declared policy is internally consistent.

CVE test cases are added after the vulnerability is discovered, not
before. The upstream `%check` suite prevents regressions of those
fixes rather than catching zero-days. That is still valuable: it
ensures that a patch is not accidentally dropped during a version
bump or a rebase.

More than a few configuration bugs and at least one NOPASSWD
regression would have been caught by these tests. The real value is
cumulative: once any bug is fixed, the build-time and runtime tests
help keep it fixed.

### What is next

Once the packaging change lands, every sudo build in Factory will
run the upstream unit tests. Once the openQA agnostic test PR is
merged, the existing `sudo.pm` can be replaced with an optimized
test that has broader coverage.

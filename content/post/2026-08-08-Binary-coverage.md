---
title : "Measuring binary function coverage of openSUSE integration test"
subtitle : "Why the trend matters more than the number"
date : 2026-08-08T05:00:00+02:00
tags : ["openSUSE", "Quality engineering", "openQA"]
type: post
---
Let's be clear about what this is and what it isn't.

A typical openSUSE Tumbleweed installation has roughly 2,600 ELF executables in `/usr/bin` and `/usr/sbin` (more or fewer depending on the installation pattern). Our test repository has 2,343 test modules. We measure coverage on 141 of those binaries using 98 of those test modules. That's a fraction of the whole picture and we're not claiming this is the total test coverage of the distribution.

But that's not the point.

The absolute numbers (about 31% coverage on `curl`, 61% on `man`, 13% on `gdb`) are interesting as a baseline, but they're not what makes this valuable. What makes this valuable is the **trend**.

If we measure coverage today and again next month, two things can change:

- **We add tests or improve existing ones** and coverage goes up. We exercised functions that were previously untouched. That's progress.
- **Upstream projects add new functionality** and the binary gains new functions from a new release, but our tests don't exercise them. Coverage goes down. That's a signal worth paying attention to.

Both directions carry information. A drop doesn't mean our tests got worse, it might mean the software grew and our tests didn't keep up. A rise doesn't mean the software is fully tested, it means we're covering more of what's there.

Think of it like blood pressure monitoring. A single reading tells you something, but not much. It's the trend over time that matters. Systematically rising or falling numbers are the signal: they tell you to look closer, not that the diagnosis is already made.

This is a different dimension from our existing test quality metrics. We already track pass/fail rates and softfail rates in openQA. Those tell us whether a test *worked*. Function coverage tells us *how much of the binary the test actually touched*. A test can pass perfectly while exercising only 3% of the binary's code. That's not a failure, but it's useful to know.

This is what we're building toward: a continuous, automated measurement that shows the direction of change. Not a final verdict on quality, but an early warning system that tells us when the gap between what we ship and what we test is growing.

## The technology

Andrea Manzini explored several approaches to measuring binary test coverage in his earlier work ([integration test coverage](https://ilmanzo.github.io/post/measuring-coverage-of-integration-tests/), [gcov-based coverage](https://ilmanzo.github.io/post/measuring-test-coverage-on-binaries/), [pin-based function tracing](https://ilmanzo.github.io/post/pintool-function-tracing/)). The approach that proved most practical is [`funkoverage`](https://github.com/ilmanzo/BinaryCoverage), which uses Linux eBPF uprobes to trace function calls at runtime.

How it works: you have a compiled binary like `/usr/bin/tar`. You install its `-debuginfo` package, which contains the debug symbols, a mapping of memory addresses to function names. funkoverage reads those symbols, attaches a tiny eBPF probe to each function's entry point, then runs the binary normally. When the binary exits, funkoverage reports which functions were entered and which weren't. No recompilation, no source code, no modification to the binary. This is function-level coverage, not branch or line coverage. It tells us a function was entered, not which paths through it were exercised. That's a limitation of the eBPF uprobe approach, but for tracking trends across hundreds of binaries at integration test scale, it's a practical trade-off.

Andrea built a proof of concept (described in his [eBPF coverage post](https://ilmanzo.github.io/post/measuring-test-coverage-with-ebpf/)) that tracked 7 binaries (vmstat, md5sum, gzip, top, tar, unzip, ld-linux) through a single [openQA](https://openqa.opensuse.org) test job, measuring 6,012 functions and finding 701 called, an 11.66% average coverage.

My goal was to try to extend this and see how big the coverage can go. How many of the binaries in a Tumbleweed installation can we actually measure? How many of the 2,343 test modules in our [test repository](https://github.com/os-autoinst/os-autoinst-distri-opensuse/) exercise binaries that funkoverage can instrument?

## Running openQA in a podman container

Instead of using [openqa.opensuse.org](https://openqa.opensuse.org) or any other production openQA instance (which run thousands of jobs daily for the actual distribution testing), I deployed a private openQA instance inside a podman container on my developer laptop. This keeps the experiment isolated and doesn't consume shared infrastructure resources.

I ran a single `podman` container with the openQA web UI, worker, and scheduler, with KVM passthrough for hardware-accelerated virtualization. The test VM gets 2 CPU cores, 2GB RAM, and a 20GB virtual disk. Networking uses QEMU's SLIRP user-mode stack, so no bridge setup and no root privileges for networking. The whole workflow is documented in [howto-reproduce.md](https://github.com/bzoltan1/binary-coverage/blob/main/howto-reproduce.md).

This is deliberately minimal, and it comes with real limitations:

- **SLIRP networking**: the VM gets outbound connectivity through NAT but no inbound. No raw ICMP forwarding, no IPv6 default route, no bridge mode. Tests that need real network topology (ping with capabilities, traceroute with raw sockets, multi-machine setups) fail or softfail.

- **Single VM only**: production openQA can spin up multi-machine test scenarios (client + server, cluster nodes). I have one VM. Any test that needs a second machine is out.

- **Serial terminal only**: production jobs use VNC for graphical needle matching (screenshot comparison). My setup relies on the serial console for all interaction. Tests that use `assert_screen` to verify visual state don't work. This rules out all GUI tests and a few console tests that use VNC for user login.

- **No persistent workers**: when the container restarts, the DNS configuration (resolv.conf) resets. I had to fix DNS to `1.1.1.1` before every job because VPN sessions left stale internal nameservers in the container.

- **ISO snapshot drift**: I used a Tumbleweed DVD ISO from July 29. The debug repositories track the *latest* rolling snapshot. As days pass, the debuginfo packages in the repo no longer match the binaries installed from the older ISO. This caused sporadic installation failures and limits how long a single ISO remains usable. In practice the window is about a week. In production on openqa.opensuse.org, ISOs are refreshed with every new snapshot, so this is only a problem for local development.

These constraints shaped every decision in the project. A binary that works perfectly on production openQA might be untestable in my container because it needs VNC, a second VM, or bridge networking. The 141 binary paths I confirmed are the ones that work within these boundaries, and they'd work on any similarly constrained openQA deployment.

## The recipe

The complete setup runs in a podman container on a developer laptop, no production infrastructure needed. The [howto-reproduce.md](https://github.com/bzoltan1/binary-coverage/blob/main/howto-reproduce.md) walks through the 8 steps: start the container, clone the [test distribution branch](https://github.com/bzoltan1/os-autoinst-distri-opensuse/tree/binary-coverage-141-targets), download a Tumbleweed ISO, and fire the job. The full run takes about 2 to 3 hours.

## Scanning for binaries

The test repository has 2,343 test modules. Each module is a Perl file that calls openQA API functions like `assert_script_run("tar xf archive.tar")` or `script_output("dig example.com")`. The first word of each command is a binary name.

I made a [scanner](https://github.com/bzoltan1/binary-coverage/blob/main/scan_test_binaries.py) that parses every `.pm` file for these patterns and extracts the binary names. Running it against the full repository finds **721 unique binary names** across all test modules.

But not all of those are usable. After filtering out shell builtins (`echo`, `test`, `cd`), interpreters (`perl`, `python3`, `bash`), and system plumbing (`systemctl`, `mount`, `ip`), we're left with roughly 500 actual application binaries referenced by the tests.

From those 500, I systematically evaluated each one:

- Is it an ELF binary or a script? (scripts can't be traced with uprobes)
- Does it have a `-debuginfo` package in the Tumbleweed repositories?
- Is the test module compatible with our serial-terminal-only QEMU setup?
- Can the binary be safely wrapped in a funkoverage shim without breaking functionality?

This evaluation, running across roughly 50 test jobs over several days, narrowed 500 candidates down to **141 confirmed binary paths** across 86 packages, exercised by 98 stock test modules.

## What doesn't work (and why)

Not everything with a command name in `/usr/bin` is measurable:

- **Scripts**: Python, Perl, Bash, Ruby executables have no ELF functions to probe. Examples: `firewall-cmd`, `aa-status`, `iotop`, `semanage`. About 30% of executables in `/usr/bin` are actually scripts.

- **System-critical binaries**: shimming `systemctl`, `mount`, or `ip` would break the system mid-test. These are called thousands of times by the test infrastructure itself.

- **Long-running daemons**: if a daemon like `chronyd` or `sshd` gets restarted through a shim, the service manager loses track of the process. The shim wrapper changes the PID chain.

- **Raw network tools**: `tcpdump` and `tshark` use raw sockets and BPF filters for packet capture. The shim's eBPF probe setup interferes with their own BPF filter attachment. They start but capture zero packets.

- **Hardware-dependent binaries**: `bpftrace`, `nvme`, `tpm_selftest` need hardware (or kernel features) that QEMU's virtual machine doesn't provide.

- **Multi-machine tests**: `ansible`, `salt`, `rsync` over SSH need a second machine or a running SSH server, which my minimal single-VM setup doesn't have.

- **GUI tests**: `weechat`, `aplay`, `firefox` need a graphical display or VNC needle matching, not available in the serial-terminal-only setup.

## The struggle

This was not a victory march. Some examples:

- **Package naming**: the debuginfo package for `gcc` is `gcc16-debuginfo`, not `gcc-debuginfo`. `postgresql` is actually `postgresql18`. Every versioned meta-package needed manual lookup.

- **Serial terminal buffer overflow**: installing 80+ debuginfo packages in one zypper command floods the serial console buffer (~64KB). I had to chunk installs into batches of 20 packages and 15 binary shims.

- **ISO age vs debug repo**: my ISO snapshot was from July 29. The debug repository serves the *latest* snapshot's debuginfo. As Tumbleweed rolls, package versions drift and debuginfo installation fails for older binaries.

- **Module ordering matters**: the `journalctl` test floods the serial buffer with ~64KB of log output, corrupting every module that runs after it. Discovered the hard way, removed permanently.

- **Daemon shim race conditions**: `vsftpd` and `slpd` start through systemd, which calls the shimmed binary. The shim adds ~2 seconds of latency. The test module's connection attempt arrives before the daemon finishes starting. Zero coverage.

- **QEMU limitations**: `openvswitch` and `ovn` tests need BPF `map_create` which the QEMU kernel denies. AppArmor tests all softfail because the guest boots with SELinux, not AppArmor. `traceroute` fails silently under SLIRP networking (no raw ICMP).

- **~50 test jobs**: I ran roughly 50 test jobs (each taking 20 to 90 minutes) to iteratively discover, test, and confirm which modules and binaries work. Many failed for reasons only visible in the serial log after the job finished.

## The coverage report

The most recent full run used 110 modules (9 install/boot, 3 infrastructure, 98 test modules) and 141 shimmed binary paths. The coverage report lists 132 unique binaries because some paths are symlinks to the same underlying binary (for example `restorecon` points to `setfiles`, `run0` points to `systemd-run`). The aggregate report shows:

| | Binaries | Functions | Called | Coverage |
|-|:--------:|:---------:|:------:|:--------:|
| Application binaries | 132 | 74,964 | 12,323 | 16.4% |
| Shared libraries | 260 | 111,314 | 10,772 | 9.7% |
| **Total** | **392** | **186,278** | **23,095** | **12.4%** |

A note on the shared library numbers: the report includes every `.so` that gets loaded by the measured binaries, even if our tests never touch their functionality. Libraries like `libvirt.so` (6,126 functions) or `libovn.so` (7,730 functions) show 0% because they were linked but never called during the test run. This inflates the denominator. Some libraries appear multiple times in the report because different binaries load them in different contexts. That's actually meaningful: `libcrypto.so` called from `openssl` exercises different code paths than `libcrypto.so` called from `gpg`. The more contexts a library is tested in, the better. Still, the binary-only line (16.4%) is the more honest number for what our tests actually exercise. The full [aggregate report](https://github.com/bzoltan1/binary-coverage/blob/main/coverage-report.html) is available for inspection.

For comparison, Andrea's proof of concept measured 7 binaries, 6,012 functions, 701 called, 11.66% coverage in roughly 20 minutes.

I scaled from 7 to 132 binaries, from 6,012 to 74,964 tracked functions (binary-only), from 701 to 12,323 functions called. That's **17x more functions exercised**. The test run got longer, from ~20 minutes to ~2.5 hours, mostly because of the 98 test modules, each installing packages and running through their test scenarios.

The interesting finding: **the coverage percentage barely moved**. Andrea's 7 binaries showed 11.66%. My 132 binaries show 16.4% on binary-only, 12.4% overall. As I discovered more binaries and added them in batches, the average coverage stayed in the 12-17% range. Every new batch brought in new functions at roughly the same ratio of called to total.

This consistency is actually meaningful. It suggests that our test suite exercises about 12-17% of the function surface of the binaries it touches, regardless of which binaries we measure. Some individual binaries score much higher (`irqbalance` at 74%, `cron` at 69%, `kmod` at 64%, `man` at 61%) and some much lower (`redis-server` at 3.2%, `vim` at 14.8%, `gdb` at 13.4%). But the average is stable. It's also possible that this stability reflects the nature of openQA console tests: they tend to be smoke-level checks that verify a binary's main workflow without exploring edge cases. A different test suite with deeper functional tests would likely show different numbers.

The top and bottom of the range tell their own stories. High-coverage binaries tend to be small, focused tools where our test exercises most of the functionality (`arping`: 6 of 7 functions, 85.7%). Low-coverage binaries tend to be large, feature-rich applications where our test only touches one workflow (`vim`: 1,192 of 8,076 functions, we open a file, edit, save, quit, and 85% of vim's code never runs).

Is 12-17% good or bad? Honestly, we don't know yet and that's not really the right question. These are integration tests, not unit tests. They're designed to verify that the software works in a real installed system, not to exercise every code path. An integration test for `tar` creates an archive and extracts it. It doesn't test every compression algorithm, every edge case in sparse file handling, every error path. That's what upstream unit tests are for. Our job is to catch regressions in the installed product, and function coverage tells us how wide that net actually is.

A few modules consistently softfail (ping, sudo, gdb, lsof, perf) due to shim overhead or environment limitations, but they still produce valid coverage data. These are known and tracked, not ignored noise.

That 12-17% is now our baseline. If it drops next month, something changed: either upstream added features we're not testing, or a test module broke. If it rises, we improved coverage. The absolute number matters less than the direction.

## What we have now

As one of my former bosses (at a different Linux distro) used to say: "Ship it."

The method works. I have one data point, which is not yet a trend. A trend requires at least two. The next step is to get this running in production so the second measurement happens automatically, and the third, and the hundredth.

I have a [branch](https://github.com/bzoltan1/os-autoinst-distri-opensuse/tree/binary-coverage-141-targets) with 141 binary targets, 98 test modules, and all the patches needed to run it (the full list of binaries and their packages is in [`tw_confirmed.yaml`](https://github.com/bzoltan1/os-autoinst-distri-opensuse/blob/binary-coverage-141-targets/schedule/coverage/tw_confirmed.yaml)). The [reproduction recipe](https://github.com/bzoltan1/binary-coverage/blob/main/howto-reproduce.md) is there for anyone to follow. The exploration phase (roughly 50 test jobs over several days, discovering which binaries and modules work) was a one-time investment. Going forward the marginal cost of adding a new binary is a line in a YAML file and a quick validation run.

In the upcoming days we will put this larger set of binaries and test modules into production on [openqa.opensuse.org](https://openqa.opensuse.org) and see how it holds up in the real world. During this study I noticed that coverage jobs, like any other openQA test job, are sensitive to infrastructure hiccups: stale DNS, package repository drift, serial console buffer limits, timing races. Running on a developer laptop in a podman container is forgiving. Running on shared infrastructure with real scheduling pressure is the actual test.

Once the coverage reporting is in production, we will set up a dashboard (management does love dashboards) and start tracking how the numbers change over time. We will celebrate when coverage grows. And we will investigate and fix tests when we see it declining.

And naturally we have the ambition to subscribe more binaries and more test modules to this list. It is most likely that in a smarter test environment (not in a minimal QEMU on my laptop) we can cover more binaries and even some SLES specific ones. From a business perspective, SLES is the revenue product, and bringing this coverage measurement to SLES testing is a clear next step.

The 12-17% baseline is our starting line, not our finish line.

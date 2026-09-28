---
title : "Speeding up tests - no fear, no cargo cult"
subtitle : "Two tracks, roughly 54,000 machine-hours a year"
date : 2026-09-28T05:00:00+02:00
tags : ["openSUSE", "SLES", "SUSE", "Quality engineering", "openQA"]
type: post
---

The openQA test suite for openSUSE and SLE has about 2,400 test modules and 1,500 schedule files. Over a 12-month window they run a few million times and burn about 710,000 machine-hours of worker time across two instances, the public [openqa.opensuse.org](https://openqa.opensuse.org) and SUSE's internal openQA. For the last two months I have been trying to shrink that number, cheap wins first.

I kept finding fossils. A comment header in `hostname` describing code deleted in 2019. A network restart timeout that crept from 10 to 120 seconds over two years of bug reports. A `make -B` workaround from 2018 for a bug fixed upstream that same year. Each made sense when it was written. But they add up, and they run on every job.

Two tracks came out of this work. One is infrastructure, the setup modules that run early in nearly every job, where replacing the VGA root console with the serial terminal or removing a dead workaround saves a fixed chunk of every run. The other is the agnostic conversion, rewriting interactive console tests as self-contained pytest or gotestsum suites that run as a single process and hand JUnit XML back to openQA, so a hundred assertions do not mean a hundred console round trips.

| Test module | Before (median) | After (median) | Saved / run | Saved h / yr |
|---|---|---:|---:|---:|
| `system_prepare` | 1m 42s | ~2s | 65s | ~14,500 |
| `hostname` (`set_hostname`) | 1m 27s | ~27s | 60s | ~10,000 |
| `sudo` | 8m 51s | seconds | large | ~10,000 |
| `hostname` | 27s | 1s | 26s | ~4,300 |
| `salt` | 1m 54s | ~80s | ~70s | ~2,360 |
| `first_boot` | 48s | no call | ~10s | ~1,350 |
| `consoletest_setup` | 20s | ~13s | 7s | ~1,470 |
| `zypper_ar` | 52s | 1s | 50s | ~1,430 |
| `system_state` | 34s | 1s | 32s | ~1,270 |
| `zypper_log` | 30s | 2s | 28s | ~695 |
| `prepare_test_data` | 50s | 43s | 7s | ~640 |
| `prepare_system_for_update_tests` | 31s | 1s | 30s | ~530 |
| `sysstat` | 1m 00s | ~35s | ~38s | ~530 |
| `publiccloud` (3 modules) | varies | varies | 0.1 to 10s | ~480 |
| `zypper_up` | 16s | 1s | 15s | ~330 |
| `prepare_test_data` | 52s | 49s | 3s | ~275 |
| `ansible` | 5m 27s | 54s | ~273s | ~3,835 |

Roughly **54,000 machine-hours a year** once everything lands. The before/after columns are medians from the 12-month sample, so a single row hides real spread across architectures and load. `Saved / run` is a separate per-run estimate, not the difference between the two median columns.

## The infrastructure modules

The setup modules everyone runs and nobody reads.

`system_prepare` runs in 638 schedules. It only runs shell commands, no `assert_screen`, no `check_screen`, nothing that needs a screen. Yet it selected the VGA root console. On a QEMU guest that means waiting for the VGA to come up, logging in, setting up the prompt, installing serial markers. Between 55 seconds and two minutes depending on architecture. The serial terminal is already there, costs about a second. One line change and `system_prepare` goes from a median of 1m 42s to about a second.

Same thing in [hostname](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26817), a handful of [console](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26834) and [update](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26836) modules, and [prepare_test_data](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26838). Then there was [set_hostname](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26649), which turned out to be restarting the entire network on every call, per device, with 120 second timeouts and a journal dump, 600,000 times a year. That one deserves its own writeup.

## The agnostic tests

A small wrapper installs pytest or gotestsum, runs the suite on the SUT, and parses the JUnit XML back into openQA.

Why is that faster? Not magic, just plumbing. The classic test talks to the machine one command at a time. Every `assert_script_run` types the command, waits for the exit code marker on the serial device, and only then moves on. A hundred assertions is a hundred round trips. Many tests also redo setup per assertion, a zypper call here, a service restart there.

The agnostic version runs one command. Inside it, pytest collects everything, runs it, writes one result file. Setup happens once. No per-assertion round trip, no Perl-to-shell translation.

[sudo](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26609) went from 9 minutes of `expect` prompts to a few seconds of pytest, with broader coverage (I wrote about that one [earlier]({{< ref "2026-09-08-Optimizing-the-sudo-test.md" >}})). [salt](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26727) dropped the round trip per assertion. The agnostic version is a flat 67 to 90 seconds regardless of architecture, the original ranged from just over a minute on s390x to nearly five minutes on aarch64. [sysstat](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26737) runs the samplers in parallel now and uses a 1 second interval instead of 5. [ansible](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26739) follows the same shape.

## What broke

I broke things. It is worth being specific.

When I stopped restarting the network in `hostname` by default, the multi-machine tests that genuinely need it broke. A [follow-up](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26702) restored the restart where it matters.

When `hostname` moved to the serial terminal, it exposed a 90 second wait that was already sitting in `consoletest::post_run_hook` on every aarch64 serial module. The old VGA path hid it. [#26818](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26818) removed it, and that by itself saved about 29 minutes per aarch64 job.

The agnostic runner broke production on SLE 15 SP6 LTSS. It asked zypper for `python3-pytest`, which needs `python3-wcwidth`, and on LTSS that package sits in the Public Cloud Module which is not enabled. Two production jobs failed before [#26726](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26726) added a pip3 fallback. That was my worst one. The old `sudo.pm` had zero Python dependencies and would have passed fine.

The sudo test also needed to handle a different PAM error message in FIPS environments. I only found out when the SP6 FIPS verification run failed.

Every one of these was found within hours and fixed the same day. The point is not that nothing broke. The point is that we caught it fast, and the net result was still a large saving. If you never break anything, you are probably not touching anything that matters. Fear of touching old code is how a test suite turns into a cargo cult. We need to be brave enough to question old decisions and clean up what no longer works.

## The font check discussion

This one was not a bug, it was a discussion. When I removed `check_console_font` from `consoletest_setup`, a reviewer said we cannot drop a test just because it has not failed recently. Fair concern, but wrong premise. I went back through the git history. The original commit ([dd6f8e1be4](https://github.com/os-autoinst/os-autoinst-distri-opensuse/commit/dd6f8e1be4), 2015) was called "workaround for sle12 bug". It unconditionally ran `systemd-vconsole-setup` to fix a broken console font, no check, no assertion. The needle-based detection was added later to make the workaround smarter. When the font is broken, the code silently fixes it and moves on. No `record_info`, no softfail, no bug reference. That is workaround behavior, not a test.

The underlying bug was in systemd 210/228 and was fixed upstream years ago. I offered a compromise, move the check into a dedicated module on a canary schedule, run it once per build instead of 756,000 times a year. The change landed with other approvals. The deeper lesson here is to keep infrastructure setup and product testing separate. Mixing them is how a workaround ends up running 756,000 times a year for a decade.

## What the saving is worth

54,000 fewer worker-hours is about 8% of the suite's total machine time. Six workers running flat out for a year. The same fleet can run 8% more jobs per build, or do the same work with less hardware.

A colleague asked what 54,000 hours means in electricity. Honestly, I cannot give a straight number. If these were dedicated machines, 54,000 hours at 200 W would be about 11 MWh, roughly the electricity of three European homes. But most of this is QEMU time on shared workers, and a saved VM hour does not always map to saved wall power. The capacity number is the real one.

Two publiccloud optimizations are different though. Cloud tests spin up real instances and the meter runs until they are destroyed. On the small instance type that is about $0.067/hour; the SAP and GPU suites cost a lot more. There, saved hours come straight off the cloud bill.

## The legacy

`sudo` had a `make -B` workaround from 2018 for a bug fixed upstream that same year. The network restart carried debug journal dumps from a 2024 investigation and a disconnect timeout that had been bumped from 10 to 120 seconds over two years of bug reports. `hostname` had a comment header describing code deleted in 2019. None of this is anybody's fault. When a change can break thousands of jobs, the safe move is to leave it alone and add one more workaround. I understand that. But it accumulates.

When you touch a test, read it. Ask why each line is there. If the bug it worked around was fixed years ago, removing it is cleanup, not risk.

## What's next

Of the roughly 1,550 test modules that run in production, **242 are straightforward to convert** to agnostic (text mode, single machine, no needles, no reboots). Together they account for about **84,940 machine-hours a year**. That is the next chunk.

After that, the smaller stuff adds up. The cloud audit found per-reboot and per-call waste in the publiccloud helpers. There is a long tail of setup modules still doing slow things for no good reason. The recipe does not change. Find a slow module, understand why it is slow, fix it, verify it, move on.

---
title : "Binary function coverage part 2: scaling up, fixing daemons, and asking the kernel"
subtitle : "From 7 binaries to 172, and what I learned along the way"
date : 2026-08-30T05:00:00+02:00
tags : ["openSUSE", "Quality engineering", "openQA"]
type: post
---
In the [first post]({{< ref "2026-08-08-Binary-coverage.md" >}}) I described the setup: funkoverage eBPF tracing, a podman container running openQA, and the first coverage report with 141 binary targets. That was the starting line. This is what happened next.

## Daemon shimming

The biggest limitation in the first round was daemons. Services like sshd, cups, postgresql, and rpcbind couldn't be shimmed because funkoverage's wrapper broke systemd's service lifecycle. Two specific problems: the shim used SIGKILL instead of forwarding SIGTERM to the child process, so daemons couldn't clean up sockets on restart. And the shim didn't relay sd_notify, so Type=notify services timed out on start.

Andrea fixed both in [BinaryCoverage PR #148](https://github.com/ilmanzo/BinaryCoverage/pull/148), released as coverage-tools 0.8.2. Signal forwarding and sd_notify relay. After that fix, sshd, tcpdump, apache, and snapper all started producing coverage data.

Not everything was fixed in 0.8.2. cups, postgresql, and rpcbind still failed. Andrea followed up with coverage-tools 0.8.3, which fixed cups and rpcbind. PostgreSQL needs more startup time once instrumented but works with extended timeouts. The remaining daemon shimming issues are tracked on [BinaryCoverage#143](https://github.com/ilmanzo/BinaryCoverage/issues/143).

## Upstream contributions

I sent 14 pull requests to the [test distribution repository](https://github.com/os-autoinst/os-autoinst-distri-opensuse). All merged. Most were small: adding `test_flags` with `fatal => 0` and `no_rollback => 1` so modules don't abort the coverage job on softfails. The valgrind module needed SDK cleanup moved from the test body to `post_run_hook` and `post_fail_hook`. The nginx module needed a `stop_service()` in `post_run_hook` to release port 80 before apache runs.

These are the kind of changes that don't alter what a test does, but make it play nicely with coverage measurement. No custom test modules, no changes to test logic. Just plumbing.

## Running on openqa.opensuse.org

The podman container on my laptop was good for development but not for continuous measurement. I deployed the coverage schedule on [openqa.opensuse.org](https://openqa.opensuse.org) using a fork branch with `CASEDIR` pointing to the test distribution. Same schedule, same modules, just running on real infrastructure with real workers.

The jobs work. They also get cancelled when the o3 scheduler decides to obsolete unassigned jobs for new product builds. That's a known cost of using `_GROUP_ID=0`. Resubmit and move on.

The mirror flakiness was more annoying. The debug repository serves the latest Tumbleweed snapshot's debuginfo. When the ISO is a few days old, the repo metadata references packages that no longer exist on the mirrors. zypper returns error 106 or 8, and the whole setup phase fails. The fix was retry logic: if a zypper install chunk fails with a network error, refresh the repo metadata and retry once. That made the jobs survive the inevitable mirror hiccups.

## Asking the kernel what binaries we're missing

After manually adding binaries for weeks, I ran out of obvious candidates. The question became: what binaries does the test schedule execute that are not being measured?

The Linux kernel knows. Every `execve` syscall goes through the kernel. I just needed to listen.

First attempt: the audit subsystem. `auditctl -a always,exit -F arch=b64 -S execve` logs every binary execution. Clean, structured output. But the audit system adds per-syscall overhead. After 100 test modules and tens of thousands of audit events, the serial console became unresponsive. The sshd test hung at port forwarding because the system was too slow to establish SSH tunnels. I tried exclusion filters (skip bash, grep, sed, cat), increased the backlog buffer to 65536, set backlog_wait_time to 0. Still too slow.

Second attempt: ftrace. The kernel's `sched_process_exec` tracepoint writes to a ring buffer with near-zero overhead. No per-syscall evaluation, no userspace daemon, no log file I/O. 16MB per CPU allocated, tracepoint enabled, the full 111-module schedule ran, buffer read at the end. Every binary that was executed during the run appeared in the trace with its full path and call count.

The result: 516 unique binaries executed, 416 of them ELF. After filtering out coreutils and shell noise, 33 new binary targets emerged across 11 packages that the tests were already exercising but not measuring. Plus 22 additional binaries from packages already tracked.

Not all of them survived verification. Shimming `/usr/libexec/ssh/sshd-session` kills the serial terminal because it handles the SSH connection that openQA uses for console access. Shimming `my_print_defaults` prevents mariadbd from starting because it calls that binary during service initialization. One test module (`postfix_tls`) turned out to be orphaned -- not in any upstream schedule, referencing test data files that don't exist.

But most of them worked. After verification and adding the survivors to the production schedule ([o3 job #6190943](https://openqa.opensuse.org/tests/6190943)):

| Metric | Before | After | Delta |
|--------|-------:|------:|------:|
| Total targets | 438 | 489 | +51 (+12%) |
| Binaries tracked | 139 | 172 | +33 (+24%) |
| Total functions in scope | 186,290 | 208,163 | +21,873 (+12%) |
| Functions called | 24,580 | 26,826 | +2,246 (+9%) |
| Non-zero binaries | 96 | 111 | +15 (+16%) |
| Binary functions in scope | 78,297 | 94,500 | +16,203 (+21%) |
| Binary functions called | 13,550 | 15,616 | +2,066 (+15%) |

The coverage percentage dipped slightly (13.2% to 12.9%) because the denominator grew faster than the numerator -- 21,873 new functions added to the scope. But the absolute numbers are what matter: 2,246 more functions exercised, 33 more binaries measured, 15 more binaries with non-zero coverage. The full [aggregate coverage report](/coverage-report-2026-08-30.html) is available for inspection.

## Shim limitations

The funkoverage shim works by moving the real binary to `/var/coverage/bin/` and placing a wrapper in its place. This is elegant but has consequences:

- **Helper binary path breakage**: programs that locate helpers relative to their own path fail. tshark hardcodes dumpcap's location relative to itself. After the shim moves tshark, it can't find dumpcap. Dead end.

- **Per-connection daemons**: sshd forks `sshd-session` for each connection. Shimming the session handler means every SSH connection (including the serial terminal) goes through the wrapper. When sshd restarts during a test, the wrapper kills the session handling the serial console. The console dies.

- **Startup dependencies**: mariadbd calls `my_print_defaults` to read config before starting. Shimming `my_print_defaults` adds latency and changes the process tree. The service fails to start.

- **SELinux enforcement**: coverage_setup sets SELinux to permissive mode. Without that, SSH port forwarding is blocked by SELinux policy. I discovered this when the execve trace schedule (which doesn't run coverage_setup) had sshd failing on port forwarding. The fix was one line: `setenforce 0`.

These are fundamental limitations of the binary shimming approach, not bugs. They define which binaries can and cannot be measured.

## What's next

The current state: 172 binaries tracked across 101 packages, exercised by 111 test modules. The schedule produces roughly 500 coverage data points per run. The numbers are stable across runs on the same snapshot.

The immediate next step is running this continuously. Not just when I remember to submit a job, but as part of a regular schedule. The interesting question is how the numbers behave across Tumbleweed snapshots. When upstream projects release new versions with new functions, coverage should drop slightly. When tests are added or improved, it should rise. Both signals are valuable.

After that, a dashboard. The aggregate HTML report is fine for manual inspection but not for trend tracking. A time series is needed: date, snapshot, binary, functions total, functions called. Something to plot and set alerts on. Management does love dashboards, and in this case one would actually be useful.

The 12-17% baseline from the first post is now 12.9% with a wider net. The direction matters more than the number. I'll keep watching.

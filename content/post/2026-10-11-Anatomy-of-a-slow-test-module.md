---
title : "Anatomy of a slow test module"
subtitle : "A support server case study, from 7m 37s to 5m 10s"
date : 2026-10-11T05:00:00+02:00
tags : ["openSUSE", "SLES", "SUSE", "Quality engineering", "openQA"]
type: post
---

Last week [Antonios Pappas](https://github.com/a-kpappas) told me that if I am already chasing slow tests, I should look at [`tests/support_server/setup.pm`](https://github.com/os-autoinst/os-autoinst-distri-opensuse/blob/master/tests/support_server/setup.pm). It is known for wasting time. He also gave me the rule of thumb that made me take it seriously: every wasted second in the support server costs about three seconds, because the parallel nodes wait for the master to finish its setup.

That is the difference between a slow test and an expensive test. A slow single-machine test wastes one job. A slow support server wastes the whole cluster, and it does it before the real test even starts.

The SLE 15 SP7 SAP HANA scenario was running this module in 7m 37s. The cluster is the support server plus two nodes, and both nodes block on a mutex that the support server creates at the end of setup. So every second spent here is paid three times.

The module also has a reputation. When I posted the first commit, we had a short discussion in the team channel. A teammate caught a real mistake in my verification runs. When you clone a job you have to change the build and the group, otherwise you effectively replace the regular production jobs that use the canonical test code. That was a good catch, and I fixed my cloning command.

Another teammate warned me that the module is a beast. The last person who worked on it, earlier this year, to make it usable on 15-SP7, kept coming back to the channel to curse it. I understand why. When a change can break thousands of multimachine jobs, the safe move is to leave the module alone and add one more workaround. They also confirmed my first disappointing result: batching the package installs saved no time at all, because the packages ship in the image already, but it is still a code quality improvement and worth keeping.

That discussion is the reason the method mattered more than the ideas. I went in expecting to be wrong more than once.

## The method

I did not push one big change. I did ten-plus small rounds. Each round was one idea, one commit, one real verification run, one measurement, and only then the next idea. When a round did not save time, or broke something, I dropped it and moved on.

I started with a code review and a list of ten improvement opportunities. I ranked them by assumed saving and by risk. The low-risk obvious ones came first. The risky ones, the network setup and the iSCSI server, came later, once I trusted the method.

The order saved me twice. More about that below.

## The patterns

The waste was not exotic. It was the same handful of patterns, repeated. I expect many long setup modules in the suite have some of them.

### One command, one round trip

Every `assert_script_run` is a round trip. Type the command, wait for the exit marker, continue. The module built the iSCSI server with eleven separate `targetcli` calls, plus four `parted` calls, plus a package check per service role, plus a `firewall-cmd` and a reload per service. Dozens of round trips for work that is mostly waiting.

The package check is a good example. It was called once per role:

```perl
sub setup_dns_server { chk_req_pkgs('bind bind-utils'); ... }
sub setup_ntp_server { chk_req_pkgs('chrony'); ... }
```

Now there is one list and one call for all roles:

```perl
my %pkgs;
$pkgs{chrony} = 1 if exists $server_roles{ntp};
$pkgs{bind} = $pkgs{'bind-utils'} = 1 if exists $server_roles{dns};
...
chk_req_pkgs(keys %pkgs) if %pkgs;
```

The firewall had the same shape. Every service added its rule and reloaded the firewall. Now the rules are collected and applied with one reload at the end:

```perl
sub firewall_cmd_add { push @firewall_rules, @_ if is_firewall_active() }

sub firewall_cmd_apply {
    return unless @firewall_rules;
    assert_script_run('firewall-cmd ' . join(' ', @firewall_rules) . ' --permanent');
    assert_script_run('firewall-cmd --reload');
    @firewall_rules = ();
}
```

### The serial console types one character at a time

This one surprised me. The console does not paste, it types, and it types at roughly thirty characters per second.

The iSCSI setup built one long shell command that chained all the `targetcli` calls with `&&`. It was 822 characters long. That command took about 30 seconds to type and 5 seconds to run. The typing was the cost.

The fix is to upload the script and run it with a short command:

```perl
write_sut_file('/tmp/iscsi_lio_setup.sh', "set -ex\n" . join("\n", @tcli_cmds) . "\n");
assert_script_run('bash /tmp/iscsi_lio_setup.sh', timeout => 120);
```

`write_sut_file` uploads the content through the worker's HTTP server, so the script never goes over the serial console at all. Only the short `bash /tmp/x.sh` is typed. If a command is long, upload it.

### A restart is not the same as a reload

`apache2` was stopped and then started. Two round trips and two service transitions for something `systemctl restart` does in one. In `setup_nfs_server` the code started `rpcbind` and `nfs-server` and then restarted them again a few lines later. In the DNS setup, `dhcpd` was restarted right after the DHCP setup had already started it.

None of it is wrong. It is just expensive, and it runs on every job.

### Probe once, use many times

`is_networkmanager()` is a system probe. It was called four times inside one function, once per helper. Each call is a round trip.

```perl
my $is_nm = is_networkmanager();
configure_static_ip(ip => $ip, is_nm => $is_nm);
configure_default_gateway(is_nm => $is_nm);
configure_static_dns(get_host_resolv_conf(), is_nm => $is_nm);
restart_networking(is_nm => $is_nm);
```

Same story for `check_os_release()`. The module asked whether the system is version 12, 12.3 or 15 up to nine times per run. I added a small local cache:

```perl
my %os_release_cache;
sub is_os_release {
    my ($version) = @_;
    return $os_release_cache{$version} //= check_os_release($version, 'VERSION_ID');
}
```

A reviewer caught an important detail here. I first put the cache into `lib/version_utils.pm`, globally, so every caller would benefit. The reviewer pointed out that `/etc/os-release` is not always constant during a run. There is a SLES to SLES for SAP migration scenario where it changes. A global cache would return a stale answer. The cache is local to the support server now, where the release is known to be constant during setup.

### Diagnostics are work too

Three `record_info` calls at the end of the network setup each fetched one command output in its own round trip: `ip route`, `ip addr`, `iptables -v -L`. They are diagnostics, not assertions, but they still cost three round trips. They run in one script now.

### Read the image, not only the code

My first idea was to batch the package installs. It looked like an easy 30 to 60 seconds of saving. It saved nothing. The support server images already ship all the needed packages, so the check hit the fast path and never called zypper. The change stayed, because it is cleaner code and it helps on fresh images, but the time saving was zero.

The machine is part of the system. Before optimizing a call, check what it does on the real image.

## The failure

The biggest single idea was to replace the eleven `targetcli` calls with one, because that is the call that takes about 30 seconds to type.

I built the rtslib saveconfig JSON in Perl and restored it with a single `targetcli restoreconfig` call. The verification runs passed. The command returned exit code 0. It looked like the cleanest change of all.

It created nothing.

```text
o- iscsi ....................................................... [0 Targets]
```

The reason is that rtslib's `restore()`, which `targetcli restoreconfig` drives, is not fail-fast. Every object that fails to load goes through an error function, gets appended to a list of warnings, and is skipped. `targetcli` still exits 0. I got a green command and an empty target table.

The real damage was further down. The setup module only recorded the target list in a `record_info`, it never asserted it. So setup passed with zero targets, and the multimachine test failed later, in `ha/barrier_init`, with an empty IQN and a confusing error about a missing sysfs path.

I reverted the change and kept the working `&&`-chain. But I also added a guard, so a silent empty target cannot pass setup again:

```perl
my $target_path = "/sys/kernel/config/target/iscsi/$iscsi_iqn:$iscsi_identifier";
assert_script_run("test -d $target_path", fail_message => 'iSCSI LIO target was not created');
assert_script_run("test \$(ls -1 $target_path/tpgt_1/lun | grep -c '^lun_') -eq $num_luns",
    fail_message => "iSCSI LIO target does not export the expected $num_luns LUN(s)");
```

Two lessons. Verify the effect, not the exit code. And when a step can pass while doing nothing, add the assertion that makes it fail loudly.

## What it saves

The SAP HANA scale-up scenario is a single test name, but it runs a lot. In a six week window on our internal instance it ran 1,172 times across five product versions, about 194 runs a week, so roughly 10,000 runs a year.

The setup module went from 7m 37s to 5m 13s on SLE 15 SP7, from 7m 38s to 5m 11s on SP6, and from 7m 34s to 5m 09s on 12 SP5. Call it 2m 25s per run.

| What | Per run | Per year |
|---|---:|---:|
| support server setup | 2m 25s | about 400 machine-hours |
| the two nodes that wait for it | up to 2m 25s each | up to about 800 more |
| whole cluster | | up to about 1,200 machine-hours |

The 400 hours for the support server is measured. The node part is an upper bound. The nodes do their own boot while the support server sets up, so they only save the full 2m 25s when the setup is still on the critical path at the moment they are released.

So this one scenario, on one instance, gives back about 400 machine-hours a year on its own, and up to 1,200 once the two waiting nodes are counted. Every result that depends on it also arrives about 2m 25s earlier, which matters more to the engineers. The other scenarios that use the module save less, because they use fewer roles, but they add up too. The full PR is [os-autoinst/os-autoinst-distri-opensuse#26987](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/26987).

For the wider picture of this effort, I wrote about it [here]({{< ref "2026-09-28-Speeding-up-tests.md" >}}).

## What I would do differently

I did the rounds sequentially, in one pull request. That is good for safety. Each round is verified on top of the previous one, and I never had to untangle two changes that conflict.

The price is wall clock. Every round needs a verification run, and a multimachine verification takes about an hour. Ten rounds is ten hours, mostly waiting. I could have opened the safe, independent ideas as separate PRs, verified them in parallel, and merged them one by one. That would have been faster for me.

For this module I still think the sequential mode was right, because the risk was high and all the changes touch the same file. For a lower-risk module, parallel PRs would win.

## What's next

The PR is complete and waiting for human review. The obvious next candidates are the other setup-heavy modules that run early in many jobs, and the multimachine support servers that still do work in the wrong order.

Two concrete ones are already on my list. The first is to merge the two network restarts in this module, which I tried and had to drop because it broke the SLE 12 SP5 path. It needs a way to bring up the fixed network before the test network scripts run. The second is to bake the SLE 12 SP3 repositories into the support server image. That would remove the repo block that costs about 63 seconds on every 12 SP5 run, and it is an image change, not a code change.

The recipe does not change. Find a slow module, understand why it is slow on the real image, fix one thing, verify it, measure it, and only then move on. Keep the guard. And remember the rule of thumb from the start: in a multimachine job your wasted second is somebody else's three.

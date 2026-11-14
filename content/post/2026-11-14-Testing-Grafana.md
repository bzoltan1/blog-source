---
title : "Testing Grafana, not only using it"
subtitle : "From a coverage dashboard to a Grafana test suite"
date : 2026-11-14T05:00:00+02:00
tags : ["openSUSE", "SLES", "SUSE", "Quality engineering", "openQA", "Grafana"]
type: post
---

I was working on the functional binary coverage dashboard for our new coverage job group. The group is here: [openqa.opensuse.org/group_overview/145](https://openqa.opensuse.org/group_overview/145). I wrote about the coverage measurement [here]({{< ref "2026-08-08-Binary-coverage.md" >}}) and [here]({{< ref "2026-08-30-Binary-coverage-part2.md" >}}). We use Grafana to show the metrics of our QA factory, so it was natural to build this dashboard in Grafana.

I also use Grafana at home. I have a small setup that watches the temperature of my hot water boiler. I wrote about that [here]({{< ref "2025-11-30-I-want-to-have-a-hot-shower.md" >}}).

Grafana is one of the most popular dashboard tools in IT. Infrastructure monitoring, DevOps, SRE, cloud native systems, application observability. It is open source, easy to install, and it has a big and active community. At this point it is almost everywhere.

After I finished the coverage dashboard I asked a simple question: how good is our own test coverage for Grafana?

The answer was not what I expected.

## The helmet on the shelf

We have one Grafana test in os-autoinst-distri-opensuse: [`tests/sles4sap/grafana_dashboards.pm`](https://github.com/os-autoinst/os-autoinst-distri-opensuse/blob/master/tests/sles4sap/grafana_dashboards.pm). It installs Grafana and the SAP/HA dashboard packages on a SLES4SAP GNOME image, starts the service, checks the log for failed dashboard imports, then opens Firefox, logs in, and checks that the SUSE dashboard renders.

It is a smoke test. It is also not scheduled in any job group. In the last 12 months the module had zero production executions.

It feels like having a cool bicycle helmet that I keep on the shelf instead of putting it on when I go out for a ride. I am sure there is a reason for it, but let's move on.

## What Bugzilla says

As always, I started with Bugzilla. These are live numbers from 2026-11-14, filtered to the SUSE Linux Enterprise products and SUSE Security Incidents, from SLES 15 GA in July 2018. I looked at Grafana bugs only.

| What | Number |
|---|---:|
| Unique bugs | 235 |
| Security incidents | 230 |
| Unique CVEs | 243 |
| SLES/SAP/HPC functional bugs | 5 |
| Fixed | 101 |
| Bugs with a maintenance incident | 76 |
| Unique maintenance incidents | 41 |
| INVALID | 29 |
| NEW | 47 |

Of the 235 bugs, 101 are fixed, 100 are still open, 47 of them are NEW, and 29 are INVALID. 76 bugs reference a maintenance incident, which is the strongest sign that SUSE engineering accepted the issue and shipped a fix.

A large part of the 243 CVEs are not in Grafana's own code. They are in Go libraries bundled into Grafana: `golang.org/x/net`, gRPC, go-jose, mongo-driver, OpenTelemetry, and others. These are real security fixes, but they are not observable through the HTTP API.

The five functional bugs are the interesting ones for a test writer. For example: the pie chart panel did not work ([#1166790](https://bugzilla.suse.com/show_bug.cgi?id=1166790)), the SAP NetWeaver dashboard missed a variable ([#1177229](https://bugzilla.suse.com/show_bug.cgi?id=1177229)), the SAP HA dashboard packages were missing on 15-SP3 ([#1183690](https://bugzilla.suse.com/show_bug.cgi?id=1183690)), a HANA exporter dashboard had an empty panel ([#1187243](https://bugzilla.suse.com/show_bug.cgi?id=1187243)), and Grafana was missing from SLES HPC SP4 ([#1202182](https://bugzilla.suse.com/show_bug.cgi?id=1202182)).

So the answer was clear. Grafana is important for SLES users. They care about the quality and the support of Grafana. We should test it.

## What Fedora does

I took the lazy approach first. I checked if another Linux distribution has open source Grafana tests that we can use or at least learn from.

Fedora has a good set. It lives in a separate tmt/beakerlib repository: [gitlab.com/redhat/centos-stream/tests/grafana](https://gitlab.com/redhat/centos-stream/tests/grafana). I am grateful that they publish these tests. The tests are imported into Fedora dist-git, and they gate Bodhi updates. The RPM `%check` is not the coverage: the frontend Jest and the backend Go tests are commented out there.

Fedora tests these things:

| Fedora test | What it checks |
|---|---|
| general | service start, `/login` 200, basic auth `/api/org` 200, admin password reset |
| cipher-check | HTTPS with generated certificates, `nmap ssl-enum-ciphers`, no weak ciphers |
| grafana-home | the `grafana` user home and `DATA_DIR` |
| licence-check | SPDX license validation |
| loading-unsigned-plugins | the `allow_loading_unsigned_plugins` PCP list |
| selinux-unconfined | no Grafana-owned process runs unconfined |
| systemd-sysusers | the systemd-sysusers config exists |
| verify-systemd-units | `systemd-analyze verify` on the shipped unit |
| coredumps | baseline before the run, report and reset after |

This is useful. Fedora is strong on packaging and distro integration. The functional part is only a smoke test: service up, login page, basic auth. There is no datasource, dashboard, provisioning, RBAC, or CVE test.

## What we test now

I built a new test that covers the package from the outside, over the HTTP API. It has three groups.

| Group | Checks |
|---|---|
| Functional/API | `/api/health`, admin login, bundled testdata plugin, datasource query, dashboard create/read/search, file provisioning, restart persistence, Viewer RBAC |
| CVE regressions | plugin asset path traversal, anonymous and Viewer snapshot creation, SQL expressions `LOAD_FILE`, oversized plugin resource body |
| System/packaging | systemd unit verification, TLS cipher allowlist, grafana user/data directory, license metadata, conditional SELinux confinement, coredump detection |

The CVE cases are the ones that can be checked from the outside. Path traversal (CVE-2021-43798/43813/43815), anonymous and Viewer snapshot creation (CVE-2019-15043, CVE-2026-33379), SQL expressions file read (CVE-2026-33380), and the plugin resource body limit (CVE-2026-28383).

Some CVEs need more than a single virtual machine. Grafana Live and the auth proxy IPv6 allow list are still on the list. They need a WebSocket client or a more complex network setup.

The system checks are the ones I borrowed from Fedora: `systemd-analyze verify`, a TLS cipher scan, the user and data directory, the license, SELinux if it is enabled, and coredumps.

Fedora is still better on the Fedora-specific packaging checks: full SPDX validation, the systemd-sysusers mechanism, and the unsigned PCP plugin list. We have the portable ones now. On the product side our test is broader. Fedora's functional test stops at service, login, and basic auth. We test datasource, dashboard, provisioning, RBAC, and the CVEs. The two sets are complementary. We are not better everywhere, and that is fine.

## The implementation

I could write this as a normal Perl test module. I did not. I used the openQA agnostic model.

There are two reasons. The first is performance. A classic console test often talks to the SUT one command at a time. In the simplest case every assertion is a round trip over the serial console. The agnostic test runs one pytest process and writes one JUnit XML file. The whole suite took about 21 seconds in my local container.

The second reason is portability. I want to run this test at home too, and at home I do not have openQA. The agnostic test is just Python and pytest. I can run it on any SUSE system.

The artifact is in `data/console/openqa_agnostic/python/testGrafana/`. There is a small wrapper in `tests/console/oqa_agnostic/grafana_agnostic.pm`. The wrapper installs `grafana`, `curl`, `nmap`, and `openssl`, then calls `agnosticTestRunner`.

The test restores what it changes. It restores the provisioning directory, removes the temporary systemd drop-ins, and stops the service if it started it. When a prerequisite is missing, the test skips with a visible reason. For example, SELinux is skipped on Tumbleweed because it is not enabled there.

The test uses the default `admin/admin` login. It does not reset the admin password. Grafana locks the account after a few failed logins, and the lockout is stored in the `login_attempt` table. It survives a service restart. I learned this the hard way during development.

The PR is here: [os-autoinst/os-autoinst-distri-opensuse#27383](https://github.com/os-autoinst/os-autoinst-distri-opensuse/pull/27383). At the time I am writing this post the PR is still open. It is not reviewed and not merged yet.

## Does it work?

I validated the full suite on a systemd-enabled Tumbleweed container with the real `grafana-12.4.11-3.1.x86_64` RPM. 21 tests passed and 1 was skipped. The skip was SELinux, because it is not enabled in the container. The run took 21 seconds.

On [openqa.opensuse.org](https://openqa.opensuse.org) the Tumbleweed run passed the same 21 tests and skipped SELinux in 47 seconds. The 21 tests include the systemd unit, the TLS ciphers, the user and data directory, the license, and the coredumps. The job is here: [openqa.opensuse.org/tests/6342437](https://openqa.opensuse.org/tests/6342437)

The full artifact also passed on SLE 15-SP7. The module took 68 seconds on SLE 15-SP7 x86_64, 71 seconds on aarch64 and 89 seconds on s390x. It also passed on SLE 16.0 (59-62 seconds). The SLE 16.1 verification was blocked before the new module by a PackageHub registration failure. That is an SCC infrastructure problem, not the test.

The existing SLES4SAP test also ran in a verification job, but it fails on an old needle: `select_grafana_left_sidebar`. It gets through install, service start, and login, and then the GUI screenshot does not match. That is a test maintenance problem for QE-SAP, not a product bug.

## Testing the test

A test that only passes on a fixed version does not prove much. I wanted to see it fail on a version that still has the bug.

The version gates skip checks on versions that predate the fix. That is correct for production, but for validation I added a switch: `GRAFANA_TEST_FORCE_CVE=1` bypasses the gates. I also added a small `validate_cve.sh` script that runs the vulnerable/fixed matrix in localhost-only containers.

I used the path traversal case (CVE-2021-43798):

| Grafana | Expected | Result |
|---|---|---|
| 8.2.6 | test fails | failed with `path traversal returned /etc/passwd ... (HTTP 200)` |
| 8.2.7 | test passes | passed, the request returned 404 |
| 8.3.0 | test fails | failed with the same `/etc/passwd` leak |
| 8.3.1 | test passes | passed, the request returned 404 |

Grafana 8.2.7 is the fixed 8.2.x backport and 8.2.6 is the last vulnerable 8.2.x. Grafana 8.3.1 is the upstream fix and 8.3.0 is the last vulnerable 8.3.x. The requests on 8.2.6 and 8.3.0 returned the container's `/etc/passwd`, and the test caught them. On 8.2.7 and 8.3.1 the same request returned `404 {"message":"Plugin file not found"}` and the test passed.

I also tried the Viewer snapshot check on Grafana 12.4.0. It passed, so that version does not allow the Viewer snapshot creation in the way the CVE describes, or the test's viewer scenario does not hit the vulnerable path. That is useful too: not every old version is a negative test.

The validation is container-only: bind to 127.0.0.1, use `--rm`, and the only file exposed is the throwaway container's own `/etc/passwd`.

## What is next

The PR needs review and merge. After that the test will run in the Tumbleweed textmode, SLE PackageHub and SLE QAM maintenance (`mau-extratests-phub`) schedules on every build where those suites run.

The existing SLES4SAP test still needs a production job group entry. That is a separate job-group change and it needs QE-SAP sign-off. The needle also needs a refresh.

SLE 16.1 needs the PackageHub registration issue fixed. The SUMA-bundled Grafana still has no agreed owner. And the CVEs that need a WebSocket or a more complex network setup are still on my list.

The dashboard was the reason I started this. Now the test suite is the reason I can trust the dashboard. The bicycle helmet is finally off the shelf and on my head.

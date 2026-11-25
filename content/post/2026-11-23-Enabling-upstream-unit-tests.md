---
title : "Enabling upstream unit tests, one container per package"
subtitle : "A prompt, a reviewer and a lot of build logs"
date : 2026-11-23T05:00:00+02:00
tags : ["openSUSE", "SLES", "SUSE", "Quality engineering", "LLM", "OBS"]
type: post
---

In my view, quality engineering is not only about testing and writing automated tests. Quality engineering is much broader than that. When you think about the quality of a bicycle you do not just think about the stress test of the frame or the puncture test of the tyres. Quality of a bike is about what materials and parts are used to build it. How it works in different conditions, in rain, in snow, on terrain. How it feels compared to the other brands. In general, can we talk about the quality of a bike without knowing how other bikes work, feel and look? Hardly, I believe.

It is not much different when it comes to operating systems, or more specifically when it comes to Linux distributions. I like to look at different aspects of their quality and see how openSUSE and SLES are doing. [Andrea Manzini](https://ilmanzo.github.io/post/compare_package_versions_across_distros/) for example compares package versions across distributions. How fresh a distribution is, is a quality question. Just like food, fresher means better.

This time I picked a very simple but often overlooked detail. The unit tests of our upstream projects. One cheap way to improve the quality of the distribution is to make the tests that upstream already wrote actually run when we build. This post is the story of doing that, first by hand, then with containers. So far that is five packages, three changes accepted or merged, two requests waiting, and nothing declined.

## The question

I wrote before about shipping upstream functional tests as installable packages [here]({{< ref "2026-07-25-Enabling-upstream-functional-tests.md" >}}). The topic here is the tests that run during the build.

I do admit that one motivation was that management was talking about how much it values unit tests. I could not agree more, so I decided to take a look at this topic.

I wanted to know how many packages we support and maintain in openSUSE and in SLES where the upstream project does have unit tests, but for some reason we do not run them at build time. I was especially interested in the cases where our competitors, the other Linux distributions, do run the same unit tests. The benefit of running unit tests at build time feels so obvious that one may think it does not make sense not to run them. Running the upstream tests during the OBS build checks that the software really works on your distribution, with your compilers, your libraries and your CPU architectures. Broken packages are caught before the users get them.

But let's be fair. It is totally understandable that sometimes package maintainers decide not to enable unit tests. Some tests need network access or special hardware. Some tests are flaky and keeping them green is a pain. Running tests takes time and loads the build infrastructure. And most importantly, if a CI policy does not require it then some people simply do not bother.

I do.

## The landscape

I did some quick research about the landscape. I am not going to publish a scary percentage here, because a number without context never helped anyone. Without finger-pointing to specific packages I can say that there are enough packages in SLES and openSUSE where this work is worth taking seriously. There are packages where we run unit tests and others do not. There are packages where others run unit tests and we do not. Hardly a surprise that I am interested in the second group. About the first group I expect the Fedora, Ubuntu and Debian folks to write a blog post.

For the method, I compared our SLE 15 and Factory packages against the current Fedora spec files and the Debian packaging, per package and with a script. The first pass gave close to two hundred candidates. That is not two hundred broken packages. Many of them are wrapper packages, some name a different upstream, and some already run their tests. What matters is how many of them are fixable with a test switch and a skip list, and that number is much smaller.

Later I also checked the list against the real build logs of the shipped SLE packages. A script follows the link to the maintenance incident that built the package, cuts the `%check` section out of the build log, and looks for test results. That check corrected a few rows. Some packages that looked untested were running their tests all along. A few that looked tested were only building the test programs, or hiding the failures. The log is the only reliable source here. The list is a hypothesis, the build log is the evidence.

## How the order was decided

The list is not alphabetical and it is not sorted by how easy the change is. It is sorted by how much pain each package causes, per hour of work. The input is Bugzilla.

For every candidate package I collected its bug history and scored each bug. A bug that ended as fixed or verified weighs more than one that was closed as invalid or as a duplicate. A recent bug weighs more than an old one, with an exponential decay, so bugs older than five years count as zero. A bug in a security incident weighs more than a normal one. A bug that led to a maintenance update or a PTF weighs more than one that stayed open. Then I divided the total by the effort the test enablement needs, which I had estimated earlier, and ranked the result.

This is why the first packages are not the trivial ones. They are the packages where the history says the package is troublesome and the fix looks affordable. It is a data driven decision, and I can explain every rank. It also means the ranking is only as good as the bug data. A package nobody reports bugs against can be broken and still rank low. I use the ranking as a starting point, not as a truth.

## The first three packages

I took a few easy-looking examples, and fixed up their spec files.

The [freerdp change](https://build.opensuse.org/request/show/1397741) needed `-DBUILD_TESTING=ON` because CMake builds no tests without it. The spec already had a `%ctest` call that ran zero tests, which is worse than having no tests, because it looks like the tests run. It also needed `LD_LIBRARY_PATH` in `%check` because the spec disables RPATH, and `xvfb-run` for one test that wants a display. 183 tests pass on Tumbleweed x86_64 and i586. In SLE 15 the same project is packaged as `freerdp2`, from the 2.x branch.

The [xrdp change](https://build.opensuse.org/request/show/1397742) needed `--enable-tests`, a `%check` with `make check`, and `check-devel` and `libcmocka-devel`. 213 tests pass.

The [gvfs change](https://src.opensuse.org/GNOME/gvfs/pulls/13) was the interesting one. Upstream registers exactly one unit test under `-Dunit_tests=true`. One test is still one test. When I applied the same change to the older SLE 15 code streams the test failed, because the old test expected the SMB password to survive the URI re-encoding and the current upstream test expects it to be dropped. The backport of the test fix is part of the change. The pull request is merged now, and the GNOME maintainer has forwarded it to Factory in [request 1399909](https://build.opensuse.org/request/show/1399909), which is in review.

## Recording everything

While working on these three packages I kept recording every step, with a clear goal. I wanted to create a smart prompt, and later a skill file, so that LLMs can repeat the same work. I do like fiddling with spec files and test builds. But honestly, if there is one thing where AI is really good, it is following a well written recipe and doing repetitive tasks.

After three packages I had a quite detailed prompt. I tested it with dry runs and I was satisfied with the results.

The recipe starts with the test history. Were the tests ever enabled? Were they turned off later? Why? Is the reason still valid? A package that had tests turned off in 2019 because of a flaky test may have that test fixed since then. A package that never had tests is a different case.

Then research the test setup. Read how Fedora runs the tests in its spec and how Debian does it in `debian/rules`. Ubuntu follows Debian here. Find the switch of the upstream build system, `BUILD_TESTING`, `--enable-tests`, `meson test`, `make check`, `pytest`. Understand the harness before skipping anything. Some suites have an aggregator test that loops over other tests and stops at the first failure. With such a harness one build shows you one failure at a time, which is a very slow way to learn.

Derive the skips from causes, not from trial and error. If a test needs the network, a display, a TPM, or a five gigabyte test data repository that nobody ships, skip that test with a comment that says why. Do not gut the suite to get a green build.

Edit the spec minimally. Add the switch, a `%check` section, and only the BuildRequires the tests need. If the change would break older code streams that build from the same spec, guard it.

Then the local gate. Nothing goes to OBS before a full local build passes with `%check` actually running, and the test counts are read from the log. Seeing `Executing(%check)` in the log only proves that the section ran, which is why freerdp is a good example. Its `%ctest` ran zero tests. The test counts in the log are the proof.

And write the `.changes` and the request message in a fixed style. Short and plain. What changed, why, and the test result. If the tests were turned off before, say when, why, and why the reason no longer applies.

Then Factory first. The openSUSE change goes to the devel project and to Factory. Only after that comes the SLE 15 maintenance request, which references the Factory request and adapts the change to the older version, because the test set and the dependencies are different there.

## From a prompt to a container

The next step was doing all this in an agentic mode. That is something I have not done so far, so I wanted to be careful. I needed a well confined container with limited access to tokens, keys and services. When it comes to AI driven and especially agentic AI, security comes first.

So I wrote a prompt for the container itself. I wanted a container that gets one package name from the list and the prompt, enables the unit tests with a worker model, builds the package locally and reads the build result, and iterates until the tests run or until the task is proven impossible. Then a reviewer model reads the change, the worker fixes what the reviewer found, and it rebuilds. The review round repeats until the reviewer is satisfied. After that the container makes the pull request or the submit request for openSUSE and for SLE, watches the builds, reads the failing logs, fixes, pushes, and watches again. When everything is green it returns and writes the links to the list with a note.

I also wanted to start as many containers as I wish, each on a different package, and they should not step on each other.

What I ended up with is [github.com/bzoltan1/unit-test-enabler](https://github.com/bzoltan1/unit-test-enabler), public and GPL-2.0-or-later. The parts that are specific to the SUSE internal infrastructure are configuration, not code, so the same thing runs against another build service with a different config file.

## Inside the container

The image is Tumbleweed with osc, the build script, qemu and the opencode agent. There is no privileged mode. The only device the container gets is `/dev/kvm`, because the local build is a real `osc build` in a KVM virtual machine, the same way OBS builds it. The build root is then a plain file, no loop mount is needed, and rootless podman runs that VM as my user.

Credentials never go into the image. They are podman secrets, created from files on my machine, and installed into the container home at start. The image itself has no token and no key.

There is a guard in front of `osc`, `git`, `git-obs` and `tea`. It allows writes only to my home project and to my forks. It blocks creating a request or a pull request until the orchestrator opens the gate for that target. The gate opens only after the reviewer approved and the builds are green. For SLE it opens only after the Factory request exists. The guard blocks accept, decline and merge.

A reviewer model reads every proposed change in a separate, read-only session. It cannot edit files, it cannot commit, it cannot push. It reads the diff, the build logs and the report, and it writes a verdict with findings. On a claim that the tests already run, it has to check the build log, not the spec text. A change that fails the review does not get a request from the container.

Nothing in the container asks for permission. Every permission is allow or deny. Git and ssh run non-interactive. There is no pager and no editor.

I want to list the limits too, because a containment story that only lists the good parts is not useful. The tokens in the container are my personal tokens, with the scopes I already had. They are not scoped to the home project and they are not short lived. The guard is a set of wrappers in `PATH`, so a model can still call `/usr/bin/osc` directly, or write a small Python script and use the token with curl. The reviewer is read-only because of its opencode permissions and a deny list on the publishing commands, not because of an OS sandbox. There is no network egress restriction. And the build logs and upstream test files are untrusted input, so a prompt injection hidden there is a real risk, not a theoretical one. This is acceptable for now because the whole thing runs on my machine, with my credentials, on packages I am allowed to touch. A model that follows an instruction hidden in a build log could do more with my token than submit a request. It could accept or merge my own requests, read projects I have access to, or send the token out. My plan is a dedicated token with the smallest scope that still does the job, a short lifetime, and an egress limit on the container. Until that exists, the honest description is a policy layer in front of my full credentials, not a sandbox.

## Running it

Setup is one command. It checks the host, creates the secrets, builds the image, and runs a small end-to-end smoke test that logs in to both build services, does a real KVM build of a tiny package, and pings every model. When it prints `smoke test passed`, the host is ready.

Then you ask for the next packages and start a container per package.

```
$ host/next-packages.sh 2
grub2 containerd
$ host/run.sh grub2
started te-grub2  (podman logs -f te-grub2; results .../results/grub2)
$ host/run.sh containerd
started te-containerd  (podman logs -f te-containerd; results .../results/containerd)
```

Some hours later the fleet looks like this. The snapshots in this post were not all taken at the same moment, and I trimmed the lines for width.

```
$ host/status.sh
te-openssl     Exited (0) 11 hours ago
te-dovecot     Exited (0) 6 hours ago
te-gvfs        Exited (0) 10 hours ago
te-grub2       Up 6 hours
te-containerd  Up 4 hours

containerd  ready_for_human_review  factory:submitted rev=1 obs=0 | sle:dry_run_ready rev=2 obs=0 | calls=12
dovecot     ready_for_human_review  factory:submitted rev=3 obs=1 | sle:already_enabled | calls=11
gvfs        ready_for_human_review  factory:submitted rev=1 obs=0 | sle:dry_run_ready rev=1 obs=0 | calls=9
grub2       running                 factory:running rev=2 obs=0
openssl     already_enabled         factory:already_enabled | sle:already_enabled | calls=3
```

`rev` is the number of review rounds, `obs` is the number of build rounds after a failed build. One of these packages came back as `already_enabled`. openssl is a noarch wrapper, and the real source `openssl-3` already runs its tests. The agent came back with evidence instead of a spec change, which is a valid result and cheaper than a wrong request.

I also ran gvfs through a container after the hand-made pull request, as a check that the tool reaches the same result on a change I already knew worked. The branch and the pull request already existed, so that run did not open a second request. The `factory:submitted` row in the table is that check run, the SLE 15 plan it prepared is still waiting for me, and I will submit it after reading it. That is also why gvfs appears twice.

containerd is the best example of the history check. Its `go test` was dropped from `%check` in 2018 because it failed randomly (boo#1095817). The current version passes 2258 tests and subtests, so the old reason no longer holds. The agent found the reference in the history, checked the current release, and turned the tests back on.

## What it costs

Every model call is logged with its token counts. A small script prices them.

```
$ host/costs.py
package        calls   tokens      usd  unpriced
containerd        12  35.89M     1.65   34.00M
dovecot           11  43.16M     2.41   39.91M
grub2              6  13.69M     1.23   11.91M
gvfs               9  43.50M     1.13   42.17M
openssl            3   4.46M     0.00    4.46M

per model
  helmcode/deepseek-v4-flash            in 2.72M out 0.23M cache rd 132.59M wr 0.00M  no price
  google-vertex/claude-opus-5-5@default in 0.00M out 0.06M cache rd   7.85M wr 0.68M  $6.42

total priced  $6.42  (list price estimate)
```

Read this carefully. The prices are public list prices. They are here to show the scale, not what I pay. Some numbers and details in the example outputs are sanitized or rounded on purpose too, so treat all of this as scale and not as an invoice or a log extract.

The $6.42 is only the paid reviewer, which at that time was Opus. About 94 percent of the tokens belong to the worker, and in this setup the worker has no per-token price. It runs on a flat subscription that I already pay for, so the marginal cost of a run is zero for me, but on a metered plan those tokens would cost money too. Most of them are cache reads, which is what a long build log review looks like. openssl stopped at the evidence stage, so there was no change and no paid review.

The real cost is time, and most of the wall clock goes to waiting for OBS, not to model tokens.

The containers have hard limits. A cost limit in dollars, a wall clock limit per package, a number of local builds per phase, and a number of build fix rounds. If a container reaches a limit it stops and writes down the evidence instead of guessing. There is also a disk limit. The first version kept a forty gigabyte build root per build and filled the machine. Now there is one build root at a time and it is deleted at the end of the run.

## Which model should review

I wanted to know whether a different model would review as well as the one I started with. Since the containers run in parallel, the reviewers can run in parallel too. A shadow reviewer gets the same prompt and the same checkout as the main reviewer, writes its own verdict, and does not influence the run. Then a report compares them.

The first attempt was wrong, and I am glad I looked. The shadow reviewer had read access to the main reviewer's verdict file. Nothing stopped it from copying, and a reviewer that can read the other opinion is not a second opinion. Now each shadow works in its own view of the results, with links to the diff and the build logs, and no access to the other verdicts. The worker and the main reviewer cannot read shadow verdicts either.

The second lesson was about expectations. A shadow that runs only on finished packages sees only the last round, which is almost always an approval. So it measures false alarms, not whether a model finds real problems. Qwen 3.8 27B invented a Unicode homoglyph in a test file and asked for changes. The bytes in the tarball were plain ASCII. That is exactly the kind of review nobody wants.

With those caveats, the numbers from eight replayed rounds. `verdict` is the number of reviews that produced a usable verdict. `agrees with main` means the same decision, approve or confirm against changes requested or dispute.

| model | reviews | verdict | agrees with main | findings per review | usd per review | minutes |
|---|---|---|---|---|---|---|
| Claude Opus 5.5 | 8 | 8 | main | 3.0 | 0.37 | n/a |
| Gemini 3.5 Flash | 8 | 8 | 8 of 8 | 1.9 | 0.54 | 2.3 |
| DeepSeek V4 Flash | 8 | 8 | 8 of 8 | 3.1 | 0.00 | 4.7 |
| Qwen 3.6 | 8 | 7 | 7 of 7 | 1.4 | 0.00 | 4.8 |
| Qwen 3.8 27B | 8 | 8 | 7 of 8 | 1.2 | 0.00 | 4.2 |

Opus cost less per review than Gemini, so the expensive model was not the expensive reviewer. DeepSeek and Qwen 3.6 were not worse on agreement than the paid models, but agreement is not quality and most of these rounds were approvals, so my claim is narrow. On this small sample DeepSeek and Qwen 3.6 did not disagree more. Qwen 3.8 is the exception, it agreed with the main reviewer seven times out of eight. That was enough for me to switch the reviewer to DeepSeek V4 Flash, which comes with the subscription I already have, so the marginal cost is zero. The weakness is that the same model now writes the change and reviews it. I keep Qwen 3.8 as a free shadow because of that.

One more practical finding. Qwen 3.8 wrote its verdict in the reply instead of the file in six of the eight rounds. The harness recovers those, but it is not what I asked for.

## How many rounds

```
$ host/review-report.py
package     phase   outcome       appr post bld  t_appr t_phase  rounds
containerd  factory submitted      1    0   0     20m    6.2h  A(3m,1n)
containerd  sle     dry_run_ready  2    0   0     48m    5.6h  CR(1M,3n) > A(1m,2n)
dovecot     factory submitted      2    1   1    2.5h   17.2h  CR(2M,5m,1n) > A(2m,2n) > A(2m,1n)
gvfs        factory submitted      1    0   0     15m   17.1h  A(1m,2n)
gvfs        sle     dry_run_ready  1    0   0     30m   16.7h  A(4m,2n)
grub2       factory running        2    0   0     51m     53m  CR(1M,2m,1n) > A(2m,1n)
```

`A` is approve, `CR` is changes requested. `M`, `m` and `n` are major, minor and nit findings. `appr` is the number of review rounds until the first approval, so dovecot (packaged as `dovecot24`) has a `2` there and three rounds in the sequence, because the third round came after a build failed on a repository that the devel project has disabled. `t_phase` is much larger than `t_appr` because the container sits waiting for OBS.

Six phases are in the table, and the two gvfs rows come from the check run and not from a new package. Three phases passed the first review, which is half. The mean is 1.5 rounds to the first approval. In all three cases where the first review asked for changes, the next review approved. That is the number I care about. So far a unit test enablement needs one to two review rounds before it waits for a human.

## What is left for the human

The guard blocks accept, decline and merge, so the containers only create requests and write down the links.

```
$ host/requests.sh --open
PLAN     containerd  sle      results/containerd/sle/submit_plan.md
         dry run, submit it from the plan
PLAN     gvfs        sle      results/gvfs/sle/submit_plan.md
         dry run, submit it from the plan
WAITING  containerd  factory  https://build.opensuse.org/request/show/1399862
         reviews a maintainer:new, a review group:new
WAITING  dovecot     factory  https://build.opensuse.org/request/show/1399809
         state new
```

This reads the current state from the build service and from Gitea. For a request it shows the state, the reviews and the comments, and it follows the supersede chain. For a pull request it shows the reviews and the commit status. Each item gets a label. WAITING is open with nothing new. ACTION means somebody has to look, for example a decline, a requested change, or a build failure. PLAN means a SLE dry run waits for me. LANDED means accepted or merged. Bot comments, and build failures the target package already has, are ignored, otherwise half of the list is red for no reason.

The SLE run is in dry run mode. It prepares the branch, builds it on all the architectures the package builds for in the internal build service, and writes a plan with the exact commands and messages instead of submitting. I read the plan and submit it myself. One day that can be a flag.

For the openSUSE devel projects it is different. Once the reviewer approved and the builds are green, the container opens the request under my name. I read every request right after it is opened, and I answer the review.

So far the hand-made freerdp and xrdp changes are accepted in Factory, the gvfs pull request is merged and its Factory request is in review, two devel-project requests from the containers are waiting for a maintainer, and the SLE plans are waiting for me. Nothing has been declined.

## Proving it mattered

Passing tests are not the same as useful tests. After the first packages I wanted to know whether the tests we were enabling would have caught real problems, not only whether they pass today.

So I built a second job over the same list. For one package it takes a bug that was fixed later, the exact source that customers had, and the test from that source. It proves that the test existed at the time, that it fails on the affected code for the reason in the bug, and that it passes with the fix. A second model audits each case and reruns the build. A case counts only when the test was in the shipped source and the failure matches what the customer saw.

The job ran over the full list. Most candidates did not survive, because there was no fixed source to compare, because the test never reaches the changed code, or because the fix arrived as part of a larger version bump. But four packages gave six proven cases. In each one the package was built green and shipped while a test that would have failed that same build was sitting in the source. The tests were not run, the build stayed green, and the bug reached customers. That is the strongest argument for this work I can give, and it is the reason I keep looking for more.

## Learnings

Every run ends with a learnings file, whatever the outcome. The worker writes what is specific to the package, what is reusable for other packages, and where the prompt or a tool misled it. The orchestrator adds the parts that do not need a model, such as the rounds, the findings the reviewer marked as blockers, the commands the guard blocked, and the cost.

I read those files and promote what holds up into the prompt or into the lessons files. Nothing goes into the containers automatically, because one model's wrong conclusion should not spread to every later run. That is the same discipline as any other code review.

## What is next

The list is long and the containers are cheap, but I am not going to flood the maintainers. My own rule is at most three open Factory requests from the containers at a time. After that I wait for reviews, fix what they find, and only then start the next batch. I also want to fold the lessons from these packages into the prompt, so the next batch starts smarter than this one did.

The code is at [github.com/bzoltan1/unit-test-enabler](https://github.com/bzoltan1/unit-test-enabler). It is generic. With your own credentials, your own package list and your own prompt, it runs against your infrastructure. If you maintain packages in any distribution, the idea probably transfers.

If you are one of the maintainers who received one of these requests, thank you for reading it. Most of these changes are test-only, and a test-only change is boring to review. A few, like dovecot, also carry real fixes. It is the kind of boring that keeps the next update from breaking.

The agents propose and people decide. That is the design. With my own tokens in the container it is a rule the guard keeps, not a boundary the system enforces.

---
title: "From Shell Script to Python Runtime"
date: 2026-04-04
description: "850 lines of Python embedded in EC2 user-data, a 16KB size limit, and why testability matters more than velocity. How the runner bootstrap evolved over 7 months."
tags: [devops, python, github-actions, aws, automation]
---

This is the post where I admit that I kept things in bash for too long.

In [Part 2](/2026/03/26/ami-baking-pipeline.html) I mentioned the November 6th marathon — 22 commits in a day fixing AMI cleanup edge cases, all written in inline bash inside GitHub Actions workflows. I said I knew I'd need to rewrite it, but I had other priorities. That was true. It's also true that by the time I finally did the rewrite, the bash had grown into something genuinely hard to maintain.

This post is about what that migration looked like, why it mattered more than I expected, and what the Python runtime actually does.

## How It Started

The first version of the runner infrastructure was mostly bash. The EC2 user-data script that bootstraps a new instance was a shell script. The AMI baking logic was inline bash in the GitHub Actions workflow. The pre-job cleanup hooks were bash. Even the runner registration and deregistration was handled by shell commands piped together with `&&` and `||`.

For a while this worked fine. Shell scripts are great for stringing together AWS CLI calls and systemd commands, and when the system was simple — one instance, one runner process, no autoscaling — there wasn't enough complexity to justify anything else.

The problems crept in gradually. The first sign was the pre-job hook. We run persistent runners, which means the same runner picks up job after job without being destroyed between runs. That's great for performance, but it means the workspace directories accumulate state between jobs. Git lock files from crashed builds, stale LFS hooks from repos that used to use LFS but don't anymore, repo-local git config that bleeds across unrelated jobs.

The hook started as a two-line script that deleted `.git/index.lock` before each job. Then it needed to handle nested lock files. Then it needed to clean up LFS hooks that were causing checkout failures on runners that didn't have `git-lfs` installed. Then it needed to scrub repo-local git config. Each addition was a few more lines of bash, and each one had to be careful about paths, error handling, and not breaking the cases it wasn't trying to fix.

By the time the hook was "done" it had been through 5 iterations across multiple PRs, and it was still fragile. A missing quote here, a glob that expanded wrong there. Every fix felt like it might introduce a new edge case.

## The Breaking Point

The real breaking point wasn't a single incident. It was the accumulation of several things happening around the same time.

The AMI baking pipeline had grown from a simple "stop instance, create image" workflow to a multi-step orchestration with detach, patch, pre-pull, cleanup, snapshot, roll forward, and smoke test. All in bash. With error handling that was either missing or wrong.

The user-data bootstrap script had evolved to support multiple concurrent runner processes per instance, with systemd service templating, signal handling for graceful shutdown, and runner registration token lifecycle management. In bash.

And I was hitting POSIX compatibility issues. The EC2 instances run commands via SSM, and SSM's `AWS-RunShellScript` document uses `/bin/sh`, not `/bin/bash`. Features I was relying on — `pipefail`, arrays, certain parameter expansion syntax — either didn't work or behaved differently. I'd fix something locally, push it, and watch it fail in a completely different way on the instance.

The tipping point was when I needed to add autoscaling support. The Lambda metric publisher needed to be Python anyway (it's a Lambda function), and looking at the codebase, I realized I was about to have the autoscaling logic in Python, the baking logic in bash, the bootstrap in bash, and the cleanup in a mix of both. Three different languages and paradigms for what was essentially one system.

So I carved out a week and rewrote everything.

## What the Python Runtime Actually Does

The core of the system is now a Python package called `runner_ops`. It has a CLI entrypoint with subcommands for each operational task, and a set of modules that handle specific domains.

**The bootstrap runtime** is the big one — about 850 lines that get base64-encoded and embedded directly into the EC2 user-data via cloud-init. When an instance boots, the cloud-init script decodes the Python runtime, writes it to disk, and runs it. The runtime handles everything the bash used to do and more:

It queries the EC2 instance metadata service to figure out its own instance ID and region. It downloads and extracts the GitHub Actions runner archive. It creates systemd service units for however many concurrent runner processes the instance is configured to run — if the config says 4 concurrent runners, it generates 4 separate services, each with its own working directory and registration. It handles runner token registration with GitHub, including generating unique runner names based on the instance ID and slot number so there are no naming conflicts across the fleet.

The persistent runner loop is where the bash-to-Python difference matters most. Each runner process runs in a loop — register, start the listener, wait for it to exit, clean up, re-register. In bash, the signal handling for graceful shutdown was a mess of `trap` statements that sometimes worked. In Python, it's a signal handler that sets a flag, and the loop checks the flag between iterations. If a SIGTERM arrives while a job is running, the runner finishes the current job and then exits cleanly instead of being killed mid-build.

**The pre-job hook** went from a brittle bash script to a Python function that systematically walks the workspace directories and cleans up git lock files, LFS hooks, LFS config, and repo-local git configuration. It logs what it finds and what it removes, which made debugging checkout failures significantly easier. The bash version would silently succeed or silently fail. The Python version tells you exactly what state it found.

**The AMI baking module** is the 900-line orchestrator from [Part 2](/2026/03/26/ami-baking-pipeline.html). The `finally` block alone justified the rewrite — it ensures the ASG min_size gets restored and detached instances get terminated even if the bake fails halfway through. In bash, there was no equivalent. If the script failed after detaching the instance but before the cleanup, I'd have a dangling EC2 instance accumulating charges until I noticed.

**The AMI cleanup module** is the safety-capped deletion logic. **The autoscaling metric publisher** is the Lambda from [Part 3](/2026/04/02/autoscaling-without-webhooks.html). **The notification module** handles Slack webhook calls for pipeline status. **The rollout module** handles zero-downtime instance replacement after a prod apply.

## The 16KB Problem

There was one genuinely tricky part of the migration: EC2 user-data has a 16KB size limit. The bash bootstrap was already pushing against it, and 850 lines of Python is significantly larger than the shell version it replaced.

The solution was compression. The Terraform template base64-encodes the Python runtime and a JSON config blob. The cloud-init script — which is still a thin shell wrapper, because it has to be — decodes the base64, decompresses it, writes the Python files to disk, and starts a systemd service that calls the Python entrypoint.

The shell wrapper is about 60 lines now. Its only job is to unpack the Python runtime and hand off control. All the actual logic lives in the Python, which means all the actual logic is testable.

## The Part That Actually Mattered

The testability is what made the biggest difference in practice. Every function in `runner_ops` that computes something has unit tests. The cleanup plan logic, the shell command builders, the tag verification, the polling utilities, the metric publisher. The AMI baking module has tests for the error handling paths that were completely untested when they were bash.

When I added the LFS cleanup in March — the fix for checkout failures on persistent runners — I wrote the test first, implemented the cleanup function, and verified it worked before deploying. In the bash era, I would have written the fix, deployed it to dev, triggered a bunch of jobs, and hoped for the best. Then probably spent another 2-3 iterations fixing edge cases I didn't think of.

The other thing that mattered was the CLI. Having `runner_ops` as a proper CLI tool with subcommands meant the GitHub Actions workflows became thin wrappers:

```yaml
- name: Bake and roll dev AMI
  run: python -m runner_ops.cli bake-dev-ami --workspace dev ...
```

The workflow doesn't contain logic anymore. It passes configuration to the CLI, and the CLI does the work. If something goes wrong, I can reproduce it locally by running the same command with the same arguments. That was never possible with inline bash workflows where the logic was split across workflow YAML, shell scripts, and SSM commands.

## What I Should Have Done Differently

I should have started with Python. Not because bash is bad — it's great for simple automation. But the moment I had multi-step orchestration with error handling and cleanup requirements, bash was the wrong tool. I knew this intellectually. I just kept thinking "I'll rewrite it after this next feature" and then the next feature added more bash.

The migration itself took about a week of focused work. That's not nothing, but it was less painful than I expected because the system's behaviour was well-understood by that point. I wasn't designing something new — I was translating something that already worked (mostly) into a language that could handle the edge cases properly.

If I were starting this project over, I'd write the cloud-init wrapper in shell — because you have to — and everything else in Python from day one. The initial velocity of bash isn't worth the maintenance cost once the system gets beyond a few hundred lines.

## What's Next

The tooling is in good shape now, but there's one more piece of the story: how do you safely manage all of this as a single engineer? Dev/prod isolation, deployment gates, automated rollouts, and the practices that prevent a solo operator from accidentally breaking production. That's [Part 5](/2026/04/06/safe-infra-changes-team-of-one.html).

---

*This is **Part 4** of a series on building production-grade self-hosted GitHub Actions runners at a robotics startup.*

- ***Part 1:** [How a Single EC2 Instance Runs Our Entire CI Pipeline](/2026/03/20/single-ec2-entire-ci.html)*
- ***Part 2:** [We Replaced Packer With a GitHub Actions Workflow](/2026/03/26/ami-baking-pipeline.html)*
- ***Part 3:** [Autoscaling GitHub Runners Without Webhooks](/2026/04/02/autoscaling-without-webhooks.html)*
- ***Part 4:** From Shell Script to Python Runtime (you are here)*
- ***Part 5:** [Safe Infrastructure Changes With a Team of One](/2026/04/06/safe-infra-changes-team-of-one.html) — dev/prod isolation and deployment gates without a platform team*

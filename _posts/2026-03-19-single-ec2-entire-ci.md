---
title: "How a Single EC2 Instance Runs Our Entire CI Pipeline"
date: 2026-03-19
description: "One m7i.xlarge, four concurrent runners, zero image pull time. How we built a production CI pipeline for a robotics startup for less than $170/month."
tags: [devops, github-actions, aws, ci-cd, robotics]
---

Look, I'm not going to pretend I set out to build something clever here. I'm a DevOps engineer at a robotics startup. We needed CI that worked. GitHub-hosted runners didn't. So I built something that did.

7 months and 166 commits later, we're running our entire CI pipeline — builds, tests, linting, Docker-based jobs, the lot — on a single EC2 instance. One. Not a fleet. Not Kubernetes. Not some managed runner service that charges per minute. One `m7i.xlarge` in `eu-central-1`.

And it handles everything.

## A 23GB Container Image and Nowhere to Put It

Here's the thing nobody warns you about when you're building robotics software with GitHub Actions: your devcontainer image will be enormous, and GitHub really does not want you to use it.

Our devcontainer image is about 23GB. It has ROS 2 Humble, a full robotics toolchain, simulation dependencies, OpenCV, BLAS libraries, `colcon` build tools — everything you need to build and test a robotics monorepo. It's big because robotics is big. There's no way around it.

Standard GitHub-hosted runners have 14GB of disk space. Our image literally does not fit. So right from the start, we were forced onto GitHub's `Large-GPU-Runner` — a beefier hosted runner that costs significantly more per minute. And even on the large runner, every single CI run started by pulling that 23GB image from scratch. That pull alone took 6-8 minutes. Every. Single. Run.

We looked at caching it. GitHub's built-in action cache has a 10GB limit. The Docker cache action is backed by the same 10GB store. Our image is 23GB. Dead end.

So before a single line of our code even started compiling, we were already 6-8 minutes into the job and burning money on oversized hosted runners. Multiply that by every PR, every push, every developer on the team. At a startup, time and money are the same thing, and we were wasting both.

## The Obvious Answer (and Why I Didn't Use It)

Self-hosted runners were the obvious fix. But the question was how.

I looked at the options. There are managed self-hosted runner services. There's the actions-runner-controller for Kubernetes. There are Terraform modules with hundreds of variables that try to cover every possible use case.

They all felt like overkill for what we needed.

We're a small team. We don't have a Kubernetes cluster to throw runners onto (and honestly, running K8s just to host CI runners is a special kind of infrastructure madness). The managed services charge per-minute pricing that, once you do the maths, isn't that different from just paying GitHub directly. And the big open-source Terraform modules were so configurable that configuring them felt like a project in itself.

So I did what any engineer does when the existing tools don't fit. I opened a new repo, wrote an initial commit, and started building.

## The Architecture (It's Embarrassingly Simple)

Here's the entire infrastructure:

- 1 x `m7i.xlarge` EC2 instance (4 vCPUs, 16 GiB RAM)
- 1 x Auto Scaling Group (min 1, max 2)
- 1 x Launch Template with a custom AMI
- 1 x Lambda function for autoscaling signals
- A handful of CloudWatch alarms

That's it. No load balancer. No container orchestration. No service mesh. The instance boots, registers itself with GitHub, and starts picking up jobs.

The key insight — and this took me a while to land on — is **runner concurrency**. A single GitHub Actions runner process can only handle one job at a time. But nothing stops you from running multiple runner processes on the same machine. So I run 4. One per vCPU.

When the instance launches, 4 separate runner processes register with our GitHub org. GitHub sees 4 idle runners, all with the same labels, and distributes jobs across them automatically. 4 concurrent CI jobs, one instance. Each runner gets about 1 vCPU and 4 GiB RAM, which is plenty for our workloads.

## The Real Win: Zero Image Pull Time

The concurrency trick is nice, but the real win is what's baked into the AMI.

Remember that 6-8 minute image pull? Gone. Completely gone. Our custom AMI has the 23GB devcontainer image pre-pulled into the Docker cache. When a CI job starts and specifies `container: image: ghcr.io/...devcontainer:stable`, Docker finds it locally and starts it in seconds. No network transfer, no waiting, no praying that GHCR doesn't throttle you.

We bake a fresh AMI weekly. The pipeline detaches the current instance from the ASG, patches it via SSM, pre-pulls the latest devcontainer image, snapshots it, rolls the ASG forward to the new AMI, runs a smoke test, and terminates the old instance. No Packer, no separate build infrastructure — just GitHub Actions orchestrating AWS APIs.

That AMI baking pipeline deserves its own post (the number of edge cases we hit building it was genuinely absurd), but the point is: eliminating the image pull is what made the biggest single difference. Everything else is optimisation on top.

On the very first day I set this up, I actually ran a side-by-side comparison — the same test suite running on both the `Large-GPU-Runner` and the new self-hosted runner in parallel. The difference was immediately obvious. All that time we'd been sinking into image pulls, environment setup, and cold caches just vanished.

## From Ephemeral to Persistent (and Why That Matters)

My first version used ephemeral runners. Fresh registration for every job, clean state every time. It's what most guides recommend and it feels "correct" from a security standpoint.

It was also slow.

Every job had to re-register the runner with GitHub, pull a fresh token, configure the runner binary. That's 30-60 seconds of overhead per job, and when you're running dozens of jobs a day across 4 runner slots, it adds up. Worse, you lose all your workspace caches between jobs. Git clones start from scratch, dependency caches are gone, build artifacts disappear.

About two months in, I switched to persistent runners. The runner processes start when the instance boots and they stay up. They pick up jobs, run them, pick up the next one. State persists between runs, which means git fetches are incremental instead of full clones, Docker layers stay warm, and builds that benefit from cached outputs get those benefits.

The tradeoff is that you need to actively manage workspace hygiene. Stale git lock files from killed jobs, leftover LFS hooks from previous checkouts, repo-local git config that bleeds between projects — all real problems we hit. I ended up writing a pre-job hook that runs before every single job and cleans all of this up. It was annoying to build, but it runs in under a second and it solved the reliability issues completely.

That's the thing about self-hosted runners that nobody writes about. The initial setup takes a day. Getting it production-ready takes months. The edge cases that only surface when you've been running hundreds of jobs a week — those are where the real work is. But once you've solved them, they stay solved.

## What About Cost?

This is the part everyone actually cares about, so let me be real about it.

An `m7i.xlarge` on-demand in `eu-central-1` runs about $170/month. We could cut that significantly with reserved instances or savings plans, but we haven't bothered yet because even at on-demand pricing, the maths works out easily.

Compare that to what we were paying before. The `Large-GPU-Runner` is one of GitHub's premium hosted runner tiers — you're paying a meaningful premium per minute for those. And we had no choice but to use them because standard runners couldn't even fit our image. With self-hosted, we went from paying per-minute for oversized hosted runners to a flat monthly cost for a machine that's always warm and always ready.

But the real cost saving isn't the compute bill. It's developer time. When every CI run was burning 6-8 minutes just pulling a container image before doing anything useful, that's dead time on every single push. Across a team, across a day, across a sprint — it compounds into something massive. Eliminating that wait was worth more than any line item on the AWS bill.

## The Autoscaling Story

For the first 6 months, we just ran a single instance with `max_size = 1`. No autoscaling. One box, 4 runners. It handled everything.

Recently, as the team and the number of pipelines grew, we started occasionally saturating all 4 slots. So I added job-driven autoscaling: a Lambda function runs every minute, calls the GitHub API to count how many runners with our label are currently busy, and publishes that as a CloudWatch metric. Standard ASG scaling policies react to it — scale out by 1 when all 4 runners are busy, scale back in when they've been idle for 5 minutes.

The whole autoscaling layer is about 170 lines of Python and some Terraform resources. It costs almost nothing to run (a Lambda invocation per minute is well within free tier), and it means we can burst to 8 concurrent jobs when we need to without paying for that second instance when we don't.

I deliberately chose this polling approach over webhook-based autoscaling. Webhooks mean exposing a public endpoint, managing authentication, handling delivery failures. The polling model is simpler, cheaper, and "good enough" at 60-second granularity. We also did a bunch of work to get the ASG instance startup time down — started at around 5 minutes, got it to just over a minute. I'll dig into the detail of that in a later post in this series.

## What I'd Do Differently

If I started this project over, I'd skip ephemeral runners entirely and go straight to persistent. The security argument for ephemeral runners makes sense in theory, but in practice, if you're running your own infrastructure, you're already trusting the machine. The performance difference is too significant to ignore.

I'd also write the bootstrap runtime in Python from day one instead of starting with shell scripts. The first version was a cloud-init template with bash doing all the heavy lifting — fetching tokens, registering runners, managing systemd services. It worked, but it was brittle and nearly impossible to test. The Python rewrite gave us proper error handling, structured logging, and actual unit tests. Should have done that from the start.

Everything else — the Terraform structure, the ASG-based approach, the concurrency model — I'd keep exactly the same. It's been running in production for 7 months, it's handled thousands of CI jobs, and the total infrastructure cost is less than what most teams spend on a single SaaS tool.

## The Takeaway

You don't need a complex orchestration layer to run self-hosted GitHub Actions runners. You don't need Kubernetes. You don't need a managed service. For most small-to-medium teams, a single well-configured EC2 instance with multiple concurrent runner processes will handle more than you think.

The trick is getting the foundations right: a custom AMI with your dependencies baked in, persistent runners for fast job starts, good workspace hygiene between jobs, and just enough autoscaling to handle burst load.

Start simple. You can always add complexity later. But you probably won't need to as soon as you think.

---

*This is **Part 1** of a series on building production-grade self-hosted GitHub Actions runners for a robotics startup. Coming up next:*

- ***Part 2:** We Replaced Packer With a 40-Line GitHub Actions Workflow — our zero-downtime AMI baking pipeline and the edge cases that nearly broke it*
- ***Part 3:** Autoscaling GitHub Runners Without Webhooks — how a 170-line Lambda became our entire scaling layer*
- ***Part 4:** From Shell Script to Python Runtime — how the runner bootstrap evolved over 7 months*
- ***Part 5:** Safe Infrastructure Changes With a Team of One — dev/prod isolation and deployment gates without a platform team*

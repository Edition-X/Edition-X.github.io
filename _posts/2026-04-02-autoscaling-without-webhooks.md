---
title: "Autoscaling GitHub Runners Without Webhooks"
date: 2026-04-02
description: "A 170-line Lambda, two CloudWatch alarms, and a polling model that's more reliable than any webhook. How we added burst capacity for under $1/month."
tags: [devops, github-actions, aws, autoscaling, lambda]
---

In [Part 1](/2026/03/20/single-ec2-entire-ci.html) I explained how we run our entire CI on a single EC2 instance with 4 concurrent runner processes. In [Part 2](/2026/03/26/ami-baking-pipeline.html) I covered how we bake and promote AMIs without Packer. Both of those posts describe a system that works well under normal load.

The problem was burst load. When multiple developers push at the same time, or someone triggers a batch of integration runs, all 4 runner slots fill up and everything else queues. During one busy week we saw jobs waiting over an hour and a half for a free slot. CPU was hitting 100% on the instance and the queue just kept growing.

We needed autoscaling. And the standard advice for self-hosted GitHub runners is to use webhooks — listen for `workflow_job` events, spin up instances when jobs are queued, tear them down when they finish. GitHub's own docs recommend this pattern, and tools like `actions-runner-controller` are built around it.

We went a different direction.

## Why Not Webhooks?

Webhooks are the obvious answer, and for larger teams with existing infrastructure they're probably the right one. But for us, a small team with no public-facing endpoints and one DevOps engineer, the webhook approach had some problems.

First, you need something to receive the webhooks. That means either a public endpoint with authentication and TLS, or a more complex setup with API Gateway or a queue in front of it. Either way you're maintaining infrastructure whose sole job is to receive HTTP callbacks from GitHub.

Second, webhook delivery isn't guaranteed. GitHub has retry logic, but if your receiver is down during a burst, you miss the scaling signal entirely. Now you need to handle missed webhooks, which means you need a fallback mechanism anyway — at which point you're maintaining two scaling signals instead of one.

Third, the coupling is tight. Your scaling logic is now driven by GitHub's event schema. If GitHub changes the webhook payload, or if delivery latency spikes, your scaling reacts differently. You're building reliability on top of someone else's delivery pipeline.

What if we just asked GitHub "how busy are you right now?" on a schedule and scaled from there?

## The Polling Approach

The idea is simple. A Lambda function runs every minute, calls the GitHub API to check how many runners are currently busy, and publishes that number as a CloudWatch metric. Standard CloudWatch alarms watch the metric and trigger ASG scaling policies when the count crosses a threshold.

That's the whole architecture. No public endpoints, no webhook receivers, no event-driven complexity. Just a scheduled function that publishes a number, and AWS does the rest.

The Lambda itself is about 170 lines of Python. It fetches a GitHub PAT from SSM Parameter Store, paginates through the org runners API filtering by label, counts how many are busy, and publishes the count to CloudWatch with dimensions for the ASG name and runner label. EventBridge triggers it every minute.

The scaling logic is two CloudWatch alarms and two ASG policies. When the busy runner count hits 4 — meaning all slots on our instance are full — the scale-out alarm fires and adds one instance. When the count drops to 0 and stays there for 5 consecutive minutes, the scale-in alarm fires and removes one. SimpleScaling policies with a 60 second cooldown on scale-out and 300 seconds on scale-in.

## Doing The Maths First

Before writing any code I wanted to know whether this was actually worth building. I pulled 14 days of GitHub Actions history from the API and built a queue-wait report — looking at how often all 4 runner slots were occupied, how long jobs were waiting, and what the actual queuing pattern looked like.

The data made it pretty clear. There were regular bursts where all 4 slots were full and jobs were backing up. Not constantly, but often enough that developers were noticing and occasionally scheduling their pushes around each other to avoid the queue.

I also ran cost projections for the additional instance. With a scale-out threshold of 4 and a 5 minute scale-in window, the extra instance would run for maybe a few hours per month during peak periods. The projected cost was under $3/month. For context, the time developers were losing to queue waits was worth significantly more than that.

The report made the decision easy. This wasn't speculative infrastructure work — there was a concrete problem with measurable impact, and the fix was cheap.

## The Implementation

The Lambda handler is straightforward. The interesting decisions are mostly in the configuration.

**Why poll every minute?** Because CloudWatch alarms evaluate on metric periods, and a 1 minute period gives us a 1 minute reaction time to a full queue. Our instances take about a minute and a half to launch and register as runners, so the total time from "all slots full" to "new runner available" is around 2-3 minutes. That's fast enough that most queued jobs only wait one cycle.

**Why a scale-out threshold of 4?** Because we run 4 runner processes per instance. When all 4 are busy, the next job to arrive has nowhere to go. Setting the threshold at 4 means we trigger scale-out exactly when the queue would start building. Setting it lower would mean scaling out while there are still free slots, which wastes money. Setting it higher isn't possible — you can't have more than 4 busy runners on a 4-runner instance.

**Why scale-in at 0 for 5 minutes?** Because we don't want the extra instance oscillating up and down between bursts. If a developer pushes 3 PRs in sequence, we don't want to scale in after the first batch finishes and then immediately scale out again when the next batch starts. The 5 minute window with a 300 second cooldown gives enough buffer for back-to-back pushes without keeping the instance running for hours after everyone's gone home.

**Why SimpleScaling instead of StepScaling?** Honestly, because StepScaling doesn't support cooldown periods and I only discovered that when the Terraform apply failed halfway through. SimpleScaling does exactly what we need — add one instance when busy, remove one when idle. There's no need for graduated steps when the ASG is scaling between 1 and 2 instances.

**What about `treat_missing_data`?** Both alarms use `notBreaching`. If the Lambda fails to publish a metric for a few minutes — maybe the GitHub API is slow, maybe the Lambda times out — we don't want that to trigger a scale-in. Missing data means "we don't know," not "nobody's busy." This is a small detail that prevents a bad scenario where a Lambda failure causes the ASG to scale down and drop running jobs.

## Validating It

I tested this in dev first by opening 5 draft PRs at once against the robot software repo. Each PR triggers an integration workflow that uses the self-hosted runners, so 5 PRs means 5+ jobs competing for 4 runner slots.

The first dev test showed the autoscaling working but exposed the bootstrap latency. The scale-out alarm fired within about 15 seconds of all slots filling up, and the ASG launched a new instance almost immediately. But the new instance took about 5 minutes from launch to picking up its first job — the runner binary had to download, register with GitHub, and start the listener processes. That first-job latency was too slow for what I wanted.

The AMI baking work from [Part 2](/2026/03/26/ami-baking-pipeline.html) is what fixed this. With the runner binary and dependencies pre-baked into the AMI, the bootstrap got much faster. By the time I validated in production, the scale-out alarm was firing about 1 minute 42 seconds after the burst started, and the new instance was launching 8 seconds later. Total time from burst to new capacity was under 2 minutes.

The scale-in worked cleanly too. After all the PR jobs finished, the busy runner count dropped to 0, the alarm waited through its 5 evaluation periods, and the ASG scaled back to 1. No manual cleanup, no orphaned instances.

## What It Costs

The production autoscaling has been running since mid-March 2026. In the first few days of real usage, the extra instance ran for about 2.5 hours total during burst periods. That's roughly 60 cents.

For a system that eliminates queue waits during peak development hours and requires zero manual intervention, that's a pretty good trade. The Lambda itself runs within the free tier — 128MB of memory, 30 second timeout, once per minute. The CloudWatch metrics and alarms add negligible cost.

The most expensive part of the whole setup was the time I spent building the queue-wait report to justify it.

## The Whole Stack

For anyone thinking about doing something similar, here's what the infrastructure looks like:

One Lambda function (Python 3.12, 128MB, 30s timeout) triggered every minute by EventBridge. It reads a GitHub PAT from SSM, paginates the org runners API, counts busy runners by label, and publishes a CloudWatch custom metric.

Two CloudWatch alarms watching that metric — one for scale-out (busy count >= runner concurrency), one for scale-in (busy count = 0 for N consecutive periods). Both use the Maximum statistic and treat missing data as not breaching.

Two ASG SimpleScaling policies — one that adds 1 instance, one that removes 1. Cooldowns of 60 seconds and 300 seconds respectively.

The whole thing is behind an `enable_job_autoscaling` feature flag in Terraform, so it can be toggled per environment. Dev and prod each have their own Lambda, alarms, and policies, publishing metrics with different runner label dimensions so they don't interfere with each other.

## What's Next

The autoscaling is running and the cost is negligible, but there's a bigger story here about how the tooling around it evolved. The Lambda handler was one of the first things I wrote in Python for this project. The rest of the infrastructure — the runner bootstrap, the AMI baking, the operational CLI — started as bash and stayed as bash longer than it should have. That migration, and why it mattered, is [Part 4](/2026/04/04/shell-script-to-python-runtime.html).

---

*This is **Part 3** of a series on building production-grade self-hosted GitHub Actions runners at a robotics startup.*

- ***Part 1:** [How a Single EC2 Instance Runs Our Entire CI Pipeline](/2026/03/20/single-ec2-entire-ci.html)*
- ***Part 2:** [We Replaced Packer With a GitHub Actions Workflow](/2026/03/26/ami-baking-pipeline.html)*
- ***Part 3:** Autoscaling GitHub Runners Without Webhooks (you are here)*
- ***Part 4:** [From Shell Script to Python Runtime](/2026/04/04/shell-script-to-python-runtime.html) — how the runner bootstrap evolved over 7 months*
- ***Part 5:** [Safe Infrastructure Changes With a Team of One](/2026/04/06/safe-infra-changes-team-of-one.html) — dev/prod isolation and deployment gates without a platform team*

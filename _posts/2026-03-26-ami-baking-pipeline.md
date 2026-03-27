---
title: "We Replaced Packer With a GitHub Actions Workflow"
date: 2026-03-26
description: "In-place AMI baking, 22 commits in 24 hours, and the safety cap that stopped us from deleting everything."
tags: [devops, github-actions, aws, ami, python]
---

In [Part 1](/2026/03/20/single-ec2-entire-ci.html) I talked about how we run our entire CI pipeline on a single EC2 instance with 4 concurrent runner processes. The secret ingredient was a custom AMI with our 23GB devcontainer image pre-baked into the Docker cache, eliminating the 6-8 minute image pull that was killing our build times.

What I didn't talk about was how we build that AMI. And honestly, this is where things got interesting.

We don't use Packer. We don't use EC2 Image Builder. We don't have a separate build server or a dedicated image pipeline. Instead, we bake our AMIs in-place — we take the running dev instance, patch it, update the cached Docker images, snapshot it, and roll forward. The whole thing is orchestrated by a single GitHub Actions workflow.

It works really well now. Getting here was a different story.

## Why Not Just Use Packer?

Packer is the standard answer to "how do I build AMIs" and for good reason — it's mature, well-documented, and handles a lot of the complexity for you. But when I looked at what we actually needed, Packer felt like it was solving a bigger problem than we had.

Our setup is a single EC2 instance running GitHub Actions runners. The AMI doesn't need to be built from scratch each time — it needs to be *updated*. New OS patches, a fresh pull of the devcontainer image, maybe a new version of the runner binary. The base doesn't change. The delta is small.

Building from scratch with Packer would mean spinning up a temporary instance, running a full provisioning script, waiting for a 23GB Docker image to download over the network, creating the AMI, and tearing it all down. That's 30+ minutes of build time and a bunch of infrastructure to maintain for something we only do once a week.

What if instead we just took the instance that's already running — the one that already has the 23GB image cached, the runner binary installed, all the dependencies in place — and turned *that* into the next AMI?

## The Bake-in-Place Approach

Here's what the pipeline actually does, step by step:

**1. Detach the instance from the ASG.** We lower the ASG min_size to 0 and detach the current instance. This is important because we need to stop the instance to create a clean snapshot, but we don't want the ASG to panic and launch a replacement while we're working.

**2. Patch via SSM.** Once the instance is detached, we run `AWS-RunPatchBaseline` through Systems Manager. This applies any pending OS security patches without needing SSH access. The instance is still running at this point, we're just using it as a build target.

**3. Pre-pull the latest devcontainer image.** This is the step that matters most for our use case. We send an SSM command that logs into GHCR, pulls the latest `devcontainer:stable` image, and verifies it landed correctly. Because the instance already has the previous version cached, Docker only downloads the changed layers. What would be a 23GB pull from scratch is usually a few hundred MB of deltas.

**4. Clean up runner state.** Before snapshotting, we need to remove anything that's specific to this particular instance — runner registration credentials, systemd service state, work directories, cloud-init caches. If any of this ends up in the AMI, the next instance that boots from it will have identity conflicts.

**5. Stop the instance and create the AMI.** We stop the instance (not terminate), create an image from it, tag the AMI and its snapshots, and wait for it to become available.

**6. Update the SSM parameter and roll forward.** The new AMI ID gets written to an SSM parameter. Then we run `terraform plan` and `terraform apply`, which picks up the new AMI ID and updates the launch template. The ASG launches a fresh instance from the new AMI.

**7. Smoke test.** We wait for the new instance to come up, verify via SSM that the runner service is healthy, and only then terminate the old detached instance.

The whole process takes about 15-20 minutes, most of which is waiting for the AMI to finish creating and the new instance to boot. The actual "work" — patching, pulling, cleaning — is maybe 5 minutes.

## The Day Everything Broke (22 Commits in 24 Hours)

The first version of this pipeline worked on the happy path. The problem was that AMI operations on AWS have a lot of unhappy paths, and I found most of them on November 6th, 2025.

I was adding AMI cleanup — the part of the pipeline that deletes old AMIs so they don't pile up and cost money. Simple enough in theory: list all AMIs matching a naming pattern, keep the newest ones, delete the rest along with their EBS snapshots.

The first version used inline bash in the GitHub Actions workflow, shelling out to the AWS CLI and piping through `jq` for JSON parsing. It worked locally. It did not work in CI.

Here's what went wrong, roughly in the order I discovered it:

**Two machines, two different toolsets.** The GitHub-hosted runner where the orchestration runs and the EC2 instance where the commands actually execute are two completely different environments with different tools installed. What's available on one isn't necessarily available on the other, and I was making assumptions about both.

**The AWS CLI `--query` parameter is unforgiving.** I had a filter that was supposed to exclude the newest AMI from the deletion list, but the sort order wasn't doing what I thought it was. The cleanup was keeping the wrong AMI and trying to delete the one it should have kept. These are the kinds of bugs you don't catch until you have enough data to exercise the edge case — in this case, more than 2 AMIs.

**Eventually-consistent API responses.** Sometimes `describe-images` returns an empty list briefly after a new AMI is created but before it's fully registered. My code assumed the list would always have at least one entry and crashed with an index error. Needed retry logic for empty state responses.

**Bash arithmetic will betray you.** I had a timeout calculation where operator precedence silently changed the meaning of the expression. The timeout was effectively infinite for the first few iterations and then immediately expired.

**Snapshot lifecycle quirks.** When you deregister an AMI, its EBS snapshots don't automatically get deleted — you have to delete them separately. But sometimes the snapshot is briefly "in use" even after the AMI is deregistered, and the delete call fails. More retry logic, more error handling.

**Inconsistent JSON output.** The AWS CLI's `--output json` flag formats things differently depending on whether there's one result or many. A single AMI ID comes back as a string, multiple come back as an array. The bash script was handling one case but not the other.

That was 22 commits in a single day. Every time I'd fix one issue, the pipeline would get a little further and hit the next one. By the end of the day I had a working cleanup process, but the bash was getting messy. I knew I'd need to rewrite it properly at some point, but I had other priorities and it was working — I'll talk more about that migration in [Part 4](/blog/shell-script-to-python-runtime) of this series.

## The Safety Cap That Saved Us

One of the better decisions I made during that November 6th marathon was adding a safety cap to AMI deletions. The cleanup takes a `max_delete_per_run` parameter, and if the number of AMIs to delete exceeds that limit, it refuses to run and throws an error instead.

I started with a cap of 3. Within a few weeks, normal operation pushed us past that — we had a stretch where the weekly bake ran but cleanup kept failing silently, so AMIs accumulated. Bumped it to 5 in December. Hit the limit again in January after another cleanup regression. Bumped it to 10.

The cap seems conservative, but it's there to prevent a bug in the AMI listing logic from accidentally deleting everything. If some API response comes back wrong and the "keep" list is empty, the cap ensures we delete at most N images instead of all of them. It's a safety net for exactly the kind of API edge cases that caused all those November problems.

The logic itself is simple: the AMI list comes in sorted by creation date, we keep the newest N, and we refuse to delete more than the safety cap. That's it. The simplicity is the point — when something guards against accidental deletion, you want it to be easy to reason about.

## Dev/Prod Promotion: The Full Pipeline

The bake pipeline only touches the dev environment. It creates the AMI from the dev instance, writes the new AMI ID to an SSM parameter, and rolls the dev ASG forward. Prod is a completely separate concern.

Getting the new AMI into production used to be a manual step — go to the GitHub Actions UI, pick the `promote_ami` action, trigger it. That worked fine, but it meant someone had to remember to do it. Now the whole thing is automated as part of the weekly scheduled run.

Every Sunday, the pipeline runs end-to-end: bake the dev AMI, promote the AMI ID from the dev parameter to the prod parameter, run `terraform apply` against prod to pick up the new launch template, roll any outdated runner instances, and send a Slack notification with the result. If any step fails, the pipeline stops and Slack gets a failure notification instead. The manual promote is still there in the prod workflow as a fallback if something needs to jump the queue.

The promotion itself is deliberately boring — it reads one SSM parameter and writes the value to another. But the pipeline around it means dev always bakes first, and if the bake fails, prod doesn't get touched. Running in dev for a week before promoting to prod has caught real problems that the smoke test didn't.

## What's Next

This post covered the AMI pipeline — how we build it, what broke along the way, and how we promote changes safely from dev to prod. There's more to say about the bash-to-Python migration specifically: the runner bootstrap went through the same journey, and that rewrite touched a lot more than just the AMI cleanup code. I'll cover that properly in [Part 4](/blog/shell-script-to-python-runtime).

But first — there's a question I've been dodging: how does the system know when to scale? We don't use webhooks. We don't use GitHub's built-in autoscaling. We have a Lambda function that runs every minute, checks how many runners are busy, and publishes a CloudWatch metric. The ASG scales off that metric. It's about 170 lines of code and it's been more reliable than any webhook-based approach I've seen.

---

*This is **Part 2** of a series on building production-grade self-hosted GitHub Actions runners at a robotics startup.*

- ***Part 1:** [How a Single EC2 Instance Runs Our Entire CI Pipeline](/2026/03/20/single-ec2-entire-ci.html)*
- ***Part 2:** We Replaced Packer With a GitHub Actions Workflow (you are here)*
- ***Part 3:** Autoscaling GitHub Runners Without Webhooks — how a 170-line Lambda became our entire scaling layer*
- ***Part 4:** From Shell Script to Python Runtime — how the runner bootstrap evolved over 7 months*
- ***Part 5:** Safe Infrastructure Changes With a Team of One — dev/prod isolation and deployment gates without a platform team*

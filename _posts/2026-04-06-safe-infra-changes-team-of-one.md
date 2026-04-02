---
title: "Safe Infrastructure Changes With a Team of One"
date: 2026-04-06
description: "Dev/prod isolation, promotion gates, zero-downtime rollouts, and least-privilege IAM. How a solo DevOps engineer deploys infrastructure without breaking production."
tags: [devops, aws, terraform, infrastructure, security]
---

Throughout this series I've been describing a system — self-hosted GitHub Actions runners with custom AMIs, autoscaling, and a Python runtime — without really addressing the elephant in the room. There's one person managing all of this. Me.

There's no platform team. No infrastructure review board. No second pair of eyes on a Terraform apply. When I push a change to the runner infrastructure, I'm the one who wrote it, reviewed it, approved it, and deployed it. If something goes wrong in production, I'm the one who gets the Slack notification and I'm the one who fixes it.

This post is about the practices and guardrails that make that work without constant anxiety. None of this is groundbreaking — it's mostly just discipline and good defaults. But the specific combination matters, and I think it's useful for anyone else running infrastructure as a solo operator or a very small team.

## Separate Everything

The most important decision was treating dev and prod as completely separate environments from the start, not just logically but in every layer of the stack.

Terraform workspaces keep the state files isolated. The dev workspace and the prod workspace each have their own state, their own resource naming, and their own configuration. A `terraform destroy` in dev can't touch prod resources because they don't exist in the same state file.

Each environment has its own tracked tfvars file with explicit configuration. Dev uses a `t3.xlarge` with a 100GB root volume. Prod uses an `m7i.xlarge` with the default 300GB. Dev runners have the label `sunrise-dev`. Prod runners have the label `sunrise`. This means a dev job physically cannot be picked up by a prod runner, and vice versa. The labels are the isolation boundary at the GitHub level, and the Terraform workspaces are the isolation boundary at the AWS level.

Even the AMIs are separated. Dev bakes from the dev instance and writes to a dev SSM parameter. Prod reads from a different SSM parameter that only gets updated through an explicit promotion step. There is no path where a dev AMI accidentally ends up in production without someone intentionally promoting it.

I also have a third Terraform workspace — `ci-iam` — that owns the account-global resources like the GitHub OIDC provider and the CI IAM role. These are resources that both dev and prod depend on, but that shouldn't live in either workspace. Having them in their own workspace means I can update IAM policies without touching runner infrastructure, and I never risk accidentally duplicating the OIDC provider across environments.

## No Long-Lived Credentials

The CI pipeline authenticates to AWS using GitHub's OIDC integration. There are no AWS access keys stored in GitHub secrets. The GitHub Actions runner assumes an IAM role via OIDC, and the trust policy is scoped to this specific repository.

This matters for a solo operator because credential rotation is one of those tasks that's easy to forget about and catastrophic to get wrong. With OIDC, there's nothing to rotate. The trust relationship is between GitHub's identity provider and the AWS IAM role, and it's verified on every run.

The GitHub PAT that the runners use to register with GitHub is stored in SSM Parameter Store as a SecureString. It's the one long-lived credential in the system, and it's the only thing I'd need to rotate manually. Everything else is ephemeral.

## The Promotion Gate

The weekly pipeline runs end-to-end on Sunday: bake the dev AMI, promote it to prod, apply Terraform against prod, roll the runner instances, and notify Slack. But the key design choice is that each stage is a separate job with an explicit dependency on the previous one succeeding.

If the dev bake fails, promotion doesn't happen. If promotion fails, prod apply doesn't happen. If the apply fails, the runners don't get rolled. At no point can a downstream stage execute if an upstream stage failed. This sounds obvious, but it's easy to get wrong with GitHub Actions — the default behaviour for dependent jobs is to run unless you explicitly check the outcome of the dependency.

The manual fallback is there too. If something needs to go to prod outside the Sunday cycle, I can trigger the `promote_ami` action from the prod workflow UI, then run an apply. But the default path is the automated pipeline, which means prod changes are consistent, predictable, and always preceded by a week of dev validation.

This has caught real issues. There was a week where the dev bake succeeded but the AMI had a subtle problem with the LFS cleanup that only showed up under specific repository configurations. Because dev was running that AMI for a week before promotion, we caught it during normal dev CI usage. If I'd been baking directly into prod, that would have been a production incident.

## Zero-Downtime Rollout

Not every change can be deployed through AMI promotion. When the runner bootstrap code changes — the Python runtime that gets embedded in the EC2 user-data — the fix lives in the launch template. An AMI promotion won't pick it up because the existing instances are already running. You need to replace the instances.

The rollout module handles this automatically after a prod apply. It surges the ASG by one instance, waits for the new instance to be fully ready — InService in the ASG, SSM agent online, runner services active in systemd, and registered with GitHub — and only then drains and terminates the old instance. If the drain fails for any reason, it leaves the extra instance running. Availability over perfect cleanup.

This was one of the later additions to the system. Before this, a bootstrap code change in prod meant manually terminating the instance and waiting for the ASG to replace it. That's a few minutes of downtime where no runner is available, which is fine at 2am but not great if someone triggers it during working hours. The automated rollout means I can merge to main, trigger a prod apply, and the instance replacement happens safely without me babysitting it.

## Least-Privilege IAM

The CI IAM role has three managed policies, each scoped to exactly what it needs. The S3 backend policy can read and write the Terraform state bucket. The AMI cleanup policy can deregister and delete snapshots, but only for resources tagged with the dev environment — it physically cannot delete a prod AMI. The workload policy covers EC2, ASG, Lambda, and the other resources the runner infrastructure needs, with ARN patterns scoped to the project's naming convention.

I won't pretend this was easy to set up. The initial deployment involved a lot of "apply, see what fails, add the missing permission, apply again." And the policies had to be refactored when they started hitting the AWS managed policy size limit. But the result is a CI role that can only affect the resources it's supposed to affect, which is important when that CI role is the primary deployment mechanism and there's nobody reviewing the applies in real time.

One specific decision that paid off: the AMI cleanup policy is explicitly restricted to dev-tagged resources. The safety cap in the cleanup code (from [Part 2](/2026/03/26/ami-baking-pipeline.html)) is the application-level guard against deleting too many AMIs. The IAM policy is the infrastructure-level guard against deleting the wrong ones. Two layers of protection for the same destructive operation.

## Reusable Workflows

The CI logic is structured as a shared workflow that both dev and prod call with different inputs. The shared workflow handles init, format checking, validation, planning, and apply. The environment-specific workflows handle the inputs — which workspace, which var file, which actions are available.

This means there's exactly one place where the Terraform plan/apply logic lives. If I fix a CI bug, both environments get the fix. If I add a new validation step, both environments get the step. The alternative — copying workflow YAML between dev and prod files — is how you end up with environments that slowly drift apart until something works in dev and breaks in prod for reasons that take an hour to debug.

## Slack Notifications

The Sunday pipeline sends a Slack notification when it completes, regardless of whether it succeeded or failed. The message includes the workflow name, the repository, the workspace, and a link to the run.

This is the simplest form of observability, and for a solo operator it's one of the most important. If the Sunday bake fails, I find out in Slack on Sunday evening, not on Monday morning when someone complains that their build is slow because the AMI is a week out of date. If it succeeds, I get a quiet confirmation that the system is healthy and I don't need to think about it.

The notifications only fire on scheduled runs. Manual runs don't notify, because if I'm running something manually I'm already watching it.

## What Makes It Work

If I had to summarize what makes solo infrastructure management work, it would be this: make the safe path the easy path.

Promoting to prod should be easier through the pipeline than around it. Rolling instances should happen automatically, not manually. Destructive operations should have safety caps. Credentials should be ephemeral. Environments should be isolated by default, not by convention.

None of these are novel ideas. But the difference between knowing them and implementing them is the difference between a system you're confident in and a system you're afraid to touch. Building this infrastructure over the past 7 months — from a single bash script to the system described across these 5 posts — the thing I'm most satisfied with isn't any individual feature. It's that I can push a change on a Wednesday afternoon and not spend the rest of the day worrying about whether I just broke CI for the whole team.

---

*This is **Part 5** of a series on building production-grade self-hosted GitHub Actions runners at a robotics startup.*

- ***Part 1:** [How a Single EC2 Instance Runs Our Entire CI Pipeline](/2026/03/20/single-ec2-entire-ci.html)*
- ***Part 2:** [We Replaced Packer With a GitHub Actions Workflow](/2026/03/26/ami-baking-pipeline.html)*
- ***Part 3:** [Autoscaling GitHub Runners Without Webhooks](/2026/04/02/autoscaling-without-webhooks.html)*
- ***Part 4:** [From Shell Script to Python Runtime](/2026/04/04/shell-script-to-python-runtime.html)*
- ***Part 5:** Safe Infrastructure Changes With a Team of One (you are here)*

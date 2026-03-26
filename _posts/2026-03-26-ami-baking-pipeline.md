---
title: "We Replaced Packer With a GitHub Actions Workflow"
date: 2026-03-26
description: "In-place AMI baking, 900 lines of Python, and the day 22 commits taught us to stop parsing JSON in bash."
tags: [devops, github-actions, aws, ami, python]
---

In [Part 1](/2026/03/20/single-ec2-entire-ci.html) I talked about how we run our entire CI pipeline on a single EC2 instance with 4 concurrent runner processes. The secret ingredient was a custom AMI with our 23GB devcontainer image pre-baked into the Docker cache, eliminating the 6-8 minute image pull that was killing our build times.

What I didn't talk about was how we build that AMI. And honestly, this is where things got interesting.

We don't use Packer. We don't use EC2 Image Builder. We don't have a separate build server or a dedicated image pipeline. Instead, we bake our AMIs in-place — we take the running production instance, patch it, update the cached Docker images, snapshot it, and roll forward. The whole thing is orchestrated by a single GitHub Actions workflow backed by about 900 lines of Python.

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

The first version used inline bash in the GitHub Actions workflow, shelling out to the AWS CLI and piping through `jq` for JSON parsing. It worked on my machine. It did not work in CI.

Here's what went wrong, roughly in the order I discovered it:

**jq wasn't installed on the runner.** The `ubuntu-latest` GitHub-hosted runner where the bake job runs has `jq`, but I was doing the JSON parsing *inside* an SSM command on the EC2 instance, which didn't have it. First commit of the day: add a jq availability check.

**The AMI query was wrong.** I was trying to exclude the latest AMI from the deletion list, but the `--query` filter had a logic error that excluded the *oldest* one instead. So the cleanup was keeping the oldest AMI and trying to delete the newest one. That's the kind of bug that only shows up when you have more than 2 AMIs.

**Empty responses from the AWS API.** Sometimes `describe-images` returns an empty list briefly after a new AMI is created but before it's fully registered. My code assumed the list would always have at least one entry and crashed with an index error. Added retry logic for empty state responses.

**Arithmetic precedence bug in the timeout calculation.** Bash arithmetic. I had something like `$TIMEOUT - $ELAPSED / $INTERVAL` when I needed `($TIMEOUT - $ELAPSED) / $INTERVAL`. The timeout was effectively infinite for the first few iterations and then immediately expired. This one took me longer to find than I'd like to admit.

**Snapshot deletion failures.** When you deregister an AMI, its EBS snapshots don't automatically get deleted — you have to delete them separately. But sometimes the snapshot is briefly "in use" even after the AMI is deregistered, and the delete call fails. Needed retry logic and error handling here too.

**JSON output parsing.** The AWS CLI's `--output json` flag formats things differently depending on whether there's one result or many. A single AMI ID comes back as a string, multiple come back as an array. The bash script was handling one case but not the other.

That was 22 commits in a single day. Every time I'd fix one issue, the pipeline would get a little further and hit the next one. By the end of the day I had a working cleanup process, but I'd learned a pretty clear lesson: shelling out to the AWS CLI and parsing JSON in bash is a terrible way to build something that needs to be reliable.

## The Safety Cap That Saved Us

One of the better decisions I made during that November 6th marathon was adding a safety cap to AMI deletions. The cleanup function takes a `max_delete_per_run` parameter, and if the number of AMIs to delete exceeds that limit, it refuses to run and throws an error instead.

I started with a cap of 3. Within a few weeks, normal operation pushed us past that — we had a stretch where the weekly bake ran but cleanup kept failing silently, so AMIs accumulated. Bumped it to 5 in December. Hit the limit again in January after another cleanup regression. Bumped it to 10.

The cap seems conservative, but it's there to prevent a bug in the AMI listing logic from accidentally deleting everything. If some API response comes back wrong and the "keep" list is empty, the cap ensures we delete at most N images instead of all of them. It's a safety net for exactly the kind of JSON parsing and API edge cases that caused all those November problems.

The cleanup logic itself ended up being surprisingly concise once I moved it to Python:

```python
def compute_cleanup_plan(image_ids, *, keep_count, max_delete_per_run):
    if len(image_ids) <= keep_count:
        return CleanupPlan(keep_ids=image_ids[:], delete_ids=[])

    delete_ids = image_ids[:-keep_count]
    if len(delete_ids) > max_delete_per_run:
        raise ValueError(
            f"Refusing to delete {len(delete_ids)} AMIs; "
            f"safety cap is {max_delete_per_run}"
        )
    return CleanupPlan(keep_ids=image_ids[-keep_count:], delete_ids=delete_ids)
```

That's the entire cleanup decision logic. The AMI list comes in sorted by creation date, we keep the newest N, and we refuse to delete more than the safety cap. Simple, testable, hard to get wrong. Compare that to the 22-commit bash version.

## From Bash to Python (and Why It Mattered)

After that November experience, I eventually rewrote the entire bake pipeline from inline bash in the workflow to a proper Python CLI module. The workflow now calls a single command:

```bash
python -m runner_ops.cli bake-dev-ami \
  --workspace dev \
  --var-file environments/dev/terraform.tfvars \
  --region eu-central-1 \
  --runner-pat-parameter /sunrise/github/cr_pat \
  --dev-ami-parameter /sunrise/ami/github-runner-dev \
  --cleanup-name-pattern 'gha-runner-dev-*' \
  --keep-count 1 \
  --max-delete-per-run 5 \
  --creation-max-attempts 120
```

Behind that single command is about 900 lines of Python across `ami_bake.py`, `ami_cleanup.py`, and `aws_wait.py`. It handles the full bake lifecycle: Terraform operations, instance detachment, SSM commands, patching, Docker image pre-pull, AMI creation and tagging, cleanup, rollout, and smoke testing. There's a generic polling utility that handles all the "wait for X to be ready" loops with configurable timeouts, success predicates, and terminal state detection.

The Python version has real error handling. If the bake fails at any step, the `finally` block restores the ASG min_size and terminates the detached instance so we don't leave dangling resources. If patching fails, it logs a warning but continues — patches are nice to have but shouldn't block a bake. If the new instance comes up on the wrong AMI (which can happen if the ASG hasn't picked up the launch template update yet), it recycles the instance and waits for a correct one, up to 3 attempts.

Most importantly, the Python version has unit tests. Every function that computes something — cleanup plans, shell command builders, tag verification logic — has tests. The bash version was untestable by definition.

## Dev/Prod Promotion: One-Way Gates

The bake pipeline only touches the dev environment. It writes the new AMI ID to `/sunrise/ami/github-runner-dev` in SSM Parameter Store. It does not touch prod. Ever.

Getting the new AMI into production is a separate, explicit action. Someone goes to the GitHub Actions UI, picks the `promote_ami` action from the dropdown, and runs it. That copies the dev AMI parameter value into the prod parameter. Then either the weekly scheduled apply picks it up on Sunday, or someone triggers a manual apply.

This is deliberately conservative. The bake pipeline does a lot of things — patches the OS, pulls new Docker images, cleans up state, creates a snapshot, rolls an ASG. Any of those steps could introduce a subtle issue that only shows up under load. Running in dev for a week before promoting to prod has caught real problems that the smoke test didn't.

The promotion is a one-liner in Python:

```python
def promote_ami_parameter(*, source_parameter, destination_parameter):
    image_id = get_ssm_parameter(source_parameter).strip()
    put_ssm_parameter(destination_parameter, image_id)
    return image_id
```

Nothing clever. It reads the dev AMI ID, writes it to the prod parameter. That's it. The simplicity is the point — the promotion step should be the most boring, predictable part of the whole pipeline.

## What I Learned

If I had to distill this whole experience down, it would be three things.

First, bash is fine for simple automation but terrible for anything that needs error handling. The moment you're parsing JSON, managing state, or handling partial failures, you need a real programming language. I should have started with Python.

Second, AWS APIs have a lot of eventually-consistent behaviour that doesn't show up in the docs. AMI state transitions, snapshot availability, the describe-images results being briefly empty after a create-image call — you need polling loops with retries and sensible timeouts everywhere.

Third, safety caps on destructive operations are worth their weight in gold. The AMI cleanup safety limit has prevented at least two incidents where a bug in the listing logic would have deleted AMIs we needed. It costs nothing to implement and it saves you from the worst-case scenario.

The pipeline has been running weekly since August 2025, with 68 commits touching the AMI baking code over that time. It's stable now, but the git history is a pretty honest record of how much iteration it took to get there.

---

*This is **Part 2** of a series on building production-grade self-hosted GitHub Actions runners at a robotics startup.*

- ***Part 1:** [How a Single EC2 Instance Runs Our Entire CI Pipeline](/2026/03/20/single-ec2-entire-ci.html)*
- ***Part 2:** We Replaced Packer With a GitHub Actions Workflow (you are here)*
- ***Part 3:** Autoscaling GitHub Runners Without Webhooks — how a 170-line Lambda became our entire scaling layer*
- ***Part 4:** From Shell Script to Python Runtime — how the runner bootstrap evolved over 7 months*
- ***Part 5:** Safe Infrastructure Changes With a Team of One — dev/prod isolation and deployment gates without a platform team*

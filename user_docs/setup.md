---
type: UserGuide
title: PL8 Setup
description: Deploy PL8 to your own AWS account and install the CLI
generated: { by: agent:claude-opus-5, at: 2026-09-22T00:00:00Z }
---

# Setup

PL8 is BYOC (bring your own cloud): you deploy it into an AWS account you
control. This guide takes you from an empty account to a working `pl8`
command.

It has three parts:
1. [Deploy the backend](#1-deploy-the-backend) from
   [pl8-services](https://github.com/dchenstealth/pl8-services) with OpenTofu.
2. [Grant access](#2-grant-access) to the people and agents who will use it.
3. [Install the CLI](#3-install-the-cli) and point it at your deployment.

## Prerequisites

* An AWS account, and credentials that can create DynamoDB, Lambda, IAM,
  SQS, EventBridge, CloudWatch and S3 resources
* [OpenTofu](https://opentofu.org/docs/intro/install/)
* [uv](https://docs.astral.sh/uv/getting-started/installation/)
* The [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
  and [jq](https://jqlang.org/), for the smoke test
* An S3 bucket for OpenTofu state. Any bucket you can write to works.

### Choose an environment name

Every resource PL8 creates is prefixed with an environment name, for example
`prod` or `dev`. The CLI uses the same name to find your deployment. You can
run several environments side by side in one account. This guide uses `prod`.

## 1. Deploy the backend

### Clone pl8-services

```bash
git clone https://github.com/dchenstealth/pl8-services.git
cd pl8-services
```

### Configure

pl8-services reads its configuration from three files. Two are specific to
your account, so they live outside the repo. This guide keeps them in
`~/pl8-config/`:

```bash
mkdir -p ~/pl8-config
```

**Backend** (`~/pl8-config/prod.tfbackend`): where OpenTofu keeps its state.

```hcl
bucket       = "<your-tfstate-bucket>"
region       = "us-east-1"
use_lockfile = true
```

**Account variables** (`~/pl8-config/prod.tfvars`): the region to deploy to
and the environment name.

```hcl
aws_region  = "us-east-1"
environment = "prod"
```

**Service variables** (`infra/tfvars/prod.tfvars`): sizing for the table,
queues and Lambdas. Start from the defaults that ship with the repo:

```bash
cp infra/tfvars/dev.tfvars infra/tfvars/prod.tfvars
```

The defaults suit a small team. Settings you may want to change:

| Variable | Default | Notes |
| --- | --- | --- |
| `pl8_table_on_demand_read_request_units`, `pl8_table_on_demand_write_request_units` | `1000` | Caps table throughput, which also caps cost |
| `*_log_retention_days` | `14` | CloudWatch Logs retention for each Lambda |
| `alarm_actions` | `[]` | SNS topic ARNs to notify when an event fails (see [Monitoring](#monitoring)) |

See [`infra/variables.tf`](https://github.com/dchenstealth/pl8-services/blob/main/infra/variables.tf)
for every variable.

### Build and apply

Authenticate to your AWS account first, for example with `aws login` or by
setting `AWS_PROFILE`. Then build the shared Lambda dependency layer. The
layer must be built before planning, because OpenTofu packages it at plan
time.

```bash
src/build-layer.sh
```

Then plan and apply:

```bash
cd infra
tofu init -backend-config ~/pl8-config/prod.tfbackend
tofu plan \
  -var-file tfvars/prod.tfvars \
  -var-file ~/pl8-config/prod.tfvars \
  -out plan.tfplan
tofu apply plan.tfplan
cd ..
```

This creates:

* A DynamoDB table, `prod-pl8-table`, which holds all of your PL8 data
* Three Lambdas:
  * `prod-pl8-interface`, which the CLI invokes
  * `prod-pl8-stream-handler` and `prod-pl8-event-handler`, which apply
    lifecycle updates in the background
* An EventBridge bus, `prod-pl8-events`, plus SQS queues with dead-letter
  queues and CloudWatch alarms on those dead-letter queues

### Verify

The smoke test runs PL8's full lifecycle against your deployment. It creates
a temporary Space, checks that finishing or deleting a blocking Issue
unblocks the Issue it blocks, checks that nothing was dead-lettered, and then
cleans up whether it passes or fails.

```bash
scripts/smoke-test.sh prod
```

The last line should be `PASS`. Run it with the same admin credentials you
deployed with, since it also reads the dead-letter queues.

## 2. Grant access

PL8 has no user accounts of its own. Access is controlled entirely by IAM:
anyone with `lambda:InvokeFunction` on `<environment>-pl8-interface` has
full read and write access to your PL8 data. Attach a policy like this to
the IAM users or roles for the people and agents who should use PL8:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UsePL8",
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:<account-id>:function:prod-pl8-interface"
    }
  ]
}
```

The interface function is also tagged `Type=PL8Interface`. If you run
several environments, you can grant access to all of them by matching the
tag instead of listing each function:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "UseAllPL8Environments",
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "*",
      "Condition": {
        "StringEquals": { "aws:ResourceTag/Type": "PL8Interface" }
      }
    }
  ]
}
```

Grant agents their own role with only this permission. They don't need
access to DynamoDB or anything else PL8 creates.

## 3. Install the CLI

The CLI is published to PyPI as [`pl8-cli`](https://pypi.org/project/pl8-cli/)
and needs Python 3.11 or later. uv will download a suitable Python if you
don't have one.

Install the `pl8` command:

```bash
uv tool install pl8-cli
```

Or run it without installing:

```bash
uvx pl8-cli --help
```

Point it at your deployment. `PL8_ENV` names the environment, and the CLI
invokes `<PL8_ENV>-pl8-interface`:

```bash
export PL8_ENV=prod
pl8 space list
```

A new deployment returns `{"ok": true, "data": {"items": [], "cursor": null}}`.

| Setting | Flag | Environment variable |
| --- | --- | --- |
| Environment (invokes `<ENV>-pl8-interface`) | `--env ENV` | `PL8_ENV` |
| Function name or ARN, instead of an environment | `--function-name NAME` | `PL8_FUNCTION_NAME` |
| Default Space for bare Issue ids | `--space SPACE` | `PL8_SPACE` |
| Creator recorded on what you create | `--creator WHO` | `PL8_CREATOR` |
| AWS credentials and region | `--profile`, `--region` | Standard AWS chain (`AWS_PROFILE`, `AWS_REGION`, `~/.aws/config`, ...) |

Flags take precedence over environment variables, and a function name takes
precedence over an environment. Use `--function-name` with a full ARN to
reach a deployment in another account or region.

Next: [Usage](usage.md).

## Monitoring

Some PL8 work runs in the background, such as moving an Issue from BLOCKED
to TODO when its blockers finish. If a background step keeps failing, its
event ends up in one of two dead-letter queues:

* `<environment>-pl8-stream-handler-dlq`
* `<environment>-pl8-event-handler-dlq`

A message in either queue means an Issue may be stuck, for example BLOCKED
with no active blockers. A CloudWatch alarm watches each queue. Set
`alarm_actions` to one or more SNS topic ARNs to be notified when an alarm
fires. Check the Lambdas' CloudWatch logs to find the cause. Once it's
fixed, redrive the messages.

## Upgrading

Pull the latest pl8-services, rebuild the layer, and plan and apply again:

```bash
git pull
src/build-layer.sh
cd infra
tofu plan \
  -var-file tfvars/prod.tfvars \
  -var-file ~/pl8-config/prod.tfvars \
  -out plan.tfplan
tofu apply plan.tfplan
```

Only Lambdas whose code changed are redeployed. Upgrade the CLI with
`uv tool upgrade pl8-cli`.

## Removing PL8

**This permanently deletes all of your PL8 data.** The table has deletion
protection enabled, so turn it off first:

```bash
aws dynamodb update-table --table-name prod-pl8-table --no-deletion-protection-enabled
cd infra
tofu destroy \
  -var-file tfvars/prod.tfvars \
  -var-file ~/pl8-config/prod.tfvars
```

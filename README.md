# PL8

PL8 is a lightweight issue tracker built for AI agents and the people working
alongside them. Issues live in Spaces, can block one another, and move
through a small, strict status lifecycle (`TODO`, `BLOCKED`, `IN_PROGRESS`,
`DONE`). When an Issue's blockers are all finished, PL8 moves it back to
`TODO` for you and emits an event, so agents can pick up work as it becomes
ready.

PL8 has no web UI. You use it through [`pl8-cli`](https://github.com/dchenstealth/pl8-cli),
which prints one JSON document per command and signals failures with
well-defined exit codes, so agents can drive it without scraping text.

> **Status:** alpha (0.0.x). Interfaces may change between releases.

## Bring your own cloud

PL8 is **BYOC (bring your own cloud)**. There is no hosted PL8 service to sign
up for. You deploy PL8 into your own AWS account, and:

* **Your data stays in your account.** Issues are stored in a DynamoDB table
  you own. Nothing is sent to a third party.
* **Access is plain AWS IAM.** There are no separate PL8 users or API keys.
  Anyone (or any agent) with `lambda:InvokeFunction` on your PL8 function can
  use it.
* **You pay AWS directly.** PL8 is fully serverless (DynamoDB on-demand,
  Lambda, SQS, EventBridge), so costs scale with usage and are close to zero
  when idle.
* **You can extend it.** PL8 publishes lifecycle events to an EventBridge bus
  in your account, which you can route to your own workflows.

## Getting started

See the [user docs](user_docs/):

1. [Setup](user_docs/setup.md): deploy PL8 to your AWS account and install the CLI
2. [Usage](user_docs/usage.md): Spaces, Issues, blockers, and automating with agents

## Repositories

| Repo | What it is |
| --- | --- |
| [pl8-docs](https://github.com/dchenstealth/pl8-docs) | This repo: user docs and architecture docs |
| [pl8-services](https://github.com/dchenstealth/pl8-services) | OpenTofu infrastructure and Lambda code you deploy |
| [pl8-cli](https://github.com/dchenstealth/pl8-cli) | The `pl8` command line ([PyPI](https://pypi.org/project/pl8-cli/)) |
| [pl8-base](https://github.com/dchenstealth/pl8-base) | Python data layer and source of truth for the data model ([PyPI](https://pypi.org/project/pl8-base/)) |

## About this repo

This repo follows the [OKF format](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md).

### Content
* User docs ([user_docs/](user_docs/)): setup and usage for people running PL8
* Architecture ([architecture/](architecture/)): how PL8 is built
  * Loosely follows [C4 model](https://c4model.com/abstractions):
    * System: Highest level abstraction, analogous to "application" or "product"
    * Container: Sub-system that is part of the product. Examples: "Web app", "Backend"
    * Component: Logical groupings within a container. Example: set of AWS lambda functions
    * Code: Not tracked in this repo (code-level docs belong in the repos)

## License

[MIT](LICENSE)

# PL8 Docs

Documentation repo for PL8, a lightweight issue tracker backed by DynamoDB.

Follows [OKF format](https://github.com/GoogleCloudPlatform/knowledge-catalog/blob/main/okf/SPEC.md)

## Content
* Architecture
  * Loosely follows [C4 model](https://c4model.com/abstractions):
    * System: Highest level abstraction, analogous to "application" or "product"
    * Container: Sub-system that is part of the product. Examples: "Web app", "Backend"
    * Component: Logical groupings within a container. Example: set of AWS lambda functions
    * Code: Not tracked in this repo (code-level docs belong in the repos)

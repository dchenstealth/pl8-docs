---
type: ComponentOverview
title: PL8 Backend Overview
description: Component overview of PL8 backend
generated: { by: agent:claude-opus-5, at: 2026-09-26T00:00:00Z }
---

# Backend overview
Repos:
* pl8-base: data layer, source of truth
* pl8-services: IaC + lambda code

## pl8-interface lambda
Main interface for agents, invoked via CLI (see [cli](../cli/)).

## pl8-stream-handler lambda
DynamoDB stream handler, sends events on entity changes.
Events go to eventbus -> SQS -> pl8-event-handler lambda.
See [Events](events.md).

## pl8-event-handler lambda
Applies async entity lifecycle updates.

## Stores
DynamoDB holds every entity and is the source of truth.
One S3 bucket holds IssueAttachment objects; the rows that describe them stay in
DynamoDB. pl8-interface signs the URLs callers upload through, and
pl8-event-handler deletes objects when their rows go away.
See [Storage](storage.md).

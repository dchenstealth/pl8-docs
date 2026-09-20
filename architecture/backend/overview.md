---
type: ComponentOverview
title: PL8 Backend Overview
description: Component overview of PL8 backend
generated: { by: human:dchen, at: 2026-09-19T00:00:00Z }
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

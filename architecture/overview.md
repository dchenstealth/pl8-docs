---
type: SystemOverview
title: PL8 System Overview
description: System overview of PL8, a lightweight issue tracking system
generated: { by: human:dchen, at: 2026-09-08T00:00:00Z }
---

# System Overview
PL8 is a lightweight issue tracking system backed by DynamoDB.

## Architecture
PL8 is hosted in AWS, backed by DynamoDB, EventBridge, SQS, and AWS Lambda.

## Events
PL8 sends EventBridge events on specific entity lifecycle changes.
These events are intended both to trigger internal updates *and* allow consumers
to extend the service by triggering custom workflows. See [backend](backend/)
for details.

## Interface
PL8 has *no UI* and is intended to be accessed through the [CLI](cli/),
authenticated through AWS IAM.

---
type: ComponentOverview
title: PL8 CLI Overview
description: Component overview of PL8 CLI
generated: { by: agent:claude-opus-5, at: 2026-09-27T00:00:00Z }
---

# CLI overview
One repo: pl8-cli

## Purpose
Agent-friendly wrapper to invoke pl8-interface lambda (see [backend](../backend/)).

## Command surface
Subcommands are grouped by entity: `space`, `issue`, `comment`, `attachment`,
`blocker`. Each wraps one interface operation. `pl8 invoke` sends any operation
by name, so nothing pl8-interface exposes is unreachable and nothing the CLI
exposes bypasses it. The reference is in the
[user docs](../../user_docs/usage.md#commands).

## Client-side work
Three commands do more than wrap an operation, each for a reason that keeps it
off the backend:

* `attachment add` drives all three steps of an upload and sends the bytes to S3
  itself, since they never pass through a Lambda (see
  [storage](../backend/storage.md)).
* `attachment get` fetches through a presigned URL and writes it to a path via a
  temp file and a rename, so a failed download leaves no truncated file. It
  never prints the URL, which is a five-minute bearer token.
* `comment wait` polls on an interval, because a Lambda that blocked would bill
  wall clock for waiting.

## Output contract
One JSON document per command on stdout, and an exit code saying whether a retry
is safe: reads are retried on transient faults, writes never are, since a write
that timed out may already have been applied. The codes are in the
[user docs](../../user_docs/usage.md#output-and-exit-codes).

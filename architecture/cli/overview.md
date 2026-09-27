---
type: ComponentOverview
title: PL8 CLI Overview
description: Component overview of PL8 CLI
generated: { by: agent:claude-opus-5, at: 2026-09-26T00:00:00Z }
---

# CLI overview
One repo: pl8-cli

## Purpose
Agent-friendly wrapper to invoke pl8-interface lambda (see [backend](../backend/)).

## Command surface
Subcommands are grouped by entity: `space`, `issue`, `comment`, `attachment`,
`blocker`. Each is a thin wrapper over one interface operation, plus
`pl8 invoke`, which sends any operation by name for the ones that have no
subcommand yet. Nothing the CLI exposes is unreachable through `pl8 invoke`, and
nothing it exposes bypasses pl8-interface. The full reference, with flags and
the rules each command enforces, is in the
[user docs](../../user_docs/usage.md#commands).

## What the CLI does that the backend does not
Three commands do real work on the client, and each is on the client for a
reason.

`attachment add` is one command over the three steps of an upload: it asks
pl8-interface to reserve the row and sign a target, sends the file to S3 itself,
then asks pl8-interface to confirm. The bytes never pass through a Lambda, so
the CLI is the only thing that sees the whole file. It takes the size from the
file rather than from the caller, because the size is what the upload is signed
for (see [storage](../backend/storage.md)).

`attachment get` fetches the object through a presigned URL and writes it to the
path the caller named, via a temp file in the same directory and a rename, so a
failed download cannot leave a truncated file where a whole one is expected. The
URL is what it keys off: pl8-interface returns one only for an UPLOADED
attachment (see [storage](../backend/storage.md)), so a response with no
`download_url` is the CLI's signal to refuse rather than a status it has to
interpret. It never prints the URL. Stdout is the command's JSON envelope, and
that URL is a five-minute bearer token for the object: printing it would put a
live credential into scrollback, CI logs and agent transcripts.

`comment wait` polls pl8-interface on an interval until new comments appear or
the caller's deadline passes. It polls from the client because a Lambda that
blocked would bill wall clock for waiting, and it treats an empty result as
success so a caller can loop on it.

## Output contract
One JSON document per command on stdout, and an exit code that says whether a
retry is safe. The CLI retries reads on transient faults and never retries
writes, since a write that timed out may already have been applied. The codes
are documented in the
[user docs](../../user_docs/usage.md#output-and-exit-codes).

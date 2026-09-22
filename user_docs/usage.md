---
type: UserGuide
title: PL8 Usage
description: Use PL8 from the CLI, from agents, and through its events
generated: { by: agent:claude-opus-5, at: 2026-09-22T00:00:00Z }
---

# Usage

This guide assumes you have [set up](setup.md) PL8 and can run
`pl8 space list`.

## Concepts

* **Space**: a named group of Issues, such as a team or project. You choose
  its id: 1-64 characters from `[A-Za-z0-9_-]`, for example `ENG`.
* **Issue**: a unit of work with a title, description and status. PL8
  generates its id. Every Issue belongs to a Space, and you refer to it as
  `SPACE/ISSUE_ID`, for example `ENG/abc123`.
* **Status**: one of `TODO`, `BLOCKED`, `IN_PROGRESS` or `DONE`. `DONE` is
  final.
* **Blocker**: "Issue A blocks Issue B". Blockers can cross Spaces. While B
  has unfinished blockers it stays `BLOCKED`, and PL8 moves it back to
  `TODO` when they are all finished.

The full rules are in [Entities](../architecture/entities.md).

## A quick tour

Set your environment and a default Space, so you can use bare Issue ids:

```bash
export PL8_ENV=prod PL8_SPACE=ENG
```

Create the Space, then add Issues to it:

```bash
pl8 space create ENG --name "Engineering" --description "Engineering work"

schema=$(pl8 issue create --title "Design the schema" --description "..." | jq -r .data.issue_id)
api=$(pl8 issue create --title "Build the API" --description-file api.md | jq -r .data.issue_id)
```

The API can't start until the schema is designed:

```bash
pl8 blocker add --blocking "$schema" --blocked "$api"
pl8 issue get "$api"        # status is now BLOCKED
```

Work on the schema, then finish it:

```bash
pl8 issue transition "$schema" --status IN_PROGRESS
pl8 issue transition "$schema" --status DONE
```

A few seconds later, PL8 moves the API Issue back to `TODO` on its own:

```bash
pl8 issue list --status TODO
```

## Commands

Every command has `--help`, which lists its options and the rules it
enforces.

```
pl8 space create SPACE_ID --name NAME --description TEXT
pl8 space get SPACE_ID
pl8 space list
pl8 space update SPACE_ID --name NAME --description TEXT [--if-version N]
pl8 space delete SPACE_ID

pl8 issue create --title TITLE --description TEXT [--status STATUS]
pl8 issue get ISSUE
pl8 issue list --status STATUS
pl8 issue update ISSUE --title TITLE --description TEXT [--if-version N]
pl8 issue transition ISSUE --status STATUS [--if-version N]
pl8 issue delete ISSUE

pl8 blocker add --blocking ISSUE --blocked ISSUE
pl8 blocker remove --blocking ISSUE --blocked ISSUE
pl8 blocker list (--blocked ISSUE | --blocking ISSUE)

pl8 invoke OPERATION [--params JSON | --params-file PATH]
```

* **Issue references.** `ISSUE` is `SPACE/ISSUE_ID`, or a bare `ISSUE_ID`
  that takes its Space from `--space` or `PL8_SPACE`. A Space written in the
  reference always wins, so blockers across Spaces need nothing extra:
  `pl8 blocker add --blocking OPS/k8s123 --blocked abc456`.
* **Long text.** Use `--description-file PATH` in place of `--description`
  to read a description from a file, or `--description-file -` to read it
  from stdin. This avoids shell quoting problems.
* **Lists.** Every list returns `{"items": [...], "cursor": ...}`. Pass the
  cursor back with `--cursor` to get the next page, set the page size with
  `--limit` (1-100, default 50), or use `--all` to fetch every page.
  `pl8 issue list` returns the Issues in one status, with the Issue that has
  been in that status longest first.
* **Updates.** `update` replaces both the name or title and the description,
  so pass both.
* **Raw operations.** `pl8 invoke` sends any operation with JSON params,
  for operations that don't have a subcommand yet.

### Rules to know

* Create a Space before creating Issues in it.
* `DONE` is final: an Issue can't move from `DONE` to any other status.
* An Issue with unfinished blockers can't leave `BLOCKED`. To unblock it,
  finish or delete its blockers, or remove them with `pl8 blocker remove`.
* An Issue that is `DONE` can't be added as a blocker, and an Issue can't
  block itself.
* Delete a Space's Issues before deleting the Space.
* Blocking cycles (A blocks B, and B blocks A) are allowed but deadlock both
  Issues. Remove one of the blockers to break the cycle.

### Background updates

Some changes happen in the background, a few seconds after the command that
caused them:

* When a blocking Issue becomes `DONE` or is deleted, the Issues it blocked
  return to `TODO` once they have no other unfinished blockers.
* When an Issue is deleted, the blockers that name it are removed.

If a background change never happens, see
[Monitoring](setup.md#monitoring).

### Concurrent edits

Each Space and Issue has a `version` that increases on every write. To
avoid overwriting someone else's change, pass the version you last read:

```bash
pl8 issue update ENG/abc123 --title "..." --description "..." --if-version 3
```

If the item has changed since then, the write fails with
`DDBVersionConflictError`. Re-read the item and try again.

## Output and exit codes

Every command prints exactly one JSON document to stdout. Add `--pretty` to
indent it.

```json
{"ok": true, "data": {"issue_id": "abc123", "status": "TODO", ...}}
{"ok": false, "error": {"type": "DDBStillBlockedError", "message": "..."}}
```

The exit code tells you what kind of failure happened and whether retrying
is safe:

| Exit | Meaning | What to do |
| --- | --- | --- |
| 0 | Success | |
| 1 | PL8 rejected the request, for example because an item is missing, a version is stale, or a rule was broken | Fix the request. Retrying it unchanged won't help. |
| 2 | The command line was invalid. Nothing was sent. | Fix the command. |
| 3 | No reliable answer, because of a network failure or a server fault | A write may or may not have been applied. Read the item again before retrying. |
| 4 | A temporary conflict that PL8 already retried. Nothing was applied. | Retry the same request. |

The CLI retries reads on transient errors. It never retries writes
automatically, because a write that timed out may already have been
applied.

## Using PL8 with agents

The CLI was designed for agents: output is always JSON, failures have a
stable `error.type`, and the exit code says whether a retry is safe. To give
an agent access to PL8:

1. Give the agent AWS credentials with only `lambda:InvokeFunction` on your
   interface function (see [Grant access](setup.md#2-grant-access)).
2. Set `PL8_ENV`, and `PL8_SPACE` if the agent works in one Space, in the
   agent's environment.
3. Tell the agent to use `pl8` and to read `pl8 --help` and
   `pl8 <command> <subcommand> --help` for details. For example:

   > Track your work in PL8 with the `pl8` CLI. Run `pl8 --help` to learn
   > the commands. Pick up Issues from `pl8 issue list --status TODO`, move
   > an Issue to `IN_PROGRESS` before you start it and to `DONE` when you
   > finish it, and record dependencies with `pl8 blocker add`.

## Reacting to events

PL8 publishes events to the EventBridge bus `<environment>-pl8-events` in
your account, with source `pl8`. You can add your own EventBridge rules to
that bus to trigger workflows, for example starting an agent whenever work
becomes ready.

The event intended for your workflows is `IssueReady`. PL8 sends it when an
Issue is created with status `TODO`, or when an Issue moves to `TODO`,
including when its last blocker finishes. Match it with this event pattern:

```json
{
  "source": ["pl8"],
  "detail-type": ["IssueReady"]
}
```

The event's `detail` names the Issue:

```json
{
  "type": "IssueReady",
  "type_version": "0.0.1",
  "event_id": "5f0c1c1e-...",
  "sent_at": "2026-09-22T12:00:00.000Z",
  "space_id": "ENG",
  "issue_id": "abc123"
}
```

Events are delivered at least once, so the same event can arrive more than
once. Make your handlers safe to run twice, for example by checking the
Issue's current status before acting on it.

PL8 uses its other events (`IssueDone`, `IssueDeleted`,
`IssueNumActiveBlockersZeroed`) internally. See [Events](../architecture/backend/events.md).

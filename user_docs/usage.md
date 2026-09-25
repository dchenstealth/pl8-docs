---
type: UserGuide
title: PL8 Usage
description: Use PL8 from the CLI, from agents, and through its events
generated: { by: agent:claude-opus-5, at: 2026-09-24T00:00:00Z }
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
* **Comment**: a note on an Issue, for discussion or for recording what an
  agent did. PL8 generates its id. Comments are listed oldest first, and are
  deleted along with the Issue they are on.
* **Creator**: a label naming who made a Space, Issue or Comment, which you
  set when you create it. PL8 records it and never checks it.

The full rules are in [Entities](../architecture/entities.md).

## A quick tour

Set your environment, a default Space so you can use bare Issue ids, and the
creator to record on what you create:

```bash
export PL8_ENV=prod PL8_SPACE=ENG PL8_CREATOR=alice
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
pl8 comment add "$schema" --body "Went with a single-table design."
pl8 issue transition "$schema" --status DONE
```

Anyone picking the work up can read the thread, oldest note first:

```bash
pl8 comment list "$schema"
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

pl8 comment add ISSUE --body TEXT
pl8 comment get ISSUE COMMENT_ID
pl8 comment list ISSUE
pl8 comment update ISSUE COMMENT_ID --body TEXT [--if-version N]
pl8 comment delete ISSUE COMMENT_ID

pl8 blocker add --blocking ISSUE --blocked ISSUE
pl8 blocker remove --blocking ISSUE --blocked ISSUE
pl8 blocker list (--blocked ISSUE | --blocking ISSUE)

pl8 invoke OPERATION [--params JSON | --params-file PATH]
```

* **Issue references.** `ISSUE` is `SPACE/ISSUE_ID`, or a bare `ISSUE_ID`
  that takes its Space from `--space` or `PL8_SPACE`. A Space written in the
  reference always wins, so blockers across Spaces need nothing extra:
  `pl8 blocker add --blocking OPS/k8s123 --blocked abc456`.
* **Comment references.** A comment is named by its Issue and then its
  `COMMENT_ID`, as two separate arguments rather than one `/`-joined
  reference: `pl8 comment delete ENG/abc123 0199f3a1-...`. The `ISSUE` part
  is an ordinary Issue reference, so it can be bare.
* **Creator.** `space create`, `issue create` and `comment add` record who
  created the item. Set `PL8_CREATOR` once in your environment, or pass
  `--creator WHO` on the command. PL8 stores the label and never checks it,
  so it says who *claims* to have created something; see
  [Rules to know](#rules-to-know).
* **Long text.** Use `--description-file PATH` in place of `--description`
  to read a description from a file, or `--description-file -` to read it
  from stdin. `comment add` and `comment update` take `--body-file` the same
  way. This avoids shell quoting problems.
* **Lists.** Every list returns `{"items": [...], "cursor": ...}`. Pass the
  cursor back with `--cursor` to get the next page, set the page size with
  `--limit` (1-100, default 50), or use `--all` to fetch every page.
  `pl8 issue list` returns the Issues in one status, with the Issue that has
  been in that status longest first. `pl8 comment list` returns one Issue's
  comments, oldest first.
* **Updates.** `update` replaces both the name or title and the description,
  so pass both. `comment update` replaces the body. An update never changes
  the creator.
* **Raw operations.** `pl8 invoke` sends any operation with JSON params,
  for operations that don't have a subcommand yet.

### Rules to know

* Create a Space before creating Issues in it.
* `DONE` is final: an Issue can't move from `DONE` to any other status.
* An Issue with unfinished blockers can't leave `BLOCKED`. To unblock it,
  finish or delete its blockers, or remove them with `pl8 blocker remove`.
* An Issue that is `DONE` can't be added as a blocker, and an Issue can't
  block itself.
* Delete a Space's Issues before deleting the Space. Comments are the other
  way round: deleting an Issue deletes its comments for you, however many it
  has.
* You can comment on an Issue in any status, including `DONE`. `DONE` stops
  an Issue moving to another status; it doesn't close the discussion.
* The creator is a label, not a login. PL8 doesn't verify it against your AWS
  identity, nothing stops two callers using the same one, and no command is
  allowed or refused on the basis of it. Use it to see who did what, not to
  control who may do what — for that, use IAM.
* Blocking cycles (A blocks B, and B blocks A) are allowed but deadlock both
  Issues. Remove one of the blockers to break the cycle.

### Background updates

Some changes happen in the background, a few seconds after the command that
caused them:

* When a blocking Issue becomes `DONE` or is deleted, the Issues it blocked
  return to `TODO` once they have no other unfinished blockers.
* When an Issue is deleted, the blockers that name it are removed, and so are
  its comments.

If a background change never happens, see
[Monitoring](setup.md#monitoring).

### Concurrent edits

Each Space, Issue and comment has a `version` that increases on every write.
To avoid overwriting someone else's change, pass the version you last read:

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
   agent's environment. Set `PL8_CREATOR` too, to a name for that agent: give
   each agent its own, and a thread of comments tells you which agent wrote
   what.
3. Tell the agent to use `pl8` and to read `pl8 --help` and
   `pl8 <command> <subcommand> --help` for details. For example:

   > Track your work in PL8 with the `pl8` CLI. Run `pl8 --help` to learn
   > the commands. Pick up Issues from `pl8 issue list --status TODO`, move
   > an Issue to `IN_PROGRESS` before you start it and to `DONE` when you
   > finish it, and record dependencies with `pl8 blocker add`. Leave what
   > you learned on the Issue with `pl8 comment add`, and read
   > `pl8 comment list` before starting work someone else has touched.

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

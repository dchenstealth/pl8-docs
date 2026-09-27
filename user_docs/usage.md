---
type: UserGuide
title: PL8 Usage
description: Use PL8 from the CLI, from agents, and through its events
generated: { by: agent:claude-opus-5, at: 2026-09-27T00:00:00Z }
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
  agent did. PL8 generates its id. Comments are listed oldest first unless you
  ask for `--desc`, and are deleted along with the Issue they are on.
* **Attachment**: a file on an Issue, up to 100MB, optionally tied to one of
  that Issue's comments. PL8 generates its id and keeps the file in an S3
  bucket in your account. Attachments are deleted along with the Issue they are
  on, and along with the comment they are tied to.
* **Creator**: a label naming who made a Space, Issue, Comment or Attachment,
  which you set when you create it. PL8 records it and never checks it.

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
note=$(pl8 comment add "$schema" --body "Went with a single-table design." | jq -r .data.comment_id)
pl8 attachment add "$schema" --file schema.png --comment "$note"
pl8 issue transition "$schema" --status DONE
```

Anyone picking the work up can read the thread, oldest note first, and pull
down what was attached to it:

```bash
pl8 comment list "$schema"
att=$(pl8 attachment list "$schema" | jq -r .data.items[0].attachment_id)
pl8 attachment get "$schema" "$att" --output schema.png
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
pl8 issue update ISSUE [--title TITLE] [--description TEXT] [--if-version N]
pl8 issue transition ISSUE --status STATUS [--if-version N]
pl8 issue delete ISSUE

pl8 comment add ISSUE --body TEXT
pl8 comment get ISSUE COMMENT_ID
pl8 comment list ISSUE [--desc]
pl8 comment update ISSUE COMMENT_ID --body TEXT [--if-version N]
pl8 comment delete ISSUE COMMENT_ID
pl8 comment wait ISSUE [--after COMMENT_ID] [--interval SECONDS] [--max-wait SECONDS]

pl8 attachment add ISSUE --file PATH [--comment COMMENT_ID] [--name NAME]
                         [--content-type TYPE]
pl8 attachment get ISSUE ATTACHMENT_ID --output PATH [--force]
pl8 attachment list ISSUE [--comment COMMENT_ID] [--desc]
pl8 attachment delete ISSUE ATTACHMENT_ID

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
* **Creator.** `space create`, `issue create`, `comment add` and
  `attachment add` record who created the item. Set `PL8_CREATOR` once in your
  environment, or pass `--creator WHO` on the command. PL8 stores the label and
  never checks it, so it says who *claims* to have created something; see
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
  comments and `pl8 attachment list` its attachments, or one comment's
  attachments with `--comment`. Both list oldest first, and both take `--desc`
  for newest first.
* **Updates.** `space update` replaces both the name and the description, so
  pass both. `issue update` replaces the title, the description, or both, and
  leaves whichever you omit unchanged. `comment update` replaces the body. An
  update never changes the creator.
* **Attachments.** `pl8 attachment add` is one command for what is really
  three steps: PL8 reserves the attachment and signs an upload, the CLI sends
  the file to S3, and PL8 marks it uploaded. The size is taken from the file
  and is never yours to declare, because it is what the upload is signed for;
  files over 100MB are refused. The name defaults to the file's name and the
  content type is guessed from it, unless you pass `--name` or
  `--content-type`. If a step after the first fails, the attachment already
  exists, and the error envelope carries its `attachment_id` — see
  [Output and exit codes](#output-and-exit-codes).
* **Downloads.** `pl8 attachment get` requires `--output PATH` and writes the
  file there; `-` is refused. It never prints the download URL, which can read
  as an inconvenience until you notice what that URL is: a five-minute bearer
  token for the object, usable by anyone who has it. Stdout is not the place
  for one — it ends up in terminal scrollback, in CI logs and in agent
  transcripts. If you genuinely want the URL, `pl8 invoke
  get_issue_attachment` returns it. The download is written to a temp file
  beside `PATH` and renamed into place, so an interrupted download never leaves
  a half-written file where your file should be; pass `--force` to overwrite an
  existing one. An attachment whose upload never finished has no download URL —
  PL8 doesn't issue one for a file that isn't there — so `attachment get`
  refuses it instead of writing you an empty file.
* **Waiting for comments.** `pl8 comment wait` returns when new comments appear
  on an Issue. It polls from your machine rather than blocking in AWS, because
  a Lambda that sat waiting would bill you for the wall clock it spent doing
  nothing. `--interval` defaults to 15 seconds, is clamped to 5-60 and is
  jittered so several waiters don't line up; `--max-wait` defaults to 300
  seconds. Nothing arriving is not a failure: the command exits 0 with an empty
  `items` list, so a loop can simply call it again. It prints one JSON document
  when the wait ends, not a comment at a time.
* **Resuming a wait.** `--after COMMENT_ID` asks for comments after that one;
  omitted, you get the thread from the start. Taking your next `--after` from
  the last id of a batch is the obvious way to follow a thread, and it is
  almost right: ids order comments by the millisecond they were written, so a
  comment written in the same millisecond as the one you stopped at can sort
  before it and be missed. If you can afford to see a comment twice but not to
  miss one, resume from the second-to-last id in the batch instead and ignore
  the comments you have already read.
* **Raw operations.** `pl8 invoke` sends any operation with JSON params,
  for operations that don't have a subcommand yet.

### Rules to know

* Create a Space before creating Issues in it.
* `DONE` is final: an Issue can't move from `DONE` to any other status.
* An Issue with unfinished blockers can't leave `BLOCKED`. To unblock it,
  finish or delete its blockers, or remove them with `pl8 blocker remove`.
* An Issue that is `DONE` can't be added as a blocker, and an Issue can't
  block itself.
* Delete a Space's Issues before deleting the Space. Comments and attachments
  are the other way round: deleting an Issue deletes its comments and its
  attachments for you, however many it has, and deleting a comment deletes the
  attachments tied to that comment.
* You can comment on an Issue in any status, including `DONE`. `DONE` stops
  an Issue moving to another status; it doesn't close the discussion.
* The creator is a label, not a login. PL8 doesn't verify it against your AWS
  identity, nothing stops two callers using the same one, and no command is
  allowed or refused on the basis of it. Use it to see who did what, not to
  control who may do what — for that, use IAM.
* Blocking cycles (A blocks B, and B blocks A) are allowed but deadlock both
  Issues. Remove one of the blockers to break the cycle.
* An attachment belongs to its Issue for good. It can't be moved to another
  Issue, and the comment it is tied to is fixed when you create it.
* An attachment counts as an attachment only once its upload has finished.
  `pl8 attachment add` finishes it for you; an upload that died part way leaves
  a `PENDING` attachment, which is listed, is counted nowhere, can't be
  downloaded, and is cleaned up with its bytes about a day later.
* An attachment name is at most 128 characters and can't contain `"`, `\` or
  control characters, because PL8 puts the name in the filename header of the
  download. A content type must be bare, like `text/plain`: parameters such as
  `; charset=utf-8` are refused.
* A download URL is a bearer token, not a permission check. It carries PL8's
  own authority over that one object for five minutes, so anyone who obtains it
  can fetch the file whatever their IAM identity. This is why
  `pl8 attachment get` writes to a file instead of printing the URL.

### Background updates

Some changes happen in the background, a few seconds after the command that
caused them:

* When a blocking Issue becomes `DONE` or is deleted, the Issues it blocked
  return to `TODO` once they have no other unfinished blockers.
* When an Issue is deleted, the blockers that name it are removed, and so are
  its comments and its attachments.
* When a comment is deleted, the attachments tied to it are deleted too.
* An upload that is started and never finishes is cleaned up about a day after
  it was started, along with any bytes it managed to send.

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

Counters are deliberately outside this. An Issue's `num_comments` and
`num_attachments`, and a comment's `num_attachments`, change as comments and
attachments come and go without bumping the item's `version`, so someone
attaching a file never makes your `--if-version` write fail. Note that
`num_attachments` counts finished uploads only: it can read 0 while an upload
is still in flight.

## Output and exit codes

Every command prints exactly one JSON document to stdout. Add `--pretty` to
indent it.

```json
{"ok": true, "data": {"issue_id": "abc123", "status": "TODO", ...}}
{"ok": false, "error": {"type": "DDBStillBlockedError", "message": "..."}}
```

`pl8 attachment add` is the one command that can fail after it has already
created something. If the upload or the final step fails, the attachment row
exists, and the error envelope carries its `attachment_id`. Use it: `pl8 invoke
resign_issue_attachment_upload` gets you a fresh upload target for that same
attachment, and `pl8 attachment delete` throws it away. Ignoring it leaves a
`PENDING` attachment that nobody can name until it expires.

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
   > you learned on the Issue with `pl8 comment add`, attach logs or output
   > with `pl8 attachment add`, and read `pl8 comment list` before starting
   > work someone else has touched. If you need an answer from someone before
   > you can continue, ask in a comment and wait for the reply with
   > `pl8 comment wait`.

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
`IssueCommentDeleted`, `IssueAttachmentDeleted`,
`IssueNumActiveBlockersZeroed`) internally. See [Events](../architecture/backend/events.md).

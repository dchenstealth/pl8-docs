---
type: ComponentDetails
title: PL8 Events
description: Events sent by PL8
generated: { by: agent:claude-opus-5-5, at: 2026-09-28T00:00:00Z }
---

# PL8 events
pl8-stream-handler sends events on entity changes.
Events are sent as JSON over an EventBridge eventbus.
Event definitions live in pl8-base.

Events:
* IssueNumActiveBlockersZeroed:
  * Core lifecycle event
  * Sent when Issue.num_active_blockers becomes 0
  * Triggers Issue to be moved from BLOCKED to TODO
* IssueDeleted:
  * Core lifecycle event
  * Sent when Issue is deleted
  * Carries space_id, issue_id
  * Triggers cleanup of linked IssueBlockers and of the Issue's IssueComments
    and IssueAttachments
* IssueCommentDeleted:
  * Core lifecycle event
  * Sent when an IssueComment is deleted
  * Carries space_id, issue_id, comment_id
  * Triggers cleanup of that IssueComment's IssueAttachments
* IssueAttachmentDeleted:
  * Core lifecycle event
  * Sent when an IssueAttachment row is removed, by any means, including
    expiry of its TTL
  * Carries space_id, issue_id, attachment_id
  * Triggers deletion of the attachment's S3 object (see [Storage](storage.md))
* IssueDone:
  * Core lifecycle event
  * Sent when an Issue is transitioned to status=DONE
  * Triggers its IssueBlockers to be marked satisfied and the counters on
    the Issues they block to be decremented
* IssueReady:
  * Consumer-facing event
  * Sent when a new Issue is created with status=TODO, or when
    an Issue is transitioned to status=TODO.

## Space event queues
A space event queue is an SQS queue a watcher long polls for one Space's events.

Rules:
* Each queue MUST receive only the events it is configured for, and only those
  of its own Space, matched on the event's space_id.
* Queues are configured per environment in pl8-services, each with its
  Space, event types, message retention and long poll wait time.
* Delivery is at-least-once, so watchers MUST tolerate duplicates.
* A message no watcher deletes expires after the queue's retention period;
  there is no dead-letter queue. This costs a watcher that is down longer than
  the retention period the events it missed.

---
type: ComponentDetails
title: PL8 Events
description: Events sent by PL8
generated: { by: agent:claude-opus-5, at: 2026-09-26T00:00:00Z }
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

---
type: ComponentDetails
title: PL8 Events
description: Events sent by PL8
generated: { by: human:dchen, at: 2026-09-19T00:00:00Z }
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
  * Triggers cleanup of linked IssueBlockers
* IssueDone:
  * Core lifecycle event
  * Sent when an Issue is transitioned to status=DONE
  * Triggers its IssueBlockers to be marked satisfied and the counters on
    the Issues they block to be decremented
* IssueReady:
  * Consumer-facing event
  * Sent when a new Issue is created with status=TODO, or when
    an Issue is transitioned to status=TODO.

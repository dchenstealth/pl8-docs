---
type: SystemDetails
title: PL8 Entities
description: Descriptions of PL8 entities
generated: { by: human:dchen, at: 2026-09-22T00:00:00Z }
---

# Entities
## Issue
An Issue is the fundamental unit of work in PL8.
An Issue:
* MUST belong to a Space
* MUST have a Status
* MUST have a title and description

### Status
A Status indicates what state an Issue is in. There are four Statuses:
* TODO
* BLOCKED
* IN_PROGRESS
* DONE

### Status lifecycle rules
* An Issue MUST NOT be allowed to transition from DONE to another status.

## IssueBlocker
Represents a blocking relationship between two Issues. Issues MAY have
blocking relationships across spaces.

Rules:
* When an IssueBlocker is created, the blocked Issue status MUST be transitioned to BLOCKED.
* An IssueBlocker MUST NOT name an Issue with status DONE as the blocking issue.
* An Issue MUST NOT be transitioned out of BLOCKED while it has active IssueBlockers
* When a blocking Issue is transitioned to DONE and the Issue it blocked has no
  other active blockers, the unblocked Issue MUST be transitioned to TODO.
* An IssueBlocker MUST NOT name the same Issue as both the blocking and the
  blocked issue.
* Blocking cycles between distinct Issues (for example A blocks B and B blocks A)
  are permitted. Issues in a cycle are deadlocked, since neither can reach DONE
  while the other blocks it. Deleting one of the IssueBlockers in the cycle is the
  supported remedy.
* When an Issue is deleted, every IssueBlocker naming it MUST be deleted, whether
  it is the blocking or the blocked issue.

## Space
Spaces are places to group Issues.
A Space:
* MUST have an id, a name, and a description
* MUST be enumerable together with every other Space
* MUST track how many Issues belong to it (`issue_count`)

### Space ids
A space id is supplied by the caller rather than generated, because it is what
every Issue names to say which Space it belongs to.

Rules:
* A space id MUST be unique. Creating a Space with an id already in use MUST
  fail rather than replace the existing Space.
* A space id MUST be between 1 and 64 characters drawn from `[A-Za-z0-9_-]`.
  `#` is excluded because it separates key groups.

### Referential integrity
An Issue MUST belong to a Space. A Space's `issue_count` is what enforces that,
atomically with every write that could break it.

Rules:
* Creating an Issue MUST fail if its Space does not exist. The existence check
  and the `issue_count` increment MUST be atomic with the Issue write.
* Deleting an Issue MUST decrement its Space's `issue_count` atomically with the
  delete.
* Deleting a Space MUST fail while its `issue_count` is nonzero, whatever the
  statuses of its Issues. Deleting a Space never deletes or changes Issues; the
  caller deletes them first.
* Maintaining `issue_count` MUST NOT change the Space's version, so a
  version-fenced Space update is not failed by Issue writes in that Space.

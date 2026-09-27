---
type: Guidelines
title: PL8 Architecture Doc Guidelines
description: How the PL8 architecture docs are structured and what belongs where
generated: { by: agent:claude-opus-5, at: 2026-09-27T00:00:00Z }
---

# Architecture doc guidelines
The architecture docs say what PL8 is and the rules it keeps. A reader should
be able to learn what an entity or component *is* without wading through every
edge case. The edge cases are enforced, and explained, in the code.

## Layers
The docs loosely follow [C4](https://c4model.com/abstractions). Each fact has
one home, at the highest layer that owns it.

* **System** (`architecture/`)
  * `overview.md`: what PL8 is and what it is built on.
  * `entities.md`: the entities, their fields, and the rules relating them.
* **Container** (`architecture/<container>/`, e.g. `backend/`, `cli/`)
  * `overview.md`: the container's parts and what each does.
  * One doc per component or mechanism, e.g. `backend/events.md`,
    `backend/storage.md`: keys, events, wire formats, the technology involved.
* **Code**: not in this repo. Why an implementation is shaped the way it is
  lives in docstrings and comments in pl8-base, pl8-services and pl8-cli.

## What belongs where
* Entity docs name no storage technology. "Atomically" is the right level of
  detail; a transaction, a key, a bucket or a header is not. When a rule needs a
  mechanism to make sense, state the rule and link to the component doc.
* Component docs are where a technology is named and a mechanism explained.
  They link to `entities.md` for the rules they implement rather than restating
  them.
* The docs state rules; the code enforces them and argues for them. A doc
  gives a reason when a reader would otherwise get the rule wrong, in a
  sentence. Races, retries, failure ordering and why an alternative lost belong
  in the code.
* User docs (`user_docs/`) say what a caller does and what happens. Mechanism
  appears there only where a caller's behaviour depends on it.

## Structure of a section
* Open with a sentence or two saying what the thing is.
* List its required and optional properties as MUST/MAY bullets.
* Group further rules under `###` headings, each a `Rules:` list.
* Supporting prose comes after the rules, and is the first thing cut when a
  section grows.
* A rule that is the same as one already written links to it (e.g. "Attachment
  ids follow the same rules as comment ids") instead of restating it.
* Where a limitation is accepted rather than fixed, say so and say who it costs.

A new section that resembles an existing one should read at a comparable
length. If it is much longer, the extra is usually restatement or edge cases
that belong in the code.

## Reasons must be true
A reason in the docs is checked against the code before it is written. A
plausible reason that is not true is worse than none: it gets copied into
comments and tests and defended later. If the real reason is a product choice,
say that.

## Files
* Every doc carries frontmatter: `type`, `title`, `description` and
  `generated: { by, at }`. Agent edits use `by: agent:<model>`; `at` is the
  date of the change.
* `type` is one of `SystemOverview`, `SystemDetails`, `ComponentOverview`,
  `ComponentDetails`, `UserGuide` or `Guidelines`.
* Every directory has an `index.md` listing its docs, one line each, using the
  doc's `description`.

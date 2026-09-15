# ADR-020: Setlists are an ordered sequence on the event, not a set of relationship edges

**Status:** Proposed · 2026-09-15 · Split out of [ADR-019](ADR-019-musician-domain.md) §1's
context, per that ADR's own alternatives section. Not yet reviewed by the
owner and not yet implemented.

## Context

`docs/domains.md` §2a states the engine's relationship model plainly:
**relationships are an unordered set.** A `Walk` gathers whatever entities a
relationship name and direction reach; nothing about an edge says "this one
comes third." The engine's existing rule for the one case that already needs
order — an attachment sequence — is to keep it *inline*: "one attachment is
an entity; an ordered sequence stays inline… the test: if removing an item
would change what the remaining items mean, it is a sequence" (`docs/domains.md`
§2, base vocabulary section).

A setlist fails that test in the most literal way possible. Reordering two
songs changes the set's energy arc and key transitions; removing a song
shortens the set and may break a planned key transition either side of the
gap; the position of a song (opener, mid-set, before the break, closer,
encore) is often *the* fact a musician wants recorded about it, more than
which songs happen to appear. A setlist is not "the songs this gig involved,"
which a `Walk` over an unordered `included` relationship would faithfully
answer — it is "the songs, in this order, with these breaks."

The question is where that order lives and how it survives the same temporal
questions this engine already answers elsewhere (ADR-004's `snapshot(T)`, "what
did we play last time we played this venue").

## Decision

### 1. A setlist's songs live in `content.songs[]`, ordered, on the setlist entity

A `setlist` entity (ADR-019 §2) carries its song sequence as an ordered array
in `content`, following the existing attachment-sequence rule exactly:

```json
{
  "entity_type": "setlist",
  "content": {
    "label": "The Anchor, Sat night",
    "songs": [
      {"song_id": "song-123", "set": 1, "position": 1},
      {"song_id": "song-456", "set": 1, "position": 2, "key_override": "D"},
      {"song_id": "song-789", "set": 1, "position": 3},
      {"song_id": "song-321", "set": 2, "position": 1, "note": "false start, restarted"},
      {"song_id": "song-654", "set": "encore", "position": 1}
    ]
  }
}
```

Each entry names a `song_id` (a reference to a `song` entity — a plain id
reference in `content`, not a relationship, exactly as a `document`'s
`blob_id` is a plain reference rather than a relationship to the blobs
table). `set` and `position` carry the ordering and the set-break structure
(Set 1 / Set 2 / Encore) that a relationship edge has no field for. Optional
per-performance overrides (`key_override`, a `note` for "this one went
sideways") live on the entry, not on the `song` itself — the song's own
record stays the same across every band and every night that plays it; what
varies night to night is the setlist entry.

### 2. The setlist is attached to its occasion by `content.event_id`, not a walk

A setlist belongs to exactly one `event` (a `gig` or `rehearsal`). That
reference is a plain id on the setlist's `content`, the same shape as the
song references inside it — not a `happened_to` relationship. Two references
of different cardinality and different query needs (a Walk that fans out to
many, versus a single-parent pointer) do not need to be forced into one
mechanism just because both happen to name another entity; a relationship
buys traversal and `find_similar`-style discovery, which nothing here needs —
a setlist has exactly one event and is found by "get the setlist for this
event," a direct lookup. If this proves wrong once real gigs' data
accumulates (§4), promoting `event → setlist` to a proper `happened_to`
relationship is a one-line manifest change, not a data migration, since the
foreign key already exists as a plain value.

### 3. A planned setlist and the setlist actually played are two entities

Following ADR-016 §1.3's full-lifecycle discipline (a vehicle's pre-purchase
evaluation is recorded even when the purchase never happens): a setlist drawn
up before the gig and the set as actually played — with a song dropped for
time, an unplanned cover thrown in, a reordering on the night — are not the
same fact and are not overwritten into each other. Two entities:
`setlist` with `content.kind: "planned"` and one with `content.kind:
"played"`, both referencing the same `event_id`. Nothing is lost: what was
intended, and what happened, are both permanent. (Whether these should
instead be one entity with plan-vs-actual as a versioned edit — using the
engine's existing version history rather than two entities — is exactly the
kind of question §4 defers to a real walk: the vehicles domain's evaluation
records are a close precedent for "two records, not one edited record,"
because the *difference* between plan and actual is itself often the
interesting fact, the way an evaluation that did not lead to a purchase is
still worth keeping in full.)

### 4. Deriving whether this is enough

This ADR commits to inline ordering as the mechanism; it does not commit to
the exact field names or to plan-vs-played being two entities forever.
Following ADR-016 §4's discipline: do real post-gig walks (ADR-019 §4),
including at least one gig where the played set diverged from the plan and
one gig at a repeat venue, then check this shape against real questions
("what did we play last time here," "how has our closer changed over a
year," "did we ever manage to fit both of these songs in the same key
sequence"). Promote or revise only from what those walks show.

## Consequences

- Ordering, set breaks and per-performance overrides are representable from
  day one without inventing an ordered-relationship concept the engine does
  not otherwise have — the fix stays inside `content`, which is exactly what
  the existing attachment-sequence rule is for.
- A setlist's songs are **not** reachable by a generic `Walk` (ADR-012 §2 /
  `docs/domains.md` §2a) the way `fitted_to` parts are — `get_setlist(event)`
  must be a `handler`-backed `DomainTool` that reads and orders
  `content.songs[]`, not a declarative `Walk`. This is the same trade-off the
  engine already accepts for ordered attachment sequences: declarative tools
  cover relationship traversal; a handler covers everything content-shaped.
- "Which setlists include this song" is a `content` search (FTS or a content
  scan), not a relationship traversal — slower and less indexed than a Walk
  over an edge would be. Acceptable at the expected volume (a band's book and
  gig count are small numbers by construction — ADR-019 §1's volume
  boundary); revisit only if that stops being true.
- Plan-vs-played as two entities roughly doubles the setlist row count per
  gig where both are recorded, which is negligible at this domain's volume
  and buys back the same "nothing is silently overwritten" property ADR-011
  and ADR-016 both rely on elsewhere.

## Alternatives considered

- **A relationship (`includes`, setlist → song) plus a `position` integer in
  edge properties** — the shape `docs/domains.md` §2's own sequence rule
  explicitly warns against ("nowhere to put its order except a `step` integer
  in edge properties that every reader must know to sort by"); rejected on
  the engine's own stated precedent, not a new argument invented for this
  ADR.
- **A `song` carries `set_position` per setlist as a property on a
  many-to-many join entity** — reinvents a join table inside a graph engine
  that already has a first-class inline-sequence mechanism; more moving parts
  for no extra capability.
- **One setlist entity per gig, edited in place as the night progresses**
  (no plan-vs-played split) — rejected: loses the pre-gig plan the moment the
  set diverges from it, the same information loss ADR-016 rejected for
  vehicles by keeping evaluations that did not lead to a purchase.
- **Store the setlist as an ordered list of song *titles* (free text), not
  song references** — rejected: breaks "what have we played this song with
  this band" and "how has our closer changed" (ADR-019's motivating
  questions), and reintroduces the exact fragmentation (`"Wonderwall"` vs
  `"wonderwall"` vs a cover's alternate title) the manifest vocabulary exists
  to prevent (`docs/domains.md` §2, "why not arbitrary strings").

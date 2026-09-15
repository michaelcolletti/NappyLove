# ADR-019: The musician domain — a walk-first knowledge history graph for bands, gigs and songs

**Status:** Proposed · 2026-09-15 · Drafted following the vehicles domain
precedent ([ADR-016](ADR-016-vehicles-domain.md)); not yet reviewed by the
owner and not yet implemented. The vocabulary in §2 is a deliberately loose
*capture* vocabulary, to be revised from real walks (§4) before this ADR is
marked Accepted, exactly as ADR-016 §4 requires for vehicles.

## Context

[ADR-012](ADR-012-domain-abstraction.md) proved the engine domain-blind with
`house` and `vehicles`. A third domain — musicians, bands, instruments and the
gigs they play — is a further proof of the same abstraction, and a genuinely
different shape of data from either prior domain:

- **House** is mostly *what is where, and does it control what*: a snapshot
  with light history.
- **Vehicles** is a *knowledge history graph* about a small number of
  long-lived things (ADR-016): full lifecycle, low write volume, no ordering
  problem — a car either has a part fitted or it does not.
- **Musicians** is also a knowledge history graph, but it has a shape neither
  prior domain does: a **setlist** is not a set of songs, it is a *sequence*
  of songs, and the sequence is the point — a set builds energy, manages key
  and tempo transitions, and paces a night. The engine's relationships are an
  explicitly unordered set (`docs/domains.md` §2), so this domain cannot reuse
  the vehicles pattern of "a relationship per association" for its central
  artefact without losing the one thing that matters about it. That question
  is significant enough to be its own decision — [ADR-020](ADR-020-setlists-as-ordered-sequences.md) — rather than
  folded into this ADR's vocabulary table.

The domain exists to answer questions like: *what have we played this song
with this band, and when did we last play it? who has played this venue
before, and what was the setlist? what gear does this musician own versus
borrow, and who has the good DI box this week? when did this lineup change?*
— the same "walk round it, ask what happened" pattern as vehicles, applied to
people, instruments, songs and shows instead of cars.

## Decision

### 1. Principles (following ADR-016 §1, applied here)

1. **Walk-first.** A `musician-walk` — same mechanism as `room-walk` and
   `vehicle-walk`: session file on disk, diffs accumulated, review, `confirm`,
   every created entity read back before the session archives. The natural
   walks are *after a gig* (what did we play, how did it go, what broke) and
   *gear check* (what do we own, what needs fixing, what did we borrow or
   lend). Photos, audio snippets and transcribed notes are first-class walk
   outputs, not afterthoughts.
2. **A knowledge history graph, with a volume boundary.** The graph holds
   *facts about a musician's, band's or song's life*, not a stream. The test
   is the same as ADR-016 §1.2, translated: *would a person write this in the
   band's logbook?* — a gig played, a lineup change, a song added to the book,
   a piece of gear bought or broken, yes. Every rehearsal's individual take,
   a click track, raw multitrack audio — no; those live wherever recording
   happens, and the graph may hold a `document` pointing at a mixdown or a
   single reference recording of a performance.
3. **Full lifecycle, append-only.** A band's record starts before its first
   gig (formation, early rehearsals) and continues through lineup changes,
   hiatuses and reunions; a musician's record spans every band they have been
   in; a song's record spans every arrangement and every band that has played
   it. Nothing is deleted — a band that stops playing gets an end date on its
   membership intervals, not a tombstone; a retired song leaves the setlists
   it was ever on exactly as they were.
4. **Derive the vocabulary from examples.** Ship the loose capture vocabulary
   in §2, do real walks (§4), then promote what recurs with structure —
   exactly ADR-016 §1.4's discipline, so this domain does not repeat the
   house's original mistake of designing from imagination (ADR-013).
5. **The record is durable; the interface is disposable** (ADR-016 §1.5,
   unchanged here). The append-only graph outlives the walk skill and the
   declared tools around it.
6. **One walk skill, prompt packs by occasion**, not by instrument or genre:
   *post-gig* (setlist actually played, what worked, what broke, a note per
   song if it went sideways), *gear* (what came in, what went out, what needs
   fixing), *new song* (who wrote or arranged it, key, tempo, structure
   notes), *lineup change* (who joined, who left, when, why if worth
   recording). A five-piece covers band, a solo singer-songwriter and an
   orchestral sub all use the same mechanism; only the questions differ.

### 2. Capture vocabulary (loose on purpose)

| Type | Role | Notes |
|---|---|---|
| `musician` | A person, across their whole playing life | `aliases` (stage names), `instruments` (free text list of what they play), `contact` |
| `band` | A named act — a duo, a covers band, an orchestra section, a pickup gig | `content.kind` (band / duo / solo / pit / session pool …), `aliases`, `status` (active / hiatus / disbanded) |
| `instrument` | A physical instrument or major piece of gear with its own identity | Kept as an asset the way `part` is in vehicles: `content.kind` (guitar, bass, kit, keys, PA, amp, pedal, mic …), `identity` (serial number, if any), `spec` (make, model, tuning, modifications), `status` (owned / borrowed / loaned out / sold) |
| `song` | A piece in the book, independent of any one performance | `title`, `aliases` (also known as), `content.key`, `tempo`, `structure` notes, `original_artist` (blank if an original), `written_by` reference via `wrote` |
| `venue` | Where things are played — **may be anywhere**, not just a fixed room | `kind` (bar / hall / house show / festival / studio / street), address, `capacity`, house-gear notes (does it have a PA, a piano) |
| `setlist` | An ordered plan or record of songs for one occasion | See [ADR-020](ADR-020-setlists-as-ordered-sequences.md) for why this is *not* a set of relationship edges; an interval-free entity whose `content.songs[]` is the ordered sequence, each entry carrying the song reference, an optional key/tempo override for that night, and set-break markers (Set 1 / Set 2 / Encore) |
| **`event`** | **Something that happened to a musician, a band, an instrument or a song, on a date** | `kind` open: `gig`, `rehearsal`, `recording_session`, `audition`, `lesson`, `lineup_change`, `purchase`, `repair`, `loan`, `sale`, `song_added`, `song_retired` …; `when`, `cost`, `currency`, `pay` (what the gig paid, if applicable), `where` (a venue id or text), free text |
| `note` | Transcribed speech, free text | The walk's post-gig transcript is a note on the `event` |
| `photo` | base attachment | Stage plots, gear photos, the crowd |
| `document` | This domain's attachment type | `kind`: chart, lyric sheet, contract/rider, mixdown, setlist scan, press photo … |

Relationships, minimal: `member_of` (**interval**; musician → band — a
lineup is who is a member *when*, exactly like `fitted_to` tracks what part
is on which vehicle when), `plays` (**interval**; musician → instrument, for
who currently has which piece of gear — borrowed and returned is two
interval edges, not a deletion), `performed_at` (event → venue), `booked`
(event → band), `wrote` (musician → song), `arrangement_of` (song → song, a
cover or rearrangement pointing at its source), `happened_to` (event →
musician / band / instrument / song, whichever the event is actually about),
`documented_by` (anything → note / document), base `has_photo`. A setlist's
songs are **not** a relationship at all — see ADR-020 — but a setlist is
still attached to the event it belongs to via `happened_to` (event → setlist)
or, more simply, `content.setlist_id` on the `event` itself; §4 decides which
once a walk exercises it.

**Why one `event` type with an open `kind`**, exactly ADR-016's reasoning:
`gig`, `rehearsal`, `lineup_change`, `purchase` and `repair` are five guesses
at structure a walk has not yet earned. What is not loose from day one:
`event.when` and both interval relationships (`member_of`, `plays`), because
those are what answer "who was in the band when this was recorded" and "whose
amp was that, that night" — the temporal spine, same as vehicles.

**Merge rules**: none identified yet beyond the engine defaults; unlike a
vehicle's odometer, nothing in this vocabulary has an obvious monotonic
field. Revisit once a walk shows a concurrent-edit case.

**Tools**: the engine's 18 generic tools, plus a small declared set that
mirrors the vehicles pattern with different constants — `get_band_history`
(every event, membership change and gig, oldest first, `at`-aware),
`get_events(band|musician|song, kind, since, until)`, `get_current_lineup(band,
at)`, `get_gigs_at_venue(venue)`, `get_songs_played_with(band, since)`, and
`where_is(instrument)` — who has it and since when, `at`-aware, the direct
analogue of the vehicles tool of the same purpose. `get_setlist(event)` reads
back the ordered `content.songs[]`. Anything more specific (`get_set_length`,
`get_key_transitions`, `get_repertoire_overlap` between two bands) is added
once its need is real.

### 3. The walk

`domains/musician/skills/musician-walk/SKILL.md`: the room-walk phases
(orient → discovery loop → review → confirm → handoff), with occasion packs
rather than kind packs (§1.6): *post-gig*, *gear*, *new song*, *lineup
change*. Every answer becomes a `musician`, `band`, `instrument`, `song`,
`venue`, `setlist` or an `event` of some `kind`, with photos and a note; the
pack changes only which questions are asked.

### 4. Deriving the real vocabulary

Do at least two real walks per occasion (post-gig, gear, new song, lineup
change) before promoting anything beyond §2, following ADR-016 §4's
discipline exactly: walk first, read the graph, then promote what recurs with
structure into typed entities and tools. Specifically undecided until a walk
answers it: whether a setlist is its own entity (as modelled in §2) or inline
content on the `event` it belongs to — both keep the ordering property ADR-020
requires; which is more convenient in practice (querying "every setlist with
this song" versus keeping a gig's record self-contained) is worth finding out
from real data rather than guessing here.

### 5. Feeds and apps

Setlist.fm-style services, streaming royalty statements, and tab/chart
libraries are the obvious future `app` entities `manage`-ing a band or a song,
writing `event`s at logbook granularity (a gig logged, a royalty statement
event with an amount and currency). None are integrated by this ADR; the
first walks are manual.

## Consequences

- A third domain, alongside `house` and `vehicles`, further exercising the
  isolation test (`tests/test_domain_isolation.py`) and the manifest-driven
  tool catalog (ADR-012 §2, `docs/domains.md` §2a) — no engine change is
  needed to add it.
- The domain introduces the first genuinely order-sensitive artefact in the
  system (the setlist); ADR-020 resolves that at the vocabulary level so the
  engine's unordered-relationship model is not stretched to fit it.
- Like vehicles, this domain is deliberately under-designed at entities the
  walks have not yet exercised (song structure, gear maintenance history in
  detail); the cost of being wrong is a manifest edit and a re-typing
  migration (ADR-013's precedent), not a schema change or lost data.
- A gig's `pay` and `cost` fields reuse the currency-per-event pattern ADR-016
  established for vehicles' fuel/charge events, rather than inventing a new
  money shape.

## Alternatives considered

- **Typed entities per event kind (`gig`, `rehearsal`, `lineup_change`) from
  day one** — rejected for the same reason ADR-016 rejected it for vehicles:
  guessed structure ahead of any walk.
- **Model a band's lineup as a plain (unversioned) list on the band entity**
  — rejected: it cannot answer "who was in the band on this date," which is
  exactly the kind of question this graph exists to answer; `member_of` as an
  interval edge is the same mechanism vehicles uses for `fitted_to`.
- **Fold musicians into the vehicles or house domain** ("it's just another
  collection of things with history") — rejected: distinct vocabulary,
  distinct owners in practice, and ADR-012's whole premise is that unrelated
  domains stay isolated databases; a shared domain would recreate the
  coupling ADR-012 was written to avoid.
- **Design the setlist ordering mechanism inline in this ADR** — rejected in
  favour of a dedicated ADR-020: it is a decision about the engine's
  ordered-vs-unordered data rule (`docs/domains.md` §2), not merely a fact
  about musicians, and deserves its own record and its own alternatives.

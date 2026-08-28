# Android build plan

How the notes and data in this repo become **Woodshed**, an Android practice
app. Written after re-reading `DESIGN.md` and checking its claims against the
two data files, which is where the two findings below came from.

Decisions already made:

| | |
| --- | --- |
| Stack | Kotlin, Compose, Room |
| Shape | Flash cards *and* a timed session mode |
| Scope | Local-only, single user, offline. No account, no server, no sync |
| Seed | `data/standards.json` (201), `data/exercises.json` (44), shipped as assets |

Kotlin over a Rust core via UniFFI: the two files worth salvaging
(`roller.rs`, `keys.rs`) are small and well-specified, `DESIGN.md` already
expects the roller not to survive the product change, and an NDK build plus a
cross-language debug story is a lot of ceremony for a few hundred lines.

---

## Finding one: there are no flash cards in the data yet

`exercises.json` reads like a card deck and isn't one. Every one of the 44
entries is a **procedure**, not a prompt:

> **Lick around the circle** — Take one lick and move it through the circle of
> fourths until it stops needing thought.

That tells you what to do with a lick. It does not contain a lick. Same for
*Melody from memory*, which is a thing you do **to** a tune you already chose.

**The move: don't author a deck, compose one.** A card is a tune × a procedure ×
a written key. `standards.json` supplies 201 subjects, `exercises.json` supplies
the verbs, transposition supplies the key:

```
All The Things You Are  ·  changes without the melody  ·  written F
```

This needs one small authoring pass — an `applies_to_tune` flag hand-set across
44 rows. The two `Standards` procedures obviously qualify; `Improvisation`,
`Ear Training` and `Transcription` probably do; `Tone` and `Air & Breathing` do
not. Half an hour of work that turns 201 rows into a few hundred distinct cards.

**The gap that stays open.** The vocabulary deck has no content and cannot be
composed. Licks and ii-V-I lines are the one place where the card *is* the
material, and none of it exists in this repo. That makes lick capture — text, a
photo of a scrap of manuscript, a short audio memo — a real sub-feature rather
than a detail, and the reason vocabulary lands in M4 and not M1.

## Finding two: the five-phase model only works at an hour or more

`DESIGN.md` flags the phase shares as an unvalidated guess and asks for them to
be questioned. They break before you get to whether they match how you practice:
**the percentages are arithmetically infeasible below about 45 minutes.** Each
phase's share of a short session comes out smaller than the shortest exercise
that phase can draw.

Minutes allocated per phase. `!` marks a phase that cannot fit even its
shortest exercise.

| Session | Warmup<br>15% · min 5m | Fundamentals<br>20% · min 6m | Vocabulary<br>25% · min 8m | Repertoire<br>30% · min 10m | Cooldown<br>10% · min 6m |
| --- | --- | --- | --- | --- | --- |
| 20 min | 3.0 ! | 4.0 ! | 5.0 ! | 6.0 ! | 2.0 ! |
| 30 min | 4.5 ! | 6.0 | 7.5 ! | 9.0 ! | 3.0 ! |
| 45 min | 6.8 | 9.0 | 11.3 | 13.5 | 4.5 ! |
| 60 min | 9.0 | 12.0 | 15.0 | 18.0 | 6.0 |
| 90 min | 13.5 | 18.0 | 22.5 | 27.0 | 9.0 |

Cooldown fails at every length under an hour. A 20-minute session fails
everywhere. The shares were fitted to the 45-minute session the old planner
assumed — but the whole premise of moving to a phone is *short, frequent,
interruptible* practice, so the common case is exactly the broken one.

**Fix: allocate by whole exercises, not by percentage.** Treat the shares as
weights for a greedy fill. Walk phases in order; at each step pick the phase
whose *served* minutes are furthest below its target share; take one exercise
that fits the remaining time; repeat until nothing fits. A phase that never gets
a slot simply doesn't appear.

This preserves the load-bearing property `DESIGN.md` identifies — short sessions
keep warmup and fundamentals and drop from the end — because the end phases are
the ones that lose the tie-break when time runs out. It also produces a sane
20-minute session (one warmup, one fundamental, one tune) instead of five
impossible slots.

Below roughly 15 minutes, stop planning a session at all and hand the user card
mode. Two products for two shapes of free time is the point of building both.

---

## Domain model

Four pieces carry over. They are small, well-specified, and each has a failure
mode that is invisible until you are on the stand.

### Instruments and transposition

Eleven instruments across four families, each with a semitone offset from
concert to written pitch. `written = (concert + offset) mod 12`. C is 0, Bb is
+2, G is +5, Eb is +9.

**Keep the test that asserts each offset agrees with the pitch name the
instrument is built in.** A wrong offset makes every displayed key silently
wrong and nothing else in the app catches it.

The split that matters: **repertoire stores concert pitch, practice history
stores written keys.** A ii-V-I in written G counts toward G whether it happened
on alto or on flute. Two columns, never one.

### Key spelling needs two tables

Major prefers flats (C Db D Eb E F Gb G Ab A Bb B), minor prefers sharps
(C C# D Eb E F F# G G# A Bb B). One table produces names no musician writes,
like Db minor for what is plainly C# minor.

Parsing is **fallible and must stay fallible**. The real data contains
`"Eb, C"` for On Green Dolphin Street, which genuinely modulates. Return null
and render the raw string; a confidently wrong transposition is worse than a
visible unknown. In Kotlin that is `Key?` plus a preserved `rawKey: String`,
not an exception.

Key rotation walks the circle of fourths (C F Bb Eb Ab Db Gb B E A D G) rather
than picking at random, because that is the order keys actually get practiced
in. It is the sequence behind both the `CYCLE` policy below and the "around the
circle" instruction four of the exercises already assume.

### What a card is

The one genuinely new piece of modelling. A card is an item plus, sometimes, a
key — and how the key behaves differs by material. Putting that on a per-item
`KeyPolicy` keeps one scheduler serving all three cases.

| Policy | Scheduling | Key comes from | Applies to |
| --- | --- | --- | --- |
| `FIXED` | One card per item | The tune's `common_key`, transposed | Repertoire, by default |
| `CYCLE` | One card per item | Next step round the circle each rep | Scales & patterns; tunes you have promoted |
| `PER_KEY` | Twelve independent cards | Fixed per card | Licks, ii-V-I — where key-specific fluency *is* the skill |

Materialising 201 tunes × 12 keys would mean seeing a tune in all twelve before
any of it counted as learned. `FIXED` with a per-tune promotion to `CYCLE`
matches how repertoire actually gets learned and keeps the row count honest.
`PER_KEY` is affordable precisely because vocabulary is a handful of items, not
two hundred.

## Scheduling, and the 80/20 guess it replaces

`DESIGN.md` describes the problem exactly: with ~201 tunes at "wishlist",
drawing by staleness hands over a brand-new tune every session and nothing is
ever learned. The fix used was to draw an in-progress tune about 80% of the
time — a magic number, acknowledged as a guess.

**Spaced repetition already solves this, rigorously.** The standard mechanism is
a separate, rate-limited new-card queue: due cards are served first, and at most
*N* new items get introduced per day. Set N to 1 or 2 for repertoire and you
have encoded "learn a tune before starting another" as a dial you can tune
against logged reps, rather than a probability you have to guess.

Give each tune an explicit lifecycle — `wishlist → learning → repertoire →
retired`. Only `learning` and `repertoire` enter the review queue; `wishlist`
**is** the new-card pool. That maps the old "in progress" idea onto SRS
vocabulary with no fudge factor left over.

### Keep the scheduler swappable

FSRS is the better algorithm, but it is fitted on recall-and-forget data for
discrete facts, and nobody has shown that transfers to motor skill on a horn.
Don't bet the schema on it. Ship a plain SM-2 variant behind an interface, and
log every rep as an event rich enough to replay history against a different
scheduler later.

```kotlin
interface Scheduler {
  fun next(state: CardState, grade: Grade, at: Instant): CardState
}
```

```sql
-- the event log is the real asset; card state is a derived cache
review_events(
  id, card_id, item_id, item_kind,
  key_written,           -- nullable: not every card has a key
  grade,                 -- again | hard | good | easy
  reviewed_at, elapsed_ms,
  scheduler_id, scheduler_version,
  interval_before, interval_after, ease_before, ease_after
)
```

Storing the scheduler's inputs and outputs alongside the grade is what makes the
swap cheap: you can re-derive every card's state under a new algorithm from
history instead of starting over. This is also what `DESIGN.md` is asking for
when it says to drop the plan-is-the-log design in favour of a plain event log —
sessions become a view over events, not a table that history hangs off.

## Session mode

The second mode, and the reason the exercise data survives. Long tones for ten
minutes with a tuner drone is not a flash card; it is a timer with audio. The
phase-to-category mapping in `DESIGN.md` is complete and exact against the data
— all 14 categories land in a phase, no phase is empty — so it ports directly.

- **Planner.** Greedy weighted fill as above, seeded from the exercise pool
  filtered by the user's instrument families. Remember that empty `families`
  means "applies to everything", which is 37 of the 44 rows — the common case,
  so make that the default in code rather than a special case.
- **Runner.** A foreground service so the timer survives the app going to
  background mid-session, with a notification showing current exercise and
  remaining time. Screen kept awake while the runner is active.
- **Output.** The session emits the same `review_events` rows as card mode. Two
  front ends, one history.
- **Interruption is the normal case.** Persist runner state on every tick.
  Someone will walk in halfway through the fundamentals block.

## Data and seeding

Both JSON files ship as assets and seed on first launch. The natural keys are
clean — all 201 slugs are unique, all 44 exercise names are unique — so
upsert-by-natural-key works with no disambiguation needed.

```sql
-- author-owned: refreshed on every re-seed
standards(id, slug UNIQUE, title, composer, year, bars,
          time_signature, form?, style?, common_key_raw?, common_key?)

-- user-owned: never touched by re-seed
tune_state(standard_id, lifecycle, key_policy, cycle_position,
           notes, archived_at)

exercises(id, name UNIQUE, category, description, minutes,
          difficulty, tempo_bpm?, families, applies_to_tune)
```

Four rules, all of them lessons the old branches paid for:

- **Upsert by natural key; never delete-and-reinsert.** History points at
  exercises by id. The old codebase's seed pattern rebuilt rows every boot,
  which was safe only because nothing referenced them. Here it would orphan
  every past session.
- **Archive, don't delete.** Anything dropped from a future data file gets an
  `archived_at`, so old events still resolve to a name.
- **Split author-owned from user-owned before the first seed, not after.**
  Re-seeding must refresh scraped metadata and leave lifecycle, notes and
  archived state alone. Getting this boundary explicit up front removed a whole
  class of bug last time.
- **Keep "unknown" distinguishable from "empty."** 44 tunes have no documented
  form, 3 no key, 1 no style. Nullable columns, not empty strings — collapse the
  distinction on import and it is gone permanently.

**Known quirks: preserve, don't normalise.** Compound styles (`Latin/Swing`,
`Swing,Ballad`) will tempt you into splitting on a delimiter; two rows is not
worth the loss. Minor keys carry a trailing hyphen (`F-`, `C#-`), sharps occur,
and `"Eb, C"` does not parse at all. Store the raw string next to the parsed
value in every case.

## Audio: metronome and drone

Not optional — 24 of the 44 exercises carry a `tempo_bpm`, and the tone
exercises explicitly call for a tuner drone. This is the one place where the
"just use the framework" answer is wrong.

- **Don't schedule clicks from a timer callback.** Handler, coroutine `delay`
  and `SoundPool` all jitter audibly at 80 bpm — and a metronome that drifts is
  worse than none, because you will trust it.
- **Render clicks into the PCM stream at sample-accurate offsets** and feed a
  streaming `AudioTrack` from a dedicated thread. At 48 kHz a beat at 80 bpm is
  exactly 36,000 frames; place it there. Solid timing without dropping to the
  NDK.
- **Reach for Oboe only if measurement says so.** It is the right tool for low
  round-trip latency, but the metronome is output-only and self-scheduled, so it
  likely does not need it. Measure before adding a C++ toolchain to the build.
- **Drone is easy** — a looped sine or sampled tone at the exercise's pitch.
  Ship it with the metronome; same audio path.
- **A tuner is a different animal.** Pitch detection is real DSP with real edge
  cases on a saxophone's spectrum. Defer it, and don't let it creep into the
  metronome milestone.

## Designing for someone holding a saxophone

The constraint that separates this from every other flash-card app, and the one
most likely to be discovered too late: during the part of the loop that matters,
both of the user's hands are full and their mouth is occupied. Nothing about the
standard Compose review screen accounts for that.

- **Bluetooth pedal support.** Page-turner pedals are common kit and present as
  keyboards — handle `KEYCODE_PAGE_DOWN`, `KEYCODE_SPACE` and the media keys. A
  foot switch meaning "next card" or "reveal" makes the app usable mid-rep, and
  it is a couple of hours of work.
- **Screen stays awake, always.** A screen that sleeps between reps is a session
  ender.
- **Landscape, at music-stand distance.** The device is propped a metre away,
  not held. Type sizes and touch targets both want to be larger than the
  defaults; test at arm's length, not at desk distance.
- **Auto-advance with a visible countdown** in session mode, so a normal run
  needs no input at all.
- **Grade buttons sized for a fast, imprecise tap** — horn in the left hand, one
  thumb free. Full-height columns, not a row of pills.
- **Nothing on a swipe gesture alone.** Every gesture needs a tap equivalent,
  for the same reason.

## Testing, per the traps already paid for

`DESIGN.md` records four mistakes that cost real time. Each has an Android
analogue that produces a green suite proving nothing.

| The trap, as it happened | The Android version |
| --- | --- |
| Unit tests that skip the real boundary — checkbox lists never hit the form deserializer | Testing a ViewModel against a fake repository proves nothing about the DAO. Run the seed importer and the DAOs against a **real in-memory Room database**, parsing the **actual asset JSON** — not hand-built lists. TypeConverters for `families` and nullable keys are exactly where this breaks. |
| A linter that cannot fail is decoration — the validator was green because it truncated its own output | Write the **negative test first**: feed the seed validator deliberately corrupted JSON and confirm it fails. A validator you have never seen fail is not evidence. |
| Verify the fix, not the intent — Chromium ignores `line-height` on a `<select>` | Compose has the same class of silent no-op: a modifier order that discards a constraint, a recomposition that never fires. **Assert the resulting state** in UI tests; don't assume the call had its effect. |
| Seed reconciliation must preserve ids | Covered by the schema above — but test it. Seed, record a review event, re-seed, assert the event still resolves. |

Plus one this app adds: **transposition gets exhaustive property tests.** All 11
instruments × 12 concert keys × both spelling tables is 264 assertions and runs
instantly. The cheapest possible insurance against every key in the app being
silently a tone out.

---

## Build order

Ordered so each milestone is independently useful, and the riskiest assumption —
that the phase model matches how you actually practice — gets tested with real
reps before anything is built on top of it.

**M0 — Data in, and provably intact.** Room schema, seed importer from assets,
instrument picker, transposition and key-spelling domain with the exhaustive
test suite, browse screens for 201 tunes and 44 exercises. No practice loop yet.
*Proves the data survives the boundary. Ends with the re-seed-preserves-ids test
passing.*

**M1 — Repertoire cards.** The `applies_to_tune` authoring pass, composed
tune × procedure × key cards, card model with `KeyPolicy`, SM-2 behind the
scheduler interface, the event log, tune lifecycle, daily new-card cap, review
screen with grade buttons. *First usable app. Start logging reps — everything
after this benefits from real data.*

**M2 — Session mode.** Greedy weighted phase planner, foreground-service runner
with timer and notification, keep-awake, auto-advance, interruption-safe state.
Emits into the same event log. *Now you can compare planned sessions against
card sessions on your own logged history.*

**M3 — Audio.** Sample-accurate metronome on a streaming `AudioTrack`, drone at
arbitrary pitch, tempo control per exercise. Measure latency before considering
Oboe. *Session mode stops needing a second device on the stand.*

**M4 — Vocabulary deck.** Lick capture — title, notes, optional photo of
manuscript, optional audio memo — plus `PER_KEY` cards over the circle of
fourths. The one milestone that needs new content from you, which is why it is
last rather than first. *Closes the gap in finding one.*

**M5 — Living with it.** Stats derived from the event log, JSON export as real
backup, Bluetooth pedal, landscape polish, practice reminder. Revisit the phase
shares and the new-card cap against several months of actual reps. *The first
point at which the guesses can be replaced with measurements.*

---

## Open questions

Carried forward from `DESIGN.md`, plus what this plan adds. All of these are
answerable with logged reps and none are answerable now, which is the argument
for M1 shipping early and thin.

**Do the five phases match how you actually practice?** `DESIGN.md` flags this
as a guess built from a verbal description and the first thing to question. It
is still unvalidated. If you always end on tunes, or never cool down, or warm up
differently on doubles day, the model is wrong in a way no test notices. M2
should make phase order configurable rather than hard-coding a guess twice.

**Is spaced repetition the right frame for motor skill at all?** SRS intervals
are fitted to memory decay for discrete facts. Playing a tune well is not
recall. Intervals may still be a good *surfacing* mechanism even if the
underlying model is wrong — the event log is the hedge that lets you find out.

**Is the instrument-family scoping right?** Only 7 of 44 exercises are
family-scoped, and `DESIGN.md` already doubts the calls — plenty of sax players
double-tongue. Cheap to fix in data, so worth a second opinion from someone who
teaches.

**What does the app do with 201 tunes you will never learn?** A wishlist that
large is a browsing corpus, not a queue. Something has to help you choose —
filter by style, form or key, or a curated starting set. Unaddressed in the old
design and still unaddressed here.

**Does the composed card actually read as a card?** Finding one is the
load-bearing bet of this plan. If *All The Things You Are · changes without the
melody · written F* feels like a chore assignment rather than a prompt, M1 needs
rethinking before M4 builds on the same model.

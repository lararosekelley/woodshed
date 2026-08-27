# Design notes

What was decided, why, and what turned out to be wrong. Written for whoever
builds the Android flash-cards app, which may well be a future me.

## The parts that transfer

The interaction model changes completely from web forms to flash cards, but the
domain does not. These four things were the expensive part and are worth lifting
whole.

### 1. Instruments carry a transposition offset

Eleven horns, each with the semitone distance from concert to written pitch:
C is 0, Bb is +2, G is +5, Eb is +9. Written key = concert key + offset, mod 12.

| Family | Instruments |
| --- | --- |
| Saxophone | soprano (Bb), alto (Eb), tenor (Bb), bari (Eb) |
| Clarinet | Bb, bass (Bb) |
| Flute | piccolo (C), flute (C), alto (G) |
| Brass | trumpet (Bb), trombone (C) |

A test asserted each offset agreed with the key the instrument is pitched in,
because a wrong offset makes every displayed key silently wrong and nothing else
catches it. Keep that test.

**Repertoire stores concert pitch; practice records written keys.** That split
matters. A tune is documented in concert (what the rhythm section calls), but
what you want to track is what you actually read on that horn, so playing a
ii-V-I in written G counts toward G whether it happened on alto or flute.

### 2. Keys need two spelling tables

Major prefers flats: C Db D Eb E F Gb G Ab A Bb B.
Minor prefers sharps: C C# D Eb E F F# G G# A Bb B.

Using one table produces key names no musician writes, like Db minor for what is
obviously C# minor. This was not obvious up front and is cheap to get right.

Parsing must be **fallible**. Real data contained `"Eb, C"` for On Green Dolphin
Street, which genuinely modulates. Returning `None` and showing the raw text beats
guessing, because a wrong transposition is invisible until you are on the stand.

The circle of fourths (C F Bb Eb Ab Db Gb B E A D G) is the order keys actually
get practiced in, so "next key" should walk it rather than picking at random.

### 3. Five phases give a session its shape

| Phase | Share | Categories |
| --- | --- | --- |
| Warmup | 15% | Tone, Air & Breathing, Articulation |
| Fundamentals | 20% | Technique, Scales & Patterns, Rhythm & Time, Etudes |
| Vocabulary | 25% | Licks, ii-V-I & Progressions |
| Repertoire | 30% | Standards |
| Cooldown | 10% | Improvisation, Transcription, Ear Training, Reading |

Phase order is load-bearing: when the time budget cannot cover every phase, drop
from the **end**, so a short session keeps its warmup and fundamentals rather
than its cool-down.

**This was never validated against real practice.** It is a guess built from a
verbal description, and it is the first thing to question. If you actually always
end on tunes, or never cool down, or warm up differently on doubles day, the model
is wrong in a way no test notices.

### 4. Repertoire selection has to invert the staleness bias

Everywhere else, "pick what you have neglected" is right: weight by days since
last practiced, treat never-practiced as maximally stale.

Repertoire is the exception. With ~201 tunes sitting at "wishlist", drawing by
staleness hands over a brand-new tune every session and nothing is ever learned.
The fix was to draw from tunes already in progress about 80% of the time and
introduce a new one the rest. That 80 was a guess and wants tuning against real
use.

This generalises directly to flash cards: a deck where every unseen card is
maximally urgent will never let you finish learning anything. Spaced repetition
solves the same problem more rigorously, and is probably the better frame for the
Android app than the staleness weighting used here.

## The data files

`data/standards.json` came from scraping [standardrepertoire.com](https://standardrepertoire.com)
(a reference site by David Miller), taking only factual metadata and deliberately
not lyrics or chord charts. Credit it if you ship it.

Field coverage across the 201 tunes, which is the useful part to know before
designing around it:

| Field | Distinct values | Absent |
| --- | --- | --- |
| form | 10 (AABA dominates at 76) | 44 |
| style | 12 | 1 |
| time_signature | 3 | 0 |
| common_key | 20 | 3 |

**Absence is common and meaningful.** 44 tunes have no documented form. Model
these as nullable and keep "unknown" distinguishable from "empty", or you will
lose the distinction permanently on first import.

Known quirks left as-is: `Latin/Swing` and `Swing,Ballad` are compound style
values, minor keys use a trailing hyphen (`F-`), sharps occur (`C#-`), and Take
Five was listed as 5/5 upstream and hand-corrected to 5/4.

`data/exercises.json` is 44 self-contained exercises, no method books needed.
`families` scopes an exercise to instrument families; **empty means it applies to
everything**, which is the common case. Only genuinely instrument-specific work
is scoped: overtones and altissimo to sax, harmonics to flute, register-break
crossings to clarinet, lip slurs to brass, double tonguing to brass and flute.

That scoping was my guess and is worth a second opinion. Plenty of sax players
double-tongue, for one.

## Traps that cost real time

Web-specific, but the shape of the mistake generalises.

**Unit tests that skip the real boundary prove nothing.** Instrument checkboxes
posted as a repeated field, which the web framework's form extractor cannot
deserialize into a list. It failed the entire request. The unit tests passed
throughout because they built the list directly and never touched the
deserializer. Test through the actual parser, not around it.

**A linter that cannot fail is decoration.** I wrote validation for the scraped
data and it looked green. It was green because my check was truncating its own
output, not because the checks fired. Deliberately break the input and confirm
the failure before trusting a validator.

**Verify the fix, not just the intent.** I fixed a select-height mismatch by
setting `line-height`, and re-measuring showed the value unchanged: Chromium
ignores that property on a `<select>`. Two more attempts followed because I kept
capping the wrong element. Measure after every change.

**Seed reconciliation must preserve ids if anything references the rows.** The
existing pattern in the codebase deleted and reinserted seed rows on every boot,
which it could afford because nothing pointed at them. Practice history pointed
at exercises by id, so that would have orphaned every past session. Upsert by
natural key instead, and archive rather than delete anything dropped from the
list.

**Decide what is author-owned before you first seed.** Re-seeding refreshed the
scraped metadata but had to leave mastery, notes, and archived state alone. Being
explicit about that boundary early avoided a class of bug entirely.

## What I would not carry over

- **The plan-is-the-log design.** A completed session plan doubled as the history
  entry, which was elegant for a web app with one form per page. A flash-card app
  has a different session shape and probably wants a plain event log.
- **The whole roll-a-plan concept.** It suits sitting down for a deliberate
  45-minute session. Flash cards suit short, frequent, interruptible reps. These
  are different products, and the planner may not survive the change.
- **Reference recordings as YouTube embeds.** Fine on a website, wrong on Android.
  If listening matters, use the platform's media handling.

# woodshed

Notes, data, and a build plan for **Woodshed**, an Android practice app for
horn players. Carried out of an abandoned attempt to build jazz practice tools
into [larakelley.com](https://github.com/lararosekelley/larakelley.com).

Nothing here runs yet. It is a design record, a plan, and two data files that
took real effort to produce.

## What is here

| Path | What |
| --- | --- |
| `DESIGN.md` | The decisions worth keeping, and the traps that cost time |
| `ANDROID_PLAN.md` | How this becomes the app: stack, domain model, build order |
| `data/standards.json` | 201 jazz standards with composer, year, bars, meter, form, style, key |
| `data/exercises.json` | 44 instrument-scoped practice exercises |

## Credit

`data/standards.json` is factual metadata scraped from
[standardrepertoire.com](https://standardrepertoire.com), a jazz standards
reference by **David Miller**. Only factual fields were taken — composer, year,
bars, meter, form, style, common key — deliberately not lyrics and not chord
charts. If any of this ships, that credit ships with it.

`data/exercises.json` is original, written for this project.

## Where the code is

Not here. It lives in ten closed-but-undeleted branches on the site repo,
`feat/practice-*`, stacked in this order:

```
scraper -> schema -> vocab -> library-pages -> seed
        -> recordings -> roller -> log -> e2e -> form-polish
```

Roughly 11,500 lines of Rust, Maud templates, and CSS, with 241 tests. It was
finished and green, not half-built: the reason it is not merged is that the
feature belongs in a phone app, not on a website.

Worth reading if you want it: `src/practice/roller.rs` (the session planner) and
`src/practice/keys.rs` (transposition) are the two files with real thinking in
them and no web coupling.

## Still open on the site repo

Issue #73, unrelated to practice: a production environment-detection bug, found
while working on this. Details are in the issue. Independent of whether any of
this ever gets built.

Three fixes were made to shared site CSS during this work and died with the
branches. Each stands alone if it is ever worth redoing:

- `header nav` has no `flex-wrap`, so with the admin-only links visible the page
  scrolls sideways at 768px.
- `.btn` has no `text-align`, so an `<a class="btn">` inherits its cell's
  alignment while a real `<button>` centres. They look different side by side in
  a right-aligned table cell.
- A `<select>` sizes to its widest option and ignores `line-height` in Chromium,
  so selects render shorter than text inputs and can overflow a narrow container.

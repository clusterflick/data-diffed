# data-diffed

This repository contains the automated workflow for recording what changed
between consecutive runs of the Clusterflick pipeline.

## Purpose

Every pipeline run publishes a fresh
[data-transformed](https://github.com/clusterflick/data-transformed) release,
but the release on its own says nothing about how it differs from the one before
it. This workflow compares the two most recent transformed releases and
publishes a single JSON blob describing the difference, so the site can offer a
"recent additions" feed and an RSS feed without recomputing the diff itself.

## How It Works

The workflow downloads the two most recent `data-transformed` releases and runs
the diff command:

```bash
npm run diff -- <current-tag> <previous-tag> <current-published-at>
```

This command:

- Reads both releases from `transformed-data/current` and
  `transformed-data/previous`
- Compares each venue's showings, matching performances by time so a shifted
  showtime reads as a reschedule rather than a removal plus an addition
- Ignores performances that had already happened when the current release was
  published — only what was still to come is a change worth reporting
- Writes `diffed-data/diffed-data.json` describing venues added, removed and
  emptied; showings added, removed and modified; and movie matches gained, lost
  and changed

The comparison logic lives in the shared
[scripts](https://github.com/clusterflick/scripts) repository, so the same diff
backs both this workflow and the release comparison report in
[data-analysed](https://github.com/clusterflick/data-analysed).

## What This Deliberately Does Not Track

The diff compares showings, performance **times**, and movie matches. It does
not compare the other per-performance fields — `format`, `accessibility`,
`status.soldOut`, `screen` or `notes`. A release where only those changed
produces no diff and therefore no release.

**This is a decision, not a gap.** When a screening's `format.source` gains
`imax-70mm`, the venue has announced nothing and nothing has become available —
we have simply started parsing a detail we previously missed. Reporting it would
make the feed a changelog of our own scraper rather than of cinema listings, and
we cannot distinguish a parser fix from a genuine venue re-announcement, so the
honest choice is to report neither. Discovery of formats belongs on the
dedicated format pages, which list them properly and sorted by date.

`status.soldOut` is excluded for a different reason: a feed exists for things a
reader can act on, and "you can no longer buy this" is not one. The case that
_does_ matter — a venue adding an extra screening because the first sold out —
is a new performance time, so it is already reported as an addition.

The one acknowledged edge: more tickets released for an _existing_ sold-out
performance (same showing, same time, `soldOut` flipping back to false) is
invisible here. That is genuinely actionable, unlike the rest, but it is rare
enough to be worth confirming against real data before building for it.

## Output

Two assets per release: `diffed-data.json`, the change set described below, and
`seen-registry.json`, described under [Seen Registry](#seen-registry).

`diffed-data.json`:

```json
{
  "metadata": {
    "currentRelease": "20260726.031204",
    "previousRelease": "20260725.031157",
    "asOf": "2026-07-26T03:20:11.482Z",
    "venueCount": 335
  },
  "summary": {
    "totalVenues": 335,
    "venuesAdded": 1,
    "venuesRemoved": 0,
    "venuesEmpty": 2,
    "showingsAdded": 84,
    "showingsRemoved": 3,
    "futurePerformancesAdded": 512,
    "futurePerformancesRemoved": 19,
    "tmdbMatchesGained": 7,
    "tmdbMatchesLost": 1,
    "tmdbMatchesChanged": 2
  },
  "venues": {
    "princecharlescinema.com": {
      "name": "Prince Charles Cinema",
      "venueAdded": false,
      "venueRemoved": false,
      "venueEmpty": false,
      "showings": { "added": [], "removed": [], "modified": [] },
      "futurePerformances": {
        "previousTotal": 210,
        "added": 12,
        "removed": 0,
        "rescheduled": 1
      },
      "tmdbChanges": { "gained": [], "lost": [], "changed": [] }
    }
  }
}
```

`currentRelease` and `previousRelease` are the `data-transformed` release tags
being compared, and `venueCount` counts the venues compared rather than the
venues that changed — `summary` describes the whole comparison even though
`venues` only lists what moved.

`asOf` is when the current release was published, and is what "still to come"
was measured against. It is deliberately not the time the diff ran: no wall
clock reaches the output, so re-running a given pair of releases produces a
byte-identical blob.

Venues are keyed by venue id — the same file name used across the pipeline.
Times are epoch milliseconds, as everywhere else in the pipeline.

Each showing entry carries enough to be rendered on its own (`title`, `url`,
`category`, `seen`, and the matched `themoviedb` / `themoviedbs` id, title and
release date) without joining against the combined release.

## Seen Registry

`seen-registry.json` exists so that a film whose run has ended keeps its page on
the website. A movie drops out of the transformed data the moment its last
performance has been and gone, which used to take its page with it — every link
to it, indexed or shared, started returning a 404.

```json
{
  "metadata": {
    "release": "20260808.180256",
    "retentionDays": 365,
    "showingCount": 1361,
    "departedCount": 402
  },
  "movies": {
    "550": { "lastSeen": "20260808.180256", "lastPerformance": 1754900000000 }
  }
}
```

It records `lastSeen` for **every** movie, showing or not, rather than only for
the ones that have gone. That is what keeps it a function of one transformed
release plus its own previous state: departure is not an event to be spotted by
comparing two releases, it is simply an entry whose `lastSeen` stopped
advancing. A movie that returns is stamped again and stops being departed on its
own, with no removal path to get wrong, and re-running the same release produces
the same registry.

Consumers should not read departure off `lastSeen`, though — subtract the ids
they can already see in the live data instead, which stays correct even when a
run publishes nothing. `lastSeen` is for ageing entries out; `lastPerformance`
is the last time the film could actually have been watched.

Only root-level `themoviedb` matches are recorded. The `themoviedbs` of a
multiple-movies event are folded into their parent by the combine stage rather
than given pages of their own, so treating them as seen would mint pages for
movies that never had one. Unmatched events are not recorded at all: there is no
TheMovieDB record to rebuild a page from.

`retentionDays` is a work budget, not a storage one — the file is tens of
kilobytes either way, but every entry costs a TheMovieDB fetch in the cache
stage and a static page at build time.

## Schedule

The workflow is automatically triggered when the
[data transformation workflow](https://github.com/clusterflick/data-transformed)
completes successfully. It can also be triggered manually via workflow dispatch
— it works out which two releases to compare on its own.

**No release is created when nothing changed.** A run that finds no differences
finishes successfully and leaves the previous release as the latest. The seen
registry is built either way, but is published only as part of a release: a run
with nothing to report also has nothing departing, so the previous registry is
still current and the next run carries forward from it.

## Downstream Triggers

- [data-cached](https://github.com/clusterflick/data-cached) - To cache
  TheMovieDB data for the release, including the departed movies only the seen
  registry knows about

This fires whether or not a release was created, so an unchanged diff does not
stall the rest of the pipeline.

## Maintenance

### Dependencies

The workflow requires API keys configured as GitHub secrets:

- `PAT` - Personal Access Token for reading `data-transformed` releases

## Licence

The code in this repository is licensed under the [MIT licence](LICENSE).

The releases are **not licensed at all**. `diffed-data.json` is an internal
build artifact — it exists so the website can render its New Listings feed and
RSS without recomputing the diff, and it inherits the third-party metadata the
transformed releases carry. It has no schema guarantees and changes without
notice.

The underlying screening data _is_ licensed. For data you can use, see the
[data licence](https://clusterflick.com/data-licence). The exact terms for this
repository are in [LICENSE-DATA](LICENSE-DATA).

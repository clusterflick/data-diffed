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

## Output

A single `diffed-data.json` asset per release:

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

## Schedule

The workflow is automatically triggered when the
[data transformation workflow](https://github.com/clusterflick/data-transformed)
completes successfully, in parallel with
[data-cached](https://github.com/clusterflick/data-cached) and
[data-calendar](https://github.com/clusterflick/data-calendar). It can also be
triggered manually via workflow dispatch — it works out which two releases to
compare on its own.

**No release is created when nothing changed.** A run that finds no differences
finishes successfully and leaves the previous release as the latest.

## Downstream Triggers

None. Nothing consumes this data automatically yet.

## Maintenance

### Dependencies

The workflow requires API keys configured as GitHub secrets:

- `PAT` - Personal Access Token for reading `data-transformed` releases

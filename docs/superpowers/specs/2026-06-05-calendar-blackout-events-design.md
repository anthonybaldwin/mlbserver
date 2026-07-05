# Calendar: show blacked-out games as link-less events

**Date:** 2026-06-05 (implemented 2026-07-05)
**Status:** Implemented — extended to provider-aware three-way labeling
**Scope:** `calendar.ics` output only (`getTVData` with `dataType == 'calendar'`)

> **Implementation note (2026-07-05):** shipped with a three-way badge instead
> of the single `[BLACKOUT]` this doc originally specified. National feeds are
> distinguished from geographic blackouts via `broadcast.isNational`, and the
> national badge surfaces the carrier (`[PEACOCK]`) when `broadcast.callSign`
> is present, falling back to `[NATIONAL]`. Only a true (regional) blackout
> lifts, so only `[BLACKOUT]` carries a lift-time note. See §2 below.

## Problem

Today, when `includeBlackouts == 'false'` (the default), `getTVData` **skips**
any broadcast/feed whose `mediaId` is in the game's `blackout_feeds`
(`session.js:2784`, `continue`). A blacked-out game silently vanishes from the
subscribed calendar, so the user has no idea a game exists or why they can't
watch it on MLB.tv.

We want the opposite for the calendar: **keep the event**, but make it
unmistakably a blackout — no playable link, and a note saying when the
blackout lifts and where the game is actually viewable.

## Goal

For `calendar.ics` only, a blacked-out feed produces a VEVENT that:

- Has a state badge in the SUMMARY (drops the misleading "Watch" prefix).
  Three-way: `[PEACOCK]`/carrier name for an identified national feed,
  `[NATIONAL]` for a national feed with no `callSign`, `[BLACKOUT]` for a
  regional/RSN blackout.
- Carries **no playable link** anywhere (no `Watch:` line, no `Alternate:`
  line, no stream-URL fallback in `LOCATION`).
- Notes the blackout type, the approximate lift time, and where the game is
  viewable, e.g.:
  `Video blackout until approx. 2.5 hours after the game (~10:23 PM): Apple TV`
- Has **no VALARM reminder** (nothing to watch at game start on MLB.tv).

`channels.m3u` and `guide.xml` are **unchanged** — they keep skipping
blacked-out feeds (an M3U entry is fundamentally a URL; a link-less channel is
meaningless there).

`&includeBlackouts=true` keeps its current meaning unchanged: emit the real
playable stream links anyway (the "try it anyway" escape hatch).

## Key facts about the existing code

- **One pass, selective output.** `getTVData` builds the channel list, the XML
  program list, and the ICS `calendar` string in a single loop, then returns
  only the one matching `dataType` (`session.js:3495-3525`). Letting a
  blacked-out feed fall through on a `calendar` request and build just its ICS
  event has **zero effect** on M3U/XML output — those structures are discarded
  for a calendar request.

- **Single skip point.** `session.js:2784`:
  ```js
  if ( (includeBlackouts == 'false') && blackouts[gamePk] && blackouts[gamePk].blackout_feeds
       && blackouts[gamePk].blackout_feeds.includes(broadcast.mediaId) ) {
    continue
  }
  ```

- **Where the link lives.** `generate_ics_event` (`session.js:5843`) surfaces the
  link in three places:
  - `Watch: <streamUrl>` line in DESCRIPTION (only when `streamUrl && streamUrl !== location`)
  - `Alternate: <altLocation>` line in DESCRIPTION
  - `LOCATION:` — which falls back to `streamUrl` when venue is unknown
    (`locationField = venue || streamUrl`, `session.js:3065`)

  Link-less therefore means: `streamUrl = ''`, `altLocation = ''`, and
  `LOCATION = venue` (no URL fallback).

- **Expiry + "where" are free.** `get_blackout_expiry` (`session.js:5794`) and
  `get_scheduled_innings` (its only dependency) are pure — no network. The
  blacked-out feed's network name is already in scope as `station`
  (`broadcast.callSign`, `session.js:2798`). Both can be computed inline in the
  calendar loop. The current calendar path calls `get_blackout_games()` with no
  args and `calculate_expiries=false`, so it does **not** compute expiries
  today; we add the expiry inline instead of changing `get_blackout_games`
  (whose expiry-matching loop only handles a single date and would be wrong for
  the multi-day calendar window).

## Design

### 1. Gate the skip by `dataType` (`session.js:2784`)

```js
let isBlackout = false
if ( (includeBlackouts == 'false') && blackouts[gamePk] && blackouts[gamePk].blackout_feeds
     && blackouts[gamePk].blackout_feeds.includes(broadcast.mediaId) ) {
  if ( dataType != 'calendar' ) {
    continue            // M3U / XML: unchanged — skip blacked-out feeds
  }
  isBlackout = true     // calendar: keep, render link-less below
}
```

`isBlackout` is consumed only in the MLB calendar-event block. Channel-object
and XML-program creation in the loop are left untouched; for a `calendar`
request they are discarded, so guarding them is unnecessary.

### 2. Render the calendar event link-less when `isBlackout` (≈ `session.js:3035-3086`)

When `isBlackout`:

- **Stream URL:** `streamUrl = ''` (skip the whole `embed.html?...` construction).
- **Location:** `locationField = venue` only — do **not** fall back to
  `streamUrl`. Empty `LOCATION` is acceptable when venue is unknown.
- **Alternate:** pass `altLocation = ''` (not `buildAltLoc(...)`).
- **Prefix:** force `prefix = ''` (no "Watch"/"Listen").
- **Badge (three-way):** discriminate on `broadcast.isNational`, then prepend
  to `subtitle`:
  - national + `station` → `[<STATION.toUpperCase()>] ` (e.g. `[PEACOCK] `)
  - national + no `station` → `[NATIONAL] `
  - otherwise → `[BLACKOUT] `
- **Note:** append a blackout note to `description` (built below).
- **Alarm:** suppress the VALARM (see §3).

Blackout note construction (as implemented — keyed off `isNational`, not
`blackout_type`, because the feed being rendered is the direct signal):

```js
let blackoutNote = ''
if ( isBlackout ) {
  if ( broadcast.isNational == true ) {
    subtitle = (station ? ('[' + station.toUpperCase() + '] ') : '[NATIONAL] ') + subtitle
    blackoutNote = station ? ('National broadcast — watch on ' + station) : 'National broadcast (not on MLB.TV)'
  } else {
    subtitle = '[BLACKOUT] ' + subtitle
    blackoutNote = 'Video blackout until approx. 2.5 hours after the game'
    const expiry = await this.get_blackout_expiry(cache_data.dates[i].games[j])
    if ( expiry ) blackoutNote += ' (~' + expiry.toLocaleString('en-US', { hour: 'numeric', minute: 'numeric', hour12: true }) + ')'
    if ( station ) blackoutNote += ': ' + station
  }
}
// appended to description after the suspended/TBD handling:
if ( isBlackout && blackoutNote ) description += blackoutNote
```

- `station` is `broadcast.callSign` (`session.js`), which for a national
  exclusive is the carrier ("Peacock", "Apple TV+") and doubles as "where it's
  viewable". Omit the `: <station>` suffix (RSN case) and the national
  "watch on" clause when `station` is falsy.
- Only a real (regional) blackout lifts, so only that branch computes an
  expiry / `(~time)` clause; national exclusives never do.

### 3. Suppress the VALARM for blackout events (`generate_ics_event`, `session.js:5843`)

Add a trailing parameter, e.g. `includeAlarm = true`, and emit the
`BEGIN:VALARM … END:VALARM` block only when it is true. The MLB calendar call
site passes `false` when `isBlackout`, `true` otherwise. All other call sites
keep the default (`true`) and are unchanged.

### 4. Keep SEQUENCE correct across blackout transitions (`stateHash`, `session.js:3076-3083`)

Add the blackout state to the hash so a change (feed becomes available, or a
non-blacked game becomes blacked) bumps `SEQUENCE` and pushes an update to
subscribers:

```js
const stateHash = JSON.stringify({
  s: ...,
  t: subtitle,          // already includes the [BLACKOUT] badge when isBlackout
  d: ...,
  e: ...,
  v: venue,
  u: streamUrl,         // '' when isBlackout — already differs from the linked state
  b: isBlackout
})
```

`streamUrl` going from a URL to `''` already perturbs the hash; `b` and the
badged `subtitle` make the intent explicit and cover the edge where only the
note text changes.

## Output example

A blacked-out national-exclusive game (Peacock), default calendar:

```
BEGIN:VEVENT
UID:mlb-822715@mlbserver
SEQUENCE:0
SUMMARY:[PEACOCK] Cubs @ Cardinals
DTSTART:20260705T163500Z
DTEND:20260705T193500Z
DESCRIPTION:Peacock. <pitchers etc.>. National broadcast — watch on Peacock
LOCATION:Wrigley Field
END:VEVENT
```

A regional/RSN blackout instead reads `SUMMARY:[BLACKOUT] …` with a
`Video blackout until approx. 2.5 hours after the game (~2:43 PM): Marquee`
note. Either way: no `Watch:`/`Alternate:` lines, no URL in `LOCATION`, no
`VALARM`.

## Out of scope / known pre-existing edges

- **Multiple feeds per game share one UID.** The calendar already keys UID on
  `gamePk` (`session.js:3075`), so a game with several qualifying broadcasts
  emits multiple VEVENTs with identical UID — clients dedupe. Blackout events
  follow the same existing pattern; typical single-team calendars match one
  feed per game. Not addressed here.
- `channels.m3u` / `guide.xml` behavior — intentionally unchanged.
- No change to `getMediaId` / the playback path — only calendar rendering.

## Testing

- **Unit-ish:** a blacked-out feed on a `calendar` request emits a VEVENT with
  a state badge, no `Watch:`/`Alternate:`/URL-`LOCATION`, and no `VALARM`.
- **Three-way label:** national feed with `callSign` → `[PEACOCK]` +
  `National broadcast — watch on Peacock`; national feed w/o `callSign` →
  `[NATIONAL]`; regional blackout → `[BLACKOUT]` + `Video blackout … (~time):
  <station>` note (only this branch has a lift time).
- **Regression:** the same blacked-out feed on `channels`/`guide` requests is
  still skipped (no channel, no programme).
- **Override:** `&includeBlackouts=true` still emits real playable links for the
  same feed (unchanged).
- **Not-entitled:** note reads `Not entitled: <station>` with no `(~time)`.
- **Non-blackout events:** SUMMARY, `Watch:` line, `LOCATION` fallback, and
  `VALARM` all unchanged.

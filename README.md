# Gallery Console — Indoor Bowls, Glasgow 2026

A single-file producer console for the world feed gallery: real match
schedule and commentator rota, live score entry, and a Gallery Wall that
tracks each rink through its session automatically.

One file: `index.html`. No `npm install`, no build step, no framework
tooling. React and Firebase load from CDN links at the top of the file.
The app's JSX is precompiled to plain JavaScript and inlined directly —
no in-browser Babel step, so there's nothing left that can break due to a
CDN version change.

If you ever edit the component code by hand, note it's plain
`React.createElement(...)` calls rather than JSX — a one-time tradeoff
for not depending on a live Babel CDN script during the event.

Real match schedule and commentator rota (all 11 days) are inlined in the
file as the `SCHEDULE` and `COMMENTATORS` constants. Everything else that
changes during the event — scores, match status, producer notes — is
live data, held in the `scores` object and synced through Firebase.

## Firebase — configured and live

Firestore and Anonymous auth are set up and enabled on the `bowls-b6ed9`
project; `FIREBASE_CONFIG` near the top of the `<script>` section already
has real values. Every score edit, status change, and producer note
writes to Firestore and pushes to any other device with the page open,
keyed by a schedule-derived match ID (`d{day}-s{session}-r{row}-k{rink}`).

Firestore's offline persistence is also on (`enablePersistence` with
`synchronizeTabs: true`), so a dropped connection queues edits in the
browser's IndexedDB and replays them automatically once it's back —
nothing to do differently if the venue wifi flakes mid-session. The
**Sync** tab shows live online/offline status and which mode you're in.

If `FIREBASE_CONFIG.apiKey` were ever blanked out, the app falls back to
this browser's local storage only — same UI, no other code changes
needed either way.

The API key visible in the config is a Firebase Web API key, which is
meant to be public — it identifies the project, not a credential. Actual
write access is gated by Firestore's rules (`firestore.rules`), which
require anonymous auth.

## Hosting

**Hosting is GitHub Pages, not Firebase Hosting.** Firebase is used only
for the Firestore database — it doesn't host the site itself. Updating
the site is just editing `index.html` in the GitHub repo (even from a
browser on a phone/iPad, no CLI needed) and committing; GitHub Pages
republishes automatically.

1. Push this folder to a GitHub repo.
2. Repo → Settings → Pages → deploy from the `main` branch (root).
3. It's live at `https://YOUR-USERNAME.github.io/YOUR-REPO-NAME`.

An earlier version of this project was briefly deployed to Firebase
Hosting at `bowls-b6ed9.web.app`. That URL still technically works (same
Firestore data behind it) but isn't being updated — nobody has the link,
so it's safe to ignore. `firebase.json` / `.firebaserc` in this folder are
left over from that and only matter now for the `firestore.rules` deploy
command below.

To (re-)deploy Firestore rules after editing `firestore.rules`:
`npx firebase-tools deploy --only firestore:rules`.

## Add it to the iPad home screen

The app has a `manifest.json` and touch icons so it can run full-screen,
without Safari's address bar, once installed:

1. Open the hosted URL in Safari on the iPad.
2. Share → Add to Home Screen.
3. Launch it from the home screen icon from then on.

Swap `apple-touch-icon.png` / `icon-192.png` / `icon-512.png` for a real
logo any time — same filenames, no other changes needed.

## How the Gallery Wall works

Each rink (1–4) tracks its own position through the currently selected
session independently — there's no single shared "row" all four rinks
step through together. A tile shows whichever match on that rink hasn't
been marked final yet (the first non-final match in that rink's session
list); once a producer marks a match final in the detail view, the tile
advances to the next one on its own. If every match on a rink is already
final, the tile just keeps showing the last one.

Rinks 1 and 4 are the world-feed rinks, and their tiles carry a
commentator chip pulled from the schedule's `tv1`/`tv4` rota for that
match (`commentatorLabel()` turns a code like `P&B` into "Paul + Barry").
Rinks 2 and 3 are in-venue only and never carry a commentator chip.

Every tile — world feed or venue — permanently lists the rest of that
rink's matches for the session below it (time, matchup, commentator where
relevant); there's no expand/collapse toggle any more. Clicking one of
those rows doesn't open the detail view — it changes the "focus" for that
rink, swapping that match into the main tile above (and the previously
focused match drops back into the list). This focus is a manual override
on top of the normal auto-advance and resets whenever you change day or
session. This is built from `buildRinkSessionMatches()`, which walks every
row of a session for one rink index and returns matches in time order.

The session picker (AM/PM) is still manual — only the within-session
row-stepping is automatic (or manually overridden via focus, as above).

Score entry no longer requires opening the detail view either: the two
world-feed tiles (rinks 1 & 4) have inline +/− steppers next to each team
for the current set, and the two venue-only tiles (rinks 2 & 3) have
direct number inputs for the same. Every tile also has its own Standby /
Live / Next set (or Tiebreak) / Final buttons (`TileStatusControls`), so
the full match lifecycle — not just the score — is driven from the wall.
The detail view's score editor and status controls are still there for
finer control (and for producer notes), but the Gallery Wall itself is
now enough for routine match management. The old "CAMERA · RINK FEED"
placeholder box in the detail view — a mockup for an embedded broadcast
preview — has been removed; the app was never going to be able to embed
the actual world-feed video, so it was just clutter.

Every tile shows the full set-by-set score line (`ScoreLine`), not just
the current set, so previous sets stay visible once you move on to the
next one — on the compact venue tiles this sits just above the score
inputs. Once a match is marked final, the winning team is called out
visually: on the full tiles the winning row is bolded with a "WINNER"
tag next to the name; on the compact tiles there's a "WINNER: [nation]"
line underneath, since there's no room there for two separate rows
(`matchWinner()` works out the winner from the set scores, accounting for
the 2-set-plus-tiebreaker format below).

## Scoring format

Ends-per-set is derived per match from its discipline code, not a single
global constant (`endsPerSet()`): singles (Men's/Women's Singles) play two
sets of 7 ends; pairs and every para discipline play two sets of 5 ends.
If those two sets split — one win each — a third, 1-end tiebreaker set
decides the match; the "Next set" button in the detail view becomes
"Start tiebreaker" when a match reaches that point, and disappears
entirely once the tiebreaker's been played (or once one side has won both
of the first two sets and no tiebreaker is needed). This logic lives in
`endsPerSet()` / `needsTiebreak()` / `canAddSet()`, so if these rules
change again it should just be those three functions.

## Standings

Computed live from real results, not hand-entered — but with a real
caveat: the source schedule has no pool/section grouping (no "Section A"
boundaries anywhere in the data), so this isn't an official sectional
table. Pick a discipline from the dropdown and it aggregates every match
marked final ("FT") for that discipline across all days into a running
W/L/shots/points record per nation (`buildDisciplineStandings()`), using
the standard bowls convention of 2 points for a win, 0 for a loss, shot
difference as the tiebreak. It'll need adjusting once the actual
sectional draw and points system are confirmed — that logic is isolated
in one function, so it should be a contained change rather than a
rewrite.

## Backup

The Sync tab has a "Download backup (JSON)" button — a manual snapshot
of every score, status, and note currently held in `scores`. There's no
other live-results source for this event, so this is the safety net if
Firestore has a problem mid-session.

## What's still manual, deliberately

- Live results still need a human — no public live-scores feed exists for
  this event as far as I could find, so the inline steppers/inputs on the
  Gallery Wall (or the score editor in the detail view) are the entry
  point.
- The session picker on the Gallery Wall (AM vs PM) is manual; within-session
  match progression auto-advances, but can also be manually overridden by
  clicking a match in a tile's "rest of session" list.

## Known open question

The commentator legend (Schedule tab) flags a code, `SIAN`, that isn't in
the original commentator rota — worth confirming with the production
team before relying on it.

## Not built yet

Athlete names. The schedule data only carries nation codes (`ENG`, `SCO`,
etc.) — no individual player names — so tiles and the detail view only
ever show country. Adding names means extending the schedule data model
first (either hand-entered per match, or a lookup table keyed by
discipline/nation if squads are fixed for the event) before any UI work.

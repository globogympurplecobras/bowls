# Gallery Console — Indoor Bowls, Glasgow 2026

One file: `index.html`. No `npm install`, no build step, no framework
tooling. React and Firebase load from CDN links at the top of the file.
The app's JSX is precompiled to plain JavaScript and inlined directly —
no in-browser Babel step, so there's nothing left that can break due to a
CDN version change.

If you ever edit the component code by hand, note it's now plain
`React.createElement(...)` calls rather than JSX — that's a one-time
tradeoff for not depending on a live Babel CDN script during the event.

Real match schedule and commentator rota (all 11 days) are already inlined
in the file. Everything else — scores, status — lives in this one file's
data and, by default, in this browser's local storage.

## Right now: zero setup

This works today with nothing configured. Open `index.html` in a browser
(or host it — see below) and it runs, using this device's local storage to
remember score edits between reloads. That's genuinely it.

## To put it online (so you can open it from anywhere, still single-device)

Simplest option — Firebase Hosting, one command, no build:

1. `npm install -g firebase-tools` (one-time, needs Node — this is the only
   tool you need, not a build system)
2. `firebase login`
3. Create a project at the [Firebase console](https://console.firebase.google.com) (free tier is plenty)
4. Replace `REPLACE-WITH-YOUR-FIREBASE-PROJECT-ID` in `.firebaserc` with that project's ID
5. From this folder: `firebase deploy`
6. It's live at `https://YOUR-PROJECT-ID.web.app`

Any static host works too (Netlify, a GitHub Pages branch, even just
double-clicking the file locally) — Firebase Hosting is just the one that
lines up with the sync option below if you want it later.

## Later, if you want another device to see the same live scores

1. In the same Firebase project: **Firestore Database** → create database.
2. **Authentication** → enable the **Anonymous** sign-in provider.
3. Project settings → General → "Your apps" → add a **Web app** → copy the config object it gives you.
4. Open `index.html`, find the `FIREBASE_CONFIG` block near the top of the
   `<script>` section, and paste the values in.
5. Reload the page. The Sync tab will show "Connected", and every score
   edit now writes to Firestore and pushes to any other device with the
   same page open.

No other code changes are needed for this — the app already checks whether
`FIREBASE_CONFIG.apiKey` is filled in and switches from local storage to
Firestore automatically.

To keep rules enforced once you do this: add a `firestore` block to
`firebase.json` pointing at `firestore.rules` (see the comment in that
file), then `npx firebase-tools deploy --only firestore:rules`.

## What's still manual, deliberately

- Live results still need a human — no public live-scores feed exists for
  this event as far as I could find, so the score stepper in the app is the
  entry point.
- Producer notes (in the match detail view) aren't saved anywhere yet —
  add that alongside Firebase sync whenever it's worth the effort.

# Flappy Bird

A single-file HTML5 Canvas game with an optional Firebase-backed global leaderboard.

Everything ships in `index.html` - no build step, no bundler, no framework. Open the file and play.

## Features

- Fixed-timestep physics, so the game plays identically on 60Hz and 144Hz displays
- Keyboard, mouse and touch controls
- Locally generated sound effects (WebAudio, no audio files)
- Personal best saved to `localStorage`
- Global leaderboard: top 10 players plus your live rank
- Works fully offline - the online features are loaded lazily and never block the game

## Running it

Open `index.html` in a browser, or serve the folder:

```bash
npx serve .
```

The game is a static page, so any static host works: GitHub Pages, Netlify, Vercel, Cloudflare Pages, or plain `python -m http.server`.

## Controls

| Action | Keys |
| --- | --- |
| Start / flap | `Space`, `ArrowUp`, or tap / click the canvas |
| Mute | `M` |
| Submit your name | `Enter` (in the username dialog) |
| Close a dialog | `Esc` |

A dialog owns the keyboard while it is open, and the run is frozen behind it, so you cannot die while reading the leaderboard.

## Online leaderboard

The leaderboard is optional. If Firebase is unreachable or disabled, the game logs a warning and keeps working offline.

### What it uses

- **Anonymous Auth** - signs in with no account, no email, no password
- **Cloud Firestore** - one document per player at `players/{uid}`
- **No Analytics** - no tracking of any kind

The Firebase SDK is loaded on demand from the pinned `12.19.0` CDN bundles. If you would rather bundle the npm packages instead, they are already listed in `package.json`; nothing in the page needs them today.

### Player document

```json
{
  "uid": "firebase-auth-id",
  "username": "Zed",
  "bestScore": 42,
  "bestScoreAt": "2026-01-01T00:00:00Z",
  "createdAt": "2026-01-01T00:00:00Z",
  "updatedAt": "2026-01-01T00:00:00Z"
}
```

Usernames are 3-20 characters, may include letters from any script and emoji, and are rendered with `textContent` so they can never inject markup.

### Ranking

The top 10 comes from a single `orderBy('bestScore', 'desc')` query. Your rank is `1 + the number of players with a strictly higher best score`, which makes it a **competition ranking**: tied players share a rank and the next rank skips accordingly.

### Deploying the security rules

`firestore.rules` is what actually protects the data. Until it is deployed, Firestore denies all reads and writes and the leaderboard stays empty.

```bash
npm install -g firebase-tools
firebase login
cd "path/to/flappy bird"
firebase deploy --only firestore:rules
```

The rules enforce:

- only signed-in players can read the board
- a player can only create, update, or read their own document
- `uid` and `createdAt` can never be changed after creation
- `bestScore` can only go up, and `bestScoreAt` must move with it
- deletes are denied outright
- usernames and scores are range- and type-validated

Because the write path is owner-only and the server compares against the stored value, editing the score in devtools does not help - the client cannot write a lower score than it already has.

## Project layout

```
index.html        the entire game
firestore.rules   Firestore security rules
firebase.json     Firebase config (declares the rules)
.firebaserc       default Firebase project
package.json      Firebase npm dependencies
```

## Configuration

The Firebase web config lives in `index.html` under `firebaseConfig`. A web API key is not a secret - it is shipped to every browser that loads the page, and access is controlled entirely by the security rules. Your data is protected by `firestore.rules`, not by hiding this key.

To point the game at a different project, either edit `firebaseConfig` or set the default project for the CLI in `.firebaserc`.

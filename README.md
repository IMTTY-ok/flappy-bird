# Flappy Bird

A single-file HTML5 Canvas game with an optional Firebase-backed global leaderboard. Every run is stored as its own game session under the name you played it with.

Everything ships in `index.html` - no build step, no bundler, no framework. Open the file and play.

## Features

- Fixed-timestep physics, so the game plays identically on 60Hz and 144Hz displays
- Keyboard, mouse and touch controls
- Locally generated sound effects (WebAudio, no audio files)
- Personal best saved to `localStorage`
- A fresh name for every single run - each game is stored as its own session
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
| Submit your name | `Enter` (in the name dialog) |
| Close the leaderboard | `Esc` |

A dialog owns the keyboard while it is open, and the run is frozen behind it, so you cannot die while reading the leaderboard. The name dialog is the one exception: it cannot be dismissed with `Esc`, because a run cannot start without a name.

## How a run works

1. You press play and the **name dialog** opens. Every run asks again - the name is not remembered as "your" name, it belongs to that one game.
2. The name is trimmed, stripped of control characters and angle brackets, and must be 2-20 characters (counted in code points, so an emoji counts as one).
3. The game waits up to 5 seconds for anonymous sign-in, writes a `gameSessions` document, and only then starts the run.
4. On game over the session is closed, your personal best is updated if - and only if - this run beat it, and your global rank is shown.

If Firebase is slow or unreachable the name is still required, the run still starts, and the session is simply skipped. The game never waits on the network.

## Online leaderboard

The leaderboard is optional. If Firebase is unreachable or disabled, the game logs a warning and keeps working offline.

### What it uses

- **Anonymous Auth** - signs in with no account, no email, no password
- **Cloud Firestore** - one document per player at `players/{uid}`, and one document per run at `gameSessions/{id}`
- **No Analytics** - no tracking of any kind

The Firebase SDK is loaded on demand from the pinned `12.19.0` CDN bundles. If you would rather bundle the npm packages instead, they are already listed in `package.json`; nothing in the page needs them today.

### Player document

One document per player, holding only their best run:

```json
{
  "uid": "firebase-auth-id",
  "bestScore": 70,
  "bestScoreName": "Rahul",
  "bestScoreAt": "2026-01-01T00:00:00Z",
  "createdAt": "2026-01-01T00:00:00Z",
  "updatedAt": "2026-01-01T00:00:00Z"
}
```

`bestScoreName` is the name that was used **on the run that set `bestScore`** - it is never the most recent name. A later run with a different name only changes it if that run also scored higher, so the leaderboard always shows who owns the score. The update runs in a transaction, so two tabs finishing at once cannot lose a best score.

### Game session document

One document per run, written when the run starts:

```json
{
  "uid": "firebase-auth-id",
  "playerName": "Rahul",
  "startedAt": "2026-01-01T00:00:00Z",
  "finalScore": 0,
  "completed": false
}
```

and updated once at game over:

```json
{
  "finalScore": 12,
  "completed": true,
  "endedAt": "2026-01-01T00:01:30Z"
}
```

`uid`, `playerName` and `startedAt` are immutable, so a session always records the name the run actually started with. Firestore generates the id, which is why every run gets its own document.

Names are 2-20 characters, may include letters from any script and emoji, and are rendered with `textContent` so they can never inject markup.

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
- a player can read and write only their own document and their own sessions
- `uid` and `createdAt` can never be changed after creation
- `bestScore` can only go up, and `bestScoreName` and `bestScoreAt` must move with it
- a session's `uid`, `playerName` and `startedAt` are immutable; only `finalScore`, `completed` and `endedAt` can be written
- a completed session can never change its score again
- deletes are denied outright
- names and scores are range- and type-validated

Because the write path is owner-only and the server compares against the stored value, editing the score in devtools does not help - the client cannot write a lower score than it already has. Note that this is leaderboard-grade protection, not anti-cheat: the score itself is still computed on the client, so a determined player could report any score from a modified client.

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

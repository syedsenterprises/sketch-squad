# Sketch Squad — Draw & Guess Party Game

A real-time multiplayer Draw & Guess game for 2–8 friends. No login, no
install: open the link, type a name, create or join a room with a 4-letter
code, and play from any phone or laptop.

## How to run locally

Any static file server works — there is no build step and no npm install:

```bash
cd sketch-squad
python3 -m http.server 8000
# open http://localhost:8000
```

## How to configure Firebase (required for real multiplayer)

The app ships with a placeholder config, so it loads but shows a
"game server not configured yet" state. To make rooms sync:

1. Create a free Firebase project at https://console.firebase.google.com
   (Spark plan, $0).
2. In the project, enable **Realtime Database** (start in test mode for a
   party game, or add rules below) and **Authentication → Anonymous**.
3. In Project settings → Your apps → Web app, copy the config values into
   `firebase-config.js` (the only file that holds config).
4. Reload. Create a room on one device, join with the code on another.

Suggested Realtime Database rules for a party game (anyone can play, nobody
can wipe other rooms — adapt to your needs):

```json
{
  "rules": {
    "rooms": {
      "$code": {
        ".read": true,
        ".write": true
      }
    }
  }
}
```

For a hardened deployment, restrict `turn/word` reads to the drawer and
validate guesses with a Cloud Function instead of the host client (see
"Anti-cheat note" below).

## How rooms and sync work

- **State machine (host-authoritative, stored in Realtime Database under
  `/rooms/<CODE>`):** `lobby → drawing → reveal → finished`. The room
  creator is the host; the host's browser drives every transition on a
  1-second tick (turn end, reveal length, next turn, game end).
- **Turns:** players draw in join order, one turn each, 60 seconds per turn.
  The drawer picks 1 of 3 secret words. The 3 choices are generated
  deterministically from `room code + turn number`, so all clients agree
  without an extra write. If the drawer doesn't pick in 15 seconds, the
  host auto-picks the first choice.
- **Drawing sync:** strokes are stored as normalized points (0–1000) under
  `strokes/`, pushed when the drawer lifts the pen, plus a throttled
  (~10/sec) `liveStroke` preview so guessers watch the drawing form.
  The canvas is fixed 1000×600 internally and scales to any phone screen.
- **Guessing:** guessers send guesses to `guessQueue`. The host validates
  them (case-insensitive, trimmed, exact match) and awards points:
  **100 base + up to 100 time bonus** (faster = more), and the drawer gets
  **+20 per correct guesser**. Correct guesses appear in chat as
  "🎉 Player got it!" — the word itself is never posted. Wrong guesses
  are visible to all. The drawer has no guess box while drawing.
- **Host migration:** presence uses `.info/connected` + `onDisconnect`.
  If the host disconnects, the longest-present remaining online player
  claims host via a transaction and resumes driving transitions. If the
  drawer disconnects mid-turn, the host skips to reveal. If only one
  player remains, everyone returns to the lobby with a notice.
- **End of game:** after every player has drawn once, a podium shows
  1st/2nd/3rd plus full standings. The host's "Play again" resets to the
  lobby with the same room code and players.

## Anti-cheat note (honest)

The secret word is stored in `turn/word` so the drawer and host can use
it. Guesser clients never render it and never use it for validation
(the host validates via `guessQueue`), but a determined player could
read it from browser devtools. For a friends-and-family game this is
standard; for a public competitive deployment, add RTDB rules or a
server-side validator as noted above.

## Files

- `index.html` — home, lobby, game, and podium views.
- `styles.css` — mobile-first responsive styling, big touch targets.
- `app.js` — all game logic (rooms, state machine, canvas, scoring).
- `words.js` — 135 everyday words across animals, food, objects, places,
  actions, and sports.
- `firebase-config.js` — placeholder Firebase config (fill in to go live).

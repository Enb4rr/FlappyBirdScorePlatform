# Flappy Bird - Game Portal + Telemetry System

A web-based game portal built with React and Firebase. The platform wraps an externally hosted Flappy Bird game in an iframe, handles authentication, persists scores to Firestore in real time, and provides an admin dashboard with live telemetry charts rendered in the browser using Python (via Pyodide).

> Screenshots / demo GIF coming soon

---

## Features

- Email/password and Google OAuth authentication
- Role-based access control: Player and Admin roles
- Live leaderboard with real-time Firestore updates
- User profile with editable display name and personal stats
- Embedded game via iframe with cross-origin auth handshake
- Admin dashboard with user management and role toggling
- Telemetry charts rendered in-browser using Pyodide + Matplotlib
- Mock data seeding script for local development and testing

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | React 19 + Vite |
| Auth | Firebase Authentication (Email/Password + Google OAuth) |
| Database | Firestore (real-time listeners) |
| Charts | Pyodide + Matplotlib (Python in the browser) |
| Game | Externally hosted iframe (cross-origin postMessage) |
| Config | Vite environment variables (`import.meta.env`) |
| Data Seeding | Node.js script (`scripts/ingestScores.js`) |

---

## Architecture

### Application Flow

```
App.tsx
├── onAuthStateChanged (Firebase Auth listener)
│   ├── Unauthenticated -> LoginForm
│   │   ├── Email/password auth
│   │   └── Google OAuth (signInWithPopup)
│   └── Authenticated -> GamePortal
│       ├── Tab: Game (iframe + Leaderboard)
│       ├── Tab: Profile (UserProfile)
│       └── Tab: Admin (AdminDashboard) [admin role only]
```

### Authentication and User Profile Creation

On every login, `App.jsx` calls `createUserProfileIfNeeded()` before rendering the portal. This function checks Firestore for an existing user document under `users/{uid}`. If it does not exist (first login), it creates one with default values: display name, photo URL, role set to `"player"`, and zeroed-out stats.

After profile creation, `App.jsx` opens a real-time `onSnapshot` listener on the user document so any role or stat changes propagate to the UI instantly without a page reload.

```js
// App.jsx - dual subscription pattern
const unsubscribeAuth = onAuthStateChanged(auth, async (firebaseUser) => {
  if (firebaseUser) {
    await createUserProfileIfNeeded(firebaseUser);
    setUser(firebaseUser);
    // Open a live listener on the user's Firestore doc
    unsubscribeFirestore = onSnapshot(userRef, (snapshot) => {
      if (snapshot.exists()) setUserData(snapshot.data());
    });
  }
});
```

Both subscriptions are cleaned up on unmount via the `useEffect` return function, preventing memory leaks.

### Cross-Origin Auth Handshake (postMessage)

The game runs as an externally hosted page loaded in an iframe. Since the game needs to write scores to the same Firestore project, it needs the user's Firebase ID token but cannot access it directly due to cross-origin restrictions.

The portal solves this with a `postMessage` handshake:

1. When the iframe fires `onLoad`, the portal calls `user.getIdToken()` to get a short-lived Firebase auth token.
2. It posts a `firebase-auth` message to the iframe's `contentWindow` with the token, UID, display name, and project ID.
3. The game listens for this message and uses the token to authenticate its own Firebase instance.
4. Once the game initializes successfully, it posts back a `firebase-auth-ack` message.
5. The portal stops retrying once it receives the acknowledgment.

A retry interval runs every 2 seconds in case the iframe is slow to initialize, with a 30-second timeout as a safety ceiling.

```js
// GamePortal.jsx - retry pattern
retryTimer.current = setInterval(sendAuthToGame, 2000);
setTimeout(() => {
  if (retryTimer.current) {
    clearInterval(retryTimer.current);
    if (!authAcknowledged.current) console.warn("Game never acknowledged.");
  }
}, 30000);
```

### Real-time Leaderboard

The leaderboard uses `onSnapshot` instead of a one-time `getDocs` fetch. This keeps the top 10 scores live: any score update by any user reflects immediately for everyone viewing the leaderboard without any polling or manual refresh.

### Role-Based Access

Roles are stored as a `role` field on each user document in Firestore (`"player"` or `"admin"`). The Admin tab in the portal is conditionally rendered based on `userData?.role === 'admin'`. Admins can toggle any user's role directly from the dashboard, and because `App.jsx` holds an `onSnapshot` on the current user's document, if an admin promotes or demotes themselves, their own UI updates immediately.

### Telemetry Charts with Pyodide

The Admin dashboard renders analytics charts by running Python inside the browser using Pyodide, a WebAssembly port of CPython. This avoids the need for a backend analytics service.

The flow:
1. Pyodide is loaded lazily on first access to the Admin tab (singleton pattern via `pyodideReady` ref).
2. Matplotlib is loaded as a Pyodide package.
3. Chart data (fetched from Firestore) is passed to Python via `window.__pyodideData` as a JSON string.
4. Python scripts (`scores_plot.py`, `sessions_plot.py`) parse the data, generate a Matplotlib figure, encode it as a base64 PNG, and inject an `<img>` tag directly into the DOM.

```python
# scores_plot.py (runs inside Pyodide)
data = json.loads(window.__pyodideData)
# ... build chart ...
target = document.getElementById('scores-chart')
target.innerHTML = f'<img src="data:image/png;base64,{img_b64}" />'
```

The Python scripts are imported as raw strings in the JSX via Vite's `?raw` import, so no separate bundling step is needed.

### Mock Data Seeding

`scripts/ingestScores.js` is a standalone Node.js script that pushes randomized player data and score sessions to Firestore for local development. It reads the `.env` file manually (since it runs outside Vite) and supports a `--clear` flag to delete all mock documents.

```bash
node scripts/ingestScores.js          # seed mock data
node scripts/ingestScores.js --clear  # remove mock data
```

All mock documents are flagged with `isMock: true` so the clear operation can target them precisely with a Firestore query.

---

## Project Structure

```
FlappyBirdScorePlatform/
├── src/
│   ├── components/
│   │   ├── LoginForm.jsx         - Email/password and Google OAuth login
│   │   ├── GamePortal.jsx        - Main portal shell, tab navigation, iframe + auth handshake
│   │   ├── Leaderboard.jsx       - Real-time top-10 leaderboard via onSnapshot
│   │   ├── UserProfile.jsx       - Player stats and editable display name
│   │   └── AdminDashboard.jsx    - User management, role toggling, telemetry charts
│   ├── plots/
│   │   ├── scores_plot.py        - Matplotlib bar chart for top player scores
│   │   └── sessions_plot.py      - Matplotlib line chart for games played over time
│   ├── App.jsx                   - Auth state management and user profile bootstrap
│   ├── firebase.js               - Firebase app initialization
│   └── main.jsx                  - React entry point
├── scripts/
│   └── ingestScores.js           - Node.js mock data seeder for Firestore
├── public/
└── index.html                    - Pyodide CDN script loaded here
```

---

## Getting Started

### Prerequisites

- Node.js 18+
- A Firebase project with the following enabled:
  - Email/Password Authentication
  - Google Authentication
  - Firestore Database

### Setup

1. Clone the repository

```bash
git clone https://github.com/Enb4rr/FlappyBirdScorePlatform.git
cd FlappyBirdScorePlatform
```

2. Install dependencies

```bash
npm install
```

3. Create a `.env` file from the template

```bash
cp .env.example .env
```

4. Fill in your Firebase credentials and game URL

```
VITE_FIREBASE_API_KEY=
VITE_FIREBASE_AUTH_DOMAIN=
VITE_FIREBASE_PROJECT_ID=
VITE_FIREBASE_STORAGE_BUCKET=
VITE_FIREBASE_MESSAGING_SENDER_ID=
VITE_FIREBASE_APP_ID=
VITE_GAME_URL=
```

5. Run the development server

```bash
npm run dev
```

### Seeding Mock Data (optional)

```bash
node scripts/ingestScores.js          # push mock players and scores
node scripts/ingestScores.js --clear  # remove mock data
```

---

## Test Accounts

| Role | Email | Password |
|---|---|---|
| Player | raf@ramenday.com | concussion1234 |
| Admin | spencer@bettleball.com | gonk1234 |

---

## Design Decisions

**Pyodide for in-browser charts:** Running Python in the browser via WebAssembly avoids the need for a dedicated analytics backend. The tradeoff is a heavy initial load (Pyodide + Matplotlib), which is acceptable here since the charts only appear in the Admin tab and load lazily on first access.

**postMessage for cross-origin auth:** The game is hosted on a separate origin, so sharing auth state via localStorage or cookies is not possible. Posting a short-lived Firebase ID token via `postMessage` lets the game authenticate independently with Firestore without exposing long-lived credentials or requiring the game to be co-hosted.

**`onSnapshot` over `getDocs` for the leaderboard:** A one-time fetch would require polling or manual refresh to stay current. Using a real-time listener means all connected users see score updates the moment they are written, which fits the competitive leaderboard use case.

**Singleton Pyodide loader:** Pyodide is expensive to initialize. The `getPyodide()` function caches the initialization promise in a module-level variable so it only runs once per session, regardless of how many times the Admin tab is opened.

**`isMock` flag on seeded data:** Rather than using a separate collection for mock data, all seeded documents carry an `isMock: true` field. This lets the clear script target them with a single Firestore query while keeping the data structure identical to real user documents, which makes testing more realistic.

---

## Known Limitations

- The game iframe must be configured to accept `postMessage` from the portal origin; third-party games without this support will not receive auth
- Pyodide adds significant load time on first Admin dashboard visit (~5-10s depending on connection)
- Role enforcement is client-side only; Firestore Security Rules should be configured to enforce role checks server-side for production use

---

## Author

Julian R. - [GitHub](https://github.com/Enb4rr) - [LinkedIn](https://linkedin.com/in/YOUR_PROFILE)

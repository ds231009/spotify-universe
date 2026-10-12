# Spotify Galaxy

Your Spotify library as a 3D galaxy. Every song is a star positioned by its audio features
(from [ReccoBeats](https://reccobeats.com)), and playlists are drawn as paths so you can see
how smoothly they flow or where they jump.

- **Backend:** FastAPI + SQLite. Spotify OAuth (PKCE), library sync, ReccoBeats enrichment, PCA/UMAP layout
- **Frontend:** React + React Three Fiber + drei, with instanced rendering and bloom

## How it works

```
Spotify Web API ──► top tracks · liked songs · your playlists
        │                      (track IDs, names, artists)
        ▼
ReccoBeats /v1/audio-features?ids=…   (Spotify IDs, 40 per call, no key)
        │   9 features: danceability energy valence acousticness instrumentalness
        │               speechiness liveness loudness tempo
        ▼
StandardScaler → PCA | UMAP | raw axes → 3D positions   (cached in SQLite)
        ▼
React Three Fiber: InstancedMesh + playlist path (segment color = jump size)
```

Playlist jump sizes are measured in the **standardized 9-dim feature space**, not in 3D,
because UMAP distorts global distances. "Smoother than random" compares the playlist's average
step to the average distance between two random songs in your library.

## Setup

### 1. Spotify app
1. Create an app at https://developer.spotify.com/dashboard and select **Web API**.
2. Add the redirect URI `http://127.0.0.1:8000/auth/callback` exactly as written (Spotify no longer accepts `localhost`).
3. Copy the Client ID. No secret is needed because this uses PKCE.

Development Mode limits (since Feb 2026): the app owner needs **Spotify Premium**, at most 5
allow-listed users (add them under *User Management*), and playlist contents are only returned
for playlists you **own or collaborate on**.

### 2. Backend
```bash
cd backend
python -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env        # paste your SPOTIFY_CLIENT_ID
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

Optional: `python seed_demo.py` fills the DB with 400 fake clustered songs and 3 playlists,
so you can work on the visuals without logging in. Delete `galaxy.db` afterwards to start clean.

### 3. Frontend
```bash
cd frontend
npm install
npm run dev                 # http://127.0.0.1:5173
```

Open **http://127.0.0.1:5173** (use the IP, not localhost), log in, then click **Sync library**.

## API

| Endpoint | What it does |
|---|---|
| `GET /auth/login` | Starts the Spotify login |
| `POST /api/sync` | Background sync: top tracks, liked songs (`MAX_SAVED_TRACKS`), own playlists, ReccoBeats features |
| `GET /api/status` | Login state, counts, sync progress |
| `GET /api/galaxy?method=pca\|umap\|axes` | Points with 3D positions, metadata, features |
| `GET /api/playlists` | Playlists that were synced |
| `GET /api/playlists/{id}/path` | Ordered tracks, jump distances, smoothness |

## Notes and next steps
- Not every track is in the ReccoBeats catalog. Misses are cached and listed as "skipped".
  ReccoBeats can also extract features from an uploaded audio file, which could fill gaps later.
- Feature values are ReccoBeats' estimates, not Spotify's original numbers.
- Ideas: add Last.fm tags for genre coloring, add an artist-level view (centroid per artist),
  visualize playlist "spread" as a convex hull, play 30s previews from Deezer, and
  add a "shuffle vs. your order" comparison.

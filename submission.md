# Project 5: Mixtape Bug Hunt Submission

## AI Usage

I used ChatGPT throughout this project as a debugging assistant and code navigation tool rather than asking it to solve the bugs directly.

Specifically, AI helped me:

- Understand the overall architecture of the project before modifying any code.
- Explain the responsibilities of the files in the `routes/` and `services/` directories.
- Trace execution flow from Flask routes to the corresponding service functions.
- Help interpret SQL queries and verify application behavior against the database.
- Explain Python logic while I determined the actual root cause myself.
- Help verify that my fixes matched the reported issue.

Whenever AI suggested a possible cause, I verified it manually by:
- reproducing the bug with `curl`
- inspecting the SQLite database
- tracing the execution through the route and service layers
- testing the application again after making changes

I did not blindly apply AI-generated code. Every fix was verified manually before committing.

---

# Codebase Map

## Main Files

### app.py
Creates the Flask application, initializes the SQLAlchemy database, and registers all application blueprints.

### models.py
Defines all SQLAlchemy models including:

- User
- Song
- Playlist
- PlaylistEntry
- ListeningEvent
- Rating
- Notification
- Friendship
- Tag

These models represent the application's database structure.

### routes/

The routes directory exposes the REST API. Routes mainly validate input and delegate business logic to the services layer.

- users.py
    - user profile
    - listening streak
    - notifications

- songs.py
    - search songs
    - rate songs
    - record listening events

- playlists.py
    - playlist creation
    - retrieve playlists
    - add songs to playlists

- feed.py
    - Friends Listening Now
    - Activity Feed

### services/

Contains almost all business logic.

- streak_service.py
    Calculates and updates listening streaks.

- feed_service.py
    Builds the Friends Listening Now feed and activity feed.

- search_service.py
    Handles song searching.

- notification_service.py
    Creates and retrieves notifications.

- playlist_service.py
    Retrieves playlist contents.

---

# Example Data Flow

### Rating a Song

```
POST /songs/<song_id>/rate
        ↓
routes/songs.py
        ↓
notification_service.rate_song()
        ↓
Create or update Rating
        ↓
Create notification for song owner
        ↓
Commit to database
        ↓
Return Rating JSON
```

Pattern observed:

Routes contain very little business logic. Nearly all application behavior lives inside the service layer.

---

# Root Cause Analysis

## Bug #1 — Listening streak resets incorrectly

### Issue

My listening streak shows 0 even though I've been listening on consecutive days.

### How I Reproduced It

Queried listening history using SQLite:

```sql
SELECT user_id, listened_at
FROM listening_event
WHERE user_id='<simone_id>'
ORDER BY listened_at DESC;
```

Then called:

```bash
curl http://127.0.0.1:5000/users/<simone_id>/streak
```

The endpoint returned:

```json
{"streak":0}
```

even though the user had consecutive listening events.

### Root Cause

`update_listening_streak()` incorrectly prevented streaks from increasing on Sundays.

```python
elif days_since_last == 1 and today.weekday() != 6:
```

### Fix

Removed the unnecessary weekday restriction.

```python
elif days_since_last == 1:
```

### Verification

Called the streak endpoint again and verified that consecutive listening correctly increments the streak.

---

## Bug #2 — Friends Listening Now shows people from yesterday

### Issue

Friends Listening Now displayed listening activity from the previous day.

### How I Reproduced It

Called:

```bash
curl http://127.0.0.1:5000/feed/<user_id>/listening-now
```

The response contained listening events from the previous day.

### Root Cause

The feed considered events from the previous 24 hours as "Listening Now," making the time window too broad for the feature's intended behavior.

### Fix

Reduced the recent threshold to 30 minutes and filtered using:

```python
ListeningEvent.listened_at >= cutoff
```

instead of comparing timestamps incorrectly.

### Verification

Created a new listening event and confirmed that only recent activity appeared in the feed.

---

## Bug #3 — Duplicate songs appear in search

### Issue

Searching sometimes returned the same song multiple times.

### How I Reproduced It

Called:

```bash
curl "http://127.0.0.1:5000/songs/search?q=<query>"
```

The same song appeared multiple times.

### Root Cause

Duplicate records were being included during search result construction.

### Fix

Updated the search logic to eliminate duplicate results before returning them.

### Verification

Repeated searches returned unique song results.

---

## Bug #4 — Rating a song does not notify the owner

### Issue

Users received notifications when songs were added to playlists but not when their songs were rated.

### How I Reproduced It

Rated another user's song:

```bash
POST /songs/<song_id>/rate
```

Then checked notifications:

```bash
GET /users/<owner_id>/notifications
```

Initially:

```json
{
  "count":0
}
```

### Root Cause

`rate_song()` saved ratings but never created a notification for the song owner.

### Fix

Added notification creation after successfully saving the rating.

### Verification

After rating a song again:

```json
{
  "count":1,
  "notifications":[...]
}
```

The owner correctly received a `song_rated` notification.

---

## Bug #5 — Last song missing from playlists

### Issue

The final song in a playlist never appeared.

### How I Reproduced It

Compared the database:

```sql
SELECT playlist_id, song_id, position
FROM playlist_entries;
```

with

```bash
GET /playlists/<playlist_id>/songs
```

The database contained seven songs while the endpoint returned only six.

### Root Cause

An off-by-one error caused the final playlist entry to be excluded.

### Fix

Corrected the playlist iteration logic so every entry is returned.

### Verification

The playlist endpoint now returns all songs, and the playlist tests pass successfully.

---

# Commits

- fix: allow listening streaks to continue on Sunday
- fix: prevent duplicate songs from appearing in search
- fix: notify song sharer when their song is rated
- fix: limit listening now feed to recent events
- fix: include last song in playlist results

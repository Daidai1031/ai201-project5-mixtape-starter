# Mixtape Codebase Map

## Main files and their responsibilities

**`app.py`** — Flask application factory. Creates the Flask app, configures SQLAlchemy (`db`), registers the four blueprints (`songs`, `playlists`, `users`, `feed`) under their URL prefixes, and calls `db.create_all()`.

**`models.py`** — Defines all SQLAlchemy models and association tables:
- `User` — has `listening_streak`, `last_listened_at`; relates to `Song` (shared), `Rating`, `ListeningEvent`, `Notification`, `Playlist`, and `friends` (self-referential many-to-many via the `friendships` table).
- `Tag` — simple name entity, linked to `Song` via `song_tags` (many-to-many).
- `Song` — has `shared_by` (FK to User), `share_note`; relates to `Rating`, `ListeningEvent`, `tags`.
- `ListeningEvent` — one row per "user listened to song" action, with `listened_at`. This is the sole data source for the feed.
- `Rating` — one row per (user, song) pair (`unique_user_song_rating` constraint), holds `score` (1–5). Ratings are their own model, not a column on `Song`.
- `Playlist` — has `created_by`, `is_collaborative`; songs attached via `playlist_entries`, an association table that carries `position`, `added_by`, and `added_at` — so playlist membership has explicit ordering and provenance, not just insertion order.
- `Notification` — `user_id` (recipient), `notification_type`, `body`, `read` flag.

**`routes/`** — One blueprint per resource, each doing only request parsing and response formatting:
- `songs.py` — `/songs/search`, `/songs/<id>`, `/songs/<id>/rate`, `/songs/<id>/listen`
- `playlists.py` — `/playlists/` (create), `/playlists/<id>`, `/playlists/<id>/songs` (get/add)
- `users.py` — `/users/<id>`, `/users/<id>/streak`, `/users/<id>/notifications`, `/users/notifications/<id>/read`
- `feed.py` — `/feed/<user_id>/listening-now`, `/feed/<user_id>/activity`

**`services/`** — All business logic lives here; routes call straight into it:
- `search_service.py` — `search_songs()` (title/artist substring match), `get_song()`
- `playlist_service.py` — `create_playlist()`, `get_playlist()`, `get_playlist_songs()` (ordered by `position`), `get_user_playlists()`
- `notification_service.py` — `add_to_playlist()`, `rate_song()`, `create_notification()`, `get_notifications()`, `mark_as_read()`
- `streak_service.py` — `record_listening_event()`, `update_listening_streak()`, `get_streak()`
- `feed_service.py` — `get_friends_listening_now()` (last 24h, one entry per friend), `get_activity_feed()` (unbounded time window, top N, no dedup)

**`seed_data.py`** — populates the DB with sample users/songs/playlists for local testing.

**`tests/`** — `test_playlists.py`, `test_search.py`, `test_streaks.py` — one test file per service area.

---

## Data flow — a friend rates a shared song and gets nothing, but adding it to a playlist does notify

Two related-but-separate flows worth tracing together, since they're often confused:

**1. Rating a song** (`POST /songs/<id>/rate`)
```
routes/songs.py: rate()
  → notification_service.rate_song(user_id, song_id, score)
      → validates score is 1–5
      → upserts a Rating row (unique on user_id+song_id)
      → db.session.commit()
```
No `Notification` is created and no `ListeningEvent` is written. Rating is an isolated write — the original sharer is not told their song was rated, and it never appears in any feed.

**2. Adding a song to a playlist** (`POST /playlists/<id>/songs`)
```
routes/playlists.py: add_song()
  → notification_service.add_to_playlist(playlist_id, song_id, added_by_user_id)
      → appends Song to Playlist.songs (via playlist_entries) if not already present
      → db.session.commit()
      → if song.shared_by != added_by_user_id:
            create_notification(user_id=song.shared_by, type="song_added_to_playlist", body=...)
```
This is the one place a `Notification` gets created automatically, and only for this one action.

**3. Listening to a song** (`POST /songs/<id>/listen`) — the only path that feeds the feed
```
routes/songs.py: listen()
  → streak_service.record_listening_event(user_id, song_id)
      → db.session.add(ListeningEvent(...))
      → update_listening_streak(user, now)   # +1 / reset / no-op depending on last_listened_at
      → db.session.commit()
```
Later, `feed_service.get_friends_listening_now()` / `get_activity_feed()` query `ListeningEvent` for the current user's friends to build the feed.

**Takeaway:** rating, playlist-adding, and listening are three independent writes with three different downstream effects (nothing / notification / feed+streak). None of them trigger each other — e.g., adding a song to a playlist does not create a `ListeningEvent`, so it never shows up as "friend activity" in the feed.

---

## Patterns noticed

- **Routes are thin, services own logic.** Every route handler does: parse request → call one service function → `jsonify()` the result (or catch `ValueError` → 400/404). No SQLAlchemy queries or business rules appear in `routes/`.
- **Services raise `ValueError` for not-found/invalid-input; routes translate that to HTTP errors.** This is the only error-handling convention in the app — no custom exception classes.
- **Every model has a hand-written `to_dict()`** used directly in `jsonify()` calls — no serialization library.
- **Association tables carry extra metadata when needed** (`playlist_entries` has `position`/`added_by`/`added_at`), but plain many-to-many tables (`friendships`, `song_tags`) don't.
- **Feed and notifications are separate, uncoupled systems** built on different tables (`ListeningEvent` vs `Notification`) — despite both being "things a user's friends might want to know about," there's no shared abstraction between them, so features like playlist-adds are visible in notifications but invisible in the feed, and vice versa.

---

## Bugs found

### Bug 1: `get_playlist_songs()` drops the last song in the playlist

**Location:** `services/playlist_service.py:66`

```python
return [song.to_dict() for song in songs[:-1]]
```

**Root cause:** `songs` is already the correctly ordered, correctly filtered query result for the playlist. Slicing with `[:-1]` unconditionally discards the last element regardless of playlist size — there's no reason for this in the surrounding logic; the docstring even says the function "returns all songs in the playlist."

**How I reproduced it:**
1. Seeded a playlist with 5 songs at `position` 1–5 (see `tests/test_playlists.py::seed_playlist` fixture — creates 1 user, 5 `Song` rows, 1 `Playlist`, and 5 rows in `playlist_entries` with `position=1..5`).
2. Called `get_playlist_songs(playlist_id)`.
3. Expected 5 songs back; got 4 — `Track 5` (the last one by position) was missing.
4. Confirmed via `tests/test_playlists.py::test_playlist_returns_all_songs`, which asserts `len(songs) == 5` with the comment `# Bug causes this to return 4`.
5. Edge case check: `tests/test_playlists.py::test_empty_playlist_returns_empty_list` — an empty playlist still returns `[]` without erroring, since slicing `[][:-1]` is safe. So the bug only manifests when a playlist has ≥1 song (any playlist with songs loses exactly one — its last-position song).

**Trigger condition:** Any `GET /playlists/<id>/songs` (or direct `get_playlist_songs()` call) on a playlist that has at least one song.

---

### Bug 2: Streak doesn't increment when the "today" of a consecutive-day listen falls on a Sunday

**Location:** `services/streak_service.py:73`

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

**Root cause:** The streak-increment branch has an extra, undocumented condition — `today.weekday() != 6` (Sunday) — bolted onto the "listened on the immediately following day" check. There's no comment or stated rule explaining why Sunday is excluded, and it contradicts the function's own docstring ("If the user listened yesterday: streak increments by 1" — no day-of-week exception is mentioned). Any consecutive-day listen where the second day is a Sunday falls through to the `else` branch and incorrectly resets the streak to 1 instead of incrementing it.

**How I reproduced it:**
1. Created a fresh `User`.
2. Called `update_listening_streak(user, saturday)` where `saturday = 2024-06-15` (`weekday() == 5`) → streak correctly set to 1.
3. Called `update_listening_streak(user, sunday)` where `sunday = 2024-06-16` (`weekday() == 6`, exactly 1 day after Saturday) → expected streak to increment to 2 (consecutive day), but it reset to 1 instead, because `days_since_last == 1` but `today.weekday() == 6` fails the `!= 6` check and falls into the `else: streak = 1` branch.
4. Confirmed via `tests/test_streaks.py::test_streak_increments_on_sunday`, which asserts `u.listening_streak == 2` with the comment `# Should increment, not reset` — this test currently fails against the existing code.
5. Contrast with `tests/test_streaks.py::test_streak_increments_on_consecutive_day` (Monday → Tuesday), which passes, showing the failure is specific to landing on a Sunday, not a general off-by-one in the date math.

**Trigger condition:** A user listens on day N, then listens again on day N+1, and day N+1 is a Sunday (UTC). The streak resets to 1 instead of incrementing — effectively, users can never grow a streak across a Saturday→Sunday boundary.

---

### Bug 3: Songs with multiple tags appear as duplicates in search results

**Location:** `services/search_service.py:25-35`

```python
results = (
    db.session.query(Song)
    .outerjoin(song_tags, Song.id == song_tags.c.song_id)
    .filter(...)
    .all()
)
```

**Root cause:** `outerjoin` to `song_tags` produces one result row per **matching tag**, not per song — a song with 3 tags yields 3 joined rows. There's no `.distinct()` on the query, so `search_songs()` returns duplicate `Song` objects (one per tag).

**How I reproduced it:**
1. Seeded a song ("Crown Heights Anthem") with 3 tags (`rap`, `hip-hop`, `boom bap`) — see `tests/test_search.py::seed_songs`.
2. Also seeded a song with 1 tag and a song with 0 tags for comparison.
3. Called `search_songs("Crown Heights")`.
4. Expected 1 result; got 3 — one row per tag on that song.
5. Confirmed via `tests/test_search.py::test_search_no_duplicates_multi_tag_song`, which asserts `len(matching) == 1` with comment `# Should be 1, bug causes it to be 3`.
6. Contrast: the 1-tag song and 0-tag song each return correctly (1 result), showing duplication scales exactly with tag count.

**Trigger condition:** Searching for any song that has 2+ tags — it appears once per tag in the result list.

---

### Bug 4: "Friends Listening Now" shows people from yesterday

**Location:** `services/feed_service.py:13` and `:32`

```python
RECENT_THRESHOLD = timedelta(hours=24)
...
cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD
...
ListeningEvent.listened_at >= cutoff
```

**Root cause:** "Listening now" implies real-time/near-real-time presence, but the window used to decide "recent" is a flat 24-hour rolling threshold. Any listen up to 23h59m old passes the filter and gets shown as if the friend is currently listening — including a play from late last night showing up all day today. The threshold is too generous for what the feature is supposed to represent (compare: it isn't capped to, say, the last 15–30 minutes, which is what "now" should mean).

**How I reproduced it (via the running server, not tests — there's no `test_feed.py`):**
1. Set up the app and seed the DB: `python seed_data.py`. This script deliberately creates two groups of `ListeningEvent`s to expose this bug (see `seed_data.py:110-130`):
   - "Recent" events, 10–20 minutes old → correctly meant to show as listening now.
   - "Older" events, at `hours = 2, 10, 18, 26, 34, 42, 50, 58` (i.e., `2 + i*8` for `i in range(8)`) ago, spread across several users — comment in the script literally says these "should NOT appear in listening now after fix."
2. Ran the app: `flask run` (with `FLASK_APP=app:create_app`).
3. Called `GET /feed/<nova's user_id>/listening-now` (nova is friends with darius, simone, kenji — see `seed_data.py:45-47`).
4. Observed that friends whose only listening event was **2h, 10h, or 18h ago** still appeared in the response, each labeled as "listening now" — even though, in calendar terms, the 18h-old one could easily be "yesterday evening" if you query this morning.
5. Confirmed the fix boundary: events at 26h+ ago were correctly excluded (outside the 24h window), but 18h and under were incorrectly included, since the cutoff logic only excludes things over 24h old, not things that are simply not "now."

**Trigger condition:** Query `GET /feed/<user_id>/listening-now` when a friend's most recent listen is anywhere from a few hours up to just under 24 hours old — they'll show up as "currently listening" even though that session ended long ago.

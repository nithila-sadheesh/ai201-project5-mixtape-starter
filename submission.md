# Project 5: Mixtape Bug Hunt — Submission

## Codebase Map

### Main files and what each one does

**`app.py`** — Flask application factory. Creates the `db` (SQLAlchemy) object at module level, then `create_app()` configures the SQLite database (`instance/mixtape.db` by default, overridable via `DATABASE_URL`), registers the four route blueprints with URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and runs `db.create_all()` inside an app context. Running it directly starts the dev server on port 5001. Because `db` lives here, every other module imports `from app import db`.

**`models.py`** — 7 SQLAlchemy models plus 3 plain association tables. All primary keys are UUID strings, and all timestamps are stored as timezone-aware UTC datetimes via `datetime.now(timezone.utc)` defaults.

- `User` — username/email, plus streak state stored directly on the user (`listening_streak` int and `last_listened_at` datetime). Has a self-referential many-to-many `friends` relationship through the `friendships` table.
- `Song` — a song is always a *share*: it carries `shared_by` (FK to User), `shared_at`, and an optional `share_note`. There is no global song catalog separate from shares.
- `Tag` — just a name; linked to songs through the `song_tags` association table (many-to-many).
- `ListeningEvent` — one row per listen: `user_id`, `song_id`, `listened_at`. This is the raw data behind both streaks and feeds.
- `Rating` — user's 1–5 score for a song, with a unique constraint on `(user_id, song_id)` so a user has at most one rating per song (re-rating updates it).
- `Playlist` — name, creator, `is_collaborative` flag. Songs attach through the `playlist_entries` association table, which is more than a plain join table: it has `position` (explicit ordering), `added_by`, and `added_at` columns.
- `Notification` — recipient `user_id`, a `notification_type` string (e.g. `song_added_to_playlist`), a text `body`, and a `read` flag.

Most models define a `to_dict()` method — that's the serialization layer; routes return these dicts directly as JSON.

**`routes/`** — four blueprints, one per resource:

- `songs.py` — `GET /songs/search`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen`.
- `playlists.py` — create a playlist, get its metadata, `GET /playlists/<id>/songs` (ordered songs), `POST /playlists/<id>/songs` (add a song).
- `users.py` — user profile, `GET /users/<id>/streak`, list notifications (with `unread_only` filter), mark a notification read.
- `feed.py` — `GET /feed/<id>/listening-now` (friends active in the last 24h) and `GET /feed/<id>/activity` (last N friend events regardless of age).

**`services/`** — all business logic:

- `streak_service.py` — `record_listening_event()` creates a `ListeningEvent` and calls `update_listening_streak()`, which compares calendar days (UTC) between now and `user.last_listened_at`: same day = no change, one day = increment, more = reset to 1.
- `feed_service.py` — `get_friends_listening_now()` queries friends' `ListeningEvent`s newer than a `RECENT_THRESHOLD` cutoff and deduplicates to the most recent song per friend; `get_activity_feed()` is the unfiltered "last 20 events" version.
- `search_service.py` — `search_songs()` does a case-insensitive `ilike` match on title/artist (with an outer join to `song_tags`); `get_song()` fetches one by ID.
- `notification_service.py` — a bit of a grab bag: besides `create_notification` / `get_notifications` / `mark_as_read`, it also owns `add_to_playlist()` (the playlist-add action itself lives here because it triggers a notification) and `rate_song()` (rating creation/update).
- `playlist_service.py` — playlist creation and retrieval, including `get_playlist_songs()` which joins through `playlist_entries` ordered by `position`.

**`seed_data.py`** — populates the DB with test users, songs, playlists, friendships, events. **`tests/`** — pytest suites for streaks, search, and playlists, using `create_app()` with a test config.

### Data flow — a friend adds my shared song to a playlist, and I get notified

1. Client sends `POST /playlists/<playlist_id>/songs` with JSON body `{"song_id": ..., "added_by": ...}`.
2. `routes/playlists.py:add_song()` validates that both fields are present, then delegates to `notification_service.add_to_playlist(playlist_id, song_id, added_by)`. Note it calls the *notification* service, not the playlist service — the add-song action lives there because notifying is part of the operation.
3. `add_to_playlist()` loads the Song, the adding User, and the Playlist (raising `ValueError` if any is missing — the route maps that to a 400). It appends the song to `playlist.songs` if it isn't already there and commits.
4. Then it checks `song.shared_by != added_by_user_id`: you don't get notified for adding your own song. If it was someone else, it calls `create_notification()` targeting `song.shared_by` — the original sharer — with type `song_added_to_playlist` and a body like "*alice added your song 'X' to the playlist 'Y'*".
5. The sharer later sees it via `GET /users/<their_id>/notifications`, which `notification_service.get_notifications()` serves ordered newest-first, optionally filtered to unread. `POST /users/notifications/<id>/read` flips the `read` flag.

### Patterns I noticed

- **Routes are thin, services are fat.** Every route does the same three things: parse input, call exactly one service function, format the response. All queries and business rules live in `services/`. This means the bugs (per the README) are all in the service layer, and each issue maps 1:1 to a service file.
- **Consistent error contract.** Services signal problems by raising `ValueError` with a message; routes catch it and return `{"error": str(e)}` with 400 (bad input) or 404 (missing resource). No custom exception classes.
- **`to_dict()` as the API boundary.** Services return plain dicts (via model `to_dict()` methods) rather than model objects, except when the route needs the created object (e.g. `rate_song` returns the `Rating` instance and the route calls `.to_dict()` itself).
- **UUID string PKs and UTC-aware timestamps everywhere**, generated by shared defaults in `models.py`. `streak_service` even defensively re-attaches `tzinfo=timezone.utc` to naive datetimes read back from SQLite, since SQLite drops timezone info on storage.
- **Association tables carry data.** `playlist_entries` isn't just a join table — `position`, `added_by`, `added_at` make playlist ordering explicit rather than relying on insertion order. (Notably, `add_to_playlist()` appends via the ORM relationship without setting `position`, while `get_playlist_songs()` sorts by it — worth watching when bug hunting.)
- **One quirky ownership choice:** `add_to_playlist` and `rate_song` live in `notification_service.py` instead of `playlist_service`/a rating service, presumably because they're the two actions that generate (or should generate) notifications.

---

## Root Cause Analyses

### Issue #1 — My listening streak keeps resetting

**How I reproduced it.** Ran the existing test suite (`pytest tests/`) before touching any code. `tests/test_streaks.py::test_streak_increments_on_sunday` failed: it simulates a listen on a Saturday, then a listen on the following Sunday, and asserts the streak becomes 2 — but it stayed at 1. So the trigger condition is: two listens on consecutive calendar days where the *second* day is a Sunday. Any streak crosses a Saturday→Sunday boundary once a week, which matches the user report of streaks "keep resetting."

**How I found the root cause.** Started from the route: `POST /songs/<id>/listen` in `routes/songs.py` calls `streak_service.record_listening_event()`, which creates the `ListeningEvent` and delegates streak math to `update_listening_streak(user, now)`. That function has only three branches keyed off `days_since_last` (calendar-day difference between now and `user.last_listened_at`): 0 = no change, 1 = increment, else = reset. The failing test is a `days_since_last == 1` case, so the increment branch had to be the problem. Reading it, the condition was `days_since_last == 1 and today.weekday() != 6` — the extra weekday clause was the confirmed cause, since Python's `datetime.weekday()` returns 6 for Sunday, exactly the day the test uses. (AI-assist: I asked what `weekday()` returns for each day to confirm 6 = Sunday, after I had already narrowed it to this line.)

**The root cause.** In `update_listening_streak()` (`services/streak_service.py`), the increment branch was guarded by `today.weekday() != 6`. Python's `datetime.weekday()` numbers Monday as 0 through Sunday as 6, so whenever the *current* listen happened on a Sunday, the consecutive-day condition evaluated false and control fell through to the `else` branch, which sets `listening_streak = 1`. In other words: a perfectly consecutive Saturday→Sunday listen was treated as a missed day. Calendar weekdays have no business in the streak rule at all — the documented rules only care about consecutive days.

**Fix and side-effect check.** Removed the `and today.weekday() != 6` clause so the branch is just `elif days_since_last == 1:` (`services/streak_service.py:73`). This is the smallest change that makes the code match its own docstring. Side-effect check: reran the full streak suite — all 5 tests pass, covering the other boundary sides too (same-day listen leaves the streak unchanged, a 2-day gap still resets to 1, first-ever listen starts at 1, and multi-day consecutive chains increment). `last_listened_at` update behavior is unchanged.

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it.** Ran `pytest tests/` before changing anything: `tests/test_playlists.py::test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` both failed — a playlist seeded with N songs came back with N−1, and it was always the highest-position (most recently added) song that was missing. That matches the user report exactly: the last song never shows up, regardless of which song it is.

**How I found the root cause.** Traced from the endpoint: `GET /playlists/<id>/songs` in `routes/playlists.py` delegates straight to `playlist_service.get_playlist_songs()`. Read that function top-down. The query itself is correct — it joins `Song` through the `playlist_entries` association table, filters by playlist, and orders ascending by the `position` column. The bug had to be after the query, and the return line made it unambiguous: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice drops the final element of the list. Since the list is sorted ascending by position, the dropped element is always the last song in playlist order — the exact symptom.

**The root cause.** A stray `[:-1]` slice on the query results in `get_playlist_songs()` (`services/playlist_service.py:66`). In Python, `list[:-1]` means "everything except the last element," so the function systematically discarded the song with the highest `position` value. It looks like a leftover from some off-by-one experiment; the function's own docstring even says "This function returns all songs in the playlist."

**Fix and side-effect check.** Changed the return to iterate over `songs` with no slice. One-character-class fix, no query changes. Side-effect check: all 3 playlist tests pass, including ordering (songs still come back ascending by `position`) and the empty-playlist case — with an empty list, the old `[:-1]` and the fix both return `[]`, so nothing else depended on the slice. Also confirmed the other caller of playlist data, `notification_service.add_to_playlist`, only imports `get_playlist_songs` but never calls it, so no behavior change there.

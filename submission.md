# Project 5: Mixtape Bug Hunt — Submission

## AI Usage

I worked on this project with Claude Code (Anthropic's CLI agent) as a pair-programming tool. Specifically how I used it, and where I had to verify things myself:

**What I asked it to explain, trace, or summarize:**
- After reading `models.py` and the route files myself, I had it walk the call chain for the playlist-add flow (`POST /playlists/<id>/songs` → `add_to_playlist` → `create_notification`) to check my understanding of the data flow written up in the codebase map below.
- For Issue #1, once I had narrowed the bug to the `elif days_since_last == 1 and today.weekday() != 6:` line, I asked what `datetime.weekday()` returns for each day of the week to confirm that 6 means Sunday (vs. `isoweekday()`, where Sunday is 7). That confirmed the guard was excluding Sundays from the increment branch.
- For Issue #4, I gave it the two sibling functions (`add_to_playlist` and `rate_song`) and asked for a structural comparison. It confirmed the only meaningful difference was the missing `create_notification` call — not a transaction-ordering or commit issue.

**What it helped me understand:** the SQLAlchemy association-table pattern (`playlist_entries` carrying `position`/`added_by` columns rather than being a plain join table), and why `streak_service` re-attaches `tzinfo=timezone.utc` to datetimes read back from SQLite.

**Where AI was incomplete or verification mattered:**
- The biggest case was Issue #3 (duplicate search results). The `outerjoin` to `song_tags` without `.distinct()` looks like an obvious duplicate-row bug, and an AI explanation of the query agreed it should duplicate multi-tag songs. But when I actually ran it — both the provided test (`test_search_no_duplicates_multi_tag_song`) and a hand-built reproduction — the duplicates never appeared at the API level, because SQLAlchemy 2.0's legacy `Query` deduplicates entity results after the join (the raw SQL does return 3 rows for a 3-tag song; the ORM collapses them). A plausible-sounding diagnosis was wrong for this environment, and only executing the code revealed it. That's why I swapped Issue #3 out for Issue #1, which I could actually reproduce and verify fixed.
- For every fix, the diagnosis was confirmed by running code, not by explanation alone: failing tests before / passing tests after for #1 and #5, and a Python-shell reproduction for #4 (0 notifications before the fix; exactly 1 after, with no duplicate on re-rating and none on self-rating).

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

### Issue #4 — Notified when a friend added my song to a playlist, but not when they rated it

**How I reproduced it.** In a Python shell against an in-memory DB (faster than HTTP): created two users, had alice share a song, then called `notification_service.rate_song(bob.id, song.id, 5)` — the path `POST /songs/<id>/rate` delegates to. Queried `Notification` rows for alice afterward: **0**. For contrast, calling `add_to_playlist()` in the same setup *does* create a notification, confirming the asymmetry the user reported.

**How I found the root cause.** The issue itself describes a working path and a broken one, so I diffed the two structurally. Both live in `services/notification_service.py`: `add_to_playlist()` performs its action, then has an explicit block — `if song.shared_by != added_by_user_id: create_notification(...)`. `rate_song()` validates the score, loads the song and rater, creates or updates the `Rating`, commits, and returns. There is no call to `create_notification()` anywhere in it, even though the module docstring says notifications are generated "when friends interact with a user's shared songs" and `create_notification`'s own docstring lists `'song_rated'` as an example type. A missing step is hard to "spot" by reading one function in isolation — comparing against the sibling function that works is what made it obvious. (AI-assist: after finding both functions myself, I asked for a structural comparison of the two blocks to confirm the only meaningful difference was the missing notification step, not some transaction/ordering subtlety.)

**The root cause.** `rate_song()` was simply never wired into the notification system. The rating was saved correctly (that part of the feature worked), but the function returned without ever creating a `Notification` for `song.shared_by`. The `'song_rated'` notification type existed in name only — referenced in a docstring, never emitted by any code path.

**Fix and side-effect check.** Added the same guarded block `add_to_playlist()` uses, after the commit in `rate_song()`: if the rating is newly created and `song.shared_by != user_id`, call `create_notification(user_id=song.shared_by, notification_type="song_rated", body="<rater> rated your song '<title>' <score>/5.")`. Two deliberate guards: no self-notification (matching the playlist behavior), and no notification on *re*-rating — the unique `(user_id, song_id)` constraint means updates hit the `existing` branch, and re-emitting there would let one user spam the sharer by repeatedly re-rating. Side-effect check: verified in the shell that a friend's first rating produces exactly one notification with the right body, a re-rate updates the score but adds no second notification, and a self-rate produces none; the full test suite (13 tests, including the pre-existing rating-related search/streak flows) still passes, and the route's response contract is unchanged since `rate_song` still returns the `Rating` instance.

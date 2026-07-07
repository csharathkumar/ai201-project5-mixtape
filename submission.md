# Mixtape — Bug Hunt Submission

## Milestone 1: Codebase Map

### Setup

Forked to `csharathkumar/ai201-project5-mixtape`, cloned locally, working branch `bugfix/mixtape` created off `main`. `app.py`, `models.py`, `seed_data.py`, all of `routes/`, `services/`, and `tests/` compile cleanly (`python -m py_compile`). App confirmed running locally via `FLASK_APP=app:create_app flask run`, responding at `http://127.0.0.1:5000`. The five open issues and their affected files are listed in `README.md`; the full issue reports (reproduction steps, expected vs. actual) are below and confirm the hypotheses formed from reading the code.

### Main files

`app.py` is the Flask application factory. It builds the app, wires up `SQLALCHEMY_DATABASE_URI` (defaulting to a local `mixtape.db` SQLite file), initializes the shared `db` object, registers four blueprints (`songs`, `playlists`, `users`, `feed`), and calls `db.create_all()` inside an app context. There's no business logic here — it's purely wiring.

`models.py` defines six SQLAlchemy models plus three association tables. `User` holds streak state directly (`listening_streak`, `last_listened_at`) rather than deriving it — so streak logic is a mutation, not a query. `Song` carries `shared_by`/`shared_at`, tying every song to the friend who shared it (this is what notification logic keys off of). `ListeningEvent` is an append-only log of plays, one row per listen. `Rating` has a unique constraint per `(user_id, song_id)`, so rating is upsert-style, not append-only. `Playlist` uses `playlist_entries` as its join table, which — unlike `friendships` and `song_tags` — carries extra columns: `position` (explicit ordering, not insertion order), `added_by`, and `added_at`. `Notification` is a flat table with a `notification_type` string and a pre-rendered `body`; there's no templating layer, each service call constructs the human-readable string inline at creation time.

`routes/` is a thin layer over four blueprints (`songs.py`, `playlists.py`, `users.py`, `feed.py`). Every route follows the same shape: parse `request.args`/`request.get_json()`, call exactly one service function, catch `ValueError` and turn it into a 400/404 JSON response, otherwise return `.to_dict()` output as JSON. No route does its own DB querying or business logic.

`services/` is where everything actually happens, split by domain: `streak_service.py` (streak increment/reset), `feed_service.py` (friends-listening-now + activity feed), `search_service.py` (song search by title/artist), `notification_service.py` (creating and reading notifications, plus `rate_song`/`add_to_playlist` which double as the "trigger" points for notifications), `playlist_service.py` (playlist CRUD-lite and ordered song retrieval).

`seed_data.py` wipes and rebuilds the DB with 5 friended users, 25 songs (deliberately split into 0-tag / 1-tag / 3+-tag groups — the comments in the file flag that the 3+-tag group is what exposes the search duplicate bug), 3 playlists of 5-10 songs each, and a mix of recent (~10-25 min old) and older (2-58 hours old) listening events to exercise the "listening now" recency filter.

`tests/` covers three of the five services (streaks, search, playlists) with fixtures that isolate an in-memory SQLite DB per test. Notably, several test docstrings/comments already name the expected bug behavior directly — e.g. `test_search_no_duplicates_multi_tag_song` says "Should be 1, bug causes it to be 3," and `test_playlist_returns_all_songs` says "Bug causes this to return 4." These read like they were written test-first against the intended fix, which makes them useful oracles once I start fixing rather than just describing symptoms.

### Data flow — user rates a song

`POST /songs/<song_id>/rate` in `routes/songs.py` pulls `user_id` and `score` from the JSON body, validates presence, and calls `notification_service.rate_song(user_id, song_id, int(score))`. That function (despite living in `notification_service.py`, not a rating-specific module) validates the score range, loads the `Song` and `User`, checks for an existing `Rating` row for that `(user_id, song_id)` pair, and either updates the existing row's score or inserts a new one — then commits and returns the `Rating`. The route serializes it back as `rating.to_dict()` with a 201.

The notable thing: `rate_song` never calls `create_notification`. Compare this to `add_to_playlist` in the same file, which — after appending the song to the playlist — explicitly checks `if song.shared_by != added_by_user_id` and creates a `song_added_to_playlist` notification for the sharer. `rate_song` has no equivalent block. This lines up exactly with issue #4 ("notified when added to playlist but not when rated") — the notification-on-rating code path simply doesn't exist yet, it's not a matter of it firing incorrectly.

### Data flow — user views a playlist's songs

`GET /playlists/<id>/songs` → `routes/playlists.py:get_songs` → `playlist_service.get_playlist_songs(playlist_id)`. That function loads the `Playlist` (404s via `ValueError` if missing), then joins `Song` to `playlist_entries` filtered by `playlist_id`, ordered ascending by `position`. The last line is `return [song.to_dict() for song in songs[:-1]]` — it drops the final element of the already-correctly-ordered list. This is issue #5 ("the last song in a playlist never shows up") and it's a one-line slicing bug independent of the query itself, which is otherwise correct.

### Patterns noticed

Routes never touch the database directly except `routes/users.py:get_user`, which calls `db.session.get(User, user_id)` inline instead of delegating to a service — the one exception to the "routes only call services" rule.

Every service function that mutates state re-validates its foreign keys by loading the referenced rows (`db.session.get(...)` then `raise ValueError` if `None`) before doing anything else, and every route converts that `ValueError` into an HTTP error response. This is a consistent, deliberate error-handling contract across the whole app, not something applied ad hoc per route.

Cross-cutting side effects — like notifications — are triggered from inside the service that owns the primary action (`add_to_playlist`, `rate_song`) rather than from a separate observer/event layer. This means whether a given user action notifies anyone at all depends entirely on whether that specific service function remembered to call `create_notification`. There's no central registry of "which actions notify" — bugs of the "some actions notify, some silently don't" shape (issue #4) are a direct consequence of this pattern, since each function is a separate hand-written call site rather than a table-driven dispatch.

Several `services/*.py` docstrings describe the *intended* behavior precisely (e.g. streak_service's docstring spells out the exact day-boundary rules; feed_service's module docstring says `get_friends_listening_now` should be "filtered by recency"), and the actual bugs are narrow deviations from those stated rules rather than wholesale misunderstandings — e.g. `update_listening_streak`'s `elif days_since_last == 1 and today.weekday() != 6:` has an extra, undocumented Sunday special-case that isn't mentioned in the docstring's "streak rules" and isn't matched by the corresponding logic anywhere else, which is consistent with issue #1.

### The five open issues and rough plan

| # | Title | Reporter | Service | Confirmed root cause |
|---|-------|----------|---------|-----------------------|
| 1 | Listening streak keeps resetting | kenji | `streak_service.py` | kenji's report (streak 12 on Saturday → 1 after listening Sunday morning, having listened every day) pins this exactly on the Sunday-specific condition in `update_listening_streak`: `elif days_since_last == 1 and today.weekday() != 6:` blocks the increment specifically when `today` is Sunday, falling through to the reset branch instead. It's not part of the documented streak rules and isn't mirrored anywhere else in the function. `test_streak_increments_on_sunday` already encodes the expected fix. |
| 2 | Friends Listening Now shows people from yesterday | nova | `feed_service.py` | nova's report ("hangs around in the feed until the same time the next day") confirms this is exactly the flat `RECENT_THRESHOLD = timedelta(hours=24)` rolling window in `get_friends_listening_now` — it should instead be a "listened today" (since local midnight) cutoff, not "within the last 24 hours." No hidden timezone bug; the window logic itself is the fix target. |
| 3 | Same song shows up twice in search | simone | `search_service.py` | Confirmed: `search_songs` does an `outerjoin` against `song_tags` and returns one row per matched tag, without `.distinct()`. A song with 3 tags returns 3 identical rows — matches simone's exact report (Crown Heights Anthem, 3 tags, showed up 3 times). `test_search_no_duplicates_multi_tag_song` already encodes the expected fix. |
| 4 | Notified on playlist-add but not on rating | aaliya | `notification_service.py` | Confirmed: `add_to_playlist` calls `create_notification`; `rate_song` does not, at all — an omission, not a logic error. Matches aaliya's report precisely (rating saves fine, no notification ever created, no delay). |
| 5 | Last song in playlist never shows up | darius | `playlist_service.py` | Confirmed: `get_playlist_songs` returns `songs[:-1]` after correctly ordering by `position` — unconditionally drops the last song by position, which is always the most recently added one. Matches darius's report exactly (missing song is always the newest; adding another song "frees" the previous one). `test_playlist_returns_all_songs` already encodes the expected fix (expects 5, bug returns 4). |

All five root causes are now confirmed and isolated to a single line or condition each — none needed deeper investigation beyond reading the service function and cross-referencing the existing tests. Original plan was to fix **#3, #4, #5** first; Milestone 2 (below) found that #3 doesn't actually reproduce against the pinned dependency versions, so it was swapped for **#1**. Final three: **#1, #4, #5**.

## Milestone 2: Reproduction

Environment: `.venv` with `SQLAlchemy 2.0.51`, `Flask-SQLAlchemy 3.1.1`, Python 3.9.6. App running via `FLASK_APP=app:create_app flask run` on `http://127.0.0.1:5000`, DB seeded via `python seed_data.py`. IDs below (users, songs, playlist) are real rows pulled directly from the seeded `instance/mixtape.db`.

### Issue #3 — NOT reproducible with current dependencies (swapped out)

How I tried to reproduce it: `curl "http://127.0.0.1:5000/songs/search?q=Anthem"` against "Crown Heights Anthem" (a seeded song with 3 tags — `rap`, `hip-hop`, `boom bap`). Expected 3 duplicate entries per the reported bug and the `search_service.py` code (`outerjoin` against `song_tags` with no `.distinct()`, which should fan out one row per tag). Actual result: `{"count":1, ...}` — no duplicates. Ran `pytest tests/test_search.py -v` to confirm: all 5 tests pass, including `test_search_no_duplicates_multi_tag_song`, which is written specifically to catch this bug.

Root cause of the non-reproduction: SQLAlchemy 2.0's legacy `Query.all()` automatically deduplicates ORM entity rows by primary key when the underlying SQL join fans out, so the duplicate rows never reach the caller. The bug is still latent in the code (the query has no explicit `.distinct()` and relies entirely on this ORM behavior to mask it), but it isn't observable as written against `requirements.txt`'s pinned `sqlalchemy>=2.0.0`. Per the milestone's guidance to swap to a different issue when one won't reproduce, moved to Issue #1.

### Issue #1 — Listening streak keeps resetting (reproduced)

How I reproduced it: couldn't use the live server for this one, since `record_listening_event` always uses `datetime.now(timezone.utc)` — there's no way to make "now" fall on a Sunday through the API without waiting for an actual Sunday. Instead, drove the underlying function directly with controlled timestamps, exactly like the existing test does: `update_listening_streak(user, saturday)` then `update_listening_streak(user, sunday)`, where `saturday = 2024-06-15` and `sunday = 2024-06-16` (consecutive calendar days).

Ran `pytest tests/test_streaks.py -v` as the reproduction: 4/5 tests pass; `test_streak_increments_on_sunday` fails with `assert 1 == 2` — after listening Saturday (streak → 1) and then Sunday (should → 2, since it's a consecutive day), the streak resets to 1 instead. This is kenji's exact report (streak 12 → 1 after a Sunday-morning listen, having listened every day with no gaps). Confirms the root cause: `update_listening_streak`'s `elif days_since_last == 1 and today.weekday() != 6:` condition excludes Sundays from the increment branch, sending them to the reset branch instead.

### Issue #4 — Notified on playlist-add but not on rating (reproduced)

How I reproduced it: picked "Midnight Drive" (shared by nova, id `815ab253-985d-4b72-afec-a922e4503e90`) and had darius (id `1629a02c-88ca-4e06-b7b9-298daff8cc21`) rate it.

1. `GET /users/<nova_id>/notifications` → baseline: 1 existing notification (a `song_added_to_playlist` one from seed data).
2. `POST /songs/815ab253-985d-4b72-afec-a922e4503e90/rate` with `{"user_id": "<darius_id>", "score": 5}` → `201`, rating saved correctly (`score: 5` returned).
3. `GET /users/<nova_id>/notifications` again → still exactly 1 notification, identical to baseline. No `song_rated` notification was created.

Matches aaliya's report exactly: rating succeeds and is visible on the song, but the sharer never gets notified, with no delay — confirms `rate_song` in `notification_service.py` simply never calls `create_notification`.

### Issue #5 — Last song in playlist never shows up (reproduced)

How I reproduced it: playlist "Late Night Vibes" (id `034b4135-0466-4fd8-806d-ee1c3f57687f`) has 7 rows in `playlist_entries` per the seeded DB (confirmed via direct sqlite query). `GET /playlists/034b4135-0466-4fd8-806d-ee1c3f57687f/songs` returns `{"count": 6, ...}` — one fewer than the DB actually has, and the missing song is whichever one sits last in `position` order. Matches darius's report exactly (playlist "says" 7, shows 6, missing one is always the most recently added).

Bonus finding (not one of the five tracked issues, not required to fix, noting for completeness): attempting the second half of the reproduction — adding a new song via `POST /playlists/<id>/songs` to watch the previously-missing song "reappear" — instead crashed with a 500. Server traceback shows `sqlite3.IntegrityError: NOT NULL constraint failed: playlist_entries.position`, from `INSERT INTO playlist_entries (playlist_id, song_id, added_at) VALUES (...)` inside `add_to_playlist` (`notification_service.py`). `playlist.songs.append(song)` only populates the two foreign keys (plus `added_at`, which has a Python-side default) — `position` and `added_by` are `NOT NULL` with no defaults, so any live attempt to add a song to a playlist currently crashes outright. This is a separate, more severe bug than #5's display-side slice, but confirms #5's own reproduction didn't need it: the count mismatch alone (6 vs. 7) is sufficient evidence.

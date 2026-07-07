# Mixtape — Bug Hunt Submission

## Milestone 1: Codebase Map

### Setup

Forked to `csharathkumar/ai201-project5-mixtape`, cloned locally, working branch `bugfix/mixtape` created off `main`. `app.py`, `models.py`, `seed_data.py`, all of `routes/`, `services/`, and `tests/` compile cleanly (`python -m py_compile`). The five open issues and their affected files are listed in `README.md`; I didn't have a separate brief document with extended descriptions, so the plan below is based on the README's one-line issue summaries plus what the code and existing tests reveal about each bug's mechanics.

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

| # | Title | Service | What I observed while reading |
|---|-------|---------|-------------------------------|
| 1 | Listening streak keeps resetting | `streak_service.py` | `update_listening_streak` has a Sunday-specific condition (`today.weekday() != 6`) on the "increment" branch that isn't part of the documented streak rules and isn't mirrored anywhere else — looks like the root cause. `test_streak_increments_on_sunday` already encodes the expected fix. |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` | `get_friends_listening_now` uses a flat `RECENT_THRESHOLD = timedelta(hours=24)` cutoff and dedups to one event per friend, but a 24-hour rolling window will legitimately include "yesterday at this exact hour," which may be what users are perceiving as stale. Need the full issue description to know if this is the actual complaint or if there's a timezone/date-boundary bug hiding underneath (module docstring implies it should feel like "now," not "within a day"). |
| 3 | Same song shows up twice in search | `search_service.py` | `search_songs` does an `outerjoin` against `song_tags` and returns `song.to_dict() for song in results` without `.distinct()` — a song joined to N tags will appear N times in the SQL result set. `test_search_no_duplicates_multi_tag_song` confirms a 3-tag song returns 3 rows. Root cause looks clear: missing dedup after the join. |
| 4 | Notified on playlist-add but not on rating | `notification_service.py` | Confirmed by reading the code directly: `add_to_playlist` calls `create_notification`; `rate_song` does not, at all. This is an omission, not a logic error. |
| 5 | Last song in playlist never shows up | `playlist_service.py` | `get_playlist_songs` returns `songs[:-1]` after already ordering correctly by `position` — drops the last song unconditionally. `test_playlist_returns_all_songs` confirms (expects 5, bug returns 4). Root cause looks clear and isolated to one line. |

Issues #3, #4, and #5 have root causes I can already point to precisely from reading the code and existing tests, with minimal risk of a hidden second cause — I plan to tackle these three first. Issues #1 and #2 are the ones I'd want the full brief's issue description for before touching: #1 because the Sunday condition could be an intentional-but-misplaced rule rather than pure dead weight (worth confirming what the "boundary" is actually supposed to be before deleting it), and #2 because "shows people from yesterday" could mean the 24-hour window itself is wrong, or could mean something else entirely (e.g. a caching/query ordering issue) that isn't obvious from `feed_service.py` alone.

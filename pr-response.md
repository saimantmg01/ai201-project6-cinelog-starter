# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used AI to :
- Inspect the existing collection implementation and tests.
- Used it to trace whether the `public` field was backed by working privacy controls
- Used it to play devil advocate to evaluate the maintainer's sort-order comment.

## Comment 1 — Rename
**What I did:**
Renamed save_to_watchlist() function to add_to_watchlist() in services/watchlist_service.py and update all call sites (routes/watchlist/watchlist.py).

**How I verified:**
Using editor's project-wide search

## Comment 2 — Deduplication
**What I did:**
Add deduplication logic to add_to_watchlist() in services/watchlist_service.py. Following the pattern from add_to_collection() in services/collection_service.py.
- Added AlreadyInWatchlistError.
- Checks for an existing entry matching user_id and film_id.
- Raises AlreadyInWatchlistError before inserting a duplicate.

**How I verified:**
Ran the pytest using ./.venv/bin/pytest -q

## Comment 3 — Missing test
**What I did:**
- Reviewed tests/test_collection.py and especially test_add_to_collection_nonexistent_film_raises to see how it works.
- Determine what import I needed for tests
- Create sample user and isolated test app with an in-memory database.
- Then added fake film id test

**How I verified:**
Ran `pytest tests/test_watchlist.py -v` and `pytest tests/ -v`.

## Comment 4 — Default visibility
**My position:**
Watchlist entries should default to `public=False` until the application has
working privacy controls.

**Reasoning:**
Although `WatchlistEntry` currently has a `public` field, that field is only
metadata. `add_to_watchlist()` does not accept a visibility choice, there is no
endpoint for changing visibility, and `get_watchlist()` does not use the field
to restrict access. The `GET /watchlist/<user_id>` endpoint also has no
authorization check. As a result, users currently cannot make an informed
visibility choice or rely on `public=False` to keep an entry private. Defaulting
to private is the safer behavior until those controls are implemented.

**Tradeoff acknowledged:**
CineLog is a community film-tracking app, so public watchlists could improve
sharing and film discovery. However, that benefit should not come from exposing
user data by default without a functioning opt-in control. Once visibility is
enforced and users can change it, the team can revisit whether public-by-default
fits the intended product experience.

## Comment 5 — Sort order
**My position:**
I agree with the maintainer and changed watchlists to sort by `date_added`
descending, so the most recently added films appear first.

**Reasoning:**
A watchlist is more useful as a record of recent intent than as an alphabetical
catalog. Users returning to it are likely to look for films they recently
decided to save, and newest-first makes those films immediately visible. This
also matches the existing `get_collection()` ordering and gives the user
a consistent experience.

**Engagement with reviewer's point:**
The maintainer's point that most users want to see recent additions is more
persuasive than keeping the current alphabetical order. Alphabetical sorting
can help someone find a known title in a long list, but that need would be
better addressed later through search or a user-selectable sort option rather
than making it the default. For now, newest-first makes the most sense.

## Comment 6 — Rebase
**What conflicted:**
The branch had a conflict in `.gitignore`. Main's UUID refactor also
replaced the older models file, which removed `WatchlistEntry` without producing
a textual merge conflict.

**How I resolved it:**
I kept the combined `.gitignore` entries, restored `WatchlistEntry` on top of
main's models, and changed its `film_id` foreign key to `db.String(36)` so it
matches the UUID primary key on `Film`. I also updated the watchlist service and
route documentation to describe film IDs as UUIDs rather than integers.

**How I verified no conflict remains:**
I ran the complete test suite, searched the watchlist code for stale integer-ID
references, checked the diff for whitespace errors, and inspected the commits
between `origin/main` and this branch with `git log --merges`. All five tests
pass and no merge commits remain in the feature branch history.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->

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

### What this PR does
Adds a Watchlist feature to CineLog, alongside the existing Collection feature. Users can:
- `POST /watchlist/<user_id>/add` — save a film to their watchlist (body: `{"film_id": "<uuid>"}`)
- `GET /watchlist/<user_id>` — view all films on their watchlist, sorted alphabetically by title, with `date_added` and `public` metadata attached to each film

It follows the same `verb_to_noun` naming convention and film-lookup pattern as the collection feature, and targets main's UUID-based `Film`/`User` models.

### Design decisions
- **Naming** — renamed the initial `save_to_watchlist()` to `add_to_watchlist()` to match the project's `verb_to_noun` convention (Comment 1).
- **Deduplication** — `add_to_watchlist()` raises `AlreadyInWatchlistError` if the `(user_id, film_id)` pair already exists, mirroring `add_to_collection()` (Comment 2).
- **Test coverage** — added a test asserting `add_to_watchlist()` raises `FilmNotFoundError` for a nonexistent `film_id`, following the pattern in `test_collection.py` (Comment 3).
- **Default visibility** — see Comment 4 for the position and full reasoning on why watchlist entries should default to private.
- **Sort order** — see Comment 5 for the position and full reasoning on `date_added` vs. alphabetical ordering.
- **Rebase onto main's UUID refactor** — resolved a silent conflict where main's model refactor dropped `WatchlistEntry`; restored it with `film_id` as `db.String(36)` to match the new UUID `Film.id` (Comment 6).

### How to manually test
1. Install dependencies and start the app:
   ```bash
   pip install -r requirements.txt
   python app.py
   ```
2. Seed a user and a film — there's no creation endpoint for either, so use a one-off shell:
   ```bash
   python3 -c "
   from app import create_app, db
   from models import User, Film
   app = create_app()
   with app.app_context():
       user = User(username='tester', email='tester@example.com')
       film = Film(title='Test Film', genre='Drama', year=2020)
       db.session.add_all([user, film])
       db.session.commit()
       print('user_id:', user.id)
       print('film_id:', film.id)
   "
   ```
3. Add the film to the watchlist:
   ```bash
   curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
   ```
   Expect `201` with the film plus `date_added` and `public` fields.
4. View the watchlist:
   ```bash
   curl http://localhost:5000/watchlist/<user_id>
   ```
   Expect a list containing the film just added.
5. Repeat step 3 with the same `film_id` — the service rejects the duplicate (`AlreadyInWatchlistError`), though it currently surfaces as a 500 due to the known limitation above rather than a clean 409.
6. Repeat step 3 with a made-up UUID for `film_id` — the service rejects it (`FilmNotFoundError`), same known limitation (500 instead of 404).
7. Run the automated test suite:
   ```bash
   pytest tests/
   ```
   All tests should pass.

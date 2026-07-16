# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

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
**How I verified:**

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
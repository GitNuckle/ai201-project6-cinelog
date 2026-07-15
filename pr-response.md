# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- fill in at the end -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` and updated its call site in `routes/watchlist/watchlist.py`. Confirmed this was the only call site by searching the codebase for `save_to_watchlist`.
**How I verified:** Ran `pytest tests/ -v` after the rename — all existing tests still passed, confirming the rename didn't break the import chain or route behavior.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception and a duplicate check in `add_to_watchlist()`, mirroring the pattern already used in `add_to_collection()` in `services/collection_service.py` — query for an existing entry with the same `user_id` and `film_id` before inserting, and raise instead of silently creating a duplicate.
**How I verified:** Wrote `test_add_to_watchlist_duplicate_raises` in `tests/test_watchlist.py` to confirm the second call raises `AlreadyInWatchlistError` and that only one entry exists in the database afterward. Ran the full suite to confirm no regressions.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` and wrote `test_add_to_watchlist_nonexistent_film_raises`, following the same fixture and assertion structure as `test_add_to_collection_nonexistent_film_raises` in `test_collection.py`. Used an integer film ID for the "fake" ID since the watchlist's `Film.id` is still an integer at this point in the branch (pre-refactor).
**How I verified:** Ran `pytest tests/test_watchlist.py -v` — both new tests pass. Ran the full suite afterward to confirm no regressions elsewhere.

## Comment 4 — Default visibility
## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default.
**Reasoning:** CineLog is a social film-tracking app — the value of a watchlist feature increases when other users can see what you're planning to watch, since it supports the kind of discovery and recommendation behavior the collection feature already relies on (visible ratings, shared activity). Defaulting to public means a new user's watchlist is useful to the community from the first entry, without requiring an extra decision before it has any social value.
**Tradeoff acknowledged:** This does mean users who want privacy by default won't get it automatically — someone building a watchlist of films tied to personal circumstances (e.g., a sensitive topic, a private project) has no opt-out until they discover the setting exists. That's a real cost. It's mitigated by the fact that this is a toggle, not a permanent choice — but only if the toggle is easy to find, which isn't guaranteed by the current API surface alone.

## Comment 5 — Sort order
## Comment 5 — Sort order
**My position:** Switched to date-added descending (most recent first), matching the reviewer's preference.
**Reasoning:** Dani-risingBW's point about finding the oldest unwatched film to finally cross off, or checking the most recent addition, describes real watchlist usage — alphabetical order doesn't support either of those tasks well. This also brings the watchlist in line with `get_collection()`, which already sorts by `date_added.desc()` — using two different sort conventions across nearly identical features would be inconsistent for no real benefit.
**Engagement with reviewer's point:** I initially chose alphabetical because it's predictable for lookup ("find film X"), but that's a weaker use case for a watchlist than a collection — a watchlist is short-lived and actively managed (add, watch, remove), while a collection is a long-term log. Recency ordering fits that active-management pattern better.

## Comment 6 — Rebase
**What conflicted:** Rebasing onto `origin/main` replayed my commits on top of the updated `models.py`, which no longer contained `WatchlistEntry` at all (it only ever existed on my branch) and had migrated `Film.id` from integer to UUID (`db.String(36)`). Git didn't flag this as a merge conflict since there was no overlapping line edit — it just silently dropped `WatchlistEntry` from the file. Running the test suite after the rebase caught it as an `ImportError`.
**How I resolved it:** Re-added the `WatchlistEntry` model to `models.py`, matching the UUID pattern already used by `CollectionEntry` — `film_id` is now `db.String(36)` instead of `db.Integer`. Also updated the fake film ID in `tests/test_watchlist.py` from an integer to a UUID string to match the new type.
**How I verified no conflict remains:** Ran `pytest tests/ -v` — all 6 tests pass. Confirmed with `git log --oneline` that the branch has no merge commits and sits cleanly on top of main's refactor commit.

## PR Description

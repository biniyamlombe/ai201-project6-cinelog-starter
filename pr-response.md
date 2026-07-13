# PR Response Doc — CineLog Watchlist Feature

## AI Usage
Used AI in Milestone 1 to orient on `models.py`, `add_to_collection()`, and the collection test patterns before reading review comments — then verified those summaries against the code.

For Comments 4 and 5, drafted my own positions first, then asked AI to play devil's advocate ("What counterargument would a careful reviewer raise? What tradeoff am I not acknowledging?"). That pushed me to (4) spell out that public-by-default only works if the privacy control is discoverable, and acknowledge spoiler/taste-leak risk more explicitly; and (5) address alphabetical scanning and FIFO (oldest-first) as real alternatives before committing to newest-first. The final arguments below are my own, grounded in CineLog as a community film tracker.

For Milestone 4, I pasted `git log --oneline origin/main..HEAD` and asked whether each message followed conventional commits and whether any commit bundled multiple logical changes. The check confirmed prefixes (`feat:`, `fix:`, `test:`, `docs:`) and one logical change per commit (rename, dedup, sort, test, and feature addition are separate). I verified that myself against CONTRIBUTING.md before finalizing.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` so it matches the project's `verb_to_noun` convention (`add_to_collection`, `remove_from_collection`, `get_collection`). Updated the import and the single call site in `routes/watchlist/watchlist.py`.
**How I verified:** Ran a project-wide search for `save_to_watchlist` — only the old definition and the route import/call existed; both were updated, and a follow-up search returned zero matches. Ran `pytest tests/ -v` after the rename.

## Comment 2 — Deduplication
**What I did:** Followed the same pattern as `add_to_collection()`: after confirming the film exists, query `WatchlistEntry` for the same `(user_id, film_id)`. If a row exists, raise `AlreadyInWatchlistError` instead of inserting a duplicate. Also wired the route to return 409 (same idea as collection returning 409 for `AlreadyInCollectionError`).
**How I verified:** Compared the new check line-by-line against `add_to_collection()` in `services/collection_service.py` (filter_by → `.first()` → raise). Ran `pytest tests/ -v` to confirm existing tests still pass.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with the same `app` / `sample_user` fixtures as `tests/test_collection.py`, and wrote `test_add_to_watchlist_nonexistent_film_raises` as the direct counterpart to `test_add_to_collection_nonexistent_film_raises` (same fake ID, same `pytest.raises(FilmNotFoundError)` assertion).
**How I verified:** Used `test_add_to_collection_nonexistent_film_raises` as the model. Ran `pytest tests/test_watchlist.py -v`, then `pytest tests/ -v` for the full suite.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default for new watchlist entries.
**Reasoning:** CineLog is positioned as a *community* film tracker — users log and share taste, not just keep a private diary. A watchlist is a natural discovery surface (“what are you hoping to see next?”). Optimizing for that social behavior means reducing friction for sharing: if every new save starts private, most lists stay invisible unless the user remembers to flip a toggle, and friends/followers never see the signal. Public-by-default matches an app that wants mutual discovery without requiring an extra step on every add.
**Tradeoff acknowledged:** Privacy-by-default is safer, and a careful reviewer would note that “want to watch” can leak spoilers, unreleased interests, or taste someone isn’t ready to broadcast. Public-by-default only works if the privacy control is obvious and easy to change; otherwise we privilege growth over consent. I’m accepting that tradeoff for v1 because the product thesis is social — and we should treat an explicit, hard-to-miss opt-out as a follow-up requirement, not an afterthought.

## Comment 5 — Sort order
**My position:** Adopt the maintainer’s preference: sort by `date_added` descending (newest first).
**Reasoning:** A watchlist behaves like a growing queue of intent, not a static catalog. When someone opens their list, the useful question is usually “what did I add lately?” not “where does *Alien* fall alphabetically?” Newest-first also matches `get_collection()`, so collection and watchlist share one mental model. Alphabetical is better for long-term browsing of a known library; that is not the primary job of “films I still mean to watch.”
**Engagement with reviewer's point:** I agree that most users want recent additions first — that is the behavior I’m optimizing for. I also considered two alternatives a reviewer might push: (1) alphabetical for easier scanning of long lists, and (2) oldest-first FIFO if we treat the watchlist as a strict “watch in the order you saved” queue. Both are valid secondary modes, but neither is the better *default* for a social tracking app. I’m changing the default to `date_added.desc()`; if we need A–Z or FIFO later, those can be query params without changing the default.

## Comment 6 — Rebase
**What conflicted:** Rebased `feature/watchlist` onto `origin/main` (`git rebase origin/main`). Git reported a clean replay (no conflict markers), but the UUID migration on `main` had already rewritten `models.py`, and the original watchlist commit never carried `WatchlistEntry` as a patch against that post-refactor file. After rebase, `Film`/`CollectionEntry` used UUID string IDs, but the `WatchlistEntry` model class was missing — imports failed (`cannot import name 'WatchlistEntry'`). Docstrings/comments still said integer `film_id`.
**How I resolved it:** Restored `WatchlistEntry` on top of main’s UUID schema with `film_id = db.String(36)` (matching `Film.id` and `CollectionEntry.film_id`). Updated `add_to_watchlist` / route docs from integer/`pre-refactor` to UUID. Left `public=True` and the newest-first sort from Comments 4–5 intact.
**How I verified no conflict remains:** `git rebase` completed; `git log --merges origin/main..HEAD` is empty (linear history, no merge commits). Grep showed no remaining integer/`pre-refactor` film ID notes in watchlist code. `pytest tests/ -v` — 5 passed.

## Commit history screenshot

`$ git log --oneline origin/main..HEAD`

![git log --oneline screenshot](docs/git-log-screenshot.png)

```
085ce86 docs: add PR response with design decisions and git log screenshot
3b828f9 test: add nonexistent film_id coverage for watchlist
1aa41ca fix: sort watchlist by date added newest first
ff8c52d fix: prevent duplicate films on watchlist add
ca5d5a4 fix: rename save_to_watchlist to add_to_watchlist
5965fb2 feat: add watchlist model and save_to_watchlist endpoint
```

No merge commits. Each commit uses conventional prefixes and one logical change.

## PR Description

### What this feature does
Adds a personal watchlist so users can save films they want to watch later. It introduces a `WatchlistEntry` model, service helpers (`add_to_watchlist`, `get_watchlist`), and REST endpoints:

- `GET /watchlist/<user_id>` — list a user’s watchlist
- `POST /watchlist/<user_id>/add` — add a film (`{ "film_id": "<uuid>" }`)

### Design decisions
1. **Default visibility (`public=True`):** Watchlist entries are public by default so CineLog stays discovery-friendly as a community film tracker. The tradeoff is weaker privacy until an obvious opt-out is exposed in the UI.
2. **Sort order (newest first):** Watchlists default to `date_added` descending so recent saves surface first, matching `get_collection()` and the “what did I add lately?” mental model instead of alphabetical title order.

### How to manually test
```bash
# Setup
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python app.py
```

Then, with the API at `http://127.0.0.1:5000`:

1. List films and pick a `film_id` and a `user_id` (create a user in the DB / use an existing one from your seed path as available):
   ```bash
   curl -s http://127.0.0.1:5000/films/ | python -m json.tool
   ```
2. Add a film to the watchlist:
   ```bash
   curl -s -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H 'Content-Type: application/json' \
     -d '{"film_id":"<film_uuid>"}' | python -m json.tool
   ```
   Expect `201` and a watchlist entry payload.
3. Fetch the watchlist (newest first):
   ```bash
   curl -s http://127.0.0.1:5000/watchlist/<user_id> | python -m json.tool
   ```
4. Add the same film again — expect `409` (duplicate rejected).
5. Add a nonexistent film UUID — expect `404`.
6. Or run the suite:
   ```bash
   pytest tests/ -v
   ```

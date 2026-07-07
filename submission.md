# Project 5: Mixtape Bug Hunt — Submission

Branch: `bugfix/mixtape`

## AI Usage

I used an AI coding assistant primarily as a **navigation and reasoning partner**, not as a code
generator. Here is an honest account of where it helped and where I had to verify things myself.

- **Reproducing before touching code.** I ran `pytest tests/` first and used the AI to help me read
  the failing assertions (e.g. `assert 4 == 5` in `test_playlists.py`) and map each failing test back
  to the exact service function it exercised. This kept me from guessing.
- **Tracing call chains.** The README says routes call services, so I asked the AI to trace paths like
  `POST /songs/<id>/rate → routes/songs.py → notification_service.rate_song()` and
  `GET /playlists/<id>/songs → routes/playlists.py → playlist_service.get_playlist_songs()`. This was
  the most useful part: it let me confirm *which* service held the bug before I opened it.
- **Explaining library behavior.** For the streak bug I asked the AI to confirm what Python's
  `datetime.weekday()` returns for each day. It correctly said Monday=0 … Sunday=6, which was the key
  fact that explained the Sunday reset. I double-checked this myself in a REPL because the whole fix
  hinged on it.
- **Where I had to verify / where AI was incomplete.** For the search duplication bug, the AI first
  suggested slapping `.distinct()` on the query as a quick fix. That would have masked the symptom but
  left an unnecessary join in place. I verified against `to_dict()` in `models.py` and saw tags are
  already loaded through the `tags` relationship (`lazy="subquery"`), so the `outerjoin(song_tags, ...)`
  was pure dead weight that fanned out one row per tag. Removing the join was the real root cause fix,
  not `.distinct()`. The notification bug also had **no test**, so I couldn't lean on the AI's "run the
  tests" loop — I confirmed it by reading `routes/songs.py` and comparing `rate_song()` to the working
  `add_to_playlist()` notification pattern by hand.

Net: AI sped up codebase navigation and confirmed language/library facts, but every root cause and fix
was verified against the actual code and test output before committing.

---

## Root Cause Analysis

### Issue #5 — The last song in a playlist never shows up

**How I reproduced it.** Ran `pytest tests/test_playlists.py`. Two tests failed:
`test_playlist_returns_all_songs` (`assert 4 == 5`) and `test_playlist_returns_songs_in_order`
(returned `['Track 1' … 'Track 4']`, missing `'Track 5'`). The `seed_playlist` fixture inserts 5 songs
at positions 1–5, so a playlist with 5 songs was consistently returning only the first 4. The missing
song was always the one with the **highest position** (the last-added track).

**How I found the root cause.** The failing tests call `get_playlist_songs`, so I opened
`services/playlist_service.py`. The SQL query itself looked correct — it joins `playlist_entries`,
filters by `playlist_id`, and orders by `position` ascending. The moment I was sure I'd found the cause
was the return line: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice was silently
throwing away the last element of an already-correct, correctly-ordered result set.

**The root cause.** The list comprehension sliced the query result with `songs[:-1]`, which returns
every element *except the last one*. Because the query orders songs by ascending `position`, the last
element is always the highest-position song. So every non-empty playlist returned `n − 1` songs and
always dropped the final track. (An empty playlist happened to work by accident, since `[][:-1]` is
still `[]`.)

**Fix and side-effect check.** Changed `songs[:-1]` to `songs` so all rows are returned.
After the change, `test_playlist_returns_all_songs` and `test_playlist_returns_songs_in_order` pass,
`test_empty_playlist_returns_empty_list` still returns `[]`, and the full suite (13 tests) passes. I
also confirmed ordering is untouched since the `.order_by(asc(position))` clause was never the problem.

---

### Issue #1 — My listening streak keeps resetting

**How I reproduced it.** Ran `pytest tests/test_streaks.py`. `test_streak_increments_on_sunday`
failed: listening on Saturday (`2024-06-15`) then Sunday (`2024-06-16`) produced a streak of `1`
instead of `2`. The general-day tests (Mon→Tue, same-day, skipped-day) passed, which localized the bug
to something specific about Sunday.

**How I found the root cause.** The failing test calls `update_listening_streak`, so I opened
`services/streak_service.py` and read the branch that decides whether to increment. The condition was
`elif days_since_last == 1 and today.weekday() != 6:`. The `!= 6` was the tell — I confirmed that
`datetime.weekday()` numbers days Monday=0 … Sunday=6, so `weekday() != 6` is specifically "is not
Sunday."

**The root cause.** `datetime.weekday()` returns `6` for Sunday. The increment branch required both
`days_since_last == 1` **and** `today.weekday() != 6`. So whenever the current listening day landed on a
Sunday — even though it was exactly one day after the previous listen — the second condition was `False`,
the `elif` was skipped, and control fell through to the `else` branch, which resets the streak to `1`.
In other words, any streak that continued into a Sunday was wrongly reset instead of incremented. There
was no legitimate reason to treat Sunday differently; the extra clause was simply wrong.

**Fix and side-effect check.** Removed the `and today.weekday() != 6` clause, leaving
`elif days_since_last == 1:`. After the fix, Saturday→Sunday increments to `2`. I re-ran all streak
tests to confirm the other branches still behave correctly: same-day listens don't double-count
(`days_since_last == 0` returns early), a skipped day still resets to `1` (`days_since_last > 1` hits the
`else`), and normal consecutive days still increment. Full suite passes.

---

### Issue #3 — The same song keeps showing up twice in search

**How I reproduced it.** Ran `pytest tests/test_search.py`. `test_search_no_duplicates_multi_tag_song`
failed: searching `"Crown Heights"` returned the song "Crown Heights Anthem" **3 times** instead of
once. The seed data gives that song exactly 3 tags, which matched the duplicate count and pointed
straight at the tags relationship.

**How I found the root cause.** The failing test calls `search_songs`, so I opened
`services/search_service.py`. The query chained `.outerjoin(song_tags, Song.id == song_tags.c.song_id)`
before filtering on title/artist. I confirmed the mechanism by checking `models.py`: `song_tags` is a
many-to-many association table with one row per (song, tag) pair, and `Song.to_dict()` already loads tag
names through the `tags` relationship (`lazy="subquery"`). That was the moment it clicked — the join
wasn't needed at all, and it was multiplying rows.

**The root cause.** The search query performed an outer join against the `song_tags` association table.
Because that table has one row per tag, the join produced one result row per tag on each matching song.
A song with 3 tags therefore appeared as 3 identical `Song` rows in the result, and the comprehension
turned those into 3 duplicate dicts. The join served no purpose: the `WHERE` clause only matches
`title`/`artist`, and tags are fetched separately by the relationship inside `to_dict()`.

**Fix and side-effect check.** Removed the `.outerjoin(song_tags, ...)` line so the query selects songs
directly with no row fan-out. I intentionally did **not** use `.distinct()`, because that would only
hide the duplicates while keeping a pointless join. After the fix: the multi-tag song appears exactly
once, and I re-ran the sibling tests to confirm a one-tag song and a zero-tag song each appear exactly
once, a basic match still returns the song, and a no-match query returns `[]`. Crucially, tags still
show up in each result dict via the relationship. Full suite passes.

---

### Issue #4 — I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it.** There is no automated test for notifications, so I reproduced it by tracing the
code path. A rating comes in via `POST /songs/<song_id>/rate` → `routes/songs.py` → 
`notification_service.rate_song()`. Reading `rate_song()`, it saved the `Rating` and returned it, but
never created any `Notification`. By contrast, `add_to_playlist()` in the same file ends with a
`create_notification(...)` call to the song's owner. So adding a song to a playlist notified the owner,
but rating a song did not — exactly the reported behavior.

**How I found the root cause.** I compared the two sibling functions in
`services/notification_service.py` side by side. `add_to_playlist()` had the pattern
`if song.shared_by != added_by_user_id: create_notification(...)` after committing; `rate_song()` had no
equivalent block at all. That asymmetry was the confirmation that the notification step was simply
missing from the rating flow, not broken somewhere downstream.

**The root cause.** `rate_song()` was missing the notification step entirely. It committed the rating
and returned without ever calling `create_notification`, so the song's original sharer received nothing
when someone rated their song. This wasn't a wrong comparison or a swallowed error — the code to notify
was never written for the rating path, even though the playlist path had it.

**Fix and side-effect check.** After `db.session.commit()` in `rate_song()`, I added a notification that
mirrors the playlist pattern, guarded so users aren't notified about rating their own songs:

```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

Side-effect check: the guard prevents self-notifications (rating your own song creates none), the
existing update path (re-rating a song you already rated) also flows through this block, and
`add_to_playlist()` behavior is unchanged. Since there's no notification test, I re-read the route and
the `create_notification` helper to confirm the arguments match its signature, and I ran the full suite
(13 tests) to confirm nothing else regressed.

# Mixtape Bug Hunt Submission

## AI Usage

I used AI tools mainly for codebase navigation, debugging support, and checking my root cause explanations. I first used AI to help summarize the purpose of the main route and service files so I could understand the structure of the codebase before making changes. I also used AI to compare similar service patterns, especially the difference between the working playlist notification flow and the missing rating notification flow.

During debugging, I used AI to help reason through suspicious code after I had already found the relevant files. For example, I used it to understand why a Sunday boundary condition could affect streak logic, why joining a song table with a tag association table could create duplicate rows, and why slicing with `songs[:-1]` removed the last playlist item. I verified the suggestions myself by reading the code, checking the tests, and running `pytest`.

I did not rely on AI alone to decide the fixes. I used the project README, the existing tests, the service code, and the seeded data comments to confirm the behavior before changing the code.

---

## Codebase Map

Mixtape is a Flask app for a social music platform where users can share songs, build playlists, view friend activity, track listening streaks, and receive notifications.

### Main Files and Responsibilities

* `app.py`: Creates the Flask app, configures SQLAlchemy, initializes the database, and registers the main blueprints for songs, playlists, users, and feed.
* `seed_data.py`: Populates the database with test users, friendships, songs, tags, listening events, playlists, and example notifications.
* `routes/songs.py`: Handles song search, song detail lookup, song rating, and listening events. It delegates most business logic to `search_service`, `notification_service`, and `streak_service`.
* `routes/playlists.py`: Handles playlist creation, playlist retrieval, getting songs in a playlist, and adding songs to a playlist. It delegates playlist logic to `playlist_service` and notification-related playlist actions to `notification_service`.
* `routes/users.py`: Handles user profile lookup, streak lookup, notification retrieval, and marking notifications as read.
* `routes/feed.py`: Handles the Friends Listening Now feed and the general activity feed.
* `services/streak_service.py`: Handles listening event creation and streak updates.
* `services/feed_service.py`: Builds the Friends Listening Now feed and the general activity feed.
* `services/search_service.py`: Handles song search by title or artist.
* `services/notification_service.py`: Creates and retrieves notifications, marks notifications as read, adds songs to playlists, and saves song ratings.
* `services/playlist_service.py`: Creates playlists and retrieves playlist metadata or ordered playlist songs.

### Data Flow Example: Rating a Song

When a user rates a song, the request goes to `POST /songs/<song_id>/rate` in `routes/songs.py`. The route reads `user_id` and `score` from the request body, validates that they are present, and calls `rate_song(user_id, song_id, score)` from `services/notification_service.py`.

Inside `rate_song`, the service checks that the score is between 1 and 5, retrieves the song, retrieves the user who rated it, and checks whether that user already rated the song. If a previous rating exists, it updates the score. If not, it creates a new `Rating` record. After my fix, the service also creates a `song_rated` notification for the original sharer if the rater is not the same person who shared the song.

### Pattern I Noticed

The app follows a route-to-service structure. The route files mostly handle request parsing, validation, and JSON responses. The service files contain the business logic. This helped narrow the bug hunt because the README also stated that the open issues are in the service layer.

---

## Bug Fixes

### Issue #1: My listening streak keeps resetting

#### How I reproduced it

I reproduced this by using the existing streak test that checks a user listening on Saturday and then Sunday. The expected behavior is that the streak increments from 1 to 2 because Sunday is the next calendar day. Before the fix, the test failed because the streak reset instead of incrementing.

#### How I found the root cause

I started from the issue description and the README, which pointed this bug to `services/streak_service.py`. I read `record_listening_event()` first because that is the function called when a user listens to a song. That function creates a listening event and then calls `update_listening_streak(user, now)`. I then focused on `update_listening_streak()` because it contains the date comparison logic.

The moment that made the root cause clear was this condition:

```python
elif days_since_last == 1 and today.weekday() != 6:
```

That condition treats Sunday differently from every other consecutive day.

#### The root cause

The streak rules say that if the user listened yesterday, the streak should increment by 1. However, the code only incremented the streak if `days_since_last == 1` and `today.weekday() != 6`.

In Python, `datetime.weekday()` returns `6` for Sunday. That means a Saturday-to-Sunday listening pattern was treated as a reset case instead of a consecutive-day case. The bug was not that the date difference was wrong. The bug was the extra Sunday condition that contradicted the stated streak rules.

#### Your fix and side-effect check

I removed the Sunday exception and changed the condition to:

```python
elif days_since_last == 1:
    user.listening_streak += 1
```

This fixes the root cause because any true consecutive calendar day now increments the streak, including Sunday. I checked related streak behavior by running the streak tests for starting a new streak, incrementing on a normal consecutive day, not double-counting the same day, resetting after a skipped day, and incrementing on Sunday.

Commit:

```bash
fix: preserve listening streak across Sunday
```

---

### Issue #2: Friends Listening Now shows people from yesterday

#### How I reproduced it

I reproduced this by checking the Friends Listening Now behavior against the seed data. The seed data creates recent listening events within the past 30 minutes and older listening events from hours or days ago. The bug occurs because older events from yesterday can still appear in the “Listening Now” feed.

#### How I found the root cause

I followed the README’s affected service list to `services/feed_service.py`. I read `get_friends_listening_now()` because that function builds the Listening Now feed. The function calculates a cutoff time using `RECENT_THRESHOLD`, then returns friend listening events where `listened_at >= cutoff`.

The suspicious value was:

```python
RECENT_THRESHOLD = timedelta(hours=24)
```

That means the feed was not really “now”; it was “within the last 24 hours.”

#### The root cause

The service used a 24-hour threshold for the Friends Listening Now feed. Because of that, a friend who listened yesterday could still appear as currently listening. The filtering logic itself was working, but the cutoff window was too large for the feature.

#### Your fix and side-effect check

I changed the threshold to:

```python
RECENT_THRESHOLD = timedelta(minutes=30)
```

This fixes the root cause because the Listening Now feed now only includes very recent activity. I also checked that `get_activity_feed()` was not changed because that function is supposed to return recent activity regardless of the Listening Now threshold.

Commit:

```bash
fix: limit listening now feed to recent activity
```

---

### Issue #3: The same song keeps showing up twice in search

#### How I reproduced it

I reproduced this with the existing search test for a song with multiple tags. The test creates a song named `Crown Heights Anthem` with three tags and searches for it. The expected result is that the song appears exactly once. Before the fix, the bug caused the multi-tag song to appear multiple times.

#### How I found the root cause

I started in `services/search_service.py` because the README listed that as the affected service. I looked at `search_songs(query)` and noticed that it used an outer join between `Song` and the `song_tags` association table, even though the search filter only checked `Song.title` and `Song.artist`.

The important part was:

```python
.outerjoin(song_tags, Song.id == song_tags.c.song_id)
```

The tests made the issue clearer because the duplicate bug only appeared for a song with multiple tags.

#### The root cause

The query joined songs against the tag association table. A song with three tags has three rows in `song_tags`, so the join can produce multiple rows for the same song. Since the search only filters by title or artist, the join was unnecessary and created duplicate search results for multi-tag songs.

#### Your fix and side-effect check

I removed the unnecessary join and searched directly on the `Song` table:

```python
results = (
    db.session.query(Song)
    .filter(
        db.or_(
            Song.title.ilike(f"%{query}%"),
            Song.artist.ilike(f"%{query}%"),
        )
    )
    .all()
)
```

This fixes the root cause because each song is queried once from the `Song` table instead of once per matching tag association row. I checked the search tests for no-tag songs, one-tag songs, multi-tag songs, matching songs, and no-result searches.

Commit:

```bash
fix: remove duplicate songs from search results
```

---

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

#### How I reproduced it

I reproduced this by comparing the playlist-add notification behavior to the rating behavior. Adding another user's song to a playlist creates a notification for the original sharer, but rating another user's song only creates or updates a rating record. No notification is created for the song sharer.

#### How I found the root cause

I started in `services/notification_service.py` because the README listed it as the affected service. I compared `add_to_playlist()` and `rate_song()`. The playlist function creates a notification after adding the song to the playlist:

```python
create_notification(
    user_id=song.shared_by,
    notification_type="song_added_to_playlist",
    body=...
)
```

Then I looked at `rate_song()` and saw that it validates the score, retrieves the song and rater, creates or updates the rating, commits, and returns the rating. There was no notification step.

#### The root cause

The rating flow saved the rating correctly, but it did not follow the same notification pattern used by the playlist-add flow. The missing step was a call to `create_notification()` after a rating is created or updated. Because of that, the original sharer was never notified when someone rated their song.

#### Your fix and side-effect check

I added a notification after the rating is committed:

```python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score}/5.",
    )
```

The condition prevents users from receiving notifications for rating their own songs. I checked that invalid scores still raise an error, missing songs and users still raise errors, existing ratings still update, and playlist-add notifications were not changed.

Commit:

```bash
fix: notify sharer when their song is rated
```

---

### Issue #5: The last song in a playlist never shows up

#### How I reproduced it

I reproduced this with the existing playlist test. The test creates a playlist with five songs and expects `get_playlist_songs()` to return all five songs in order. Before the fix, the function returned only four songs.

#### How I found the root cause

I started in `services/playlist_service.py` because the README listed it as the affected service. I read `get_playlist_songs()`, which queries songs joined through `playlist_entries`, filters by playlist ID, orders by position, and stores the result in `songs`.

The root cause became clear at the return statement:

```python
return [song.to_dict() for song in songs[:-1]]
```

The query returned the songs, but the return statement sliced off the last item.

#### The root cause

The function used `songs[:-1]`, which means “all songs except the last one.” This caused the final song in every playlist to be dropped from the response. The database query and ordering logic were not the problem; the issue was the list slicing at the end.

#### Your fix and side-effect check

I changed the return statement to:

```python
return [song.to_dict() for song in songs]
```

This fixes the root cause because the full ordered query result is now returned. I checked that playlists return all songs, songs remain in position order, and empty playlists still return an empty list.

Commit:

```bash
fix: return final song in playlist results
```

---

## Regression Test

The existing tests already include regression coverage for several of the bugs:

* `tests/test_streaks.py` includes a test that catches the Sunday streak bug.
* `tests/test_search.py` includes a test that catches duplicate search results for multi-tag songs.
* `tests/test_playlists.py` includes a test that catches the missing last playlist song.

The regression test I am highlighting is `test_playlist_returns_all_songs` from `tests/test_playlists.py`. It creates a playlist with five songs and asserts that `get_playlist_songs()` returns five songs. Before the fix, this test failed because `songs[:-1]` returned only four songs. After the fix, the test passes.

---

## Commit History

Screenshot included separately from:
## Commit History

![Git log showing separate fix commits](screenshots/git-log.png)

```bash
git log --oneline
```

Expected commits:

```bash
fix: preserve listening streak across Sunday
fix: limit listening now feed to recent activity
fix: remove duplicate songs from search results
fix: notify sharer when their song is rated
fix: return final song in playlist results
```

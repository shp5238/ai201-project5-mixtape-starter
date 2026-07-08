# Mixtape Bug Hunt Submission

## AI Usage

I used AI to help orient myself in the codebase, to generate the basic remplte of main files and responsibilities, asking about the purpose of different functions and pages and how they connected with others.

I used it to clarify programming questions like what are the possible values of today.weekday() when reproducing the bug, and understand how requests move from Flask routes into service functions. I also used it to help generate test scripts from the snippets of code I wrote for debugging to help refine my testing in an independent compiler, and refine the formalism for this md file.  

I verified the explanations by reading the actual source files and comparing them with the README and project brief. and to my tracing and testing. Lastly, I also used AI to help me think of commit messages for brainstorming. 

## Codebase Map

Mixtape is a Flask app for a social music platform. Users can share songs, rate songs, add songs to playlists, listen to songs, view friend activity, and receive notifications when friends interact with their shared songs. The app follows a clear route-service-model structure: routes handle HTTP requests and responses, services contain the business logic, and models define the database tables.

### Main Files and Responsibilities

`app.py` creates and configures the Flask application. It sets up the SQLAlchemy database connection, registers the route blueprints, and creates database tables inside the app context. The app is split into four route groups: songs, playlists, users, and feed.

`models.py` defines the database schema using SQLAlchemy models. The main models are `User`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, and `Tag`. It also defines association tables for friendships, song tags, and playlist entries. The `playlist_entries` table is especially important because it connects playlists to songs and stores extra metadata like `position`, `added_by`, and `added_at`, so playlist order is based on an explicit position instead of just the order SQL returns rows.

`routes/songs.py` defines endpoints related to songs. It handles song search, song detail lookup, rating a song, and recording that a user listened to a song. The route functions mostly parse request data and then call service functions such as `search_songs()`, `get_song()`, `rate_song()`, and `record_listening_event()`.

`routes/playlists.py` defines endpoints for creating playlists, viewing playlist metadata, getting songs in a playlist, and adding songs to a playlist. It delegates playlist creation and retrieval to `playlist_service.py`, while adding a song to a playlist goes through `notification_service.add_to_playlist()` because adding a song can also create a notification.

`routes/users.py` defines endpoints for user profiles, listening streaks, notifications, and marking notifications as read. It reads user data from the database directly for the profile route, but delegates streak and notification behavior to the service layer.

`routes/feed.py` defines endpoints for friend activity. `GET /feed/<user_id>/listening-now` calls `get_friends_listening_now()`, while `GET /feed/<user_id>/activity` calls `get_activity_feed()`.

`services/streak_service.py` contains listening streak logic. `record_listening_event()` creates a `ListeningEvent` and calls `update_listening_streak()` to update the user's streak based on the date of their previous listen. This service owns the rules for starting a streak, incrementing it on consecutive days, avoiding double-counting multiple listens on the same day, and resetting after a skipped day.

`services/feed_service.py` builds feed data from friends' listening events. `get_friends_listening_now()` finds recent listening events from the current user's friends, keeps only the most recent event per friend, and returns friend and song dictionaries. `get_activity_feed()` returns recent friend listening events without the same "listening now" recency filter.

`services/search_service.py` contains song search behavior. `search_songs()` searches `Song.title` and `Song.artist` using a case-insensitive match and returns song dictionaries. It joins through the song tag table so songs with tags can be included with their tag data.

`services/notification_service.py` creates and retrieves notifications. `create_notification()` inserts notification records. `add_to_playlist()` adds a song to a playlist and notifies the original sharer if someone else added their song. `rate_song()` creates or updates a rating for a song. The notification routes call `get_notifications()` and `mark_as_read()`.

`services/playlist_service.py` owns playlist creation and playlist retrieval logic. `create_playlist()` creates a playlist for a user. `get_playlist_songs()` queries songs through `playlist_entries` and orders them by the `position` column. `get_playlist()` returns playlist metadata, and `get_user_playlists()` returns playlists created by one user.

`seed_data.py` resets and populates the database with sample users, friendships, songs, tags, listening events, playlists, playlist entries, and example notifications. The seed data is designed to make the reported bugs easier to reproduce, including songs with multiple tags, playlists with several songs, existing friendships, and recent versus older listening events.

`tests/` contains pytest tests for streaks, search, and playlists. These tests show the expected behavior for several of the reported issues, such as Sunday streaks incrementing correctly, search results not duplicating songs with multiple tags, and playlists returning every song in order.

### Data Flow Example: Rating a Song

When a user rates a song, the request starts at `POST /songs/<song_id>/rate` in `routes/songs.py`. The route reads the JSON request body and expects `user_id` and `score`. If either value is missing, the route returns a `400` error.

If the request data is valid, the route calls `notification_service.rate_song(user_id, song_id, score)`. Inside `rate_song()`, the service first validates that the score is between 1 and 5. Then it loads the `Song` and `User` from the database. If either one does not exist, it raises a `ValueError`, which the route turns into an error response.

After validation, `rate_song()` checks whether this user has already rated this song by querying the `Rating` table for the same `user_id` and `song_id`. If a rating already exists, the service updates the score. If no rating exists, it creates a new `Rating` row and adds it to the session. Finally, it commits the database transaction and returns the `Rating` object. The route converts that rating to a dictionary and returns it as JSON.

This flow shows the app's general pattern: the route handles HTTP details, the service validates and performs business logic, and the model represents the persistent database record.


## Root Cause Analysis: Issue #1 - My Listening Streak Keeps Resetting

Reported by: kenji

### Issue number and title

Issue #1 - My listening streak keeps resetting.

### How I reproduced it

Kenji listens to something on Mixtape every calendar day. On Saturday night, his streak was 12. On Sunday morning, he listened again and then checked `GET /users/<my_id>/streak`. The expected result was that his streak would go from 12 to 13 because Saturday and Sunday are consecutive days. The actual result was that his streak reset to 1, as if he had skipped a day. When he listened again on Monday, the streak increased to 2, which showed that the counter started working again after throwing away the previous streak.

To reproduce this without running the full Flask app or installing dependencies, I isolated the streak logic into a small Python script. This recreates Kenji as a plain `User` object with a streak of 12 and a `last_listened_at` value on Saturday night, then calls `update_listening_streak()` for Sunday morning and Monday morning.

```python
from datetime import datetime, timezone


class User:
    def __init__(self, username, email, listening_streak=0, last_listened_at=None):
        self.username = username
        self.email = email
        self.listening_streak = listening_streak
        self.last_listened_at = last_listened_at


def update_listening_streak(user, now):
    today = now.date()

    if user.last_listened_at is None:
        user.listening_streak = 1
        user.last_listened_at = now
        return

    last_listened = user.last_listened_at
    if last_listened.tzinfo is None:
        last_listened = last_listened.replace(tzinfo=timezone.utc)

    last_date = last_listened.date()
    days_since_last = (today - last_date).days

    if days_since_last == 0:
        # Already updated today — no change needed
        return
    elif days_since_last == 1 and today.weekday() != 6:
        user.listening_streak += 1
    else:
        user.listening_streak = 1

    user.last_listened_at = now


kenji = User(
    username="kenji",
    email="kenji@mixtape.app",
    listening_streak=12,
    last_listened_at=datetime(2024, 6, 15, 22, 0, 0, tzinfo=timezone.utc),
)

sunday_morning = datetime(2024, 6, 16, 9, 0, 0, tzinfo=timezone.utc)
monday_morning = datetime(2024, 6, 17, 9, 0, 0, tzinfo=timezone.utc)

print("Before:", kenji.listening_streak, ", Expected", kenji.listening_streak + 1)
update_listening_streak(kenji, sunday_morning)
print("After Sunday listen:", kenji.listening_streak)
print("Debug: Sunday weekday value:", sunday_morning.date().weekday())

update_listening_streak(kenji, monday_morning)
print("After Monday listen:", kenji.listening_streak)
print("Debug: Monday weekday value:", monday_morning.date().weekday())
```

The output shows the bug:

```text
Before: 12 , Expected 13
After Sunday listen: 1
Sunday weekday value: 6
After Monday listen: 2
Monday weekday value: 0
```

This confirms the reported behavior. Kenji listened on Saturday and then Sunday, so the streak should have become 13. Instead, the code reset it to 1 because `today.weekday()` returns `6` on Sunday, and the current condition blocks streak increments on Sundays. When the same user listens again on Monday, `today.weekday()` returns `0`, so the code allows the consecutive-day increment and changes the streak from 1 to 2.

### How I found the root cause

I started from the user-facing route in `routes/users.py`, where `GET /users/<user_id>/streak` calls `get_streak()` from `services/streak_service.py`. Since the bug happened after listening to a song, I also traced `POST /songs/<song_id>/listen` in `routes/songs.py`, which calls `record_listening_event()` in `services/streak_service.py`. That function creates the listening event and then calls `update_listening_streak()`, so `update_listening_streak()` was the exact place where the streak value could incorrectly change from 12 to 1.

The moment that made the root cause clear was the condition `days_since_last == 1 and today.weekday() != 6`. The code already knew Saturday to Sunday was exactly one day apart, but it had an extra check that specifically prevented the increment when today was Sunday.

### The root cause

Python's `date.weekday()` returns `6` for Sunday. In `update_listening_streak()`, the code only increments a consecutive-day streak when `days_since_last == 1 and today.weekday() != 6`. That means a normal Saturday-to-Sunday listen is treated differently from every other consecutive-day listen. Since Sunday fails the condition, the code skips the increment branch and goes into the `else` branch, resetting the streak to 1.

### My fix and side-effect check

The fix is to remove the Sunday exclusion and let every `days_since_last == 1` case increment the streak. This matches the stated rule: if the user listened yesterday, the streak increments by 1. I checked the related behavior around the boundary: same-day listens still do not double-count because `days_since_last == 0` returns early, skipped days still reset because they fall into the `else`, and Monday after the Sunday reset increments from 1 to 2 under the old logic.

## Root Cause Analysis: Issue #2 - Friends Listening Now Shows People From Yesterday

### Issue number and title

Issue #2 - Friends Listening Now shows people from yesterday.

### How I reproduced it

Nova opened `GET /feed/<my_id>/listening-now` around 9 AM. Darius appeared in the feed even though his last listen was at 11 PM the previous night and he had not opened the app that morning. I reproduced the timing condition directly by comparing Darius's 11 PM listen against the current code's 24-hour cutoff and then against the expected start-of-day cutoff.

``python
from datetime import datetime, timedelta, timezone

RECENT_THRESHOLD = timedelta(hours=24)

# Scenario:
# Nova opens "Friends Listening Now" at 9 AM on June 17.
# Darius last listened at 11 PM on June 16, the previous night.
darius_last_listen = datetime(2024, 6, 16, 23, 0, 0, tzinfo=timezone.utc)
nova_opens_feed = datetime(2024, 6, 17, 9, 0, 0, tzinfo=timezone.utc)

# Current buggy logic: anything from the last 24 hours is included.
buggy_cutoff = nova_opens_feed - RECENT_THRESHOLD

print("Buggy 24-hour cutoff:", buggy_cutoff)
print("Darius last listened:", darius_last_listen)
print("Does Darius show up in Friends Listening Now?")
print(darius_last_listen >= buggy_cutoff)

print()

# Expected logic: only people who listened today should be included.
start_of_today = datetime(2024, 6, 17, 0, 0, 0, tzinfo=timezone.utc)

print("Expected start-of-today cutoff:", start_of_today)
print("Darius last listened:", darius_last_listen)
print("Should Darius show up in Friends Listening Now?")
print(darius_last_listen >= start_of_today)
```

expected output: 
```txt
Buggy 24-hour cutoff: 2024-06-16 09:00:00+00:00
Darius last listened: 2024-06-16 23:00:00+00:00
Does Darius show up in Friends Listening Now?
True

Expected start-of-today cutoff: 2024-06-17 00:00:00+00:00
Darius last listened: 2024-06-16 23:00:00+00:00
Should Darius show up in Friends Listening Now?
False
```

### How I found the root cause

I started from `routes/feed.py`, where `GET /feed/<user_id>/listening-now` calls `get_friends_listening_now()` in `services/feed_service.py`. In that service, the feed uses `RECENT_THRESHOLD = timedelta(hours=24)` and sets `cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD`. That matched Nova's report exactly: at 9 AM, a 24-hour window reaches back to 9 AM the previous day, so an 11 PM listen from yesterday still qualifies.

The moment that made the root cause clear was comparing the phrase "what they've played today" from the issue description with the code's "last 24 hours" cutoff. Those are not the same rule.

### The root cause

`get_friends_listening_now()` uses a rolling 24-hour cutoff instead of the start of the current calendar day. Because of that, anything from yesterday evening remains visible until the same time the next day. At 9 AM on June 17, the cutoff is 9 AM on June 16, so Darius's 11 PM June 16 listen is incorrectly included even though it was not today.

### My fix and side-effect check

The fix is to calculate the cutoff as the start of today, not as 24 hours before the current time. That keeps listens from earlier today in the feed while excluding listens from yesterday. The side-effect check is to verify both sides of the boundary: Darius's 11 PM previous-night listen should be excluded, while a friend who listened at 8 AM today should still be included.

## Root Cause Analysis: Issue #4 - Rating A Song Does Not Create A Notification

### Issue number and title

Issue #4 - I got notified when a friend added my song to a playlist but not when they rated it.

### How I reproduced it

The reported behavior says playlist-add notifications work, but rating notifications never appear. I reproduced this by tracing the two similar flows in `services/notification_service.py`: the working `add_to_playlist()` path and the broken `rate_song()` path. The important data condition is that a friend rates a song shared by another user. The rating itself is saved, but the original sharer has no new notification to retrieve from `GET /users/<my_id>/notifications`.

### How I found the root cause

I compared `add_to_playlist()` and `rate_song()` line by line because both functions represent a friend interacting with someone else's shared song. `add_to_playlist()` loads the song, the acting user, and the playlist, then calls `create_notification()` for the original song sharer when the adder is not the sharer. `rate_song()` loads the song and rater, creates or updates the `Rating`, commits the transaction, and returns the rating.

The moment that made the root cause clear was that `rate_song()` never calls `create_notification()`. The notification retrieval code was not the problem because playlist-add notifications already show up through the same `get_notifications()` function.

### The root cause

This is an architectural issue, not a typo. The rating flow saves the `Rating` record but does not perform the notification side effect that the playlist-add flow performs. `add_to_playlist()` explicitly creates a `Notification` row for the song's original sharer, but `rate_song()` commits and returns without creating any notification. Since `get_notifications()` only reads records from the `Notification` table, the sharer has nothing to see after their song is rated.

### My fix and side-effect check

The fix is to add notification creation to `rate_song()` when the rater is not the original sharer of the song. This makes rating behavior match the existing playlist-add pattern. The side-effect check is to confirm that the rating still saves or updates correctly, the original sharer receives a notification when someone else rates their song, and users do not receive notifications for rating their own songs.

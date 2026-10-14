# AI Usage Section

**How I used AI tools**: I used AI tools to help me understand functions that I haven't seen before, and confirm lines of code that looked off or potentially buggy.

**What they helped me understand**: They helped me understand python-native functions, functions related to datetimes, and how to best simulate my bugs and fixes using the command line and sqlite3.

**What I overrode or corrected**: I was able to use the information they gave me and apply it to the fixes that I needed to make. I could use the ids from the command they provided to execute the bug reproduction commands myself.

# Mixtape Codebase Map

Mixtape is a Flask app backed by SQLAlchemy. Users share songs, rate them, record listening events, build playlists, and see what their friends are listening to. The code follows a three-layer shape: **routes** parse HTTP, **services** hold the business logic, and **models** define the schema and serialization.

---

## Top-Level Files

**`app.py`** is the application factory. `create_app(config)` reads `DATABASE_URL` (defaulting to a local SQLite file) and `SECRET_KEY` from the environment, merges any config override passed in, and initializes the shared `db = SQLAlchemy()` object. It registers four blueprints under URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`) and calls `db.create_all()` on startup. The `db` object is defined here and imported by every model, route, and service module.

**`models.py`** defines seven SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, and `Notification`. It also defines three association tables:

- `friendships` is a self-referencing many-to-many on `user`. `User.friends` is a dynamic relationship over it.
- `song_tags` links `Song` and `Tag`.
- `playlist_entries` links `Playlist` and `Song` with extra columns: `position` (explicit order), `added_by`, and `added_at`. `Playlist.songs` reads through this table.

Every model uses string UUID primary keys and UTC timestamps. `Rating` has a unique constraint on `(user_id, song_id)`, so each user has at most one rating per song. Each model has a `to_dict()` method, so serialization lives with the data rather than in the routes.

**`seed_data.py`** populates the database with sample users (some with pre-set streaks), songs, and listening history for manual testing.

**`requirements.txt`** lists Flask, Flask-SQLAlchemy, SQLAlchemy 2.x, python-dotenv, and pytest.

---

## Routes (`routes/`)

Each route module is a Blueprint. Route handlers read the request, check that required fields are present (returning 400 if not), call one service function, and convert the result to JSON. If the service raises `ValueError`, the route returns 404 or 400 depending on the endpoint.

| Module                  | Endpoints                                                                                                                   | Calls                                                                                                          |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `routes/songs.py`     | `GET /songs/search?q=`, `GET /songs/<id>`, `POST /songs/<id>/rate`, `POST /songs/<id>/listen`                       | `search_service`, `notification_service.rate_song`, `streak_service.record_listening_event`              |
| `routes/playlists.py` | `POST /playlists/`, `GET /playlists/<id>`, `GET /playlists/<id>/songs`, `POST /playlists/<id>/songs`                | `playlist_service`, `notification_service.add_to_playlist`                                                 |
| `routes/users.py`     | `GET /users/<id>`, `GET /users/<id>/streak`, `GET /users/<id>/notifications`, `POST /users/notifications/<id>/read` | `streak_service.get_streak`, `notification_service`; `GET /users/<id>` reads the `User` model directly |
| `routes/feed.py`      | `GET /feed/<id>/listening-now`, `GET /feed/<id>/activity`                                                               | `feed_service`                                                                                               |

`routes/__init__.py` and `services/__init__.py` are empty package markers.

---

## Services (`services/`)

Services hold the business logic. They take plain IDs and values, query and mutate models through the shared `db.session`, and commit their own transactions. They know nothing about HTTP. Missing entities are reported by raising `ValueError`.

**`search_service.py`** does case-insensitive substring matching on song title and artist (`search_songs`), and fetches a single song as a dict (`get_song`).

**`streak_service.py`** owns listening events and streaks. `record_listening_event` checks that the user exists, creates a `ListeningEvent`, calls `update_listening_streak`, and commits. `update_listening_streak` compares the calendar date of the user's `last_listened_at` with today's date. The streak stays the same on the same day, increments by one on the next day (with a weekday condition in that branch), and otherwise resets to 1. `get_streak` returns the stored value.

**`notification_service.py`** handles notifications and ratings. `create_notification` inserts a `Notification` row. `add_to_playlist` adds a song to a playlist's `songs` collection, commits, and creates a notification for the song's sharer when someone else adds it. `rate_song` validates the 1–5 score and confirms the song and user exist, then updates the existing `Rating` row or inserts a new one. `get_notifications` and `mark_as_read` handle the notification inbox. This module also imports `Playlist` and `get_playlist_songs` inside `add_to_playlist` rather than at module level.

**`playlist_service.py`** creates playlists (`create_playlist`), returns playlist metadata (`get_playlist`), lists a user's playlists (`get_user_playlists`), and returns a playlist's songs ordered by `playlist_entries.position` (`get_playlist_songs`).

**`feed_service.py`** builds the two feeds. Both start by loading the user's friends. `get_friends_listening_now` looks at friends' listening events from the last 24 hours (`RECENT_THRESHOLD`) and keeps the most recent event for each friend. `get_activity_feed` returns the latest N friend events (default 20) with no time filter. Each feed entry bundles the friend, the song, and the timestamp.

---

## Data Flows

### 1. A user rates a song

1. Client sends `POST /songs/<song_id>/rate` with `user_id` and `score`.
2. `routes/songs.py: rate()` checks that both fields are present and converts `score` to an integer.
3. It calls `notification_service.rate_song(user_id, song_id, score)`.
4. `rate_song` checks that the score is between 1 and 5, loads the `Song` and `User` (raising `ValueError` if either is missing), and queries for an existing `Rating` with the same user and song.
5. If one exists, its `score` is updated. Otherwise a new `Rating` is added to the session.
6. The session is committed and the `Rating` is returned.
7. The route returns `rating.to_dict()` with status 201, or a 400 if a `ValueError` was raised.

### 2. A user listens to a song and their streak updates

1. Client sends `POST /songs/<song_id>/listen` with `user_id`.
2. `routes/songs.py: listen()` calls `streak_service.record_listening_event`.
3. The service loads the user, takes the current UTC time, and adds a `ListeningEvent` to the session.
4. It calls `update_listening_streak(user, now)`, which reads `user.last_listened_at` and updates `listening_streak` and `last_listened_at` in memory.
5. One commit saves the event and the updated user together.
6. The client can read the result from `GET /users/<id>/streak`, which calls `get_streak`.

### 3. A user adds a song to a playlist

1. Client sends `POST /playlists/<playlist_id>/songs` with `song_id` and `added_by`.
2. `routes/playlists.py: add_song()` calls `notification_service.add_to_playlist`.
3. The service loads the song, the adding user, and the playlist, raising `ValueError` for any that are missing.
4. If the song isn't already in `playlist.songs`, it is appended and the session is committed. The row in `playlist_entries` is written by SQLAlchemy through the association table.
5. If the adding user isn't the song's sharer, `create_notification` writes a `song_added_to_playlist` notification for the sharer and commits.
6. The route returns a confirmation message with status 201.

### 4. Reading a playlist

1. Client sends `GET /playlists/<playlist_id>/songs`.
2. `routes/playlists.py: get_songs()` calls `playlist_service.get_playlist_songs`.
3. The service checks that the playlist exists, then joins `Song` to `playlist_entries` on `song_id`, filters by `playlist_id`, and orders by `position`.
4. It returns the songs as dicts, which the route wraps with a `count`.

### 5. Friends Listening Now

1. Client sends `GET /feed/<user_id>/listening-now`.
2. `routes/feed.py` calls `feed_service.get_friends_listening_now`.
3. The service loads the user and their friend IDs. If there are none, it returns an empty list.
4. It queries `ListeningEvent` rows for those friends with `listened_at` within the last 24 hours, newest first.
5. It keeps only the first event per friend, then loads each friend and song to build the response entries.

---

## Patterns in the Code

- **Thin routes, logic in services.** Every route calls one or two service functions and then handles only request parsing and response shape. Business rules, validation of entities, and database writes live in `services/`.
- **Services report errors with `ValueError`, routes map them to HTTP.** The service layer never returns status codes. Routes translate `ValueError` into a 400 for bad input and a 404 for missing resources.
- **Services commit their own transactions.** Each mutating service function ends with `db.session.commit()`, so a single call is one unit of work.
- **Serialization is on the models.** `to_dict()` on each model defines the JSON shape, so routes and services don't build dicts by hand.
- **Service modules are named by domain, but routes cross them.** For example, `routes/songs.py` pulls from three services (search, notifications, streaks), and `notification_service.py` covers ratings and playlist additions as well as notifications. Route modules group endpoints by resource; services group functions by concern.
- **The `db` object is created once in `app.py`.** Every module imports it from there, which keeps one session and one configuration for the app.
- **Local imports break import cycles.** `notification_service.py` imports `Playlist` and `playlist_service` inside `add_to_playlist` rather than at module level.
- **Direct model access is used in some routes.** `routes/users.py: get_user()` loads the `User` model itself rather than going through a service, which is the one place a route queries the database directly.
- **Consistent conventions throughout.** String UUID primary keys, UTC `DateTime` columns, `ValueError` for not-found, and `to_dict()` for output appear in every module.
- **Tests mirror the services.** `tests/test_streaks.py` imports from `streak_service`, and the other two files are named after the search and playlist services. Test files build the app through `create_app({"TESTING": True, ...})` with an in-memory SQLite database.

---

# Bug fixes

## Issue #1 — My listening streak keeps resetting

### How I reproduced it

I ran the following script to set the listening streak to 12 and set last listened to as a Saturday for kenji's ID, then "listen" to a song on a Sunday and then print the streak. Instead of the streak being 13, the streak is reset to 1.

```Python
from datetime import datetime, timezone
from models import User
from services.streak_service import update_listening_streak

kenji = User(username="kenji", listening_streak=12)
kenji.last_listened_at = datetime(2026, 10, 17, 20, tzinfo=timezone.utc)  # Saturday

sunday = datetime(2026, 10, 18, 12, tzinfo=timezone.utc)
update_listening_streak(kenji, sunday)

print(kenji.listening_streak)
```

```
1
```

### How I found the root cause

I started in `services/streak_service.py` and looked at `update_listening_streak()`. Then I checked `tests/test_streaks.py` for the expected behavior when listening on consecutive days. The specific cause was the condition `days_since_last == 1 and today.weekday() != 6`. Sunday has a weekday value of 6, so this condition prevented the streak from increasing on Sundays, even when the last listen was on Saturday.

### The root cause

The code only increased the streak for consecutive days if the current day was not Sunday. When someone listened on Saturday and then Sunday, the condition was false, so the code moved to the `else` branch and reset the streak to 1.

### My fix

I removed `and today.weekday() != 6`, leaving `days_since_last == 1`. This lets the streak increase whenever the last listen was yesterday, including when today is Sunday.

### My side-effect check

I ran all five tests in `tests/test_streaks.py`. They checked that a first listen starts the streak at 1, consecutive days increase it, listening twice on the same day does not increase it again, skipping a day resets it, and listening on Saturday and Sunday increases it. All five tests passed.

### How I reproduced it

## Issue #2 — Friends Listening Now shows people from yesterday

First I created a listening event for Darius, a friend of Nova, at 11:59PM yesterday. Then I looked at Nova's Friends Listening Now feed. Darius is included even though the calendar day is one day ago, so he cannot be listening now.

```Shell
sqlite3 instance/mixtape.db "INSERT INTO listening_event (id, user_id, song_id, listened_at) VALUES ('$(python3 -c 'import uuid; print(uuid.uuid4())')', '<DARIUS_USER_ID>', '<SONG_ID>', datetime('now', 'start of day', '-1 second'));"
curl 'http://127.0.0.1:5000/feed/<NOVA_USER_ID>/listening-now'
```

```JSON
{
  "count": 3,
  "feed": [
    {
      "friend": {
        "id": "693b4bf6-9ba8-451b-9837-174ae12f7454",
        "last_listened_at": "2026-10-13T13:52:01.327206",
        "listening_streak": 3,
        "username": "darius"
      },
      "listened_at": "2026-10-14T13:42:01.327206",
      "song": {
        "album": null,
        "artist": "The Wanderers",
        "genre": "indie rock",
        "id": "b1dd3933-8720-4b30-8e4c-c7d70e437b42",
        "share_note": null,
        "shared_at": "2026-10-09T13:52:01.327206",
        "shared_by": "b4a2e6c6-10e4-4363-b2e6-4d69f07f081a",
        "tags": [],
        "title": "Midnight Drive"
      }
    },
    {
      "friend": {
        "id": "8f8e2131-b591-4b47-93b3-32f3762062ad",
        "last_listened_at": null,
        "listening_streak": 0,
        "username": "simone"
      },
      "listened_at": "2026-10-14T13:37:01.327206",
      "song": {
        "album": null,
        "artist": "Elara Moon",
        "genre": "ambient",
        "id": "f6964b1e-190e-4e67-93d7-b6b360fc709d",
        "share_note": null,
        "shared_at": "2026-10-09T13:52:01.327206",
        "shared_by": "b4a2e6c6-10e4-4363-b2e6-4d69f07f081a",
        "tags": [],
        "title": "Still Waters"
      }
    },
    {
      "friend": {
        "id": "794615a9-3e0d-4b73-93f0-bd675048eebc",
        "last_listened_at": "2026-10-14T10:52:01.327206",
        "listening_streak": 12,
        "username": "kenji"
      },
      "listened_at": "2026-10-14T13:32:01.327206",
      "song": {
        "album": null,
        "artist": "Coastal Highway",
        "genre": "indie",
        "id": "a9b5f9ba-6fed-4bb7-b4df-720c5a12371d",
        "share_note": null,
        "shared_at": "2026-10-09T13:52:01.327206",
        "shared_by": "b4a2e6c6-10e4-4363-b2e6-4d69f07f081a",
        "tags": [],
        "title": "First Light"
      }
    }
  ]
}
```

### How I found the root cause

I started in `routes/feed.py`, where the `/listening-now` route calls `get_friends_listening_now()` in `services/feed_service.py`. I followed that function to the cutoff calculation and the query that filters listening events. The specific cause was `datetime.now(timezone.utc) - RECENT_THRESHOLD`, with `RECENT_THRESHOLD` set to 24 hours. At 9am, this includes events back to 9am yesterday, which explains why Darius’s 11pm listen was still showing.

### The root cause

The code checks whether someone listened within the last 24 hours instead of checking whether they listened today. Yesterday’s events stay in the feed until they are more than 24 hours old, even after a new calendar day starts.

### My fix

I replaced the rolling 24-hour cutoff with midnight at the start of today:

```Python
cutoff = datetime.now(timezone.utc).replace(
    hour=0, minute=0, second=0, microsecond=0
)
```

The existing `ListeningEvent.listened_at >= cutoff` filter would then exclude yesterday’s events and include events from today. This uses UTC to match the app’s current timestamp convention.

### My side-effect check

I checked that listens from yesterday are excluded, today's listens are included, and a listen exactly at midnight is included. Each friend appears only once with their most recent song, non-friends stay excluded, and the separate activity feed still includes older events.

## Issue #3 — The same song keeps showing up twice in search

### How I reproduced it

Search the database for songs with "Anthem" in them. Got back three results of the same song.

```Shell
curl 'http://127.0.0.1:5000/songs/search?q=Anthem'
```

```JSON
{
  "count": 3,
  "results": [
    {
      "album": null,
      "artist": "Borough Kings",
      "genre": "rap",
      "id": "88993ac3-79a3-4caa-a26d-56cc7b82d207",
      "share_note": null,
      "shared_at": "2026-10-07T21:31:22.613050",
      "shared_by": "f4f355fe-8ed1-43ec-b2eb-ea6bcfe2d846",
      "tags": [
        "rap",
        "hip-hop",
        "boom bap"
      ],
      "title": "Crown Heights Anthem"
    },
    {
      "album": null,
      "artist": "Borough Kings",
      "genre": "rap",
      "id": "88993ac3-79a3-4caa-a26d-56cc7b82d207",
      "share_note": null,
      "shared_at": "2026-10-07T21:31:22.613050",
      "shared_by": "f4f355fe-8ed1-43ec-b2eb-ea6bcfe2d846",
      "tags": [
        "rap",
        "hip-hop",
        "boom bap"
      ],
      "title": "Crown Heights Anthem"
    },
    {
      "album": null,
      "artist": "Borough Kings",
      "genre": "rap",
      "id": "88993ac3-79a3-4caa-a26d-56cc7b82d207",
      "share_note": null,
      "shared_at": "2026-10-07T21:31:22.613050",
      "shared_by": "f4f355fe-8ed1-43ec-b2eb-ea6bcfe2d846",
      "tags": [
        "rap",
        "hip-hop",
        "boom bap"
      ],
      "title": "Crown Heights Anthem"
    }
  ]
}
```

### How I found the root cause

I started in `routes/songs.py`, where the `/search` route calls `search_songs()` in `services/search_service.py`. I followed the query and saw that it joins `Song` with `song_tags`. Crown Heights Anthem has three tags, so the join returns three rows for the same song. The specific cause was that `.scalars().all()` collects all three rows without removing duplicate songs.

### The root cause

The search query returns one row for each tag attached to a matching song. Because the results are not deduplicated, a song with three tags appears three times. These are copies of the same song in the response, rather than separate songs in the database.

### My fix

I added `.unique()` before `.all()`:

```Python
.scalars()
.unique()
.all()
```

This removes repeated instances of the same song from the query results. Each song would appear once while still including all its tags.

### My side-effect check

After making the fix, I searched for `Anthem` and confirm that Crown Heights Anthem appears once with all three tags. I also checked that title and artist searches still work, searches are still case-insensitive, songs without tags still appear, and a search with no matches returns an empty list.

## Issue #4 — I got notified when a friend added my song to a playlist but not when they rated it

### How I reproduced it

I added a song as Kenji, checked Aaliya’s notifications (notification was there), then rated one of her shared songs as Kenji and checked her notifications again. The rating was saved, but no new rating notification appeared.

**NOTE**: To successfully share a song and have Kenji add it to a playlist, I had to first fix a bug that did not allow songs to be added to playlists. I inserted the association row with the required `position` and `added_by` fields instead of only calling `playlist.songs.append(song)`.

```Shell
curl -X POST 'http://127.0.0.1:5000/playlists/e5e4c6d1-e0d3-41e3-a51a-3f1ae015f5cb/songs' \
  -H 'Content-Type: application/json' \
  -d "{\"song_id\":\"b499644b-2f2e-41fc-bffe-54636c158c57\",\"added_by\":\<KENJI_ID>"}"

curl "http://127.0.0.1:5000/users/<AALIYA_ID>/notifications"

curl -X POST 'http://127.0.0.1:5000/songs/<AALIYA_SHARED_SONG_ID>/rate' \
  -H 'Content-Type: application/json' \
  -d '{"user_id":"<KENJI_USER_ID>","score":5}'

curl 'http://127.0.0.1:5000/users/<AALIYA_USER_ID>/notifications'
```

```JSON
{
  "count": 1,
  "notifications": [
    {
      "body": "kenji added your song 'Aaliya Notification Test' to the playlist 'Study Mode'.",
      "created_at": "2026-10-14T15:44:20.260940",
      "id": "c498d19a-56f4-46f7-81a5-7ba46e5c40c1",
      "read": false,
      "type": "song_added_to_playlist",
      "user_id": "2ba7a408-7b8d-4a9e-8690-490fa5551ceb"
    }
  ]
}
```

### How I found the root cause

I started in `routes/songs.py`, where the rating route calls `rate_song()` in `services/notification_service.py`. I followed that function through the validation, saving the rating, and returning the result. Then I compared it with `add_to_playlist()` in the same file. The specific difference was that `add_to_playlist()` calls `create_notification()`, but `rate_song()` never does.

### The root cause

The rating function saves the rating and returns it without creating a notification for the person who shared the song. The notification is missing because no notification record is created, rather than because of a delay or a problem displaying it.

### My fix

I added a call to `create_notification()` after the rating is saved and before `return rating`. I also checked that the person rating the song is not the person who shared it, matching the existing playlist notification behavior.

```Python
if song.shared_by != user_id:
    create_notification(
        user_id=song.shared_by,
        notification_type="song_rated",
        body=f"{rater.username} rated your song '{song.title}' {score} stars.",
    )
```

This sends the notification to the song’s sharer when someone else rates their song. Placing it after the save handles both new ratings and updates to existing ratings.

### My side-effect check

I checked that Kenji rating Aaliya’s song saved the rating and creates a `song_rated` notification for Aaliya. I also check that updating a rating still works, rating your own shared song does not notify yourself, invalid scores do not create ratings or notifications, and adding someone else’s song to a playlist still creates its existing notification.

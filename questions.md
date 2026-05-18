# GameHub — Understanding the system

10 questions to test your understanding of the data flow and architecture.
Work through them in order: read the code first, then run the app, then try to break things.

---

## How to investigate

You will need three things:

**1. Read the source code**
Start with `models.py` (the schema), then `seed.py` (the data), then `app.py` (the logic).
Many questions are answered entirely by reading carefully.

**2. Run the app and interact with it**
Use the UI at `http://localhost:5000` or send requests with curl or Postman.
Observe what actually happens — don't just reason about it.

```bash
# Example: log an activity for nova (id=1) on Hollow Knight (id=1)
curl -X POST http://localhost:5000/activities \
  -H "Content-Type: application/json" \
  -d '{"user_id": 1, "game_id": 1, "action": "started"}'
```

**3. Query the database directly**
Open `gamehub.db` with a SQLite tool and inspect the actual rows.

```bash
sqlite3 gamehub.db
.tables
SELECT COUNT(*) FROM notifications;
SELECT * FROM notifications WHERE user_id = 1;
```

Or use a GUI: **DB Browser for SQLite** (free, recommended).

---

## Suggested approach

| Phase | Questions | What you are doing |
|-------|-----------|-------------------|
| Read first | 1, 4, 8, 10 | Understand the code before touching anything |
| Then run it | 3, 6, 9    | Observe actual behaviour                     |
| Then break it | 2, 5, 7  | Try things, hit walls, reason about why      |

---

## Questions

**1.** When a user logs a new activity, how many database tables are written to?
List them and explain why each one is affected.

2 database (activities and notifications):
- One row is inserted into activities table (the user's action on the game). For each friend the user has, a row is inserted into notifications table (friends are notified)

# Insert activity
"INSERT INTO activities (user_id, game_id, action, created_at) VALUES (?,?,?,?)"


# Then for each friend:
"INSERT INTO notifications (user_id, triggered_by, message, activity_id, created_at) VALUES (?,?,?,?,?)"

---

**2.** You call `DELETE FROM users WHERE id = 3` directly in SQLite.
What happens, and why? What would you need to do instead?

The DELETE will FAIL with a foreign key constraint violation. because Foreign keys is  enabled (
# Foreign keys must be enabled per connection in SQLite
conn.execute("PRAGMA foreign_keys = ON")
)

a possible solution will be:
- Delete notifications where user is receiver (user_id = 3)
- Delete notifications where user triggered them (triggered_by = 3)
- Delete activities (user_id = 3)
- Delete user_games (user_id = 3)
- Delete friendships (both directions: user_id = 3 OR friend_id = 3)
- Finally delete the user

---

**3.** User `nova` changes her username to `nova_2`.
She then checks her friends' notification feeds.
What do they see — the old name or the new one? Why?

hey see the OLD name (nova) because in app.py, notifications are created with the username stored as plain text in the message field

msg = f"{actor['username']} just {data['action']} playing {game['title']}!"

Once a notification is created, its message is static. Changing the users.username doesn't update existing notification messages. It still reference "nova" not "nova_2"
---

**4.** Trace the full journey of a `POST /activities` request.
Starting from the HTTP call, list every operation that happens before the response is returned.

The request flow in (app.py):

- Receive POST data with user_id, game_id, action
- Get database connection with foreign keys enabled
- INSERT into activities — creates new activity row with current timestamp
- SELECT back the activity to get its ID
- SELECT the actor (user) to get their username
- SELECT the game to get its title
- SELECT all friends of the actor (from friends table)
- For each friend: (Create a notification message with actor's username, action, and game title, INSERT into notifications table)
- COMMIT all changes
- JSON will return 201 status
---

**5.** `pixel_queen` opts out of activity tracking.
A teammate adds an `opted_out` boolean column to the `users` table and updates the `POST /activities` API route to check it.
Is the feature fully implemented? What did they miss?

It's NOT fully implemented:
- Notification generation still happens even if they don't log activities themselves, notifications about them might still be generated if they're the recipient (triggered by other users' activities)
- Historical activities remain visible old activities for pixel_queen are still in the database and visible in queries
- user_games records still there — past game ownership is unchanged
- Friends still see pixel_queen — friendship relationships are unchanged

What was missed: A complete opt-out option to delete or mark old activities as hidden


---

**6.** How many rows are created in the database when `nova` logs one activity, given the current seed data?
Show your working.

1 row in (activities) and 3 in nova friend list (notifications to each friend) = 4 rows

---

**7.** You need to delete `maya_r`.
In what order must you delete rows across the tables, and why does the order matter?

Deletion order :

- DELETE notifications WHERE user_id = maya_r (she's the receiver)
- DELETE notifications WHERE triggered_by = maya_r (she triggered them)
- DELETE activities WHERE user_id = maya_r (now safe, notifications are gone)
- DELETE user_games WHERE user_id = maya_r
- DELETE friends WHERE user_id = maya_r OR friend_id = maya_r
- DELETE users WHERE id = maya_r 

Why the order matters:(Foreign key constraints):

- Can't delete activities while notifications reference them
- Can't delete users while activities reference them
- Can't delete users while they're referenced in friends/notifications
---

**8.** The `notifications` table has a foreign key pointing to `activities`.
What happens if you try to delete an activity that has notifications attached to it?

- The DELETE will FAIL with a foreign key constraint violation due to (activity_id INTEGER NOT NULL REFERENCES activities(id)) on models.py
- Since foreign keys are ON, SQLite enforces referential integrity. You must first delete all notifications that reference the activity before deleting the activity itself.

---

**9.** A bug is found in the game catalog — wrong genre for one game.
You fix it and restart the app to ship the change.
What else just went down, and for how long?

Everything went down — the entire application is unavailable. This occur because (When you restart to fix a game's genre, you're restarting the entire server. There's no separation of concerns)

---

**10.** A teammate says: *"let's just move the notification logic into its own function in `app.py`"*.
Does that solve the problem described in Task 4?
What is the actual architectural issue?

NO, extracting the function solves nothing. The problem is architectural, not organizational.

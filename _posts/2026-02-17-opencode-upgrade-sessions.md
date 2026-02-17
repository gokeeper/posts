---
layout: post
title: "Hello, world"
---

# Where Did My OpenCode Sessions Go? Debugging a Silent SQLite Migration

Upgrading your dev tools is usually a straightforward process—until you launch the new version and realize your entire project history has vanished. 

Recently, after running an update for OpenCode on my Arch desktop, I opened my workspace only to find a completely blank session history. No recent chats, no context, just an empty sidebar. If you rely heavily on your AI coding assistant's memory, you know how frustrating this can be.

Here is the step-by-step breakdown of how I tracked down the missing data and mapped it back to the correct Git repositories using a little CLI detective work.

### The Red Herrings: XDG Base Directories and JSON files
My first instinct was that OpenCode finally adopted the XDG Base Directory specification. Sure enough, looking in `~/.config/opencode/` showed that my state data was no longer being updated there. 

I dug into the new data path at `~/.local/share/opencode/storage/session/global/` and found my old sessions sitting safely as `.json` files. 

Since recent versions of OpenCode scope your history to the specific Git repository you launch it from (using the root commit hash), I assumed the upgrade just failed to move these "global" files into a project-specific folder. 

I wrote a quick script to grab my Git root hash:
```bash
id=$(git rev-list --max-parents=0 HEAD | head -n 1)
```
I created a directory for that hash, moved the `.json` files into it, and even used `sed` to update the internal `"projectID"` metadata inside the JSON files. 

I relaunched OpenCode. Still nothing. The UI completely ignored the files.

### The Breakthrough: The "Tracer" Method
When software silently ignores configuration files, it's time to stop guessing paths and start watching file system events. I opened OpenCode, typed a quick "hello" to generate a brand new session, and immediately ran a `find` command to catch what files were modified in the last two minutes:

```bash
find ~ -type f -mmin -2 -not -path "*/.*cache*"
```

The output revealed the smoking gun:
```text
/home/user/.local/share/opencode/opencode.db-wal
/home/user/.local/share/opencode/opencode.db-shm
/home/user/.local/share/opencode/opencode.db
```

OpenCode hadn't just moved directories; it had completely abandoned flat JSON files in favor of a relational SQLite database. Moving my old `.json` files around did nothing because the application was no longer reading them to build the UI list.

### Digging into the Database Schema
I dumped the schema to see what I was dealing with:
```bash
sqlite3 ~/.local/share/opencode/opencode.db ".schema"
```
The schema showed a well-structured setup managed by Drizzle ORM, with foreign keys linking `project`, `session`, `message`, and `part` tables. 

I queried the `session` table directly:
```bash
sqlite3 ~/.local/share/opencode/opencode.db "SELECT * FROM session;"
```

**The Aha Moment:** The sessions *were* actually inside the database! The migration script hadn't failed to import them. However, it had imported my older sessions with a `project_id` of `'global'`. Because the OpenCode UI strictly filters the view based on the current Git repository's hash, it was intentionally hiding all of those 'global' rows whenever I launched the tool inside my project directory.

### The One-Line Fix
Since the data was already safe and sound in the database, I just needed to remap the `project_id` for those specific sessions from `'global'` to my repository's Git hash. 

```bash
# Ensure the project exists in the DB to prevent foreign key errors
sqlite3 ~/.local/share/opencode/opencode.db "INSERT OR IGNORE INTO project (id, worktree, time_created, time_updated, sandboxes) VALUES ('$id', '/home/user/codes/my-project', 0, 0, '[]');"

# Update the session's project scope
sqlite3 ~/.local/share/opencode/opencode.db "UPDATE session SET project_id = '$id' WHERE id = 'ses_my_old_session_id';"
```

Upon reloading OpenCode, the missing sessions instantly populated in the sidebar, with all chat history and file context perfectly intact.

### Bonus: Cleaning up "Ghost" Sessions
One nice side effect of this new SQLite architecture is the use of `ON DELETE CASCADE` constraints. If you have old sessions you want to nuke, you don't have to hunt down orphaned message or attachment files anymore. Deleting a session from the database cleanly wipes all associated data:

```bash
sqlite3 ~/.local/share/opencode/opencode.db "DELETE FROM session WHERE id = 'ses_unwanted_session_id';"
```

If you recently upgraded OpenCode and thought you lost your mind (or your code context), stop digging through JSON files and check your `opencode.db`. A quick SQL update might be all you need.

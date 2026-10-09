# KB · Restore an earlier version of a file

For staff and the Kestrel IT Service Desk. Tested in F3 with Priya's pay run file.

Works for files on the shared drives (F:, K:, O:, R:). It doesn't work for files saved on the PC itself, like the Desktop or Documents.

## When to use it

- Someone saved over a file by mistake.
- Someone deleted a file or folder.

## How far back can you go?

Snapshots are taken at **7am and 12pm on weekdays** (only while the server is running), and older ones are removed when they reach their space limit. You can restore the file as it was at one of those times, not any time in between. Anything saved after the last snapshot can't be brought back this way.

## Saved over a file

1. In File Explorer, right-click the file → **Properties** → **Previous Versions**.
2. Pick the version from before the mistake.
3. Click **Open** first and check it's the right one.
4. To keep both, use the arrow next to **Restore** → **Restore to…** and save it somewhere else. Plain **Restore** replaces the current file.

## Deleted a file

The file isn't there to right-click, so go one level up:

1. Right-click the **folder** it was in → **Properties** → **Previous Versions**.
2. **Open** the version from before it was deleted.
3. Copy the file back into the folder.

## Things to know

- Users can restore their own files. Ask the service desk only if no versions show up.
- **Never use Revert on the server.** It rolls back the entire `E:` volume, every department's files, not just the one you need.
- This isn't a backup. If the server's disk is lost, the old versions go with it.

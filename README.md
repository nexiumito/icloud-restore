# iCloud Restore

> **This is a patched fork of [odysseus0/icloud-restore](https://github.com/odysseus0/icloud-restore).**
> Upstream currently fails on the first API call for most accounts: it hardcodes one
> Apple server pool (`p107`) and requests 2000 items per page, which returns HTTP 421
> and HTTP 500 respectively. This fork fixes that plus lossy pagination and concurrent
> credential refresh. Fixes submitted upstream as
> [PR #5](https://github.com/odysseus0/icloud-restore/pull/5) /
> [issue #4](https://github.com/odysseus0/icloud-restore/issues/4);
> use this fork until they land.
>
> Verified on a real account: **39,165 files restored, 0 failures.**

**Restore deleted files from iCloud Drive when the web UI fails.**

```bash
uvx icloud-restore
```

> *iCloud.com's "Recently Deleted" page freezes, crashes, or shows a spinning wheel forever? This tool fixes that.*

## The Problem

iCloud.com's "Restore Files" page crashes or hangs when you have a large number of deleted files (10k+). Symptoms:

- Page shows spinning wheel forever
- Browser tab crashes or becomes unresponsive
- "Restore All" button stays greyed out (waiting for all files to load)
- Page loads partially then freezes
- Safari/Chrome runs out of memory

This happens because the web UI tries to render all deleted files at once before letting you restore them. Apple hasn't fixed this bug for years.

## The Solution

This tool bypasses the broken web UI by using Apple's API directly. It:

1. Opens your Chrome browser to iCloud's recovery page
2. You log in normally (with Keychain autofill, 2FA, etc.)
3. Fetches the list of deleted files via API
4. Restores them in batches
5. Auto-refreshes credentials when they expire (long restores can take hours)

## Installation

> **Note:** this package is not on PyPI, so `uvx icloud-restore` and
> `pipx run icloud-restore` fail with "no available version"
> ([upstream issue #2](https://github.com/odysseus0/icloud-restore/issues/2)).
> Install straight from this fork:

```bash
# Using uv (recommended)
uvx --from git+https://github.com/nexiumito/icloud-restore icloud-restore

# Using pipx
pipx run --spec git+https://github.com/nexiumito/icloud-restore icloud-restore

# Or install globally
pip install git+https://github.com/nexiumito/icloud-restore
```

Requires Python 3.10+ and Google Chrome.

## Usage

Just run the command:

```bash
icloud-restore
```

Your Chrome browser will open to iCloud's recovery page. Log in with your Apple ID (Keychain autofill works!), then the tool will:

1. Detect your login
2. Fetch the list of deleted files
3. Ask for confirmation
4. Restore all files

### Progress & Resume

The tool saves progress to local files. If interrupted (Ctrl+C, crash, etc.), just run it again to resume where you left off.

Progress files:
- `icloud_restore_checkpoint.json` - Tracks file list fetching
- `icloud_restore_progress.json` - Tracks restore progress

### Long Restores

For large restores (100k+ files), the process can take several hours. The tool will:

- Keep the browser open throughout
- Auto-refresh credentials when they expire (~every hour)
- Save progress periodically

You can leave it running unattended.

## How It Works

1. **Browser Login**: Launches Chrome with a fresh profile (macOS Keychain still works for autofill)
2. **Credential Capture**: Watches network requests to capture your session credentials
3. **API Calls**: Uses the same API endpoints that the web UI would use
4. **Batch Processing**: Restores files in batches with retries and rate limiting
5. **Auto-Refresh**: When credentials expire, reloads the browser page to get fresh ones

## Security

- **Local only**: Your credentials never leave your machine
- **No storage**: Cookies are not saved to disk (only held in memory during the session)
- **Keychain works**: Fresh Chrome profile still has access to macOS Keychain for password autofill
- **Open source**: Review the code yourself

## Requirements

- Python 3.10+
- Google Chrome installed

## Troubleshooting

### "Login not detected"

Make sure you complete the full login flow including any 2FA prompts. Wait a few seconds after logging in for the tool to detect it.

### "Auth expired" errors

The tool should handle this automatically by refreshing credentials. If it keeps failing, try closing the tool and running it again.

### Some files failed to restore

This can happen if Apple's servers are overloaded. Run the tool again - it will only retry the failed files.

### One run is not enough - expect to run it several times

Apple's tombstone pagination is *lossy*: a full pass does not necessarily return
every deleted file. On a ~39k file account, a second pass surfaced 1,187 ids the
first pass never returned. It took five passes to converge:

| pass | genuinely new files restored |
|---|---|
| 1 | 37,740 |
| 2 | 1,187 |
| 3 | 32 |
| 4 | 3 |
| 5 | 0 |

Keep re-running until a pass reports no new files. Progress is cumulative, so
later passes are quick - they skip everything already restored.

### iCloud still lists the files as deleted after a successful restore

This is expected and is **not** a failure. Apple's "Recently Deleted" list stays
stale for a long time after a restore: it kept reporting ~1,598 files as deleted
on an account where every one of them was verified present on disk at its
original path.

Trust the local filesystem, not the recovery page's counter. Note that restored
files can sit very deep - build caches like sbt's `target/streams/.../out` were
16 directory levels down, so a shallow `find -maxdepth` will miss them and look
like data loss when there is none.

### Restored files are not on my Mac yet

Restores put files back into iCloud Drive, not onto your disk directly. macOS then
syncs them down lazily, which can take hours for large restores. Make sure the Mac
stays awake (`caffeinate -dims`) and has enough free space - dependency folders
like `node_modules` come back too.

## License

MIT

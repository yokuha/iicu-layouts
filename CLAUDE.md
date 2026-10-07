# iicu-layouts

Python CLI to back up, restore and copy Intervals.icu page layouts (UI settings)
between device classes desktop, tablet, phone. License: GPL-3.0-or-later.

## Verified API findings (browser inspection, session-cookie auth)
- Read: GET https://intervals.icu/api/athlete?deviceClass=<desktop|tablet|phone> returns
  a large payload. We need only its `settings` object (~74 keys on desktop). The payload
  also contains email and subscription data. NEVER store anything except `settings`.
- Write: PUT https://intervals.icu/api/settings/<deviceClass> with JSON body
  {"<settingKey>": <full value>}. Only the sent keys are updated (per-key merge); each
  value replaces the old one as a whole. GET on that URL returns 405.
- Unknown deviceClass values silently return the DESKTOP settings. Accept only
  desktop|tablet|phone and never pass anything else through.
- Fitness page layout: settings["FitnessView.options"] with days, futureDays, activeTab,
  recentDateRanges, tabs[] (id, label, show* flags, showCustom = custom chart IDs).
  Custom chart IDs are account-level, so layouts copied between classes should resolve.
- Last writer wins. A stale open browser window rewrites its old FitnessView.options on
  load or interaction and can silently undo our write.
- Tablet and phone have their own, smaller key sets. Some keys exist only on desktop.
- These are undocumented internal endpoints and may change without notice.

## Credentials
- Source: Python `keyring` only. Service "iicu-layouts", entries "api_key" and
  "athlete_id" (already stored by the user; never ask for or handle the values).
  Read them only via keyring.get_password() inside one loader function.
- If the keyring backend is unavailable or returns nothing, stop with a clear error.
  Never fall back to plaintext files, env vars, other credential files or a file-based
  keyring backend. Do not add keyrings.alt. No alternative credential sources.
- Never print, log, or read credentials outside the loader. Do not query the keychain
  via shell commands.
- Auth is HTTP Basic: username "API_KEY", password = the key.
- Tests mock the loader. No real credentials in tests, fixtures or logs.

## Write safety
- Write commands are dry-run by default and write only with --apply.
- Snapshot the target class before every write. Re-read and verify after every write,
  re-check after ~60 s (--watch for longer) and report loudly if a stale client reverted it.
- Default --keys is FitnessView.options only. Other keys need an explicit --keys list;
  ComparePage.options etc. get a warning.
- Before writing, tell the user to close or reload other Intervals.icu windows on that class.
- The first live write test targets `phone`, never desktop.

## Engineering
- Python 3.11+, requests (or httpx), stdlib argparse, keyring, pytest. Minimal dependencies.
- .gitignore first: .env, __pycache__, venv, snapshots/ (tab names and chart IDs are
  personal). Ask before committing any snapshot. Small commits; say what each does.
- Unit tests use sanitized fixtures (fake chart IDs and labels), no live calls.

## Documentation requirements (README must contain)
- Purpose, the API findings above, and the caveat about undocumented endpoints.
- Credentials setup, step by step:
  `python -m keyring set iicu-layouts api_key` and
  `python -m keyring set iicu-layouts athlete_id` (values are prompted, nothing lands
  in shell history); the service/entry names; how to verify the entries exist without
  printing them; how to rotate or delete them.
- Platform notes: macOS Keychain shows a permission prompt on first read, "Always Allow"
  applies per Python binary, so a new venv may prompt again, and it needs a GUI session
  (not over SSH). Linux needs a running Secret Service (GNOME Keyring/KWallet); the tool
  fails loudly otherwise. Windows uses Credential Locker.
- Usage examples for every command, the dry-run/--apply model, and the stale-window warning.
- A recommended-model note is fine (Sonnet 5.5, default effort), as documentation only.

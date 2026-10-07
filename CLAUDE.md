Project: iicu-layouts.

A small Python CLI to back up, restore and copy Intervals.icu page layouts (UI settings) between
device classes: desktop, tablet, phone.

## Verified findings (from browser inspection, session-cookie auth)
- Read: GET https://intervals.icu/api/athlete?deviceClass=<desktop|tablet|phone>
  returns a big payload. The part we need is the `settings` object (~74 keys on desktop).
  The payload also holds email and subscription data. NEVER store anything except `settings`.
- Write: PUT https://intervals.icu/api/settings/<deviceClass> with a JSON body
  {"<settingKey>": <full value>}. Only the sent keys are updated (per-key merge), and each
  value replaces the old one as a whole. GET on that URL returns 405.
- Unknown deviceClass values silently return the DESKTOP settings. The CLI must therefore
  accept only desktop|tablet|phone and never pass anything else through.
- Fitness page layout is settings["FitnessView.options"]: days, futureDays, activeTab,
  recentDateRanges, tabs[] (id, label, show* flags, showCustom = custom chart IDs).
  Custom chart IDs are account-level, so a layout copied between classes should resolve.
- Last writer wins. A stale open browser window rewrites its old FitnessView.options
  on load or interaction and can silently undo our write.
- Tablet and phone have their own, smaller key sets. Some keys exist only on desktop.

## Step 0 (do this first, read-only, then stop and report to me)
Check whether HTTP Basic auth with the Intervals.icu API key (username "API_KEY",
password = key from env var ICU_API_KEY) is accepted by GET /api/athlete?deviceClass=tablet.
Report status codes and the top-level key names of the response only. Do not print values.
If the API key is rejected, do NOT build workarounds. Report back and we decide together
how to authenticate (e.g. a session cookie that I supply via env var, never committed).

## CLI (after step 0 passes), package `iicu_layouts`, command `iicu-layouts`
- dump [--classes desktop,tablet,phone]: write snapshots/<class>/<UTC timestamp>.json
  containing only `settings`. Also keep snapshots/<class>/latest.json.
- list: show snapshots and, per class, the tab labels of FitnessView.options.
- diff <a> <b>: compare two snapshots or live classes, per key (changed / only-in-one).
- restore <snapshot-file> --to <class> [--keys FitnessView.options]
- copy <from-class> <to-class> [--keys FitnessView.options] [--reset-active-tab]
- Write commands are dry-run by default and show what would change. They write only with --apply.
- Default --keys is FitnessView.options only. Writing other keys needs an explicit
  --keys list, and ComparePage.options etc. get a warning.
- Before every write: snapshot the target class first.
- After every write: re-read the server and verify the values. Then re-check after
  ~60 s (--watch for longer) and report loudly if a stale client reverted the change.
- Tell the user before writing: close or reload other Intervals.icu windows on that class.
- The first live write test must target `phone`, never desktop.

## Engineering constraints
- Python 3.11+, requests (or httpx) and stdlib argparse, minimal dependencies. pytest.
- Config via env vars (ICU_API_KEY, optionally ICU_BASE_URL); no secrets in the repo.
- Create .gitignore first (config, .env, __pycache__, venv, snapshots/ is ignored by default
  because tab names and chart IDs are personal). Ask me before committing any snapshot.
- Unit tests use sanitized fixtures (fake chart IDs and labels), no live calls.
- README: purpose, the findings above, auth, usage examples, a clear caveat that these are
  undocumented internal endpoints that may change, and the stale-window warning.
- Keep commits small and tell me what each does.

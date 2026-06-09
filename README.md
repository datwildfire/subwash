<p align="center">
  <img src="https://i.postimg.cc/QNKxWk6g/535be6fd-e324-4381-b792-c701926fc4d5.png" alt="Subwash banner" width="100%">
</p>

A single-file SRT ad/credit cleaner. No dependencies, Python 3.10+.

Combines the rule sets from [KBlixt/subcleaner](https://github.com/KBlixt/subcleaner) (purge/warning regex profiles) and [rogs/subscleaner](https://gitlab.com/rogs/subscleaner) (`AD_PATTERNS`) into one standalone script, with a removal policy hopefuly tuned to not delete dialogue on a live library.

## Why
A missed ad is cosmetic. A deleted line of real dialogue is permanent and you find out months later mid-watch. So the policy is asymmetric: aggressive on unambiguous ad tokens, conservative on anything that merely looks subtitle-ish.

- **Hard ad tokens** (OpenSubtitles, YTS, addic7ed, "Support us and become a VIP member", ...) remove a block anywhere in the file.
- **Warning vocabulary alone** ("subtitles", "edited by", "synced", ...) removes a block only when it is at a file edge **and** short (credit-shaped). A line of prose dialogue containing "edited by" is never deleted, even if earlier removals shift it to position 1.
- **Idempotent.** Re-running on an already-cleaned file removes nothing. Safe for Bazarr re-runs, hook double-fires, and library rescans.

## Install

```bash
curl -O https://example/subclean_combined.py
chmod +x subclean_combined.py
python3 subclean_combined.py --update-rules --print-rule-counts
```

`--update-rules` fetches the latest upstream rules into `~/.cache/subclean-combined/rules.json`. Without a cache it falls back to a small built-in token list. (rogs is on GitLab; if that host is unreachable the KBlixt rules still load and still cover YTS/YIFY/OpenSubtitles as bare tokens.)

## Usage

Default is dry-run. Nothing is written until you pass `--write` or `--stage`.

```bash
# preview
python3 subclean_combined.py movie.en.srt

# safe first pass over a library: writes movie.subclean.en.srt next to each
# original, never touches originals, logs every removal
python3 subclean_combined.py --stage --update-rules /library

# steady-state hook: overwrite in place, pristine .bak, audit log
python3 subclean_combined.py --write /downloads/movie.en.srt

# from find / qBittorrent
find /library -iname '*.srt' -print0 | python3 subclean_combined.py --write --stdin0
```

Promote staged candidates after review:

```bash
for f in /library/**/*.subclean.*.srt; do mv "$f" "${f/.subclean/}"; done
```

## Safety

Three layers, all scoped to files actually modified:

- **Pristine backup.** `--write` copies the original to `.srt.bak` before overwriting. The first `.bak` is never clobbered; subsequent runs timestamp instead, so a re-run can't replace your real original with a cleaned copy.
- **Audit log.** Every removed block is appended as JSONL (timestamp, mode, timecode, reason, full text) to `~/.cache/subclean-combined/removals.jsonl`. Disable with `--no-log`.
- **Empty-file guard.** If every block would be removed, the file is left untouched unless you pass `--allow-empty`.

## Flags

| Flag | Effect |
|------|--------|
| `--write` | Overwrite in place (with `.bak` unless `--no-backup`). |
| `--stage` | Write `<name>.subclean<.lang>.srt` candidate, never touch original. |
| `--out-suffix` | Suffix for `--stage`, default `.subclean`. |
| `--no-backup` | Skip the `.bak` on `--write`. |
| `--log` / `--no-log` | Audit log path / disable. |
| `--allow-empty` | Permit writing a fully-emptied file. |
| `--remove-score` | Score threshold to remove, default 3 (token match = 3, warning match = 1). |
| `--update-rules` | Refresh the rule cache from upstream. |
| `--print-rule-counts` | Show loaded rule counts and exit. |
| `--stdin` / `--stdin0` | Read paths from stdin (newline / NUL-separated). |
| `--no-recursive` | Do not descend into directories. |
| `-v` | Verbose (per-file clean/backup lines). |

## Known limitations

- **Mid-film multi-block ad clusters are only partially cleaned.** The block carrying the hard token is removed; a token-less neighbour ("Subtitles by") mid-film is kept. This is the deliberate cost of protecting dialogue. Edge clusters are fully cleaned.
- **All language profiles run on every file** (no language detection). Low false-positive risk because the merged rules are dominated by specific brand tokens, but it does inflate warning vocabulary, which is exactly why warnings alone can't delete a block.
- Brands with no hard token in upstream (e.g. some VPN sponsors) survive mid-film. Add them as purge tokens if you want them caught everywhere.

## Credits

Rules from KBlixt/subcleaner and rogs/subscleaner. This script reuses their pattern sets under a different removal engine.

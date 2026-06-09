<p align="center">
  <img src="https://i.postimg.cc/QNKxWk6g/535be6fd-e324-4381-b792-c701926fc4d5.png" alt="Subwash banner" width="100%">
</p>

<p align="center">
  <img src="https://i.postimg.cc/QNKxWk6g/535be6fd-e324-4381-b792-c701926fc4d5.png" alt="Subwash banner" width="100%">
</p>

# Subwash

A single-file SRT ad and credit cleaner.

No dependencies. Python 3.10+. Designed to remove subtitle junk without eating the actual movie.

Subwash combines rule sets from:

* [KBlixt/subcleaner](https://github.com/KBlixt/subcleaner), purge and warning regex profiles
* [rogs/subscleaner](https://gitlab.com/rogs/subscleaner), `AD_PATTERNS`

Same trash pile, different washing machine.

## Why

A missed ad is annoying.

A deleted line of dialogue is worse, because you usually discover it months later, mid-watch, wondering why everyone suddenly became mysterious.

So Subwash is deliberately uneven:

* **Hard ad tokens** like `OpenSubtitles`, `YTS`, `YIFY`, `addic7ed`, and `Support us and become a VIP member` are removed anywhere.
* **Warning-ish lines** like `subtitles`, `edited by`, or `synced` are only removed when they are short and live at the start or end of the file.
* **Dialogue is protected.** A normal line containing `edited by` should not vanish just because it looks vaguely subtitle-adjacent.
* **Re-running is safe.** Cleaned once should mean cleaned, not slowly pressure-washed into dust.

## Install

```bash
curl -O https://example/subclean_combined.py
chmod +x subclean_combined.py
python3 subclean_combined.py --update-rules --print-rule-counts
```

`--update-rules` fetches the latest upstream rules into:

```text
~/.cache/subclean-combined/rules.json
```

Without a cached rule file, Subwash falls back to a small built-in token list.

`rogs/subscleaner` lives on GitLab. If GitLab is unreachable, the [KBlixt/subcleaner](https://github.com/KBlixt/subcleaner) rules still load and still cover common junk like YTS, YIFY, and OpenSubtitles.

## Usage

Dry-run is the default. Nothing is written unless you use `--write` or `--stage`.

```bash
# Preview only
python3 subclean_combined.py movie.en.srt

# Safer first pass over a library
# Writes movie.subclean.en.srt next to the original
python3 subclean_combined.py --stage --update-rules /library

# Steady-state hook
# Overwrites in place, keeps a pristine .bak, logs removals
python3 subclean_combined.py --write /downloads/movie.en.srt

# From find, qBittorrent, scripts, cron goblins, etc.
find /library -iname '*.srt' -print0 | python3 subclean_combined.py --write --stdin0
```

Promote staged candidates after review:

```bash
shopt -s globstar nullglob

for f in /library/**/*.subclean.*.srt; do
  mv "$f" "${f/.subclean/}"
done
```

## Safety

Subwash is aggressive toward ad sludge, not toward your files.

* **Pristine backup:** `--write` copies the original to `.srt.bak` before overwriting. The first `.bak` is never clobbered. Later runs get timestamped backups instead.
* **Audit log:** every removed block is appended as JSONL with timestamp, mode, timecode, reason, and full text.
* **Empty-file guard:** if every block would be removed, the file is left untouched unless `--allow-empty` is passed.

Default audit log:

```text
~/.cache/subclean-combined/removals.jsonl
```

Disable logging with:

```bash
python3 subclean_combined.py --write --no-log movie.en.srt
```

## Flags

| Flag                  | Effect                                                                      |
| --------------------- | --------------------------------------------------------------------------- |
| `--write`             | Overwrite files in place. Creates `.bak` unless disabled.                   |
| `--stage`             | Write a cleaned candidate next to the original. Never touches the original. |
| `--out-suffix`        | Change the staged suffix. Default: `.subclean`.                             |
| `--no-backup`         | Skip `.bak` creation when using `--write`.                                  |
| `--log` / `--no-log`  | Set or disable the audit log.                                               |
| `--allow-empty`       | Allow writing a fully emptied subtitle file. Dangerous, but available.      |
| `--remove-score`      | Score threshold for removal. Default: `3`.                                  |
| `--update-rules`      | Refresh the upstream rule cache.                                            |
| `--print-rule-counts` | Show loaded rule counts and exit.                                           |
| `--stdin`             | Read newline-separated paths from stdin.                                    |
| `--stdin0`            | Read NUL-separated paths from stdin.                                        |
| `--no-recursive`      | Do not descend into directories.                                            |
| `-v`                  | Verbose output.                                                             |

## Removal policy

Subwash uses an asymmetric policy:

* Hard ad match = remove.
* Weak warning match = maybe remove, but only if it looks like a credit block.
* Edge blocks are treated more suspiciously than mid-film blocks.
* Earlier removals do not cause later dialogue to become “edge” and get deleted.

That last bit matters. A cleaner should not get braver just because it already removed something.

## Known limitations

* **Mid-film ad clusters are only partially cleaned.** The block with the hard token is removed. A token-less neighbour like `Subtitles by` may survive. This is intentional.
* **All language profiles run on every file.** There is no language detection. The rules are mostly brand tokens, so the false-positive risk is low.
* **Some sponsor junk survives.** If a VPN, casino, or other goblin has no hard token in the upstream rules, it may stay. Add it as a purge token if it annoys you enough.

## Credits

Rules from:

* [KBlixt/subcleaner](https://github.com/KBlixt/subcleaner)
* [rogs/subscleaner](https://gitlab.com/rogs/subscleaner)

Subwash reuses their pattern sets under a more cautious removal engine, but shit, don't trust me!

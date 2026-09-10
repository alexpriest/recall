# recall

Full-text search over every Claude Code conversation you have ever had.

## Status

Shipped — indexes every Claude Code conversation back to December 2025.

## License

Not licensed for reuse.

```sh
recall "vermouth trademark"              # FTS5 — words AND'd, "quote a phrase", OR / NOT
recall cartographer --since 2026-07 --project architect --limit 5
recall --stats                           # coverage, span, index size
recall --refresh                         # pick up new transcripts (also automatic)
recall --rebuild                          # discard the index and start over
```

Each hit prints a **session id**. That is the handle for the other half of the workflow:

```sh
claude --resume <session-id>                    # works from any directory
~/.claude/scripts/kit-consult.sh <session-id> "question"   # reload it and ask
```

**Find it with `recall`, then ask it with `kit-consult.sh`.** Searching gives you the words that
were said; consulting gives you the reasoning, the rejected alternatives, and what a session
actually changed on disk.

## What it searches

Two roots, together, because neither is complete alone:

| Root | What it is |
| --- | --- |
| `~/.claude/projects` | Live transcripts. Claude Code prunes these at `cleanupPeriodDays` (now **3650** in `~/.claude/settings.json`; it was unset, so the 30-day default had been deleting them). |
| `~/Code/system/conversation-archive` | Frozen snapshot, 2026-01-06 onward. Holds **~4,990 conversations that exist nowhere else** — Claude Code already deleted the originals. Gitignored, never committed, replicates via Syncthing. |

🔒 **`~/.local/share/confidant-archive` is never indexed or searched.** That is the Iris seal,
enforced twice in the source: the path is not in `ROOTS`, and any path containing `Confidant` is
skipped at walk time. **Do not "fix" this by adding it.**

## The index is a cache

`~/.local/state/recall/index.sqlite` is SQLite FTS5, rebuilt from the transcripts on demand.
Delete it whenever you like. It refreshes incrementally — a file is re-read only if its mtime or
size changed — so day-to-day runs are fast and the first run is the slow one.

**The transcripts are the only source of truth.** If the index and a transcript disagree, the
transcript is right.

## Why not the thing it replaced

`episodic-memory@superpowers-marketplace` was removed 2026-08-14. It offered semantic search,
which sounds strictly better. In practice, measured on this machine:

- It indexed **7,513 of 15,535** archived transcripts — 48% — after seven months.
- It was used **167 times in seven months**, 44% of that inside two days.
- On the one recorded test, it ranked the correct conversation **4th of 4**, below two unrelated
  threads. The index was healthy that day; that is the design's ceiling, not a misconfiguration.
- It summarised conversations by **resuming the original session** with that session's full tools
  and permissions. On 2026-08-14 one such run committed a from-scratch rebuild over five phases of
  finished work, authored under Alex's git identity. Known upstream, confirmed by seven accounts,
  unfixed at HEAD.
- It cost 6.6 GB and a background job on every session start.

Full reasoning: `~/Obsidian/alexpriest/Claude/Architect/Proposals/2026-08-14 Episodic Memory — Remove, Keep the Archive, Fix Retention.md`

## The real limitation, stated plainly

**`recall` matches words, not meaning.** It cannot find a conversation that never used your term.
If a search comes up empty, try the word Alex would actually have typed — and remember that an
empty result is not evidence the conversation doesn't exist.

That is a genuine regression from what semantic search *promises*. It is not a regression from
what that particular implementation *delivered*.

## Install on a new machine

`~/.local/bin` does not sync between machines, so the symlink is per-machine:

```sh
chmod +x ~/Code/tools/recall/recall
ln -sf ~/Code/tools/recall/recall ~/.local/bin/recall
recall --refresh          # first build; the archive is large
```

No dependencies beyond Python 3 and the SQLite that ships with macOS (FTS5 required — present in
the system sqlite3).

## Notes for whoever edits this

- **Raw transcripts are full of injected context** — `CLAUDE.md`, `MEMORY.md`, the agent roster,
  `<system-reminder>` blocks. Naive grep matches ~45% of files on almost any project term. The
  indexer reads only `user`/`assistant` text blocks and strips those wrappers first. Removing that
  filter makes the tool useless rather than noisy.
- **Recurse with `rglob`/`find`, never a fixed-depth glob.** Subagent and workflow transcripts nest
  several levels deep; a `*/*.jsonl` pattern silently misses thousands of them.
- Results are capped by `--limit` and the footer prints the true total. **A cap is never reported
  as the total** — that is a house rule and the reason the count is printed at all.

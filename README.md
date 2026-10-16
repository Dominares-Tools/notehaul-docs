# notehaul — Apple Notes to any endpoint, safe to publish

**Paid tool: $49 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/notehaul**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

notehaul turns an Apple Notes folder into synced, safe-to-publish data for any endpoint, from the Mac or an iPhone Shortcut. It keeps the tables plaintext drops, pushes with replace-all, strips [bracketed] internal text and collects standing DIRECTIVE, ALWAYS and NEVER lines.

## Get it

Buy notehaul at https://dominares.org/tools/notehaul: one payment of $49, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the notehaul Python package and command, receiver examples for FastAPI, Flask and a Cloudflare Worker, the iPhone Shortcut generator and the full test suite.

Requirements: Python 3.10 or newer, standard library only. Exporting from Apple Notes and signing the Shortcut need macOS; everything else runs anywhere. The Worker example needs a Cloudflare account.

This page documents version 1.0.2. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

`notehaul demo` prints this:

```
exported 4 notes (4 reported by the app)

1. a note with a table (plaintext alone would have been one placeholder character):
   Site | Opens
   North | 9:00
   South | 10:30

2. synced: 3 stored, 1 refused as internal (the DO NOT TOUCH note)
3. an empty batch is refused: skipped_empty_sync=True, 3 rows kept

4. publish mode strips [bracketed] text:
   before: We close early on Fridays [check with the manager before saying why]
   after:  We close early on Fridays

5. standing directives, ready to put above a prompt:
   STANDING NOTES (operator directives; follow these first):
   - [Voice of the bot] ALWAYS: answer in the visitor's language
   - [Voice of the bot] NEVER: promise a refund
```

### What it does

Notes is a good place for a team to keep the facts a bot, a website or a script needs. Getting those notes out safely is the part that goes wrong. notehaul does that part.

- **Exports a Notes folder from a Mac** (`notehaul export`). It reads each note two ways, `plaintext` and the HTML `body`, and keeps the fuller one. Apple's `plaintext` replaces a whole table with a single placeholder character; the HTML body holds it. Table rows come out as `Site | Opens`, one row per line. If the HTML conversion ever comes out shorter than the plaintext, the plaintext wins, so a conversion bug cannot empty a note.
- **Pushes with replace-all** (`apply_sync`, `notehaul push`). Each sync replaces the previous batch it owns (rows carrying its tag), so a note you delete in Notes is gone from the destination at the next sync. Rows written by anything else are never touched.
- **Refuses an empty batch.** A sync that would replace a non-empty batch with nothing is almost always a misfired Shortcut, so it is ignored and the existing notes are kept. The push client refuses before it sends anything.
- **Backs up before every overwrite.** The store is copied into a backup folder first (newest 30 kept by default).
- **Folder allow-list.** A receiver given an allow-list refuses a payload whose declared `folder` label is not on it, and (the default whenever a list is set) a payload that declares no folder at all. The generated Shortcut always sends a label. The label is what the sender says, fixed in the Shortcut file or passed to `push --folder-label`; it catches a sender set up for the wrong folder, but it does not prove which folder the notes were read from.
- **Publish mode** (`notehaul publish`, `--publish` on export and push). Strips `[bracketed]` text, which by convention is internal, and folds the bold letters Apple Notes exports as mathematical symbols and its invisible spaces back to plain text. Typography you meant (dashes, curly quotes, emoji) is left alone. A note that is entirely bracketed is dropped.
- **Standing directives** (`notehaul directives`). Lines that start with `DIRECTIVE:`, `ALWAYS:`, `NEVER:` or `PIN:` are collected into one block, capped in size, for the top of a prompt. A bolded marker still counts.
- **Keeps internal notes internal.** A note is dropped, never stored, if its title or body contains `DO NOT TOUCH` or its tags include `internal`, `private` or `do-not-touch`. The check runs after glyph folding, so a bolded marker does not slip through.
- **Receivers to start from**: a framework-free core (`notehaul.receiver`), examples for FastAPI, Flask and a Cloudflare Worker in `examples/receivers/`, and a stdlib server (`notehaul serve`) for trying it locally.
- **An iPhone Shortcut generator** (`notehaul shortcut`) for any endpoint that speaks the contract below, signed with `shortcuts sign --mode anyone` so it can be added from a link.

### Commands and options

```
notehaul export --folder NAME[:tag] [--folder ...] [--out notes.json] [--publish] [--allow-missing]
notehaul push FILE URL [--token-env NOTEHAUL_TOKEN] [--folder-label LABEL] [--publish] [--allow-empty] [--allow-insecure]
notehaul publish FILE [--out FILE]        |   notehaul publish --text "Open 9-5 [staff only]"
notehaul directives FILE [--cap 4000] [--header TEXT]
notehaul shortcut [--name N] [--folder F] [--endpoint URL] [--label L] [--auth-header H] [--auth-scheme S] [--response-key K] [--out FILE] [--unsigned]
notehaul serve --store FILE [--host 127.0.0.1] [--port 8500] [--path /sync] [--allow-folder NAME ...] [--allow-unlabelled] [--token-env NOTEHAUL_TOKEN] [--tag TAG] [--backup-dir DIR]
notehaul demo
```

- `--folder NAME:tag` on export puts `tag` on every note from that folder, so a destination can treat a filed-away folder as history. To name a folder that itself contains a colon, end the argument with a colon: `--folder "Work:Notes:"`.
- If any named folder does not exist, `export` writes nothing and exits 2, because a replace-all push of a partial export would delete that folder's notes at the destination. `--allow-missing` exports the folders that were found. Any other Notes.app error (a denied automation permission, an unreadable note) stops the export with its real message.
- Tokens are read from an environment variable, never from the command line, so they stay out of shell history and process lists. `push` also refuses to send a token over plain `http://` to any host except this machine unless you pass `--allow-insecure`, and it never follows a redirect: a receiver that answers 301/302/303/307/308 is an error, so the token is never sent on to another address.
- `serve --allow-folder` also requires every payload to declare its folder; `--allow-unlabelled` lifts that.
- `serve` refuses to start without a token, and logs nothing about requests.

### The receiver contract

Everything speaks the same small contract, so the Mac client, the Shortcut and the receivers are interchangeable.

Request: `POST` with `Authorization: Bearer <token>` and a JSON body

```json
{"folder": "Public", "notes": [{"title": "Opening hours", "body": "North | 9:00"}], "allow_empty": false}
```

`folder` and `allow_empty` are optional, except that a receiver with a folder allow-list requires `folder`. A bare JSON list of notes is also accepted by a receiver with no allow-list.

| Answer | Body |
|---|---|
| 200 stored | `{"ok": true, "rows_added": 2, "rows_replaced": 1, "backup_file": "...", "notes_received": 3, "notes_after_filter": 2}` |
| 200 refused empty batch | `{"ok": true, "skipped_empty_sync": true, "rows_added": 0, "rows_kept": 5, "warning": "..."}` |
| 401 | wrong or missing token |
| 400 / 413 | not JSON / over 20 MB |
| 422 | folder not on the allow-list, or no folder declared when the allow-list needs one |
| 500 | the store file is corrupt; nothing was written |

Each stored row is `{id, title, body, source, tags, status, created_at_utc, origin_date, origin_source_url}`; `tags` starts with the sync tag (default `notehaul-sync`).

### Use it from Python

```python
from pathlib import Path
from notehaul import apply_sync, publish_text, build_directives_block

apply_sync(Path("store.json"), {"folder": "Public", "notes": notes},
           allowed_folders={"Public"}, backup_dir=Path("backups"))
publish_text("Open 9-5 [staff only]")        # -> "Open 9-5"
build_directives_block(notes, cap=2000)       # -> "STANDING NOTES ...\n- [Title] ALWAYS: ..."
```

Also exported: `strip_bracketed`, `fold_for_publish`, `fold_for_match`, `notes_html_to_text`, `best_note_text`, `normalize_notes_entries`, `parse_notes_csv`, `parse_notes_markdown`, `redact_internal_emails`, `extract_directives`, `inject_directives`, `sanitize_note_rows_for_publish`. The runtime is the Python standard library only, 3.10 or newer.

### The iPhone Shortcut

`notehaul shortcut --endpoint https://your.host/sync --folder "Public" --label Public --out public-notes.shortcut` writes a signed Shortcut. Without `--endpoint` it asks the person adding it for one setup code, `address|key`, so nothing secret is inside the file and one file can serve many people. The Shortcut reads all notes, keeps those whose folder name equals `--folder`, posts them with `"folder": <label>` (`--label`, which defaults to the folder name), and shows either `Done. ... N` (N from the reply's `rows_added`) or a fixed plain message. It never shows the server's own words.

**Not proven on a phone yet.** The structure copies a Shortcut that is known to sync, the plist round-trips, and `shortcuts sign` accepts it on a Mac (the signed file starts with the `AEA1` signature header). This generated file has not been run on a physical iPhone. Until it has, treat these as unverified: whether the folder-name match ignores case, whether the success message reads the reply as expected on your iOS version, and the exact import-question wording. Plan one test run on a device before you hand the file to anyone.

### Limits

- Exporting and signing need macOS. The first export asks for permission to control Notes.app. Everything else (publish, sync, directives, receivers, the plist) runs anywhere.
- The export was checked by compiling its AppleScript with `osacompile` and by parsing recorded output in tests. It was not run against a live Notes library in the test suite.
- Only the folders you name are read.
- Attachments, images and drawings are not exported; only text and tables.
- The store is one JSON file with an advisory lock (POSIX). The whole file is rewritten on each sync, and the lock covers one machine, not a cluster. Put the same rules in front of your own database if you need more.
- `notehaul serve` is a reference receiver: plain http, no TLS, no rate limiting. Put it behind your own TLS in production.
- The Worker example is a port of the same rules, not the same code. It folds glyphs, strips brackets, excludes `DO NOT TOUCH`, refuses a corrupt store and counts the 20 MB limit in bytes, all checked under node; its bracket stripping is checked against the Python version on a shared set of cases. It stores the same row shape. It differs in one way: it owns its whole KV key, so a sync replaces every stored row, not only rows carrying the sync tag.
- Dates are best-effort. They are parsed from the text AppleScript prints, in the formats of a US-English Mac (12-hour and 24-hour); on another language setting the raw text is kept in `origin_date`.
- Publish mode removes brackets, not secrets. It cannot know that a phone number in plain text is private.
- Bracket stripping assumes your notes use `[ ]` only for internal remarks.

### Tests

`pip install ".[dev]"` then `pytest`. The dev extra installs FastAPI, httpx and Flask, and `node` must be on your PATH. Nothing is skipped. The tests cover every public function, the receivers through their framework test clients, the Worker under node with an in-memory KV, and a redirecting receiver. Two checks use a real Mac tool when there is one: `osacompile` on the generated AppleScript and `shortcuts sign` on the generated Shortcut. On another system those two check that the tool is absent and that the code says so. The suite passes on Python 3.10 and 3.14.

---

Docs mirror of the private repository Dominares-Tools/notehaul; updated with each release.

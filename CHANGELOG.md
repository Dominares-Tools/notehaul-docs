# Changelog

## 1.0.2 - 2026-10-04

README only: it opens with the demo output and leaves the install steps to the page you bought it from. No code change.

## 1.0.1 - 2026-10-04

Fixes from review.

- `push` no longer follows redirects. A receiver answering 301/302/303/307/308 used to turn the POST into a GET and receive the bearer token at the new address, and the CLI reported success. It is now an error that names where it was sent.
- Folder allow-list: a receiver with an allow-list now also refuses a payload that declares no folder (`ReceiverConfig.require_folder` defaults to on when `allowed_folders` is set; `serve --allow-unlabelled` opts out). The generated Shortcut always sends a folder label, defaulting to the folder name. The README now says what the label does and does not prove.
- Export: only a missing folder (AppleScript error -1728) is reported as missing; every other Notes.app error stops the export with its real message instead of producing a partial export that a replace-all push would then apply. `export` exits 2 and writes nothing if any named folder is missing (`--allow-missing` to override). A folder name containing `:::` is refused. Dates in 24-hour format parse.
- Store: a store holding entries that are not objects is refused (`StoreError`) instead of having them dropped on the next write.
- Auth: the `Bearer` scheme is matched case-insensitively (the token still compares in constant time).
- Cloudflare Worker example: folds bold, full-width and invisible characters before stripping and before matching `DO NOT TOUCH`, refuses a corrupt store (500, nothing written), counts the size limit in bytes, stores the same row shape as the Python store, and requires a folder when an allow-list is set.
- Tests: no test is skipped on any platform (the FastAPI, Flask and node tests are required by the dev extra and PATH; the two Mac-tool tests assert the absent-tool behaviour elsewhere). The README no longer hard-codes a test count.

## 1.0.0 - 2026-10-04

First release.

- Mac exporter: reads named Notes folders through AppleScript, both `plaintext` and the HTML `body`, and keeps table rows (`a | b | c`) that plaintext drops. Missing folders are reported, not fatal.
- Replace-all sync (`apply_sync`) into a JSON store: replaces only the rows carrying its tag, refuses an empty batch that would wipe a non-empty one, backs the store up before every overwrite, enforces a folder allow-list, locks the read-modify-write, refuses a corrupt store.
- Publish mode: strips `[bracketed]` text and folds Apple Notes' bold glyphs and invisible spaces.
- `DIRECTIVE` / `ALWAYS` / `NEVER` / `PIN` extraction into one capped block.
- Exclusion of `DO NOT TOUCH` notes and `internal` / `private` tagged notes, matched after glyph folding.
- Receiver core with a constant-time token check, plus examples for FastAPI, Flask and a Cloudflare Worker, and a stdlib reference server (`notehaul serve`).
- Push client (`notehaul push`) that refuses an empty batch and refuses to send a token over plain http to a remote host.
- iPhone Shortcut generator for any endpoint, signed with `shortcuts sign --mode anyone` on a Mac. Not yet run on a physical iPhone.

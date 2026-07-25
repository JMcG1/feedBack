# Plugin Metadata API — Runtime Verification & PR Preparation

Runtime verification record for the generic plugin metadata implementation (`lib/plugin_metadata.py`, the `song_info` WebSocket extension, `GET /api/song/{filename}/metadata`) in the real `core-development` environment. Originally written immediately before staging for the upstream PR; regenerated below to reflect that the PR has since been opened as [#1045](https://github.com/got-feedBack/feedBack/pull/1045). No redesign, no new functionality, no changes to Song Background Manager.

## Executive summary

This record has been regenerated to replace the "staged, not yet committed" snapshot from the prior verification pass with what has since actually happened: three DCO-signed commits — `52c9408` (feature), `ad37cb6` (CHANGELOG + docs), `7221f6a` (a fix for one automated review finding) — landed on `feature/plugin-metadata-api`, were pushed to `origin` (`JMcG1/feedBack`), and are open as pull request [#1045](https://github.com/got-feedBack/feedBack/pull/1045) against `got-feedBack/feedBack:main`.

Real `pytest` evidence now exists, closing the prior pass's biggest gap. On the contributor's own Windows machine (Python 3.14.6, pytest 9.1.1), `pytest tests/test_plugin_metadata.py tests/test_plugin_metadata_api.py -q` collected **31 items** (23 in `test_plugin_metadata.py`, 8 in `test_plugin_metadata_api.py`) and passed all 31; a separate run of the WS regression suite (`test_highway_ws_authors.py`, `test_highway_ws_instrument_routing.py`, `test_highway_ws_notation.py`, `test_ws_highway_disconnect.py`) collected and passed all 21. `git diff --check dd1927e..HEAD` is clean. This sandbox still cannot install `pytest`/`fastapi` (`pypi.org` remains blocked), so the manual harness below was re-run fresh as a cross-check rather than as the only evidence.

`tests/test_plugin_metadata.py` now collects **23 items**, up from 16 in the prior pass. The 7 added items — from 5 new test functions, one parametrized over 3 match states — cover `base_metadata`: per-field override of a populated cache, blank/zero values falling through to the cache rather than clobbering it, a partial `base_metadata` still pulling `genre` from the cache, and confirmed/unconfirmed enrichment gating being unaffected by `base_metadata` either way.

Diff scope is nine files, not three: `CHANGELOG.md`, `docs/PLUGIN_METADATA_API.md`, `docs/PLUGIN_METADATA_FINAL_VERIFICATION.md`, `lib/plugin_metadata.py`, `lib/routers/song.py`, `lib/routers/ws_highway.py`, `static/highway.js`, `tests/test_plugin_metadata.py`, `tests/test_plugin_metadata_api.py` — 1,177 insertions, 0 deletions, per `git diff --stat dd1927e..HEAD`.

**Status: SUBMITTED.** PR #1045 is open and has already absorbed one review round. This document is a historical verification record, not a pre-commit gate — it no longer recommends whether to commit or push.

## Environment

- Repository: `core-development`, the real repo (not a copy), at its current working-tree state.
- OS: Ubuntu 22.04.5 LTS (`Linux claude 6.8.0-124-generic`, x86_64).
- Python: 3.10.12 (`/usr/bin/python3`), linked into `.venv` (`.venv/bin/python` → `/usr/bin/python`).
- Package manager: `uv` 0.11.19, the project's documented tool. `pyproject.toml` has no `[project]` dependency table, so `uv sync` produces an empty environment; the documented install path is `uv pip install -r requirements.txt -r requirements-test.txt --python .venv/bin/python`. Ran exactly that command.
- Result: `error: Request failed after 3 retries in 5.5s / Caused by: Failed to fetch: https://pypi.org/simple/yt-dlp/ / ... tunnel error: unsuccessful` — this sandbox's outbound network policy blocks `pypi.org`. No packages were installed. No alternative tooling (plain `pip`, a different index, a vendored wheel cache) was substituted — the task specifically says not to introduce alternative tooling, and doing so would not be a genuine install regardless.
- Dependency changes: none. `fastapi`, `pytest`, `websockets`, `structlog`, and the rest of `requirements.txt`/`requirements-test.txt` remain absent from `.venv`.
- Side effect noted and reverted: `uv sync` rewrites `uv.lock`'s `requires-python` from `>=3.12` to `>=3.10` to match the sandbox's interpreter, each time it's run. Reverted with `git checkout -- uv.lock` after this session's environment attempt, same as the prior pass — confirmed clean (`git status --short uv.lock` produces no output).

## Runtime verification

Attempted, in order, exactly as instructed:

1. `PYTHONPATH=.:lib .venv/bin/python -m pytest tests/test_plugin_metadata.py -q` → `No module named pytest`.
2. `.venv/bin/python main.py` (an actual attempt to launch the real application) → fails at `lib/logging_setup.py`'s `import structlog` — i.e. the app doesn't even reach the point of constructing the FastAPI app, let alone opening a WebSocket, because a dependency several layers before `fastapi` itself is missing.

Both failures are dependency-installation failures, not application or feature defects — confirmed by their tracebacks pointing at `ModuleNotFoundError` for third-party packages, not at any line in this feature's code. Genuine live-server/live-WebSocket verification is not achievable in this sandbox; this is stated plainly rather than worked around.

## Unit test results

Real evidence now exists and supersedes the manual harness as primary evidence: on the contributor's Windows machine, `pytest tests/test_plugin_metadata.py -q` collected and passed all **23 items** (confirmed via the real pytest header reporting `collected 31 items` across this file plus `test_plugin_metadata_api.py` combined, 100% passed, zero failures, zero warnings).

This sandbox still cannot install `pytest`, so as a cross-check the manual harness was re-run fresh this session, extended to cover the 5 `base_metadata` test functions added since the prior pass (previously only 9 groups/16 items were covered here):

```
. test_returns_versioned_shape_for_unknown_song (161.87ms)
. test_metadata_version_is_present_and_stable (166.05ms)
. test_playback_metadata_with_no_enrichment (153.90ms)
. test_year_coercion_never_raises (136.96ms)
. test_album_artist_is_always_none (123.68ms)
. test_matched_enrichment_populates_identifiers (124.07ms)
. test_manual_pin_is_treated_as_confirmed (113.76ms)
. test_unconfirmed_states_never_leak_identifiers (118.29ms)
. test_base_metadata_overrides_populated_cache_per_field (106.20ms)
. test_blank_and_zero_base_metadata_falls_through_to_cache (109.43ms)
. test_partial_base_metadata_still_pulls_missing_fields_from_cache (119.49ms)
. test_confirmed_enrichment_still_available_with_base_metadata (118.70ms)
. test_unconfirmed_enrichment_still_hidden_with_base_metadata (117.25ms)
. test_output_is_a_plain_json_serializable_dict (121.32ms)

14 groups passed, 0 failed, 23 pytest-equivalent items covered
```

Note on count: the harness groups by function (14 groups), while real `pytest -q` collects one item per parametrized case — `test_year_coercion_never_raises` (6 cases), `test_unconfirmed_states_never_leak_identifiers` (3 cases), and `test_unconfirmed_enrichment_still_hidden_with_base_metadata` (3 cases, added this round) each count as multiple items — for a real total of 23, matching the real pytest run's own collection count exactly, not merely a projection. Per-case timings here are dominated by this sandbox's per-database WAL-mode setup cost (~100-160ms/case) rather than the query logic itself; they aren't comparable to real pytest's timing and aren't offered as a performance measurement.

## Integration test results

`tests/test_plugin_metadata_api.py` — real evidence now exists: it collected and passed all **8 items** on the contributor's Windows machine, as part of the same 31-item run cited above. This sandbox still cannot execute it (`fastapi.testclient.TestClient` unavailable, `pypi.org` blocked); the manual simulation below remains a sandbox-only cross-check, not the primary evidence anymore.

- Full read of the file: its WS/REST fixtures are adapted from `tests/test_highway_ws_authors.py` and `tests/test_art_candidates.py`'s own already-established patterns, not novel test infrastructure.
- Fresh standalone simulation of the REST route (`get_song_plugin_metadata`'s body reproduced verbatim, calling the real `_resolve_dlc_path` and `plugin_metadata_for`), run fresh this session against a newly seeded song:

```
PASS missing_song_404
PASS traversal_403_or_404
PASS real_song_200
PASS real_song_payload_shape
PASS real_song_album_year_genre
PASS malformed_filename_no_crash

6/6 passed
```

## Full suite results

Not executable in this sandbox — no `pytest` here, and `pypi.org` remains blocked. Real evidence exists for the relevant regression slice, though, not the whole repository suite: the contributor's real `pytest` run of `test_highway_ws_authors.py`, `test_highway_ws_instrument_routing.py`, `test_highway_ws_notation.py`, and `test_ws_highway_disconnect.py` collected and passed all 21 items, confirming the WS `song_info` change didn't regress the existing highway WebSocket behavior. A full-repository suite run has not been reported and is not claimed here.

## Performance observations

Re-measured fresh this session against a newly seeded real SQLite database (not reused figures from the prior pass):

- SQL statements per `plugin_metadata_for()` call: exactly 2 — one indexed `songs` read, one indexed `song_enrichment` read. Matches the "at most one indexed enrichment lookup" requirement.
- 1000 calls: 14.30ms total, 0.0143ms/call average.
- No network access anywhere in `lib/plugin_metadata.py` (grepped for `requests`/`urllib`/`socket`/`aiohttp`/`cloudscraper`/`curl_cffi`/`httpx` — no matches).
- Both WS and REST call sites route through this one function — confirmed by direct `grep` of both router files — so there is no duplicated lookup logic and no possibility of the two surfaces disagreeing.
- Effect on playback startup: cannot be measured live (no server), but the WS handler offloads the call via `loop.run_in_executor`, the same pattern already used for the rest of that handler's blocking work, so it cannot block the event loop any differently than existing calls in the same function already do.

## Compatibility assessment

- `plugin_metadata_for()` remains the sole assembly function — confirmed by `grep -rn "def plugin_metadata_for" lib/`, one result.
- Route precedence re-confirmed: `/user-meta` → `/overrides` → `/gap-fill` → `/metadata` (new) → bare `{filename:path}`, in that registration order in `lib/routers/song.py` — the greedy catch-all still cannot shadow the new route.
- `python3 -m py_compile` clean on all 5 Python feature files; `node --check static/highway.js` clean.
- Every change across the current nine-file diff (`dd1927e..HEAD`) is additive — `git diff --stat dd1927e..HEAD` shows 1,177 insertions, 0 deletions.
- Old plugins / old clients: `msg.metadata ?? null` in `static/highway.js` means an old server (no `metadata` key) degrades to `null` rather than `undefined`-chasing errors; a new server against old plugin code that never reads `metadata` is unaffected because nothing existing changed shape or name.

## Files changed

No longer a staging list — these files are committed (across the 3 commits listed in the executive summary) and pushed to `feature/plugin-metadata-api`, per `git diff --stat dd1927e..HEAD`:

```
CHANGELOG.md
docs/PLUGIN_METADATA_API.md
docs/PLUGIN_METADATA_FINAL_VERIFICATION.md
lib/plugin_metadata.py
lib/routers/song.py
lib/routers/ws_highway.py
static/highway.js
tests/test_plugin_metadata.py
tests/test_plugin_metadata_api.py
```

Nine files, 1,177 insertions, 0 deletions. `CHANGELOG.md` was added to this list in the DCO/PR-readiness pass (it wasn't part of the original 8-file staging round this section previously described); every other file was already present in the original list.

## Remaining unstaged files (confirmed excluded, not part of this PR)

- **`docs/PLUGIN_METADATA_API_PROPOSAL.md`** — new/untracked, part of this feature's own documentation trail (the design proposal this implementation was built from), but not on this task's staging list, so left unstaged per Step 7's instruction to stage only the specified list.
- **`.gitignore`, `uv.lock`, and ~430 other tracked files repo-wide** — a pre-existing CRLF/LF line-ending artifact between the committed blobs (LF) and this working tree (CRLF), confirmed via raw byte comparison and unrelated to this feature; `git diff --ignore-cr-at-eol` shows these produce no real diff. `uv.lock` specifically was touched incidentally by this session's own `uv sync` and reverted before staging.
- **`plugins/highway_3d/screen.js`, `tests/test_settings_export.py`** — real, substantial, already-uncommitted changes implementing an unrelated "Temporary Background API" feature (`window.feedBack.backgrounds.setTemporarySource`), evidenced by the untracked `PR_DESCRIPTION.md` and `docs/TEMPORARY_BACKGROUND_API.md` at the repo root. Not touched, not staged.
- **`plugins/song-background-manager/` (untracked directory), `tests/js/highway_3d_temporary_background_api.test.js`, `tests/js/song_background_manager.test.js`, `tests/test_song_background_manager_routes.py`, `Backups/`** — further pre-existing, unrelated content sitting in this working tree. Not touched, not read beyond identifying them, not staged. This task's own instruction not to modify Song Background Manager is honored by leaving all of this alone.

## Risks

- ~~No genuine live-`pytest` run exists for this feature~~ — **resolved**: real `pytest` evidence now exists for both new test files (31/31) and the WS regression slice (21/21), all on the contributor's own machine. No full-repository suite run has been reported, so that broader claim is not made here.
- Everything checked across this and prior passes — wiring, logic, compatibility, diff scope — remains consistent with no regression or new issue found in either the added tests or the real runs.

## Recommendation

The PR is open and under review; there is no pending commit/push decision left for this document to gate. If further review feedback lands, the established pattern for this branch is: fix, `git commit -s`, `git push origin feature/plugin-metadata-api` — the same flow already used once for `7221f6a`.
